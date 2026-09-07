# TP1 · Guión de exposición
### Grupo 5K10-06 · Bundesliga (Austria) · `league_id` 80

Cuatro bloques de unos cinco minutos, veinte minutos en total más preguntas. El orden sigue el del enunciado, así que cada bloque deja servido al siguiente.

| Bloque | Quién | Tema | Minutos |
|---|---|---|---|
| 1 | Carobolante Alejo | Contexto y arquitectura del pipeline | 5 |
| 2 | Saini Alejo | Filtrar la liga y fijar el snapshot | 5 |
| 3 | Podesta Isidro | Arreglar la validación | 5 |
| 4 | Dallape Vincenzo | Armar el zip y cierre | 5 |

El reparto se puede intercambiar. Lo que no conviene mover es el orden de los temas.

---

## Antes de empezar

Esto va para todo el grupo, y conviene resolverlo el día anterior.

- **Correr el DAG dos veces** con `mode` en `full`, `roster` en 260046 y `league_id` en 80. La segunda corrida es la que demuestra el reuso del bronce.
- **Tener el zip ya generado y descomprimido** en una carpeta, para poder mostrarlo sin depender de que la corrida termine en vivo.
- **Dejar abiertas tres pestañas**: la vista de grafo del DAG, los logs de `land_bronze` de la segunda corrida, y el `manifiesto.json`.
- **Tener el código a mano** en el editor, en las cuatro zonas que se tocaron. Son pocas líneas y se llega rápido si están marcadas de antemano.
- Que **todos** puedan responder qué es la capa de bronce. Es lo que más se pregunta y no queda bien que sólo lo sepa quien expuso ese bloque.

---

## Bloque 1 · Carobolante Alejo — Contexto y arquitectura

**Objetivo:** que se entienda qué había antes de tocar nada. Si este bloque sale bien, los otros tres se apoyan solos.

### Qué contar

1. **Cuál era la consigna.** No escribir un pipeline, sino leer uno ajeno, entenderlo y modificarlo. El entregable lo genera el propio DAG.

2. **Qué hace el DAG original.** Construye el dataset canónico de la materia scrapeando sofifa: 18.936 jugadores de 52 ligas, 90 columnas, una fila por jugador.

3. **Recorrer el grafo en pantalla**, tarea por tarea, sin entrar en código. Un sensor que espera a que la fuente responda, una rama que decide si se sigue por la fuente o por el respaldo, un cortocircuito que corta la corrida si no hay nada nuevo, el descubrimiento de ligas, la descarga, el parseo, la consolidación, la validación y el guardado.

4. **El modelo medallón.** Es el concepto central del bloque y hay que explicarlo despacio.
   - Bronce es el HTML tal como llegó, comprimido y guardado en carpetas por snapshot y por liga.
   - Plata es el dato tipado, una fila por jugador.
   - Oro no se construye acá, se arma en las unidades 3 y 4.

5. **Por qué se guarda el HTML crudo.** Las dos propiedades que lo justifican:
   - Una página que ya está en bronce no se vuelve a pedir, así que la segunda corrida sobre el mismo snapshot no genera ni una request.
   - Un bug en el parseo se arregla sin volver a la fuente: se corrige la plata y se reprocesa lo que ya está en disco.

6. **Los tres niveles de degradación.** HTTP directo, después navegador si Cloudflare devuelve 403, y por último el respaldo en disco. Los dos primeros no aparecen en el grafo porque son la misma página por otra puerta. El tercero sí aparece, porque ahí el dato es otro, más viejo, y eso hay que comunicarlo.

### Qué mostrar

La vista de grafo del DAG, y los logs de `land_bronze` de la **segunda** corrida, donde dice cuántas páginas se pidieron y cuántas se reusaron. Que se lea el cero en voz alta.

### Frase para cerrar el bloque

> Un pipeline de producción no tiene una fuente, tiene un plan para cuando la fuente falla. Eso es lo que se ve en el grafo.

### Preguntas probables

**¿Por qué usar Airflow y no un script?** Por reintentos, ejecución programada, paralelismo, y sobre todo por visibilidad: se ve qué tarea falló, cuándo, con qué logs, y se puede reprocesar sólo esa sin volver a correr todo.

**¿Por qué corre todos los días si casi nunca hay datos nuevos?** sofifa publica un parche cada una o dos semanas. El cortocircuito corta la corrida cuando el snapshot no cambió. La mayoría de las corridas de un pipeline sano no hacen nada, y eso está bien; lo que no está bien es no darse cuenta.

---

## Bloque 2 · Saini Alejo — Filtrar la liga y fijar el snapshot

**Objetivo:** mostrar que el filtro se puso donde había que ponerlo, y que fijar el snapshot no es un capricho del enunciado.

### Qué contar

1. **El cambio.** El DAG bajaba las 52 ligas. Nosotros necesitamos sólo la Bundesliga de Austria, que es la liga 80, con unos 348 jugadores.

