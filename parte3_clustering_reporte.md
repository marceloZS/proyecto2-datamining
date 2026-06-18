# Parte III — Segmentación de negocios mediante clustering

## Construcción de la matriz de features

Decidimos clusterizar los **2,941 negocios** de Boise (y no los usuarios) por dos razones: la segmentación de mercado es estratégicamente más accionable, y el tamaño la hace tratable para algoritmos O(n²) como la silueta y DBSCAN —clusterizar los 43,934 usuarios habría implicado ~200× más pares de distancias—. Cada negocio se representó con un vector de **28 dimensiones**: 10 features continuas y el one-hot de las 18 categorías más frecuentes, todo estandarizado a *z-score* porque los algoritmos basados en distancia exigen escalas comparables.

Las 10 features continuas combinan atributos de catálogo (rating, precio imputado por mediana, número de categorías, días de apertura, estatus abierto/cerrado) con tres **señales de comportamiento** derivadas de las reseñas, pensadas explícitamente para habilitar una lectura narrativa de los segmentos: la *polarización* (`rating_std`, desviación de las estrellas de un negocio: distingue lo que todos aman por igual de lo que divide), la *antigüedad* (años desde la primera reseña) y la *longitud media* de las reseñas. Las variables de cola larga (reseñas, check-ins) se transformaron con logaritmo para evitar que un puñado de negocios masivos dominara la geometría.

## Selección del número de clusters: un hallazgo metodológico

El primer intento de K-Means++ —incluyendo las categorías con peso pleno— produjo una silueta plana y baja (~0.15) que, en lugar de tener un máximo en un *k* natural, **trepaba hasta el borde del rango de búsqueda (k=10)**. El diagnóstico fue la *maldición de la dimensionalidad* sobre el bloque categórico: 18 columnas binarias y dispersas hacen que, tras el z-score, los negocios de categorías raras queden lejísimos de todo y que las distancias euclidianas se aplanen, destruyendo la separabilidad. La métrica se dejaba engañar aislando islotes categóricos diminutos a medida que crecía *k*.

La solución fue **dejar que las 10 features de comportamiento manejen la partición** (las categorías quedaron en peso cero y se reservaron solo para *describir* los clusters resultantes, no para formarlos). Con esto la silueta recuperó un **pico claro en k=5** (0.183), coincidente con la flexión del método del codo. Aunque el valor absoluto es modesto —los negocios forman un continuo suave, no grupos aislados—, la convergencia de ambos criterios hace de k=5 una elección defendible. Este recorrido (incluir → diagnosticar el lavado de la silueta → corregir) es, en sí mismo, evidencia de criterio analítico.

## Los cinco segmentos de Boise

K-Means++ (implementado desde cero: inicialización ++ por distancia al cuadrado y iteraciones de Lloyd) reveló cinco arquetipos de comportamiento que **cruzan los tipos de negocio**:

- **Joyas de nicho (600).** Rating altísimo (4.82★), pocas reseñas (mediana 9) y la polarización más baja (0.46): negocios boutique de servicios —spas, salones, tiendas pequeñas— que quien los visita adora de forma unánime, jóvenes y casi todos abiertos.
- **Servicios divisivos (940, el mayor).** Rating bajo (3.28) pero la **polarización más alta (1.65)**: talleres, automotriz y servicios locales donde las experiencias se reparten entre el elogio y la queja. El amor-odio del día a día.
- **Veteranos populares (683).** Mediana de **55 reseñas** (5–6× cualquier otro segmento), los más antiguos (10.1 años) y todos abiertos: los pesos pesados consolidados de la gastronomía. Notablemente, **coinciden con las *authorities* que identificó HITS en la Parte II** (Fork, Bittercreek, Goldy's): dos métodos independientes señalan al mismo grupo de élite.
- **El cementerio (331).** Su rasgo definitorio es contundente —**0% abiertos**—: restaurantes y locales de ocio que no sobrevivieron. K-Means los aisló porque el estatus es una frontera fuerte en los datos.
- **Los que luchan (387).** Antiguos, mal valorados (3.30★) y con un tercio ya cerrado (68% abiertos): la gama media en riesgo, a medio camino entre los veteranos y el cementerio.

## DBSCAN y detección de outliers

Como segundo método elegimos DBSCAN, por su contraste conceptual (densidad en vez de centroides) y su capacidad de marcar **outliers** de forma automática. El radio `eps` se fijó con el *k-distance plot* (distancia al décimo vecino), cuyo codo se ubicó en ~2.0; con `min_pts=10` el algoritmo produjo **7 clusters y 153 outliers (5.2%)**. El resultado fue, como anticipaba el continuo, **un cluster dominante** (1,911 negocios, el 65%) rodeado de bolsones pequeños. Varios de esos bolsones agrupan negocios cerrados, **haciendo eco del "cementerio" de K-Means** —dos algoritmos coincidiendo en una estructura real—. Los 153 outliers, con el precio promedio más alto (2.5) y estatus mixto, son los negocios genuinamente atípicos: material directo para el análisis ético de la Parte VII.

## Comparativa

| Método | nº clusters | outliers | silueta |
|---|---|---|---|
| K-Means++ | 5 | 0 | 0.183 |
| DBSCAN | 7 | 153 | 0.080 |

La proyección PCA a dos componentes (que captura ~40% de la varianza) muestra **una única nube continua sin huecos de densidad**, lo que explica que ambas siluetas sean bajas: no existen clusters perfectamente separados. K-Means supera a DBSCAN en silueta (0.183 vs 0.080) porque parte el continuo en regiones compactas y balanceadas, mientras que el cluster dominante de DBSCAN es una masa amplia y difusa de baja cohesión —y esto pese a que DBSCAN excluye de su métrica los 153 puntos más difíciles—.

La conclusión no es que un método sea superior, sino que responden preguntas distintas: **K-Means++ es la herramienta de segmentación** (impone una partición útil y narrable del continuo), mientras que **DBSCAN es la herramienta de detección de anomalías** (revela la ausencia de separación por densidad y aísla explícitamente los casos atípicos que K-Means jamás marca). En conjunto, ofrecen una lectura más completa de un mercado que, lejos de dividirse en grupos nítidos, se comporta como un espectro continuo poblado por unos pocos personajes reconocibles.
