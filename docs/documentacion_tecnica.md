# Documentación Técnica | detector-fraude-bancario
## Detector de Fraude en Transacciones Bancarias mediante Análisis de Ciclos en Grafos

## 1. Descripción general

El programa recibe una lista de transacciones bancarias (cuenta origen, cuenta
destino, monto) y construye un grafo dirigido donde cada cuenta es un nodo y
cada transacción es una arista con peso (el monto). Luego busca **ciclos**
dentro de ese grafo: secuencias de cuentas donde el dinero, tras pasar por
varias transacciones, regresa a la cuenta de origen. Estos ciclos se reportan
ordenados por el monto total involucrado, de mayor a menor, para priorizar la
revisión de los casos más significativos.

Este enfoque está inspirado directamente en el trabajo de Bodaghi y Teimourpour (2019), quienes 
en su investigación titulada "The detection of professional fraud in automobile insurance using social 
network analysis" demuestran que buscar ciclos en un grafo es más eficiente y preciso que buscar 
comunidades densas para detectar fraude organizado, ya que las comunidades incluyen muchas 
entidades no fraudulentas mientras que los ciclos son estructuras mucho más específicas.

## 2. Estructuras de datos (TAD)

### 2.1 Grafo (lista de adyacencia)

```c
typedef struct NodoAdy {
    int destino;
    float monto;
    struct NodoAdy* siguiente;
} NodoAdy;

typedef struct {
    NodoAdy* cabeza[MAX_CUENTAS];
    int numCuentas;
} Grafo;
```

**Justificación:** se eligió lista de adyacencia en vez de matriz de
adyacencia porque en una red de transacciones reales cada cuenta se relaciona
con relativamente pocas otras cuentas (el grafo es disperso). La matriz
ocuparía O(V²) memoria sin importar cuántas transacciones existan realmente;
la lista de adyacencia ocupa O(V + E), proporcional a las transacciones reales.

### 2.2 Ciclo y ListaCiclos

```c
typedef struct {
    int nodos[MAX_LONGITUD_CICLO];
    int longitud;
    float montoTotal;
} Ciclo;
```

Estructura simple (arreglo estático) para almacenar la secuencia de cuentas de
cada ciclo detectado y su monto acumulado, suficiente dado que el número de
ciclos y su longitud están acotados en este dominio de aplicación.

## 3. Algoritmos y complejidad

### 3.1 Detección de ciclos (DFS)

Se recorre el grafo con búsqueda en profundidad (DFS), coloreando cada nodo:

- **Blanco**: no visitado.
- **Gris**: en la pila de recursión actual (forma parte del camino activo).
- **Negro**: ya se procesó por completo.

Cuando durante el recorrido se encuentra una arista hacia un nodo **gris**,
significa que ese nodo ya está en el camino actual, es decir, se cerró un
ciclo. Se reconstruye el ciclo tomando el segmento de la pila desde ese nodo
hasta el nodo actual, y se suman los montos de las aristas involucradas.

**Complejidad:**
- Temporal: **O(V + E)**, donde V es el número de cuentas y E el número de
  transacciones, ya que cada nodo y cada arista se visitan a lo sumo una vez
  en el recorrido DFS estándar.
- Espacial: **O(V)** para los arreglos de color y de pila (además del propio
  grafo, que ya ocupa O(V+E)).

### 3.2 Ordenamiento de ciclos (Quick Sort)

Los ciclos detectados se ordenan de mayor a menor monto total mediante Quick
Sort (partición tipo Lomuto), para que los casos con mayor monto involucrado
—los potencialmente más costosos si son fraude real— se revisen primero.

**Complejidad:**
- Temporal: **O(n log n)** en el caso promedio; **O(n²)** en el peor caso
  (poco probable aquí porque el número de ciclos detectados suele ser
  pequeño en comparación con el número de cuentas).
- Espacial: **O(log n)** por la pila de recursión.

