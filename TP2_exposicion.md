# TP 2 · Guión de exposición
### Grupo 5K10-06 · Bundesliga (Austria) · `league_id` 80

Cuatro bloques de unos cinco minutos. El primero arma el marco y los otros tres son una hipótesis cada uno, en el orden del notebook.

| Bloque | Quién | Tema | Minutos |
|---|---|---|---|
| 1 | Carobolante Alejo | El marco: las dos preguntas, las plantillas y el semáforo | 5 |
| 2 | Saini Alejo | Hipótesis 1 · el valor y el overall (amarillo → verde) | 5 |
| 3 | Podesta Isidro | Hipótesis 2 · la altura y el mediocampo (rojo) | 5 |
| 4 | Dallape Vincenzo | Hipótesis 3 · el club y el valor en nuestra liga (verde) + cierre | 5 |

Es el mismo reparto que en el TP1, así que cada uno sigue con el hilo que ya venía trayendo.

---

## Antes de empezar

- **Pegar la URL del dataset** en la celda de carga y correr el notebook entero una vez más. Sin eso, el notebook no corre en una máquina limpia.
- **Reiniciar el kernel y ejecutar todo de arriba a abajo.** La numeración tiene que quedar de 1 a 12 sin saltos.
- **Tener el notebook abierto en los cuatro gráficos**, para saltar a cada uno sin buscarlo.
- Que **todos** sepan responder por qué el semáforo existe. Es la pregunta más probable y no queda bien que la sepa sólo quien expuso el bloque 1.

---

## Bloque 1 · Carobolante Alejo — El marco

**Objetivo:** que se entienda el método antes de ver un solo número. Sin esto, los otros tres bloques son tres resultados sueltos.

### Qué contar

1. **Qué pedía el TP.** No hacer un análisis exploratorio, que ya viene resuelto en la práctica, sino plantear hipótesis propias y contestarlas con evidencia. Si el notebook arrancara con un `describe()` de noventa columnas, habríamos arrancado mal.

2. **Las dos preguntas, y por qué conviene saber de cuál nace cada hipótesis.**
   - Qué queremos **responder** es la pregunta de negocio, qué determina el valor de un jugador, y termina en una conclusión.
   - Qué queremos **predecir** es la de modelado, la posición, que es el objetivo de la Unidad 3, y termina en una decisión sobre qué columnas usar.

   Nosotros pusimos dos de responder y una de predecir.

3. **Las tres plantillas, y que la medida sale de la plantilla.** Este es el punto de método más importante del bloque. La forma de la afirmación determina con qué se mide, y eso se decide **antes** de ver el resultado. Dos numéricas piden correlación. Una numérica entre dos grupos pide separación estandarizada. Una numérica entre muchos grupos pide η².

   La frase que conviene decir: no se prueban las tres medidas a ver cuál da mejor.

4. **El semáforo, y sobre todo por qué existe.** Con 18.936 filas casi cualquier diferencia da estadísticamente significativa. El semáforo no pregunta si el efecto existe, pregunta si es **lo bastante grande como para cambiar una decisión**. En verde y en rojo se termina; en amarillo hay que indagar.

5. **Los movimientos.** Amarillo habilita hasta dos, y se eligen mirando el gráfico, no a ojo. Partir la población si se ven grupos que se comportan distinto. Controlar una tercera variable si el resultado contradice algo que sabemos. Cambiar la escala si la nube sube ordenada pero curva. Y el tope de dos existe porque, partiendo el dataset lo suficiente, cualquier hipótesis termina confirmada y eso no significa nada.

6. **Presentar la tabla de las tres**, que está en la portada del notebook, y decir que cayeron una en cada zona.

### Qué mostrar

La portada del notebook con la tabla resumen, y la tabla del semáforo con los cortes.

### Frase para cerrar el bloque

> El semáforo no pregunta si el efecto existe. Pregunta si es lo bastante grande como para cambiar una decisión.

### Preguntas probables

**¿Por qué no usan un test de significancia?** Porque con casi 19.000 filas todo da significativo. La pregunta útil no es si la diferencia es distinta de cero, es si es grande. Por eso todas las medidas del TP son tamaños de efecto.

**¿Los cortes de dónde salen?** Son convenciones para elegir variables, no leyes. Sirven para esta materia y el enunciado lo dice así.

---

## Bloque 2 · Saini Alejo — El valor y la valoración general

**Objetivo:** es la única hipótesis con movimiento, así que es donde se muestra el método completo de punta a punta.

### Qué contar

1. **La afirmación.** El valor de mercado está asociado a la valoración general: a mayor `overall`, mayor `value_eur`. Plantilla asociación, dos numéricas, medida correlación de Pearson.

2. **Qué esperábamos, y decirlo antes de mostrar el resultado.** Esperábamos Pearson por encima de 0,70, verde de entrada. El `overall` es el número con el que el juego resume a un jugador, así que tendría que mandar sobre el precio.

3. **Qué salió.** Pearson **0,550, amarillo**. Más bajo de lo que esperábamos.

