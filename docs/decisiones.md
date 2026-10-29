# Decisiones de diseño y Bitácora del Proyecto

Plataforma de Monitoreo Ambiental Urbano — Control Center IoT  
**Autor:** Juan Pablo Lozada López  
**Curso:** Estructuras de Datos · IDIA  

---

## 1. Decisiones de diseño — Semanas 1 a 4 (Consolidado)

### 1.1 Punto de entrada unificado
El proyecto mantiene un único punto de entrada para la aplicación de producción:
```text
ingest.IngestaSensores.main()
```
Las semanas no crean aplicaciones fragmentadas e inconexas, sino que extienden incrementalmente la arquitectura de la plataforma. Los bancos de pruebas y experimentos (`BancoDePruebas`, `BancoDeOrdenamiento`, `BancoDeListas`) son componentes de evaluación empírica y análisis algorítmico.

### 1.2 Búsqueda y Ordenamiento
- **Búsqueda por Timestamp:** Se implementaron búsqueda lineal $O(n)$ y búsqueda binaria $O(\log n)$. La búsqueda binaria requiere que la estructura garantice orden cronológico y acceso eficiente a la mediana.
- **Búsqueda por PM2.5:** Se demostró el efecto de violar las precondiciones de la búsqueda binaria cuando los datos no están ordenados por dicha variable física.
- **Algoritmos de Ordenamiento:** Se incorporaron algoritmos elementales ($O(n^2)$ como Bubble Sort con corte temprano, Selection Sort, Insertion Sort) y avanzados ($O(n \log n)$ como Merge Sort, Quick Sort con pivote aleatorio y Heap Sort). El pivote aleatorio en Quick Sort previene la degradación patológica a $O(n^2)$ y desbordamientos de pila (`StackOverflowError`) ante datos casi ordenados.

---

## 2. Decisiones de diseño — Semana 5: La Cadena (Estructuras Dinámicas)

### 2.1 Contrato e Interfaz `Repositorio` y Polimorfismo
La plataforma desacopla la lógica de negocio (ingesta, análisis de dispersión ambiental, búsqueda y ordenamiento) del almacenamiento físico mediante la interfaz `repository.Repositorio`.
Tanto `RepositorioArreglo` (estructura secuencial contigua) como `RepositorioEnlazado` (estructura dinámica enlazada) cumplen el mismo contrato:
- `void agregarAlFinal(LecturaSensor lectura)`
- `void agregarAlInicio(LecturaSensor lectura)`
- `LecturaSensor obtener(int posicion)`
- `boolean eliminar(int posicion)`
- `int tamano()`
- `String nombre()`

Esto permite aplicar polimorfismo puro: cualquier algoritmo o servicio puede operar sobre `Repositorio` sin conocer si por debajo hay casilleros indexados contiguos o una cadena de nodos dispersos en memoria.

### 2.2 Lista Enlazada Simple (`RepositorioEnlazado`)
- **Estructura del Nodo:** Clase `model.Nodo` con encapsulation de `LecturaSensor dato` y puntero `Nodo siguiente`.
- **Inserción al inicio $O(1)$:** El nuevo nodo se enlaza al antiguo `primero` y la referencia `primero` se actualiza inmediatamente. Si la lista estaba vacía, `ultimo` también se actualiza al nuevo nodo. Costo constante sin desplazamientos.
- **Optimización de inserción al final con puntero `ultimo` $O(1)$:** En lugar de recorrer toda la lista en $O(n)$ para encontrar el extremo, se mantiene la referencia privada `ultimo`. Si la lista no está vacía, `ultimo.setSiguiente(nuevo)` y `ultimo = nuevo`, garantizando inserción en tiempo constante $O(1)$.
- **Eliminación segura sin romper la cadena:** Para eliminar en la posición $i$, se navega hasta el nodo predecesor $i-1$ y se puentea el nodo a eliminar:
  ```java
  Nodo aEliminar = anterior.getSiguiente();
  Nodo siguiente = aEliminar.getSiguiente();
  anterior.setSiguiente(siguiente);
  if (siguiente == null) {
      ultimo = anterior; // Si se eliminó el último
  }
  cantidad--;
  ```
  Esto previene la pérdida catastrófica de los nodos posteriores y asegura la sincronización entre `tamano()` y `contarRecorriendo()`.

### 2.3 Historial Navegable con Lista Doblemente Enlazada (`HistorialNavegable`)
Para la estación de monitoreo en tiempo real, el operador requiere navegar el flujo temporal de lecturas en ambas direcciones (hacia el pasado y hacia el futuro).
- Cada `NodoDoble` almacena enlaces simétricos: `siguiente` y `anterior`.
- La inserción al final actualiza bidireccionalmente:
  ```java
  ultimo.setSiguiente(nuevo);
  nuevo.setAnterior(ultimo);
  ultimo = nuevo;
  ```
