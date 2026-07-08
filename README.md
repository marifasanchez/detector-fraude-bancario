# Detector de Fraude en Transacciones Bancarias

Proyecto final del curso de Algoritmos y Estructuras de Datos. Detecta
patrones sospechosos de fraude/lavado de dinero en transacciones bancarias,
modelando cuentas y transacciones como un grafo dirigido y buscando **ciclos**
(dinero que vuelve a su cuenta de origen tras varias transacciones).

Inspirado en Bodaghi, A. & Teimourpour, B. (2019). *The detection of
professional fraud in automobile insurance using social network analysis.*

## ¿Qué hace?

1. Lee un archivo con cuentas y transacciones.
2. Construye un grafo dirigido y ponderado (TAD: lista de adyacencia).
3. Detecta ciclos con DFS (posibles anillos de lavado de dinero).
4. Ordena los ciclos detectados por monto total con Quick Sort.
5. Muestra en consola los resultados priorizados.

## Cómo compilar y ejecutar

```bash
make
make run
```

o directamente:

```bash
gcc -Wall -Wextra -Iinclude src/*.c -o detector_fraude
.\detector_fraude.exe data\transacciones.txt 
```

## Estructura del repositorio

```
.
├── src/                   # Código fuente (.c)
│   ├── main.c
│   ├── grafo.c
│   ├── ciclos.c
│   └── ordenamiento.c
├── include/                # Headers (.h)
│   ├── grafo.h
│   ├── ciclos.h
│   └── ordenamiento.h
├── data/
│   └── transacciones.txt   # Datos de ejemplo
├── docs/
│   ├── propuesta.md               # Entregable Fase 1
│   └── documentacion_tecnica.md   # Justificación de algoritmos, TAD y Big-O
├── Makefile
└── README.md
```

## Equipo

Ma Fabiana Sánchez, Corina Vezga y Victoria Pereira