4. **Mostrar el primer gráfico y leerlo en voz alta.** Acá está el corazón del bloque. La nube no sube pareja: se queda pegada al piso hasta un `overall` de 70, ahí se despega y a partir de 80 se dispara. Pearson mide cuánto se parece la nube a una **recta**, y esto no es una recta.

5. **El diagnóstico.** Spearman, que sólo mira el orden, da 0,883. La brecha entre las dos da 0,333, muy por encima del corte de 0,15. Traducción: el orden se respeta casi perfecto, lo que falla es la forma. El 0,550 estaba subestimando una relación que en realidad es muy fuerte.

6. **El movimiento: cambiar la escala.** Y justificar por qué ése y no otro, que es lo que se corrige. El enunciado dice que ese movimiento corresponde cuando la nube sube ordenada pero curva y la brecha es grande, que es exactamente lo que se ve. Los otros dos no aplicaban: no hay grupos que se despeguen, así que no había nada que partir, y el resultado no contradice nada conocido, así que no había tercera variable que controlar.

7. **El resultado después del movimiento.** Con el valor en logaritmo, Pearson pasa a **0,896, verde**. Mostrar el segundo gráfico: la misma nube, enderezada. La relación siempre estuvo ahí, lo que estaba mal era la escala en la que la mirábamos.

8. **La consecuencia.** `value_eur` entra al modelo en escala logarítmica, nunca cruda. Si no, cualquier método lineal subestima el peso del `overall`, y un puñado de jugadores de más de cien millones domina el error frente a los 18.900 restantes.

### Qué mostrar

Los dos gráficos de dispersión, uno después del otro. El contraste entre la nube curva y la nube derecha es el mejor momento visual del notebook.

### Preguntas probables

**¿Por qué logaritmo y no una raíz o un polinomio?** Porque los precios se comportan de forma multiplicativa: la diferencia entre 1 y 2 millones pesa lo mismo que entre 50 y 100. El logaritmo convierte eso en una escala aditiva. Y el resultado lo confirma, la nube queda derecha.

**¿Por qué sacaron once jugadores?** Tienen `value_eur` en cero, que no es un precio bajo sino un dato ausente en la fuente, y además rompe el logaritmo. Quedan 18.925.

---

## Bloque 3 · Podesta Isidro — La altura y el mediocampo

**Objetivo:** mostrar que una hipótesis refutada vale igual que una confirmada, y que el interesante no es el rojo sino lo que se hace con él.

### Qué contar

1. **La afirmación.** Entre mediocampistas defensivos y mediocampistas centrales, la altura difiere: los CDM son más altos. Es la hipótesis de **predecir**, porque pregunta si esa columna sirve para distinguir puestos, que es el objetivo de la Unidad 3.

2. **Por qué era razonable.** El CDM juega más cerca del área propia, disputa más pelotas aéreas y cubre a los delanteros centro en las jugadas paradas. Esperábamos entre dos y tres centímetros, o sea amarillo, alrededor de 0,40 o 0,50.

3. **Qué salió.** Separación estandarizada **0,086, rojo**. La diferencia real es de 0,48 cm sobre un desvío de 5,6. Menos de medio centímetro.

4. **Mostrar el histograma.** Las dos campanas se superponen casi por completo y las líneas de media están pegadas. No hay nada que interpretar, y eso también es un resultado.

5. **El punto de método, que es lo que conviene subrayar.** Con casi 2.500 jugadores, esa diferencia seguro daría estadísticamente significativa. Y no significa nada. Es justo el caso que el semáforo está para atajar: el efecto existe, pero es demasiado chico para cambiar ninguna decisión.

6. **Movimiento: ninguno.** En rojo se termina, y eso está bien. Forzar un movimiento acá sería buscar un subgrupo donde el efecto aparezca, que es exactamente lo que el tope de dos movimientos previene.

7. **La consecuencia, que es la parte más rica.** La columna **no se descarta**. El problema no es la columna, es la frontera que le pedimos distinguir. Lo comprobamos midiendo la misma columna en otra frontera:

   | Comparación | Separación | Zona |
   |---|---|---|
   | CDM vs CM | 0,086 | Rojo |
   | Centrales vs laterales | 1,714 | Verde |

   La conclusión no es que la altura no sirva, es que **sirve para unas fronteras y no para otras**. Entonces `height_cm` entra al modelo, pero no puede ser la que resuelva el mediocampo. Para separar CDM de CM hay que ir a atributos de rol, no de físico.

8. **Y algo que ya sabemos para la Unidad 3:** si el modelo termina confundiendo esos dos puestos, sumar variables de físico no lo va a arreglar.

### Qué mostrar

El histograma superpuesto, y después la celda de la comprobación con el 1,714.

### Preguntas probables

**¿Esa segunda comparación no es cambiar la medida después de ver el resultado?** No, y conviene tenerlo claro. La medida de la hipótesis es una sola y está declarada antes: CDM contra CM. La segunda comparación está en el campo 6 y sirve para justificar la decisión de conservar la columna, y en el notebook está rotulada así.

**¿Por qué separación estandarizada y no la diferencia en centímetros?** Porque los centímetros no se pueden comparar contra el semáforo. Hacen falta unidades de desvío, y eso es exactamente lo que hace la d de Cohen.

