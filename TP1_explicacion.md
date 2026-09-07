# TP1 · Ingeniería de datos con Airflow
### Grupo 5K10-06 · Ciencia de Datos · UTN FRM · 2026

Integrantes: Carobolante Alejo · Dallape Vincenzo · Podesta Isidro · Saini Alejo

---

## 1. Qué pide el TP

El enunciado es explícito sobre su propio objetivo: **no se trata de escribir un pipeline desde cero, sino de leer uno ajeno, entenderlo y modificarlo con criterio.**

El punto de partida es el proyecto `airflow-fifa`, que trae el DAG `fifa_ingest`. Ese DAG construye el dataset canónico de la materia: 18.936 jugadores de 52 ligas, scrapeados de sofifa.com.

La consigna pide copiarlo a un archivo nuevo, dejar el original intacto como referencia, y aplicarle cuatro cambios. El entregable es **un solo archivo .zip, generado por el propio DAG**. Nada de capturas ni de código suelto.

Se calcula que los cuatro cambios obligan a leer unas cuarenta líneas del DAG. El resto se puede ignorar.

## 2. Nuestros datos

| Dato | Valor |
|---|---|
| Grupo | 5K10-06 |
| Liga | Bundesliga |
| País | Austria |
| `league_id` | 80 |
| Jugadores aproximados | 348 |
| Snapshot a fijar (`roster`) | 260046, la actualización del 23/07/2026 |
| Columnas del esquema | 90 |

El archivo del DAG en nuestra rama es `dags/tp1_5K10_G6.py`, con `dag_id` igual a `tp1_5K10_G6`.

---

## 3. El punto de partida: cómo está armado `fifa_ingest`

Antes de explicar los cambios hay que entender qué se está tocando. Esta sección es la que más peso tiene en la exposición.

### 3.1 Vocabulario mínimo de Airflow

Airflow orquesta pipelines de datos. Un **DAG** es un grafo dirigido sin ciclos: un conjunto de **tareas** con dependencias entre ellas. Airflow decide qué correr, cuándo, en qué orden, cuántas cosas en paralelo, y qué hacer cuando algo falla.

Lo que Airflow aporta y un script suelto no tiene: reintentos, ejecución programada, paralelismo, visibilidad de qué pasó en cada corrida, y la posibilidad de reprocesar una tarea sin volver a correr todo.

### 3.2 El grafo

```
wait_for_source → check_source ──┬─→ has_new_snapshot → discover_leagues
                                 │      → land_bronze[N] → refine_silver[N] → consolidate ─┐
                                 │                                                          ├→ validate → save
                                 └─→ load_frozen ───────────────────────────────────────────┘
```

En nuestra versión se agrega una tarea más al final, después de `save`.

### 3.3 Qué hace cada tarea

| Tarea | Rol |
|---|---|
| `wait_for_source` | Sensor. Espera a que sofifa responda. Sondea cada 5 minutos hasta 30. |
| `check_source` | Rama. Decide si se sigue por la fuente o por el respaldo. |
| `has_new_snapshot` | Cortocircuito. Si el snapshot no cambió, corta y la corrida termina sin hacer nada. |
| `discover_leagues` | Lee el catálogo de ligas y calcula cuántas páginas pedir de cada una. |
| `land_bronze` | Baja el HTML crudo y lo guarda comprimido. Es la única tarea que toca la red. |
| `refine_silver` | Parsea ese HTML a filas tipadas. No toca la red. |
| `consolidate` | Junta los parciales, deduplica por `player_id` y castea tipos. |
| `load_frozen` | Rama de respaldo. Usa un CSV que ya está en disco. |
| `validate` | Chequeos duros. Si alguno falla, el DAG falla. |
| `save` | Escribe el CSV fechado y refresca el respaldo. |

### 3.4 El modelo medallón

Es la idea central del diseño, y conviene poder explicarla sin leer.