2. **Se agregó un parámetro al DAG** para no hardcodear el número. Queda visible en la interfaz al disparar la corrida.

3. **Dónde se filtra, y este es el punto fuerte del bloque.** El filtro va dentro de `discover_leagues`, inmediatamente después de pedir el catálogo. Esa tarea es la que decide el tamaño del trabajo: lo que devuelve es la lista sobre la que se crean las tareas mapeadas. Filtrando ahí se crea **una sola** tarea de descarga, y no se le pide a sofifa nada que no vayamos a usar.

   El contraste que conviene decir en voz alta: filtrar al final, por ejemplo al consolidar, daría el mismo CSV, pero habríamos bajado 52 ligas para tirar 51.

4. **Fijar el snapshot en 260046.** La razón profunda es que **el catálogo de ligas se pide para el mismo snapshot que después se scrapea**, y ese catálogo cambia entre actualizaciones. La de agosto de 2025 tenía 51 ligas y 18.035 jugadores; la de julio de 2026 tiene 52 y 18.936. Pedir el catálogo actual para scrapear un snapshot viejo daría conteos equivocados y ligas que en ese momento no existían.

5. **Dos efectos que el código ya tenía previstos.** Fijar el roster hace que el cortocircuito deje pasar la corrida sin consultar la variable, porque si se pide un snapshot puntual "el último que vi" no es la pregunta correcta. Y el roster se resuelve una sola vez y viaja con cada liga, para que dos descargas separadas por minutos no caigan en snapshots distintos y se mezclen en la misma carpeta.

### Qué mostrar

El bloque del parámetro y las tres líneas del filtro. Después, la carpeta de bronce en disco, para que se vea la ruta con el roster y la liga en el nombre de las carpetas.

### Preguntas probables

**¿Por qué el snapshot y la liga van en la ruta y no en el nombre del archivo?** Porque cada actualización es un lote independiente. Así conviven varios sin pisarse y se puede borrar uno entero sin tocar los demás. Es el mismo particionado que usa cualquier data lake, con carpetas en lugar de prefijos de bucket.

**¿Qué pasa si no se pasa la liga?** El parámetro queda vacío y el DAG procesa las 52 ligas, que es el comportamiento original. Hay que pasarlo explícitamente en la corrida de entrega.

---

## Bloque 3 · Podesta Isidro — Arreglar la validación

**Objetivo:** este es el bloque que más se corrige, porque el enunciado pide justificar la decisión. No alcanza con decir qué número pusimos, hay que decir por qué.

### Qué contar

1. **El síntoma.** Al filtrar la liga, la tarea `validate` falla. No es un error nuestro, el enunciado avisa que es a propósito.

2. **Por qué falla.** El chequeo original exige al menos 500 filas. Con 52 ligas y 18.936 jugadores, 500 es un piso trivial. Con una sola liga de 348, ese mismo piso hace fallar toda corrida correcta.

3. **Qué está protegiendo ese chequeo, que es la pregunta de fondo.** No es un conteo exacto, es un detector de corridas rotas. Hay dos escenarios concretos:
   - sofifa cambia la estructura del HTML y el parser devuelve pocas filas o ninguna, sin lanzar ninguna excepción.
   - Se corta la red a mitad del scraping y se pierden páginas.

   En los dos casos el pipeline terminaría en verde y publicaría un dataset mutilado. El umbral existe para que eso falle ruidosamente en vez de pasar en silencio.

4. **Nuestra decisión: 300.** Y acá va la aritmética, que es lo que hace defendible el número.

   sofifa pagina de a 60 jugadores por página, y ese valor está en el código en la constante `PAGE_SIZE`. Nuestra liga tiene unos 348 jugadores, o sea **cinco páginas completas más 48 jugadores de la sexta**.

   | Páginas completas | Jugadores |
   |---|---|
   | 5 | 300 |
   | 5 más el resto de la sexta | 348 |

   Poner el umbral en 300 equivale a exigir que se hayan bajado y parseado las cinco páginas completas. Si falta cualquiera de ellas, el conteo cae por debajo y la corrida falla. Al mismo tiempo deja margen para los 48 jugadores de la última página, que son los que pueden variar legítimamente entre snapshots por transferencias o bajas.

5. **La idea que conviene dejar dicha.** El umbral quedó pegado a una unidad real del pipeline, que es la página, en vez de ser un número redondo elegido a ojo. Por eso se puede defender.

### Qué mostrar

El chequeo antes y después, y el comentario con la justificación que quedó en el código.

### Preguntas probables

**¿Por qué no poner 348, que es el número exacto?** Porque el conteo real varía entre snapshots por transferencias y bajas, y un umbral exacto haría fallar corridas correctas. El chequeo busca detectar una corrida rota, no verificar un número.

