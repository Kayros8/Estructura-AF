**Propuesta 3 — Delta como Matriz Dispersa**

*Análisis antes y después de la integración de la tabla hash*
# **Parte 1 — Antes de la integración de la tabla hash**
## **Idea general**
Los estados y símbolos se convierten en índices enteros y la función de transición delta se representa como un arreglo de entradas (from, symbol, to). La idea proviene de la representación matemática de un AFD como una tabla de doble entrada donde las filas son estados y las columnas son símbolos. El término "dispersa" indica que solo se almacenan las entradas que realmente existen, no todas las combinaciones posibles de estado y símbolo.
## **Estructuras**
typedef struct {\
`    `int   from;    /\* indice del estado origen  \*/\
`    `int   symbol;  /\* indice del simbolo        \*/\
`    `Tdata to;      /\* SET de estados destino    \*/\
} TransitionEntry;\
\
typedef struct {\
`    `Tdata            Q;          /\* SET de nombres de estados   \*/\
`    `Tdata            Sigma;      /\* SET de simbolos             \*/\
`    `TransitionEntry\* delta;      /\* arreglo de transiciones     \*/\
`    `int              deltaSize;  /\* cantidad de transiciones    \*/\
`    `int              q0;         /\* indice del estado inicial   \*/\
`    `Tdata            F;          /\* SET de estados de aceptacion\*/\
} Automata;
## **Cómo funcionaba la búsqueda**
La idea conceptual sugería un acceso O(1) como en una matriz real, pero al estar implementada como un arreglo lineal de TransitionEntry, la búsqueda de delta(q, a) era en realidad lineal sobre deltaSize:

Buscar delta(1, "b"):\
\
delta[0]: from=0, symbol=0 -> (0,0) != (1,1) -> siguiente\
delta[1]: from=0, symbol=1 -> (0,1) != (1,1) -> siguiente\
delta[2]: from=1, symbol=1 -> (1,1) == (1,1) OK -> retorna to
## **El problema de la traducción nombre ↔ índice**
Cada vez que se operaba con el autómata había que traducir entre el mundo de los nombres y el mundo de los índices. Para procesar la cadena "ab" se necesitaba recorrer Sigma buscando "a" para obtener su índice, recorrer Q buscando "q0" para obtener el suyo, y luego convertir los índices destino de vuelta a nombres. Este proceso de traducción debía repetirse en cada paso del procesamiento, lo que agregaba una capa de complejidad que no existía en las demás propuestas.
## **Ventajas**
- El modelo mental de tabla es muy claro para quien viene de la teoría formal de autómatas.
- deltaSize da inmediatamente la cantidad de transiciones definidas.
- Representación compacta que solo almacena las transiciones que realmente existen.
## **Desventajas**
- Prometía O(1) pero no lo cumplía: la búsqueda real era O(n) lineal sobre el arreglo.
- Capa de traducción nombre ↔ índice presente en cada operación, fuente de complejidad y errores.
- Agregar transiciones dinámicamente requería realloc, más costoso que encadenar un nodo.
- La peor integración con el sistema Tdata de todas las propuestas analizadas.
## **Conclusión de esta etapa**
La Propuesta 3 era la que más se alejaba del sistema Tdata y la que introducía más complejidad accidental sin una ganancia real de rendimiento. Su falla no era conceptual sino estructural: le faltaba una pieza que permitiera convertir un nombre en un índice sin recorrer nada. Sin esa pieza, tanto la búsqueda de la transición como la traducción de nombres terminaban siendo búsquedas lineales.

# **Parte 2 — Después de la integración de la tabla hash**
## **La pieza que faltaba**
La tabla hash mapea una cadena a un entero en tiempo O(1) promedio. La traducción nombre → índice que tanto costaba (convertir "q0" en 0, "a" en 0) es exactamente un mapeo de cadena a entero. La tabla hash no es solo útil para la Propuesta 3: tiene la forma exacta del agujero que la propuesta tenía. Esto transforma su viabilidad de raíz, porque ambos problemas originales (la búsqueda lineal de la transición y la traducción de nombres) eran en el fondo el mismo problema, y la tabla hash lo resuelve de una vez.
## **Rediseño: matriz real con traducción por hash**
Se mantienen dos tablas hash que funcionan como diccionarios de traducción y una matriz bidimensional real para delta. Los SET Q y Sigma dejan de usarse para buscar y pasan a guardar solo los nombres; las tablas hash se encargan de la búsqueda rápida.