- **Bronce** es el dato tal como llegó, sin interpretar. Acá es el HTML de sofifa, comprimido con gzip y particionado en carpetas por snapshot y por liga, con rutas de la forma `include/output/bronze/roster=260046/liga=80/pagina_00000.html.gz`.
- **Plata** es el dato tipado y limpio, una fila por jugador, con las 90 columnas del esquema.
- **Oro** serían las features del modelo y los agregados de la app. No se construye en este TP, se arma en las unidades 3 y 4.

La separación no es decorativa. Produce dos propiedades concretas:

1. **Una página que ya está en bronce no se vuelve a pedir.** La segunda corrida sobre el mismo snapshot no genera ni una sola request. La fuente se toca una vez por dato, no una vez por corrida.
2. **Un bug en el parseo se arregla sin volver a la fuente.** Se corrige `refine_silver` y se reprocesa el HTML que ya está en disco.

Sin la capa de bronce, el HTML se pierde apenas se convierte en filas, y cualquiera de esas dos cosas obliga a scrapear todo de nuevo.

### 3.5 Los tres niveles de degradación

El DAG no tiene una fuente, tiene un plan para cuando la fuente falla.

1. **HTTP directo con `urllib`.** El camino normal, medio segundo por página.
2. **Playwright con Chromium.** Si Cloudflare devuelve 403.
3. **Respaldo en disco.** Si sofifa está caído o cambió de estructura.

Los dos primeros viven dentro de la función `fetch()` y no aparecen en el grafo: son la misma página por otra puerta, así que el reintento es transparente. El tercero sí es una rama visible, porque ahí **el dato es otro, más viejo**, y eso hay que comunicarlo.

El respaldo tiene a su vez dos capas. La semilla `include/frozen/fifa_snapshot.csv` viaja versionada en el repositorio y nunca se toca, así que el respaldo funciona desde el primer clone. El archivo `ultimo_ok.csv` lo reescribe cada corrida completa exitosa y no está versionado. La tarea `load_frozen` prefiere el segundo y cae al primero.

### 3.6 Piezas de Airflow que aparecen en el código

Vale la pena poder nombrarlas, porque es lo que se está enseñando en la unidad.

- **Sensor con `mode="reschedule"`.** En vez de ocupar un worker durmiendo cinco minutos, la tarea se libera y Airflow la vuelve a encolar. Con un sensor da igual, con cincuenta es la diferencia entre un scheduler sano y uno tapado.
- **`soft_fail=True`.** Al agotarse el tiempo la tarea queda en `skipped`, no en `failed`. Que la fuente no responda no es un error del pipeline, es una condición prevista.
- **Rama con `trigger_rule=ALL_DONE`.** La regla por defecto es `ALL_SUCCESS`, y con ella la rama nunca correría justo cuando más falta hace, porque su tarea de arriba quedó en `skipped`.
- **Cortocircuito.** Si devuelve falso, todo lo que está aguas abajo queda en `skipped` y la corrida termina bien.
- **Dynamic task mapping con `.expand()`.** La cantidad de tareas la decide la fuente en tiempo de ejecución, no el código. Con `map_index_template` cada instancia aparece en la interfaz con el nombre de su liga.
- **`trigger_rule=NONE_FAILED_MIN_ONE_SUCCESS` en `validate`.** Le llegan dos ramas y sólo una se ejecuta. Con la regla por defecto nunca correría.
- **XCom transporta rutas, no filas.** La tarea `refine_silver` escribe su liga a disco y devuelve la ruta. Pasar miles de filas por XCom satura la base de metadatos de Airflow, y es de los errores más comunes al empezar.
- **Variables.** La variable `fifa_ultimo_roster` guarda el último snapshot procesado. Es lo que permite que la corrida de mañana sepa si hay trabajo.

### 3.7 Dos decisiones del código que suelen preguntarse