## 4. Manejo de memoria dinámica

Cada transacción agregada al grafo reserva memoria dinámica (`malloc`) para un
nodo de la lista de adyacencia. Toda esa memoria se libera explícitamente al
finalizar el programa mediante `grafo_liberar()`, que recorre cada lista de
adyacencia y libera cada nodo con `free()`. No se usa memoria dinámica en la
detección de ciclos ni en el ordenamiento (se usan arreglos estáticos
acotados), lo que simplifica el manejo de memoria y evita fugas.

## 5. Limitaciones conocidas

- El número máximo de cuentas y de ciclos detectables está acotado por
  constantes (`MAX_CUENTAS`, `MAX_CICLOS`) para mantener el código simple;
  esto es suficiente para el propósito académico del proyecto pero no
  escalaría a redes bancarias reales sin usar estructuras dinámicas (p. ej.
  tablas hash para mapear cuentas, arreglos dinámicos para los ciclos).
- El algoritmo puede reportar el mismo ciclo lógico más de una vez si es
  alcanzable desde distintos puntos de entrada del grafo; para el alcance de
  este proyecto no se implementó deduplicación exhaustiva de ciclos
  isomorfos, siguiendo la simplificación de enfocarnos en la lógica central
  de detección.

## 6. Cómo compilar y ejecutar

```bash
make          # compila el proyecto
make run      # compila (si hace falta) y ejecuta con data/transacciones.txt
make clean    # elimina el binario compilado
```

Para usar un archivo de transacciones distinto:

```bash
./detector_fraude ruta/a/otro_archivo.txt
```

### Formato del archivo de entrada

```
<numero_de_cuentas>
<cuenta_origen> <cuenta_destino> <monto>
<cuenta_origen> <cuenta_destino> <monto>
...
```

## 7. Evidencia de pruebas realizadas

Inicialmente se probó el programa con un archivo pequeño (10 cuentas, 10
transacciones, 3 ciclos) para validar la lógica básica. Sin embargo, un
dataset donde todas las transacciones forman ciclos no demuestra que el
algoritmo distinga fraude de actividad normal — solo demuestra que encuentra
ciclos cuando efectivamente no hay otra cosa.

Por eso se generó un dataset de prueba más representativo, con una mezcla de
transacciones normales y transacciones fraudulentas:

MétricaValorCuentas totales150Transacciones totales260Ciclos insertados deliberadamente13 (tamaños entre 3 y 6 cuentas)Transacciones "normales" (sin ciclo)190

Las 190 transacciones normales se generaron de forma que nunca pueden
cerrar un ciclo entre sí (solo se permiten aristas de una cuenta de índice
menor hacia una de índice mayor dentro de ese bloque), garantizando que
cualquier ciclo detectado por el programa sea necesariamente uno de los 13
insertados intencionalmente, y no un falso positivo generado por azar.

Resultado obtenido en consola:

--- Grafo de transacciones (150 cuentas) ---
Cuenta 0 -> [1, $2981.75]
Cuenta 1 -> [2, $3599.93]
Cuenta 2 -> [0, $4550.20]
Cuenta 3 -> [4, $4528.48]
Cuenta 4 -> [5, $842.09]
Cuenta 5 -> [6, $4443.80]
Cuenta 6 -> [7, $4038.73]
Cuenta 7 -> [3, $2906.63]
Cuenta 8 -> [9, $4745.86]
Cuenta 9 -> [10, $2552.83]
Cuenta 10 -> [11, $3114.14]
Cuenta 11 -> [8, $2268.73]
Cuenta 12 -> [13, $991.23]
Cuenta 13 -> [14, $2697.63]
Cuenta 14 -> [15, $3277.18]
Cuenta 15 -> [16, $1886.51]
Cuenta 16 -> [12, $2219.74]
Cuenta 17 -> [18, $3085.77]
Cuenta 18 -> [19, $1777.83]
Cuenta 19 -> [17, $4131.33]
Cuenta 20 -> [21, $4408.31]
Cuenta 21 -> [22, $3081.96]
Cuenta 22 -> [23, $2093.76]
Cuenta 23 -> [24, $2403.60]
Cuenta 24 -> [25, $3125.76]
Cuenta 25 -> [20, $157.45]
Cuenta 26 -> [27, $4300.01]
Cuenta 27 -> [28, $4415.61]
Cuenta 28 -> [26, $2104.95]
Cuenta 29 -> [30, $529.66]
Cuenta 30 -> [31, $3404.27]
Cuenta 31 -> [32, $440.36]
Cuenta 32 -> [29, $1984.91]
Cuenta 33 -> [34, $4566.15]
Cuenta 34 -> [35, $1928.94]
Cuenta 35 -> [36, $1511.81]
Cuenta 36 -> [33, $4907.85]
Cuenta 37 -> [38, $4589.68]
Cuenta 38 -> [39, $2412.04]
Cuenta 39 -> [37, $2868.52]
Cuenta 40 -> [41, $4333.44]
Cuenta 41 -> [42, $3182.35]
Cuenta 42 -> [43, $4190.32]
Cuenta 43 -> [40, $107.64]
Cuenta 44 -> [45, $3781.56]
Cuenta 45 -> [46, $875.43]
Cuenta 46 -> [47, $567.44]
Cuenta 47 -> [48, $820.55]
Cuenta 48 -> [49, $3857.44]
Cuenta 49 -> [44, $2366.82]
Cuenta 50 -> [51, $3857.11]
Cuenta 51 -> [52, $3564.25]
Cuenta 52 -> [53, $199.61]
Cuenta 53 -> [54, $2208.30]
Cuenta 54 -> [50, $2018.73]
Cuenta 60 -> [137, $1491.58] [67, $273.89] [92, $471.06] [136, $4891.69]
Cuenta 62 -> [106, $2157.72] [99, $3284.70] [141, $4442.11] [129, $1594.87]
Cuenta 63 -> [140, $1120.47]
Cuenta 64 -> [101, $1128.53] [103, $713.52] [80, $4612.77] [114, $3451.98]
Cuenta 65 -> [110, $1215.38] [149, $4256.64] [148, $4390.79]
Cuenta 66 -> [75, $1772.40]
Cuenta 67 -> [125, $3709.84] [86, $3304.26] [130, $3096.87]
Cuenta 69 -> [87, $2308.67] [108, $1206.47] [125, $3421.01]
Cuenta 70 -> [138, $3781.63]
Cuenta 71 -> [142, $3442.49] [118, $738.03] [104, $4753.17] [102, $2349.60]
Cuenta 72 -> [80, $4873.38] [103, $4715.47]
Cuenta 73 -> [121, $1757.00] [102, $536.41] [79, $1446.82] [145, $1190.16] [75, $3563.76] [122, $2217.90] [91, $1558.96]
Cuenta 74 -> [144, $3615.30] [93, $1319.83]
Cuenta 75 -> [109, $3223.11] [106, $1927.53] [82, $1209.56] [116, $1234.97]
Cuenta 76 -> [129, $2544.10] [92, $1358.00] [146, $1445.88] [115, $679.68]
Cuenta 77 -> [148, $765.75] [79, $2833.73] [115, $2746.02]
Cuenta 78 -> [143, $4573.84] [79, $1097.36]
Cuenta 79 -> [144, $1068.34] [92, $828.69] [80, $2829.73]
Cuenta 80 -> [110, $1779.71]
Cuenta 81 -> [126, $4817.91] [118, $915.52] [97, $1916.05] [106, $1324.67]
Cuenta 82 -> [111, $657.20] [133, $1523.44]
Cuenta 83 -> [140, $4825.84] [101, $4280.99]
Cuenta 84 -> [103, $618.34] [148, $1966.71] [85, $1925.47] [146, $2270.69]
Cuenta 85 -> [140, $3523.69] [94, $3140.46]
Cuenta 86 -> [99, $4774.44]
Cuenta 87 -> [96, $4531.11] [108, $288.79] [113, $3649.76] [149, $960.08]
Cuenta 88 -> [115, $1331.15] [144, $379.74] [91, $4557.87]
Cuenta 89 -> [92, $390.71] [90, $4524.80]
Cuenta 90 -> [97, $4629.46] [145, $2445.59] [102, $1217.16] [139, $2866.94]
Cuenta 91 -> [143, $1697.80] [119, $697.16] [120, $3706.38]
Cuenta 93 -> [127, $3421.80]
Cuenta 94 -> [108, $2783.44] [107, $2579.92]
Cuenta 95 -> [108, $4297.30] [136, $2339.04] [119, $1791.79]
Cuenta 96 -> [105, $895.56]
Cuenta 98 -> [102, $3230.06] [132, $4504.68] [109, $468.35] [148, $2801.27]
Cuenta 99 -> [106, $3412.95] [131, $3613.80] [100, $4977.19]
Cuenta 100 -> [136, $3954.98] [143, $2814.18]
Cuenta 101 -> [125, $4401.18] [114, $751.00] [116, $1646.80]
Cuenta 102 -> [139, $2410.87] [116, $90.31]
Cuenta 103 -> [145, $1956.49] [120, $2498.56]
Cuenta 104 -> [134, $282.74]
Cuenta 106 -> [137, $3535.48] [133, $2904.11] [146, $3853.69] [118, $1931.79]
Cuenta 107 -> [138, $738.20] [133, $765.41]
Cuenta 108 -> [121, $1032.44] [119, $1733.67]
Cuenta 109 -> [136, $2226.56] [129, $340.33] [120, $623.53]
Cuenta 110 -> [138, $1037.25] [114, $2639.62] [113, $146.99] [142, $895.77]
Cuenta 111 -> [120, $156.30] [126, $902.43]
Cuenta 112 -> [134, $2180.63] [122, $2146.30] [120, $1295.93] [129, $938.93]
Cuenta 113 -> [140, $2232.59] [137, $180.56]
Cuenta 115 -> [133, $3176.90] [116, $1691.49] [142, $4884.41]
Cuenta 116 -> [121, $2657.06]
Cuenta 117 -> [124, $4413.08] [125, $1320.71]
Cuenta 118 -> [120, $4647.76]
Cuenta 119 -> [144, $4759.53]
Cuenta 120 -> [146, $154.69]
Cuenta 122 -> [144, $1200.61] [128, $3402.92]
Cuenta 123 -> [143, $1490.55] [129, $642.17]
Cuenta 124 -> [141, $709.03] [149, $946.68] [139, $691.50] [132, $3979.01] [133, $4116.26] [136, $4644.52]
Cuenta 125 -> [126, $4382.85] [129, $2668.23] [139, $58.47]
Cuenta 126 -> [135, $4296.79] [141, $1695.24]
Cuenta 127 -> [141, $195.25] [138, $1261.30] [143, $3904.18] [146, $805.93]
Cuenta 128 -> [129, $4674.14] [147, $2827.29]
Cuenta 129 -> [147, $4884.49] [138, $2419.91]
Cuenta 130 -> [145, $1625.63] [138, $3030.90] [136, $1478.23]
Cuenta 131 -> [137, $2261.11] [147, $451.79] [132, $4695.90] [133, $686.43] [141, $2847.03]
Cuenta 133 -> [144, $482.26] [139, $4632.61]
Cuenta 135 -> [140, $2036.96]
Cuenta 136 -> [138, $2385.71]
Cuenta 137 -> [138, $3735.41]
Cuenta 138 -> [147, $4074.11] [143, $4192.93] [146, $1863.79]
Cuenta 139 -> [148, $4862.19] [149, $3593.48] [142, $452.12] [147, $4205.02]
Cuenta 140 -> [142, $2520.78] [146, $3998.24] [143, $1111.82] [144, $2902.25] [148, $537.67]
Cuenta 141 -> [148, $2898.85] [149, $2605.43] [146, $2484.64] [142, $3474.70]
Cuenta 142 -> [146, $2686.74] [144, $3888.01]
Cuenta 143 -> [147, $292.04]
Cuenta 144 -> [145, $3569.05]
Cuenta 145 -> [149, $1481.72] [146, $2217.72]
Cuenta 146 -> [149, $4661.06] [148, $452.72]
Cuenta 147 -> [148, $555.50]
Cuenta 148 -> [149, $4799.68]