typedef struct {\
`    `Tdata      Q;          /\* SET de nombres, solo para iterar \*/\
`    `Tdata      Sigma;      /\* SET de simbolos, solo para iterar\*/\
\
`    `/\* Traductores O(1): idxEstado["q0"]=0, idxSimbolo["a"]=0 \*/\
`    `TablaHash\* idxEstado;\
`    `TablaHash\* idxSimbolo;\
\
`    `/\* La matriz REAL. delta[i][j] = SET destino o NULL \*/\
`    `Tdata\*\*    delta;\
`    `int        nEstados;\
`    `int        nSimbolos;\
\
`    `int        q0;            /\* indice del estado inicial \*/\
`    `Tdata      F;             /\* SET de aceptacion         \*/\
`    `int        deterministic;\
} Automata;
## **División de responsabilidades**
Antes, Q y Sigma cargaban dos trabajos a la vez: guardar los nombres y servir para buscar índices, y este segundo trabajo lo hacían linealmente. Ahora esos roles se separan: los SET guardan los nombres para iterarlos, y las tablas hash se encargan de la búsqueda en O(1). Cada estructura hace una sola cosa y la hace bien.
## **Llenado de los índices al construir**
void registrar\_estado(Automata\* A, const char\* nombre) {\
`    `A->Q = set\_append(A->Q, crear\_str(nombre));\
`    `insertar(A->idxEstado, nombre, A->nEstados); /\* "q0"->0 \*/\
`    `A->nEstados++;\
}\
\
void registrar\_simbolo(Automata\* A, const char\* simbolo) {\
`    `A->Sigma = set\_append(A->Sigma, crear\_str(simbolo));\
`    `insertar(A->idxSimbolo, simbolo, A->nSimbolos); /\* "a"->0 \*/\
`    `A->nSimbolos++;\
}
## **La función delta ahora sí es O(1) real**
Tdata delta(Automata\* A, const char\* q, const char\* a) {\
`    `int i, j;\
\
`    `/\* Paso 1: nombre de estado -> indice, O(1) por hash \*/\
`    `if (!buscar(A->idxEstado, q, &i))  return NULL;\
\
`    `/\* Paso 2: nombre de simbolo -> indice, O(1) \*/\
`    `if (!buscar(A->idxSimbolo, a, &j)) return NULL;\
\
`    `/\* Paso 3: acceso directo a la matriz, O(1) real \*/\
`    `return A->delta[i][j];\
}

En la versión original este mismo cálculo recorría potencialmente todas las transiciones del autómata. Ahora son dos consultas a tablas hash y un acceso a un arreglo, ninguno de los cuales depende del tamaño del autómata.
## **Alternativa: dispersa con clave compuesta**
Una matriz completa reserva espacio para todas las combinaciones de estado y símbolo, incluso las inexistentes. La alternativa usa una sola tabla hash con clave compuesta, combinando estado y símbolo en una cadena como "q0,a" mediante cat\_str. Como la tabla guarda int y no Tdata, se almacena un índice hacia un arreglo separado de SET: la tabla mapea "q0,a" → 3 y el SET destino vive en destinos[3]. Así solo se guardan las transiciones que existen y se encuentran en O(1), a costa de construir y liberar la cadena compuesta en cada consulta.

char\* construir\_clave(const char\* estado, const char\* simbolo) {\
`    `char\* parcial = cat\_str(estado, ",");    /\* "q0,"  \*/\
`    `char\* clave   = cat\_str(parcial, simbolo); /\* "q0,a" \*/\
`    `free(parcial);\
`    `return clave;\
}
## **Ventajas tras la integración**
- El acceso a delta pasa de O(n) a O(1) real, no prometido.
- La traducción nombre ↔ índice deja de ser un problema: ahora es una consulta hash.
- La tabla hash crece sola mediante rehashing, permitiendo agregar estados dinámicamente sin redimensionar manualmente.
- Cada estructura tiene una responsabilidad única y clara.
## **Consideraciones que permanecen**
- La matriz y las tablas hash son estructuras propias que conviven con los SET; no son SET ellas mismas.
- La matriz completa ocupa memoria proporcional a nEstados × nSimbolos, ocupe o no todas las casillas.
- Para alfabetos enormes con pocas transiciones, conviene la variante dispersa con clave compuesta.
# **Comparación: antes vs después**

|**Aspecto**|**Antes del hash**|**Después del hash**|
| :-: | :-: | :-: |
|Acceso a delta|O(n) lineal|O(1) real|
|Traducción nombre→índice|O(n), recorre Q y Sigma|O(1) por tabla hash|
|Promesa O(1)|No la cumple|La cumple|
|Crecimiento dinámico|realloc manual|Rehashing automático|
|Viabilidad general|La más débil|De las más fuertes|

## **Veredicto final**
Con la integración de la tabla hash, la Propuesta 3 deja de ser la propuesta más débil y se convierte en una de las más fuertes para el caso de simular autómatas con muchas consultas a delta. Su única gran falla era estructural y no conceptual, y la tabla hash aporta exactamente la estructura que le faltaba. Gana claramente en velocidad de consulta; para implementar algoritmos formales de transformación como la determinización, la Propuesta 4 sigue siendo más natural. Son fortalezas en ejes distintos.