- El cursor permite desplazamiento hacia adelante (`siguiente()`) y retroceso (`anterior()`) en tiempo $O(1)$ estricto, sin requerir reindexación ni recorridos desde el origen.

---

## 3. Resultados Experimentales Reales (`BancoDeListas`)

Mediciones obtenidas empíricamente en la máquina local ejecutando las 5 fases de `BancoDeListas`:

### Experimento 1: Mismo código, dos mecanismos (Polimorfismo)
Se cargaron 5 lecturas sintéticas iterando indistintamente sobre un arreglo polimórfico `Repositorio[]`:
- `RepositorioArreglo`: tamaño 5, primera lectura verificada.
- `RepositorioEnlazado`: tamaño 5, primera lectura verificada.
*Conclusión:* El contrato se cumple al 100%; el consumidor no depende de la implementación interna.

### Experimento 2: Consistencia estructural tras eliminación
- **Antes de eliminar:** `tamano()` = 10 | `contarRecorriendo()` = 10
- Se ejecutó `eliminar(5)` (nodo intermedio).
- **Después de eliminar:** `tamano()` = 9 | `contarRecorriendo()` = 9
*Conclusión:* El puenteo de nodos fue exitoso; ningún nodo posterior se perdió y el contador lógico refleja exactamente los eslabones físicos alcanzables.

### Experimento 3: Inserción masiva al inicio (`agregarAlInicio`)
Comparación entre desplazamientos del arreglo vs. saltos de la lista:

| Lecturas ($N$) | Arreglo: Movimientos | Arreglo: Tiempo (ms) | Lista: Saltos | Lista: Tiempo (ms) |
|---|---|---|---|---|
| **10.000** | 49.995.000 | 67 ms | 0 | 1 ms |
| **50.000** | 1.249.975.000 | 1.111 ms | 0 | 1 ms |
| **100.000** | 4.999.950.000 | 4.861 ms | 0 | 0 ms |

*Conclusión:* En el arreglo el costo crece cuadráticamente $O(N^2)$ ($\approx \frac{N^2}{2}$ movimientos), requiriendo casi 5.000 millones de movimientos y 5 segundos para 100k lecturas. En contraste, la lista enlazada realiza 0 saltos y tarda menos de 1 milisegundo ($O(1)$ por inserción).

### Experimento 4: Recorrido secuencial por posición (`obtener(i)`)

| Lecturas ($N$) | Arreglo: Accesos directos | Arreglo: Tiempo (ms) | Lista: Saltos de puntero | Lista: Tiempo (ms) |
|---|---|---|---|---|
| **1.000** | 1.000 | 0 ms | 499.500 | 2 ms |
| **10.000** | 10.000 | 0 ms | 49.995.000 | 82 ms |
| **50.000** | 50.000 | 0 ms | 1.249.975.000 | 1.638 ms |

*Conclusión:* Recorrer una lista llamando repetitivamente a `obtener(i)` es un antipatrón severo que degrada la operación a $O(N^2)$, acumulando 1.250 millones de saltos de puntero. El recorrido correcto sobre una lista debe ser lineal $O(N)$ iterando a través de `actual = actual.getSiguiente()`.

### Experimento 5: Búsqueda binaria sobre una cadena enlazada (50.000 lecturas)
Buscando el último elemento del lote ($N = 50.000$):
- **Búsqueda Lineal:** Posición 49.999 | Saltos entre nodos: **50.000**
- **Búsqueda Binaria sobre Lista:** Posición 49.999 | Saltos entre nodos: **749.990**

*Conclusión:* Aunque la búsqueda binaria realiza apenas 16 comparaciones de clave (igual que en un arreglo), cada cálculo de la mediana requiere navegar secuencialmente desde el inicio con `obtener(medio)`. Esto causa que la búsqueda binaria en una lista requiera **15 veces más saltos** que la búsqueda lineal. La estructura y el algoritmo deben seleccionarse armónicamente.

---

## 4. Respuestas a las Preguntas de la Bitácora (Actividad 10)

### Pregunta 1: ¿Por qué insertar al inicio es más económico en una lista enlazada que en un arreglo?
**Respuesta:**  
En un arreglo contiguo en memoria, la posición física de cada casilla está ligada a su índice. Insertar en la posición `0` obliga al sistema a desplazar los $N$ elementos ya existentes una casilla hacia la derecha para liberar el primer espacio ($O(N)$ operaciones de copia/movimiento). Además, si el arreglo está lleno, se debe reservar un bloque mayor y copiar todos los elementos.  
En una lista enlazada, la memoria no es contigua; cada nodo es un objeto independiente en el heap. Para insertar al inicio basta con crear el nuevo nodo, apuntar su referencia `siguiente` al nodo que actualmente encabeza la lista (`primero`) y actualizar la variable `primero = nuevo`. Esto toma un número fijo e invariable de instrucciones de máquina ($O(1)$), independientemente de si la lista tiene diez o diez millones de elementos.

