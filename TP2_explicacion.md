# TP 2 · Tres hipótesis sobre el dataset canónico
### Grupo 5K10-06 · Ciencia de Datos · UTN FRM · 2026

Integrantes: Carobolante Alejo · Dallape Vincenzo · Podesta Isidro · Saini Alejo
Liga del TP1: Bundesliga (Austria), `league_id` 80

Entregable: [`tp2_grupo_6.ipynb`](tp2_grupo_6.ipynb), ejecutado de arriba a abajo, 12 celdas de código, sin errores y con las salidas guardadas.

---

## 1. Qué pedía el TP

El TP es corto a propósito. La limpieza, la auditoría y el análisis exploratorio vienen resueltos en los notebooks de la práctica. Lo único que se pide es lo que no se puede dar hecho: **plantear hipótesis propias y contestarlas con evidencia.**

Las restricciones son ocho, y las cumplimos todas:

| Restricción | Cómo la cumplimos |
|---|---|
| Tres hipótesis con la ficha de seis campos | Las tres, con los seis títulos en el orden pedido |
| Al menos una de responder y una de predecir | H1 y H3 responden, H2 predice |
| No las tres con la misma plantilla | Asociación, comparación y composición |
| Al menos una amarilla con movimiento | H1: amarillo, un movimiento |
| Una tiene que usar la liga del TP1 | H3 filtra por `league_id == 80` |
| Gráficos con título y ejes con nombre | Los cuatro |
| Celda inicial con grupo, integrantes y liga | Es la portada |
| Notebook ejecutado en orden y sin errores | Numeración 1 a 12, cero errores |

## 2. Las dos preguntas

El enunciado separa dos tipos de hipótesis y conviene declarar cuál es cuál, porque terminan en cosas distintas.

- **Qué queremos responder** es la pregunta de negocio, qué determina el valor de un jugador, y termina en una conclusión.
- **Qué queremos predecir** es la de modelado, la posición, que es el objetivo de la Unidad 3, y termina en una decisión sobre qué columnas usar.

Nosotros pusimos dos de responder y una de predecir.

## 3. Las tres hipótesis de un vistazo

| # | Pregunta | Plantilla | Medida | Resultado | Zona | Movimientos |
|---|---|---|---|---|---|---|
| 1 | Responder | Asociación | Correlación de Pearson | 0,550 → 0,896 | Amarillo → Verde | 1 |
| 2 | Predecir | Comparación | Separación estandarizada | 0,086 | Rojo | ninguno |
| 3 | Responder | Composición | η² | 0,284 | Verde | ninguno |

Las tres cayeron en zonas distintas, lo cual no estaba planeado pero deja el notebook bastante completo como muestra de los tres desenlaces posibles.

---

## 4. Hipótesis 1 · El valor y la valoración general

**Afirmación.** El valor de mercado (`value_eur`) está asociado a la valoración general (`overall`): los jugadores con mayor `overall` valen más.

**Qué esperábamos.** Pearson por encima de 0,70, verde de entrada y sin movimientos. El `overall` es el número con el que el propio juego resume a un jugador.

**Qué encontramos.**

| Medida | Valor | Zona |
|---|---|---|
| Pearson | +0,550 | Amarillo |
| Spearman | +0,883 | — |
| Brecha Spearman − Pearson | +0,333 | Por encima de 0,15 |

El gráfico de dispersión explica la diferencia. La nube no sube pareja: se queda pegada al piso hasta un `overall` de 70, ahí se despega, y a partir de 80 se dispara. Pearson mide parecido a una **recta**, y esto no es una recta. Spearman, que sólo mira el orden, da 0,883: el orden se respeta casi perfecto, lo que falla es la forma.

**El movimiento: cambiar la escala.** Uno solo.

El enunciado dice que ese movimiento corresponde cuando la nube sube de forma ordenada pero curva y la brecha entre Spearman y Pearson es grande. Es exactamente el caso, con una brecha de 0,333 contra un corte de 0,15. Los otros dos movimientos no aplicaban: no hay grupos que se despeguen, así que no había nada que partir, y el resultado no contradice nada conocido, así que no había una tercera variable que controlar.

Con el valor en logaritmo base 10, Pearson pasa a **+0,896, verde**.