**Se usa `urllib` y no `requests`.** Cloudflare bloquea con 403 a los clientes construidos sobre `urllib3`, y `requests` es uno de ellos, por la huella del handshake TLS. La biblioteca estándar pasa. Cambiar una por otra rompe el pipeline.

**No se usa `ds`.** Esa variable sólo existe cuando el DAG tiene `schedule` y por lo tanto intervalo de datos. La fecha se saca del `DagRun`. Es un tropiezo clásico al pasar de Airflow 2 a 3.

---

## 4. Los cuatro cambios

### 4.1 Filtrar a nuestra liga

El DAG baja las 52 ligas del catálogo. Tiene que bajar sólo la Bundesliga de Austria.

Se agregó un parámetro al DAG:

```python
"league_id": Param(
    None, type=["null", "integer"],
    title="ID de Liga",
    description="ID de la liga específica a descargar. Vacío procesa todas.",
),
```

Y se filtra el catálogo dentro de `discover_leagues`, justo después de pedirlo:

```python
catalogo = fetch_catalog(engine=params["engine"], roster=roster)

target_league = params.get("league_id")
if target_league is not None:
    catalogo = [lg for lg in catalogo if lg["league_id"] == target_league]
```

**Por qué el filtro va ahí y no en otro lado.** La tarea `discover_leagues` es el punto donde se decide el tamaño del trabajo: lo que devuelve es la lista sobre la que `.expand()` crea una tarea mapeada por liga. Filtrar acá significa que se crea **una sola** instancia de `land_bronze`, y por lo tanto no se le pide a sofifa nada que no vayamos a usar. Filtrar más abajo, por ejemplo al consolidar, funcionaría igual pero habría bajado las 52 ligas para tirar 51.

Ojo con un detalle: el parámetro tiene default vacío, que significa *todas las ligas*. Hay que pasarlo explícitamente en la corrida de entrega.

### 4.2 Fijar el snapshot

Se corre con `roster` en 260046, la actualización del 23/07/2026.

No es cosmético. sofifa guarda 46 snapshots de FC 26, y **el catálogo de ligas se pide para el mismo snapshot que después se scrapea**. La actualización del 07/08/2025 tenía 51 ligas y 18.035 jugadores; la del 23/07/2026 tiene 52 y 18.936. Pedir el catálogo actual para scrapear un snapshot viejo daría conteos equivocados y ligas que en ese momento no existían.

Fijarlo tiene además dos efectos sobre el flujo, los dos deliberados y ya previstos en el código:

- La tarea `has_new_snapshot` devuelve verdadero sin consultar la Variable, porque si se pide un snapshot puntual, "el último que vi" no es la pregunta correcta.
- El roster se resuelve **una sola vez** en `discover_leagues` y viaja con cada liga. Si quedara en "el más reciente", dos ligas bajadas con minutos de diferencia podrían caer en snapshots distintos y mezclarse en la misma carpeta de bronce.

### 4.3 Arreglar la validación

Al filtrar la liga, `validate` falla. Es a propósito.

El chequeo original es:

```python
if len(df) < 500:
    problemas.append(f"muy pocas filas: {len(df)}")
```

Con 52 ligas y 18.936 jugadores, 500 filas es un piso trivial: si el resultado baja de ahí, algo se rompió. Con una sola liga de 348 jugadores, ese mismo piso hace fallar toda corrida correcta.

**Qué protege ese chequeo.** No es un conteo exacto, es un detector de corridas rotas. Si sofifa cambia la estructura del HTML, el parser devuelve pocas filas o ninguna sin lanzar excepción. Si hay un corte de red a mitad del scraping, se pierden páginas. En los dos casos el pipeline terminaría en verde y publicaría un dataset mutilado. El umbral existe para que eso falle ruidosamente en vez de pasar en silencio.

**Nuestra decisión: 300.**

```python
if len(df) < 300:
    problemas.append(f"muy pocas filas: {len(df)}")
```