### Pregunta 2: ¿Por qué el acceso a una posición arbitraria es más costoso en una lista?
**Respuesta:**  
El arreglo posee acceso aleatorio directo $O(1)$ gracias a la contigüidad espacial: la dirección en memoria del elemento $i$ se calcula con una operación aritmética instantánea:  
$$\text{Dirección}(i) = \text{Dirección Base} + (i \times \text{Tamaño del Elemento})$$  
En cambio, en una lista enlazada cada nodo solo conoce la ubicación del siguiente eslabón. Los nodos pueden estar ubicados en regiones físicas totalmente dispersas de la memoria RAM. Por tanto, para consultar la posición $i$ no existe una fórmula matemática directa: es estrictamente necesario situarse en `primero` y seguir secuencialmente la cadena de punteros $i$ veces ($O(N)$).

### Pregunta 3: ¿Qué diferencia existe entre un experimento de rendimiento y una prueba unitaria?
**Respuesta:**  
- **Prueba unitaria (Unit Test):** Evalúa la **corrección funcional** y el cumplimiento estricto del contrato de una porción aislada de código frente a entradas específicas y casos de borde (ej. listas vacías, eliminación en extremos, índices inválidos). Es determinista, debe ejecutarse en milisegundos y produce un resultado binario: aprobado (PASS) o reprobado (FAIL) mediante aserciones (`assertEquals`, `assertTrue`, `assertNull`).
- **Experimento de rendimiento (Benchmark):** Evalúa el **costo computacional y la escalabilidad asintótica** (tiempo de ejecución en milisegundos, saltos de memoria, movimientos de casillas, número de comparaciones) a medida que el volumen de datos ($N$) se incrementa. No verifica únicamente si el algoritmo arroja el dato correcto, sino cómo reacciona la máquina y los recursos del sistema al someterlo a alta carga.

### Pregunta 4: ¿Qué caso límite consideras más importante probar en `eliminar()` y por qué?
**Respuesta:**  
El caso límite más crítico a nivel de integridad estructural es **eliminar un elemento intermedio manteniendo la continuidad de la cadena**.  
Si al eliminar el nodo $k$ se desconecta por error asignando `anterior.setSiguiente(null)`, todos los nodos desde la posición $k+1$ hasta el final quedan huérfanos e inalcanzables en memoria, provocando una pérdida silenciosa y masiva de datos (la lista queda truncada sin generar ninguna excepción visible).  
Igualmente crítico es el caso de **eliminar el único nodo existente** ($N = 1$ en posición `0`), ya que debe garantizar que tanto `primero` como `ultimo` queden explícitamente en `null` y `cantidad = 0`, evitando que queden punteros colgantes o referencias fantasma en `ultimo` que corrompan inserciones posteriores.

### Pregunta 5: Para el histórico de lecturas de sensores, ¿qué estructura elegirías si predominan las inserciones al inicio? ¿Qué elegirías si predominan las consultas por posición? Justifica ambas decisiones.
**Respuesta:**  
- **Predominio de inserciones al inicio (Escenario de Ingesta en Tiempo Real):**  
  *Elección:* **Lista Enlazada (`RepositorioEnlazado`).**  
  *Justificación:* Cada nueva lectura que llega de la red IoT representa el evento más reciente y debe registrarse al comienzo del historial de inmediato. La lista enlazada realiza esta operación en $O(1)$ sin importar el histórico acumulado (0 saltos y < 1 ms). Un arreglo en este escenario colapsaría en rendimiento debido al constante desplazamiento de millones de lecturas previas ($O(N)$ por evento).
- **Predominio de consultas por posición (Escenario de Auditoría, Paginación y Análisis Estadístico):**  
  *Elección:* **Arreglo (`RepositorioArreglo`).**  
  *Justificación:* Si los operadores o algoritmos consultan frecuentemente posiciones arbitrarias (ej. calcular percentiles, saltar a registros por número de índice o aplicar algoritmos basados en partición como búsqueda binaria), el arreglo brinda acceso instantáneo $O(1)$ gracias a la indexación directa en memoria contigua. En una lista enlazada, cada consulta por posición obligaría a recorrer cientos de miles de enlaces ($O(N)$), haciendo que los paneles de visualización y consultas analíticas resulten inaceptablemente lentos.