**Qué hacemos con eso.** `value_eur` entra al modelo en escala logarítmica, nunca cruda. En euros crudos cualquier método que asuma relaciones rectas subestima el peso del `overall`, y un puñado de jugadores de más de cien millones domina el error frente a los 18.900 restantes. La misma lógica corre para `wage_eur` y `release_clause_eur`.

---

## 5. Hipótesis 2 · La altura y el mediocampo

**Afirmación.** Entre mediocampistas defensivos (CDM) y mediocampistas centrales (CM), la altura difiere: los CDM son más altos.

**Qué esperábamos.** Separación alrededor de 0,40 o 0,50, amarillo. El CDM juega más cerca del área propia y disputa más pelotas aéreas, así que esperábamos dos o tres centímetros de diferencia.

**Qué encontramos.**

| Grupo | Jugadores | Media |
|---|---|---|
| CDM | 1.289 | 180,50 cm |
| CM | 1.193 | 180,02 cm |

Separación estandarizada: **+0,086, rojo.** La diferencia real es de 0,48 cm, menos de medio centímetro, sobre un desvío de 5,6 cm. Los histogramas se superponen casi por completo.

**La hipótesis queda refutada,** y el enunciado dice explícitamente que eso vale lo mismo que una confirmada. Con casi 2.500 jugadores esa diferencia seguro daría estadísticamente significativa, y es justo el caso que el semáforo está para atajar: el efecto existe pero es demasiado chico para cambiar ninguna decisión.

**Movimiento: ninguno.** En rojo se termina.

**Qué hacemos con eso.** Acá está el matiz que hace útil la ficha. La columna **no se descarta**, porque el problema no es la columna sino la frontera que le pedimos distinguir. Lo comprobamos midiendo la misma columna en otra frontera:

| Comparación | Separación | Zona |
|---|---|---|
| CDM vs CM | 0,086 | Rojo |
| CB vs laterales (LB/RB) | 1,714 | Verde |

La conclusión no es que la altura no sirva, es que **sirve para unas fronteras y no para otras**. Entonces `height_cm` entra al modelo de la Unidad 3, pero no puede ser la que resuelva el mediocampo. Para separar CDM de CM hay que ir a atributos de rol como `defending_standing_tackle` o `mentality_interceptions`. Y si el modelo termina confundiendo esos dos puestos, ya sabemos que sumar variables de físico no lo va a arreglar.

Esa comprobación está declarada en el notebook como apoyo del campo 6, no como la medida de la hipótesis. Es importante que se lea así: la medida se elige antes de ver resultados, y no se cambia después.

---

## 6. Hipótesis 3 · El club y el valor, dentro de nuestra liga

Es la hipótesis que usa la liga del TP1. El filtro es por `league_id == 80`, no por nombre, como pide el enunciado.

**Afirmación.** Dentro de la Bundesliga de Austria, el club al que pertenece un jugador explica una parte apreciable de la variación de su valor de mercado.

**Qué esperábamos.** Amarillo, alrededor de 0,10 o 0,15. Es una liga chica y pareja, doce clubes con planteles de tamaño similar, sin las diferencias de presupuesto de las ligas grandes de Europa.

**Qué encontramos.** η² = **0,284, verde**. El club explica el 28 % de la variación del valor, casi el doble de lo que esperábamos.

El diagrama de cajas muestra por qué. No son doce cajas alineadas, hay un escalón:

| Grupo | Clubes | Mediana |
|---|---|---|
| Arriba | Red Bull Salzburg, LASK Linz, Sturm Graz, Rapid | 1,35 a 1,85 millones |
| Abajo | Rheindorf Altach, SV Ried | 0,54 a 0,63 millones |

Más de tres veces de diferencia entre las puntas, dentro de la misma liga. La lectura futbolística es directa: arriba están los clubes que juegan copas europeas y venden al exterior. La liga es pareja en cantidad de jugadores por plantel, no en dinero.

**Movimiento: ninguno.** En verde se termina.

**Qué hacemos con eso.** Dos decisiones separadas.