=== 13 ciclo(s) sospechoso(s) detectado(s), ordenados por monto ===
Ciclo #1 | 5 cuentas | monto total: $16759.73
  Ruta: 3 -> 4 -> 5 -> 6 -> 7 -> 3 (vuelve al origen)
Ciclo #2 | 6 cuentas | monto total: $15270.84
  Ruta: 20 -> 21 -> 22 -> 23 -> 24 -> 25 -> 20 (vuelve al origen)
Ciclo #3 | 4 cuentas | monto total: $12914.75
  Ruta: 33 -> 34 -> 35 -> 36 -> 33 (vuelve al origen)
Ciclo #4 | 4 cuentas | monto total: $12681.56
  Ruta: 8 -> 9 -> 10 -> 11 -> 8 (vuelve al origen)
Ciclo #5 | 6 cuentas | monto total: $12269.24
  Ruta: 44 -> 45 -> 46 -> 47 -> 48 -> 49 -> 44 (vuelve al origen)
Ciclo #6 | 5 cuentas | monto total: $11848.00
  Ruta: 50 -> 51 -> 52 -> 53 -> 54 -> 50 (vuelve al origen)
Ciclo #7 | 4 cuentas | monto total: $11813.75
  Ruta: 40 -> 41 -> 42 -> 43 -> 40 (vuelve al origen)