---

## Bloque 4 · Dallape Vincenzo — El club y el valor, y cierre

**Objetivo:** es la hipótesis que usa nuestra liga, así que hay que dejar claro que el filtro es por `league_id` y que el resultado es específico de esa población.

### Qué contar

1. **La afirmación.** Dentro de la Bundesliga de Austria, el club al que pertenece un jugador explica una parte apreciable de la variación de su valor de mercado. Filtramos por `league_id == 80`, no por nombre, como pide el enunciado.

2. **La plantilla.** Composición: la pregunta es **cómo se reparte** la variación del valor entre las categorías que componen la población, que son doce clubes. Por eso la medida es η² y no separación estandarizada: los grupos son doce, no dos.

3. **Qué esperábamos.** Amarillo, alrededor de 0,10 o 0,15. Es una liga chica y pareja, doce clubes con planteles de tamaño similar, sin las diferencias de presupuesto de las ligas grandes. Esperábamos que el valor lo explicara más el jugador que el escudo.

4. **Qué salió.** η² **0,284, verde**. El club explica el 28 % de la variación del valor, casi el doble de lo previsto.

5. **Mostrar el diagrama de cajas y leer el escalón.** No son doce cajas alineadas. Arriba están Red Bull Salzburg, LASK Linz, Sturm Graz y Rapid, con medianas entre 1,35 y 1,85 millones. Abajo, Rheindorf Altach y SV Ried, con medianas de 540 a 630 mil. Más de tres veces entre las puntas, dentro de la misma liga.

6. **La lectura futbolística.** Arriba están los clubes que juegan copas europeas y venden al exterior. La liga es pareja en cantidad de jugadores por plantel, no en dinero.

7. **Movimiento: ninguno.** En verde se termina.

8. **Las dos consecuencias, que son distintas.**
   - La columna del club **sale** del modelo de la Unidad 3. Explica el precio, pero el objetivo es la posición, y el escudo no dice si alguien es lateral o extremo. Además son doce categorías acá y cientos en el dataset completo, así que sólo aportaría ruido.
   - `value_eur` **arrastra el efecto del club**. Un mismo jugador vale distinto según dónde juegue. Con un η² de 0,284 eso no es un detalle: para comparar entre clubes habría que normalizar el valor dentro del club, o aceptar que la variable mezcla calidad individual con poder económico del empleador.

### Cierre del grupo

Elegí una o dos ideas, no las tres.

- **Ninguna de las tres expectativas se cumplió.** La primera esperaba verde y dio amarillo, la segunda esperaba amarillo y dio rojo, la tercera esperaba amarillo y dio verde. Escribir el campo 2 antes de correr el código sirve justamente para eso: para poder equivocarse y que se note.

- **La que más información dejó fue la que falló.** Descubrir que la altura no distingue el mediocampo nos evita perder tiempo en la Unidad 3 buscando por el lado del físico un problema que es de rol.

- **Lo que queda abierto.** No sabemos todavía separar CDM de CM. Y no medimos cuánto del efecto del club sobre el valor es en realidad efecto de la calidad de los jugadores que ese club puede pagar: es una tercera variable que empuja a las dos cosas a la vez y habría que controlarla.

### Qué mostrar

El diagrama de cajas por club, y después la celda de cierre con las columnas que entran y salen.

### Preguntas probables

**¿Por qué η² y no comparar dos clubes?** Porque la afirmación es sobre los doce, no sobre un par elegido. Quedarse con dos clubes sería elegir la comparación después de ver los datos.

**¿28 % es mucho o poco?** Contra el corte del semáforo es verde, que empieza en 0,25. Y en contexto: quiere decir que si sólo supieras en qué club juega alguien, ya explicarías más de un cuarto de la variación de su precio, sin saber nada del jugador.

---

## Preguntas para todo el grupo

**¿Por qué existe el semáforo?** Con 18.936 filas casi cualquier diferencia da significativa. El semáforo pregunta si el efecto es lo bastante grande como para cambiar una decisión, no si existe.

**¿Cómo eligieron las medidas?** Salen de la plantilla, y la plantilla sale de la forma de la afirmación. Se decide antes de ver el resultado, no se prueban las tres a ver cuál conviene.

**¿Por qué hay tope de dos movimientos?** Porque partiendo el dataset lo suficiente siempre hay un subgrupo donde el efecto aparece, y no significa nada. Si después del segundo movimiento sigue amarillo, la conclusión es inconclusa y se escribe así.

**¿Qué columnas le dan al modelo de la Unidad 3?** Entran `overall`, `potential` y los atributos de rol como `defending_marking_awareness`, `defending_standing_tackle` y `skill_moves`. `height_cm` entra sabiendo qué frontera resuelve y cuál no. `value_eur` entra sólo en logaritmo. Salen `club_name`, las columnas de identidad y las redundantes con `league_id`.

**¿Usaron alguna librería rara?** No. Pandas, numpy y matplotlib. Spearman lo calculamos como Pearson sobre los rangos, que es su definición, así que el notebook no necesita scipy y corre en cualquier máquina.