**¿Y por qué no 340, o 200?** 340 sería casi exacto y tendría el mismo problema. 200 dejaría pasar una corrida a la que le falta una página y media, que es exactamente el caso que el chequeo tiene que atajar. 300 es el único valor que se corresponde con un límite de página.

**¿Qué otros chequeos hace `validate`?** Que las columnas sean exactamente las 90 del esquema, que no haya identificadores de jugador repetidos, que las columnas obligatorias no tengan nulos, y que la valoración general esté entre 1 y 99.

---

## Bloque 4 · Dallape Vincenzo — Armar el zip y cierre

**Objetivo:** mostrar que el entregable lo produjo el pipeline y no una persona, que es literalmente lo que se corrige.

### Qué contar

1. **El cambio.** Se agregó una tarea nueva al final, después de `save`, que recibe la ruta del CSV que produjo la corrida y arma un único archivo comprimido.

2. **Qué lleva adentro**, recorriendo el zip descomprimido en pantalla:
   - El dataset de la corrida.
   - El manifiesto en JSON.
   - El listado de páginas crudas que se usaron.
   - Los logs reales de la corrida.
   - El archivo del DAG modificado.

3. **El manifiesto se lee, no se escribe.** Este es el punto que más mira la corrección, y conviene decirlo explícitamente.
   - Las filas y las columnas salen de medir el CSV con pandas.
   - El identificador de la corrida sale del contexto de ejecución de Airflow.
   - La liga y el roster salen de los parámetros con los que se disparó la corrida.
   - Lo único fijo son el grupo y los integrantes, y está bien que lo sean.

4. **Cómo se verifica que no está inventado.** Abrir el manifiesto y el CSV al lado: las filas que declara uno tienen que ser las que tiene el otro. Y los logs del zip tienen que llevar el mismo identificador de corrida que declara el manifiesto. Esa es la cadena que hace que el entregable se pueda auditar.

5. **El listado de bronce.** Se arma recorriendo la carpeta del roster y la liga. Las rutas llevan el snapshot y la liga adentro, así que el archivo demuestra por sí solo que se bajó lo que se tenía que bajar.

### Cierre del grupo

Un par de ideas para terminar, elegí una o dos y no más:

- **El bronce se entiende corriendo el DAG dos veces.** La segunda corrida no le pide nada a sofifa. Guardar el HTML crudo deja de ser una recomendación de manual y pasa a ser algo que se ve.

- **Un cambio en una punta del pipeline toca la otra.** La tarea `save` refresca el respaldo cuando la corrida es completa y los datos vienen de la fuente. Nuestra corrida cumple las dos condiciones pero baja una sola liga, así que pisa un respaldo de 52 ligas con uno de 348 filas. No es grave porque la semilla versionada sigue intacta, pero es justo el retroceso que la condición del modo de desarrollo intentaba evitar. Lo encontramos leyendo, no fallando.

- **La degradación no es teoría.** Si la fuente no responde, la corrida sigue por el respaldo y el dataset es viejo. El pipeline lo avisa en el log, y por eso antes de entregar hay que confirmar en el manifiesto que las filas son 348 y la liga es la 80.

### Qué mostrar

El zip descomprimido, y el manifiesto al lado del CSV para que se vea que los números coinciden.

### Preguntas probables

**¿Por qué la tarea va después de `save` y no en paralelo?** Porque necesita la ruta del CSV final, que es justamente lo que `save` devuelve. La dependencia sale del dato, no hace falta declararla a mano.

**¿Los logs del zip están completos?** Los de la propia tarea que arma el zip no, porque se están escribiendo mientras corre. Los de todas las tareas anteriores sí, que son las que importan para verificar la corrida.

---

## Preguntas para todo el grupo

Cualquiera puede recibirlas, así que conviene que las cuatro respuestas estén sabidas.

**¿Qué es la capa de bronce y para qué sirve?** El dato tal como llegó, sin interpretar. Sirve para no volver a pedirle lo mismo a la fuente y para poder arreglar el parseo sin volver a scrapear.

**¿Por qué por XCom viajan rutas y no filas?** Porque XCom guarda en la base de metadatos de Airflow. Pasar miles de filas por ahí la satura. Cada tarea escribe su parte a disco y devuelve la ruta.

**¿Por qué el código usa `urllib` en vez de `requests`?** Porque Cloudflare bloquea con 403 a los clientes construidos sobre `urllib3`, y `requests` es uno de ellos, por la huella del handshake TLS. La biblioteca estándar pasa.

**¿Qué hace el sensor en modo `reschedule`?** En vez de ocupar un worker durmiendo entre sondeos, libera la tarea y Airflow la vuelve a encolar. Con un sensor da igual; con muchos esperando, es lo que evita tapar el scheduler.

**¿Cuánto del DAG tuvieron que leer?** Los cuatro cambios tocan unas cuarenta líneas. El resto se puede ignorar, y esa era parte de la idea del TP.