Ciclo #8 | 3 cuentas | monto total: $11131.88
  Ruta: 0 -> 1 -> 2 -> 0 (vuelve al origen)
Ciclo #9 | 5 cuentas | monto total: $11072.29
  Ruta: 12 -> 13 -> 14 -> 15 -> 16 -> 12 (vuelve al origen)
Ciclo #10 | 3 cuentas | monto total: $10820.57
  Ruta: 26 -> 27 -> 28 -> 26 (vuelve al origen)
Ciclo #11 | 3 cuentas | monto total: $9870.24
  Ruta: 37 -> 38 -> 39 -> 37 (vuelve al origen)
Ciclo #12 | 3 cuentas | monto total: $8994.93
  Ruta: 17 -> 18 -> 19 -> 17 (vuelve al origen)
Ciclo #13 | 4 cuentas | monto total: $6359.20
  Ruta: 29 -> 30 -> 31 -> 32 -> 29 (vuelve al origen)

El programa detectó exactamente los 13 ciclos insertados, sin falsos
positivos ni ciclos faltantes, correctamente ordenados de mayor a menor
monto total mediante Quick Sort. Esto valida tanto la correctitud del DFS de
detección de ciclos como la ausencia de falsos positivos frente a una carga
de transacciones normales significativamente mayor que la de fraude (190
transacciones normales contra 13 grupos fraudulentos). El programa compiló
sin errores ni warnings con gcc -Wall -Wextra.

## 8. Referencias
Bodaghi, A., & Teimourpour, B. (2019). The detection of professional fraud in automobile insurance using social network analysis. (Manuscrito no publicado / Trabajo de investigación). School of Industrial and Systems Engineering, Tarbiat Modares University, Irán.