El razonamiento está escrito en el código y es el que hay que poder defender. sofifa pagina de a 60 jugadores por página, valor que está en la constante `PAGE_SIZE`. Nuestra liga tiene unos 348, es decir **cinco páginas completas más 48 jugadores de la sexta**.

Poner el umbral en 300 equivale a exigir que se hayan bajado y parseado **las cinco páginas completas**. Si falta cualquiera de ellas, el conteo cae por debajo de 300 y la corrida falla. Al mismo tiempo deja margen para los 48 jugadores de la última página, que son los que pueden variar de forma legítima entre snapshots por transferencias o bajas.

Es decir: el umbral queda pegado a una unidad real del pipeline, la página, en vez de ser un número redondo elegido a ojo. Eso es lo que lo hace defendible.

### 4.4 Armar el zip de entrega

Se agregó la tarea `bundle_zip`, que corre después de `save` y recibe la ruta del CSV que produjo la corrida.

```python
archivo_final = save(validate(consolidado, congelado))
bundle_zip(archivo_final)
```

Arma un único archivo con cinco contenidos:

| Contenido | Cómo se obtiene |
|---|---|
| `dataset.csv` | El CSV que devolvió `save`, agregado bajo ese nombre. |
| `manifiesto.json` | Generado por código, leyendo la corrida. |
| `bronce.txt` | Recorre la carpeta de bronce del roster y la liga, y escribe una ruta por línea. |
| `logs/` | Recorre la carpeta de logs del DAG y filtra por el identificador de la corrida actual. |
| El archivo del DAG | Se agrega resolviendo la ruta del módulo en tiempo de ejecución. |

**El manifiesto se lee, no se escribe a mano.** Es el punto que más mira la corrección.

```python
df = pd.read_csv(csv_path, low_memory=False)
filas, columnas = len(df), len(df.columns)

league_id = int(params.get("league_id") or df["league_id"].iloc[0])
meta = context["ti"].xcom_pull(task_ids="wait_for_source") or {}
roster = int(params.get("roster") or meta.get("roster", 260046))
```

- Las filas y columnas salen de medir el CSV con pandas.
- El identificador de la corrida y el del DAG salen del contexto de ejecución.
- La liga y el roster salen de los parámetros de la corrida, con el propio dataset como respaldo.
- El grupo y los integrantes son los únicos valores fijos, y está bien que lo sean.

---

## 5. Cómo correr la entrega

El entorno se levanta con Docker y Astro CLI:

```bash
astro dev start
```

Con Airflow en `localhost:8080`, se dispara el DAG con esta configuración:

```json
{"mode": "full", "roster": 260046, "league_id": 80, "engine": "auto"}
```

**El modo tiene que ser `full`.** En `subset` se baja una sola página por liga, o sea 60 jugadores, y `validate` falla contra el umbral de 300. El modo `subset` sirve para desarrollo, no para la entrega.

Vale la pena hacer lo que sugiere el enunciado: correr el DAG una primera vez y mirar el grafo mientras corre, y después correrlo una segunda vez. En la segunda, `land_bronze` termina casi al instante y sus logs dicen **0 pedidas a sofifa**, porque las páginas ya están en bronce. Esa propiedad se entiende mucho mejor viéndola que leyéndola, y es material directo para la exposición.

---

## 6. Qué se entrega y cómo se corrige

Se sube un solo archivo al campus: el zip que generó el DAG.

| Se verifica | Cómo |
|---|---|
| Que el DAG haya corrido de verdad | Los logs del zip son de la corrida que declara el manifiesto, con el mismo identificador. |
| Que sea nuestra liga | La liga del manifiesto es 80 y todas las filas del CSV son de esa liga. |
| Que el manifiesto no esté inventado | Las filas y columnas que declara coinciden con el CSV que viene al lado. |
| Que la validación tenga sentido | Se lee el código de `validate` y la justificación del umbral. |
| Que el zip lo haya armado el DAG | Está la tarea nueva en el código y aparece en los logs. |
| Que el bronce sea de nuestra liga | Las rutas de `bronce.txt` llevan el roster y la liga que nos tocaron. |

