# Quadtree — Indexacion Espacial para Consultas Geoespaciales

Implementacion en Python de un Quadtree generalizado (extensible a cualquier numero de dimensiones) diseñado para operaciones geoespaciales eficientes como busqueda por radio y busqueda de K-Vecinos mas Cercanos (KNN). Construido y evaluado como parte de un analisis comparativo frente a la busqueda por fuerza bruta.

---

## Que es un Quadtree?

Un Quadtree es una estructura de datos jerarquica en la que cada nodo interno tiene exactamente cuatro hijos. Particiona un espacio bidimensional subdividiendolo recursivamente en cuatro cuadrantes hasta que cada region contenga como maximo una cantidad de 1 puntos que son las hojas.

La intuicion clave es que el espacio en si se convierte en el indice. En lugar de recorrer cada punto almacenado, el arbol permite descartar regiones enteras del espacio en un solo paso, que es precisamente lo que lo hace poderoso para cargas de trabajo geoespaciales.

---

## Como funciona la insercion?

Al insertar un punto, el arbol desciende desde la raiz comparando coordenadas en cada nivel para determinar a cual de los cuatro cuadrantes pertenece el punto. Si el cuadrante destino ya esta lleno, se subdivide y redistribuye sus puntos existentes entre cuatro nuevos hijos. Este proceso se repite recursivamente hasta que el punto aterriza en un nodo hoja con espacio disponible.

La consecuencia practica es elegante: las zonas densas (muchas entregas agrupadas) producen subarboles mas profundos, mientras que las zonas dispersas permanecen como hojas sin dividir. La memoria y el computo solo se gastan donde realmente existe informacion.

---

## Como funciona la busqueda?

La busqueda por radio se apoya en una prueba geometrica en cada nodo: la region rectangular de este nodo, intersecta el circulo de busqueda?

Si no hay interseccion, la rama completa se descarta y ninguno de sus hijos es visitado. Si hay interseccion, el arbol desciende a los hijos y repite la prueba. Cuando se llega a un nodo hoja, se calcula la distancia euclidiana exacta de cada punto almacenado y se retornan unicamente los que esten dentro del radio.

Esto significa que en lugar de evaluar los 10.000 puntos, solo se exploran las ramas que geograficamente se solapan con la zona de interes, haciendo la busqueda significativamente mas rapida conforme crece el conjunto de datos.

---

## Por que es mejor para este problema?

| Operacion | Fuerza Bruta | Quadtree |
|---|---|---|
| Busqueda por radio | O(n) | O(log n) promedio |
| Vecino mas cercano (KNN) | O(n) | O(log n) promedio |

Con 10.000 puntos dispersos en una ciudad, la mayoria de las consultas tocan solo una fraccion pequena del arbol. La diferencia de rendimiento esta respaldada tanto teoricamente como de forma empirica y medible, y se amplifica de forma sostenida a medida que el conjunto de datos crece.

---

## Detalles de Implementacion

### Construccion del arbol

El arbol esta implementado en Python con una generalizacion que soporta cualquier numero de dimensiones, aunque este proyecto utiliza dos. El numero de cuadrantes es siempre `2^d`, donde `d` es el numero de dimensiones: un arbol 2D tiene 4 cuadrantes (Quadtree) y uno 3D tendria 8 (Octree).

Cada nodo almacena el punto medio de su region y una lista de hijos. Si un nodo es hoja, su lista de hijos esta vacia y almacena directamente el punto. La subdivision se detiene cuando un nodo contiene un solo punto o todos los puntos son identicos, lo que previene la recursion infinita en casos degenerados.

### Busqueda por radio

La busqueda por radio aplica la prueba de interseccion descrita anteriormente para podar cuadrantes irrelevantes. Esta implementada como un **generador** de Python usando `yield`, lo que significa que no construye la lista completa de resultados en memoria, sino que entrega los puntos al llamador a medida que los encuentra. Esto mantiene el uso de memoria bajo incluso cuando el conjunto de resultados es grande.

### Busqueda KNN

La busqueda KNN mantiene un **max-heap de tamanio k** con distancias negadas, lo que permite rastrear eficientemente los k mejores candidatos encontrados hasta el momento. El heap se actualiza a medida que se explora el arbol. Los hijos se ordenan por distancia al punto de consulta antes de ser visitados, de modo que las ramas mas prometedoras se exploran primero. Esta ordenacion agresiva reduce dramaticamente el numero de ramas que deben visitarse, manteniendo los tiempos de KNN notablemente estables incluso a gran escala.