La columna del club **sale** del modelo de la Unidad 3. Explica el precio, pero el objetivo es la posición, y no hay razón para que el escudo prediga si alguien es lateral o extremo. Además son doce categorías acá y cientos en el dataset completo, así que sólo aportaría ruido y riesgo de sobreajuste.

Y `value_eur` **arrastra el efecto del club**. Si se usa como variable de entrada, hay que tener presente que un mismo jugador vale distinto según dónde juegue. Con un η² de 0,284 eso no es un detalle: para comparar jugadores entre clubes habría que normalizar el valor dentro del club, o aceptar que la variable mezcla calidad individual con poder económico del empleador.

---

## 7. Decisiones de método que conviene poder defender

**Por qué cada medida.** La medida sale de la plantilla, no se elige después de ver cuál da mejor. Dos numéricas piden correlación. Una numérica entre dos grupos pide separación estandarizada, y no diferencia de medias en centímetros, porque los centímetros no se pueden comparar contra el semáforo. Una numérica entre muchos grupos pide η².

**Por qué el campo 2 no coincide con el campo 4 en ninguna de las tres.** Las expectativas se escribieron antes de correr el código y las tres fallaron: la primera esperaba verde y dio amarillo, la segunda esperaba amarillo y dio rojo, la tercera esperaba amarillo y dio verde. El enunciado avisa que ese campo se corrige y que se nota cuando dice exactamente lo que después se encontró.

**Por qué excluimos once filas en la hipótesis 1.** Hay once jugadores con `value_eur` igual a cero. Un cero ahí no es un precio bajo, es un dato ausente, y además rompe el logaritmo. Quedan 18.925.

**Por qué el notebook no usa scipy ni seaborn.** Spearman se calcula como Pearson sobre los rangos, que es su definición, así que no hace falta scipy. Los gráficos son matplotlib puro. Eso reduce a tres las dependencias y hace más probable que el notebook corra en la máquina de corrección sin instalar nada.

**Por qué no repetimos el análisis exploratorio.** El enunciado lo dice explícito: si el notebook arranca con un `describe()` de noventa columnas, arrancó mal. Sólo verificamos el tamaño del dataset, la fecha del snapshot y el conteo de la liga, que es contexto mínimo, no auditoría.

---

## 8. Para revisar antes de entregar

Cuatro cosas, en orden de importancia.

### 8.1 Falta la URL de descarga del dataset

Es lo único bloqueante. La celda de carga tiene una constante `DATA_URL` vacía:

```python
DATA_URL = ""   # <-- pegar acá la URL de los notebooks de la práctica
```

Hoy el notebook encuentra el CSV por ruta relativa dentro del repositorio, así que corre acá. Pero la corrección lo ejecuta **en una máquina limpia**, y ahí no va a haber ningún archivo al lado. Hay que abrir cualquiera de los tres notebooks de la práctica, copiar la URL desde la que se bajan el dataset, y pegarla ahí. Es una línea.

### 8.2 Confirmar el nombre del archivo

El enunciado pide `tp2_grupo_N.ipynb` con el número de grupo. Nuestro código es 5K10-06, así que lo llamamos `tp2_grupo_6.ipynb`. Si en el campus el grupo figura como 06, hay que renombrarlo a `tp2_grupo_06.ipynb`.

### 8.3 La plantilla de composición

El enunciado nombra tres plantillas, comparación, asociación y composición, pero la tabla del semáforo sólo lista cuatro medidas, y ninguna se llama de composición. Nosotros interpretamos que composición es preguntar **cómo se reparte** la variación de una numérica entre las categorías que componen la población, y la medimos con η², que es literalmente la proporción de la variación total explicada por el grupo.

Es una interpretación defendible y está explicada en la ficha. Pero si en el notebook de la práctica composición está definida de otra manera, por ejemplo como proporciones de categorías, la hipótesis 3 es la que hay que adaptar. El cambio sería sólo en el campo 3 y en la medida; la afirmación, el gráfico y la conclusión se sostienen igual.

### 8.4 Chequeo final antes de subir

Reiniciar el kernel y correr todo de nuevo de arriba a abajo. El enunciado dice que un notebook con celdas sin correr o con la numeración desordenada se devuelve sin corregir. Ahora mismo la numeración va de 1 a 12 sin saltos, pero cualquier edición posterior la puede romper.