---

## 7. Para revisar antes de entregar

Estas son observaciones concretas sobre el estado actual de la rama `tp1`. Ninguna rompe la ejecución, pero varias afectan la corrección.

### 7.1 Los nombres no siguen la convención del enunciado

El enunciado pide el patrón `tp1_<comision>_<grupo>` con el guión del código de grupo convertido en guión bajo. Para 5K10-06 eso da `tp1_5K10_06`.

| Qué | Debería ser | Está |
|---|---|---|
| Archivo del DAG | `tp1_5K10_06.py` | `tp1_5K10_G6.py` |
| Identificador del DAG | `tp1_5K10_06` | `tp1_5K10_G6` |

Además, el nombre del zip se construye así:

```python
zip_nombre = f"tp1_{grupo.replace('-', '_').lower()}.zip"
```

Esa conversión a minúsculas transforma `5K10-06` en `5k10_06`, así que sale un zip con ka minúscula, cuando el enunciado la usa mayúscula. Lo mismo pasa con el nombre del archivo de código que se guarda dentro del zip, que además no coincide con el nombre real del archivo del DAG.

### 7.2 Imports que dejó el autocompletado del editor

En el encabezado del archivo quedaron seis imports que no corresponden:

```python
from curses import meta
from multiprocessing import context
from turtle import pd
from turtle import pd
import zipfile
from fastapi import params
```

Verificamos que **no rompen nada**, porque cada función que usa pandas hace su propio import local, y los otros tres nombres se reasignan como variables locales dentro de cada tarea. Pero la línea que importa desde la librería de gráficos tortuga está repetida y en realidad trae una función de dibujo, no pandas. Conviene borrarlos: es lo primero que va a ver quien lea el archivo.

### 7.3 Un apellido mal escrito en el manifiesto

En la lista de integrantes falta una ene en el nombre de Dallape. Ese valor va tal cual al `manifiesto.json`.

### 7.4 Riesgo real: que el zip salga del respaldo

Si sofifa no responde, `check_source` deriva a `load_frozen`, que carga la semilla de 18.936 filas de las 52 ligas. Esa corrida **pasa la validación sin problema**, porque 18.936 es mayor que 300, y `bundle_zip` arma igual el zip.

El resultado sería un entregable con el dataset completo en vez de la Bundesliga austríaca, una liga sacada de la primera fila del CSV y ningún archivo de bronce propio. Antes de subir hay que confirmar en el manifiesto que dice 348 filas y liga 80, y que `bronce.txt` no está vacío.

### 7.5 Efecto lateral sobre el respaldo local

La tarea `save` refresca el archivo `ultimo_ok.csv` cuando se cumplen dos condiciones: que los datos vengan de la fuente y que el modo sea `full`. Nuestra corrida de entrega cumple las dos, pero baja **una sola liga**.

O sea que la corrida buena pisa el respaldo completo con uno de 348 filas. No es grave, porque ese archivo está en `.gitignore` y la semilla versionada sigue intacta, pero es exactamente el retroceso que la condición del modo `subset` intentaba evitar. Es un buen ejemplo de cómo un cambio en una punta del pipeline toca algo en la otra, y da para comentarlo en la exposición.

### 7.6 Un test del repositorio falla de fábrica

El archivo `tests/dags/test_dag_integrity.py` espera una tarea llamada `scrape_league`, que el DAG original ya no tiene porque fue partida en `land_bronze` y `refine_silver`. Además exige que existan exactamente dos DAGs, así que agregar el del TP hace fallar otro test. Es una inconsistencia del material de la cátedra, no nuestra, pero conviene saberlo si alguien corre la suite de pruebas.