---

## Pruebas y Visualizaciones

Para verificar la correctitud, se generaron 10.000 puntos aleatorios con coordenadas en el rango `[0, 10000]`, simulando puntos de entrega distribuidos en una ciudad.

Se definio un punto objetivo en el centro del plano `(5000, 5000)` con un radio de busqueda de 500 metros. La busqueda por radio retorno todos los puntos dentro de esa circunferencia, resaltados en verde en la visualizacion. Los 5 vecinos mas cercanos se marcan en azul con sus distancias anotadas, conectados al centro por lineas que permiten verificar visualmente que son efectivamente los mas proximos.

La grafica confirma que el arbol identifica correctamente todos los puntos dentro del radio y que los vecinos mas cercanos son geometricamente consistentes con su posicion respecto al centro, con distancias que van desde aproximadamente 31 m hasta 131 m.

Esto valida que el Quadtree funciona correctamente tanto para la busqueda por radio como para KNN, con resultados visuales claros y consistentes.

---

## Analisis Comparativo: Quadtree vs Fuerza Bruta

Se ejecutaron pruebas de rendimiento sobre conjuntos de datos que van desde 1.000 hasta 200.000 puntos, midiendo el tiempo promedio de consulta en milisegundos para busqueda por radio y KNN con `k=5`.

### Comportamiento general

Las graficas de referencia usan escala logaritmica, lo que permite apreciar claramente que la fuerza bruta crece de forma sostenida con cada aumento de `n`, mientras que el Quadtree se estabiliza rapidamente. A 200.000 puntos, la fuerza bruta tarda aproximadamente **10 veces mas** que el Quadtree tanto en radio como en KNN, confirmando empiricamente la diferencia teorica entre `O(n)` y `O(log n)`.

### Donde gana la fuerza bruta

Una grafica de divergencia con zoom permite identificar con precision el punto de cruce experimental, detectado en aproximadamente **150 puntos**. Por debajo de ese umbral, el Quadtree es mas lento que la fuerza bruta debido al costo fijo de construir el arbol y recorrer su estructura. A partir de ese punto, la poda de cuadrantes empieza a rendir frutos y el Quadtree toma ventaja de forma sostenida.

La explicacion es directa: con pocos puntos, iterar una lista plana es tan rapido que la sobrecarga de la recursion del arbol no se justifica. Con muchos puntos, descartar ramas enteras ahorra un numero creciente de comparaciones, y esa ventaja se acumula con cada consulta.

### Estabilidad del Quadtree

Un resultado particularmente notable es que el tiempo de consulta KNN del Quadtree permanece casi plano conforme crece `n`, manteniendose por debajo de **0.1 ms** incluso a 200.000 puntos. Esto se debe a que KNN solo visita los nodos que podrian mejorar el mejor candidato actual en el heap, podando agresivamente el arbol a medida que el heap se llena con buenos vecinos.

A diferencia de la busqueda por radio, que puede necesitar visitar mas nodos cuando el radio es grande o los puntos estan densamente agrupados, la busqueda KNN se beneficia enormemente de la estructura del Quadtree, manteniendo tiempos de respuesta extremadamente bajos independientemente del tamaño del conjunto de datos.

La fuerza bruta muestra un crecimiento lineal claro a lo largo de toda la escala, mientras que el Quadtree demuestra su eficiencia y escalabilidad, validando su uso en aplicaciones con grandes conjuntos de datos geoespaciales donde la velocidad de consulta es critica.

---

## Dependencias y Herramientas

El proyecto utiliza las siguientes bibliotecas y herramientas:

- **matplotlib** para generar todas las visualizaciones y graficas de rendimiento.
- **heapq** (biblioteca estandar de Python) usado internamente por la busqueda KNN para mantener el max-heap de candidatos mas cercanos.
- **numpy** generación de puntos aleatorios y operaciones vectoriales eficientes sobre coordenadas.
- **Herramientas de IA** utilizadas como apoyo durante el desarrollo de las visualizaciones y refinamientos algoritmicos.
- **time**  medición de tiempos de ejecución para el análisis comparativo entre Quadtree y fuerza bruta.
- **random**  generación de datos aleatorios auxiliares, posiblemente para casos de prueba específicos.

---

*Proyecto de Laboratorio — Estructuras de Datos Espaciales*
