# Documentación de Grafos, Heurísticas y Recursividad - Protocolo Centinela

## 1. Definición del Grafo

### 1.1 Vértices
- Cada celda de calle libre (`road[i] = 1`) es un vértice
- Total vértices: `V = count(road[i] == 1)`
- Índice lineal: `i = y * n + x` donde `n` = tamaño del mapa

### 1.2 Aristas
- Conexión 4-vecinos (von Neumann): arriba, abajo, izquierda, derecha
- Arista existe si ambas celdas son calle y no están bloqueadas
- Total aristas: `E ≈ 2 * V` (grafo planar tipo grid)

### 1.3 Costos
```
costo(u → v) = 1 + risk[v]
```
- `1`: Costo base de movimiento (nunca negativo)
- `risk[v]`: Nivel de riesgo de la celda destino (0 a 8)
- **Propiedad**: `costo ≥ 1` siempre (costos no negativos)

### 1.4 Modificaciones Dinámicas
| Evento | Efecto en Grafo |
|--------|-----------------|
| Vía bloqueada | Vértice removido (`blocked[i] = 1`) |
| Réplica (colapso) | Vértice removido + `risk` +4 en radio 2 |
| Riesgo crítico | `risk ≥ 4` desencadena re-planificación |

---

## 2. Algoritmo de Dijkstra

### 2.1 Pseudocódigo
```
function Dijkstra(G, s, t):
    dist[v] = ∞ para todo v
    dist[s] = 0
    prev[v] = null
    H = min-heap(key=dist)
    H.push(s, 0)
    
    while H no vacío:
        u = H.pop_min()
        if u == t: break
        if visited[u]: continue
        visited[u] = true
        
        para cada vecino v de u:
            si G.road[v] y no G.blocked[v] y no visited[v]:
                alt = dist[u] + 1 + G.risk[v]
                si alt < dist[v]:
                    dist[v] = alt
                    prev[v] = u
                    H.push(v, alt)
    
    return reconstruir_camino(prev, t)
```

### 2.2 Complejidad
- **Tiempo**: O((V + E) log V) con montículo binario
  - V = vértices (celdas de calle)
  - E = aristas (~2V para grid 4-conexo)
  - Operaciones heap: O(log V) cada una
- **Espacio**: O(V)
  - `dist`: Float64Array(V)
  - `prev`: Int32Array(V)
  - `visited`: Uint8Array(V)
  - Heap: hasta V elementos

### 2.3 Implementación
- Clase `Heap` personalizada (array-based binary heap)
- Arrays tipados para rendimiento: `Float64Array`, `Int32Array`, `Uint8Array`
- Búsqueda temprana: termina al extraer objetivo `t`

---

## 3. Algoritmo A*

### 3.1 Función de Evaluación
```
f(n) = g(n) + h(n)
```
- `g(n)`: Costo real desde inicio hasta n
- `h(n)`: Heurística estimada de n a objetivo

### 3.2 Heurística: Distancia Manhattan
```
h(n) = |x_n - x_t| + |y_n - y_t|
```
Donde `(x_n, y_n)` = coordenadas de n, `(x_t, y_t)` = objetivo

### 3.3 Justificación de Admisibilidad
**Definición**: h es admisible si `h(n) ≤ h*(n)` para todo n, donde `h*(n)` es el costo real óptimo.

**Demostración**:
- Cada paso en el grid cambia x o y en ±1
- Distancia Manhattan = número mínimo de pasos en grid sin obstáculos
- Costo real de cada paso = `1 + risk[v] ≥ 1`
- Por tanto: `h(n) = pasos_mínimos ≤ Σ(1 + risk) = costo_real_óptimo = h*(n)`
- ✓ **h es admisible**

### 3.4 Justificación de Consistencia (Monotonía)
**Definición**: h es consistente si `h(u) ≤ c(u,v) + h(v)` para toda arista (u,v).

**Demostración**:
- `h(u) = |x_u - x_t| + |y_u - y_t|`
- `h(v) = |x_v - x_t| + |y_v - y_t|`
- Para vecinos: `|x_u - x_v| + |y_u - y_v| = 1`
- Por desigualdad triangular: `h(u) ≤ 1 + h(v)`
- Como `c(u,v) = 1 + risk[v] ≥ 1`: `h(u) ≤ c(u,v) + h(v)`
- ✓ **h es consistente**

### 3.5 Consecuencias
- A* con heurística consistente **nunca reabre nodos** (no necesita `visited` check al hacer pop)
- Garantiza **optimalidad**: encuentra camino de costo mínimo
- Expande **menos o igual nodos** que Dijkstra (igual en peor caso: h=0)

### 3.6 Complejidad A*
- **Tiempo**: O((V + E) log V) peor caso (igual que Dijkstra)
- **Tiempo promedio**: Substantialmente menor por poda con h
- **Espacio**: O(V) (mismas estructuras)

---

## 4. Recursividad: Asignación por Utilidad

### 4.1 Problema
Asignar R robots de transporte a C víctimas conocidas (C ≤ 6) maximizando utilidad total, con restricciones:
- Un robot → máximo una víctima
- Una víctima → máximo un robot
- Robot debe tener energía suficiente para misión completa

### 4.2 Función Recursiva
```javascript
function best(i, used_mask, depth, utility_matrix, R, C, L):
    // i: índice de robot actual (0..R-1)
    // used_mask: bitmask de víctimas ya asignadas
    // depth: profundidad actual de recursión
    // L: límite de profundidad = min(R, 4)
    
    // CASO BASE
    if i >= R or depth >= L:
        return {score: 0, pairs: []}
    
    // Opción 1: No asignar este robot
    best_result = best(i + 1, used_mask, depth, ...)
    
    // Opción 2: Probar cada víctima disponible
    for j in 0..C-1:
        if not (used_mask & (1 << j)) and utility_matrix[i][j] >= 0:
            sub = best(i + 1, used_mask | (1 << j), depth + 1, ...)
            total = utility_matrix[i][j] + sub.score
            if total > best_result.score:
                best_result = {score: total, pairs: [[i,j]] + sub.pairs}
    
    return best_result
```

### 4.3 Análisis de Complejidad

#### Llamadas Recursivas
- En cada nivel i, se prueba hasta (C - i) víctimas + opción "no asignar"
- Árbol de recursión: factor de ramificación ≈ C+1
- Profundidad máxima: L = min(R, 4)
- **Total llamadas**: O((C+1)^R) acotado por L

#### Con límites prácticos (R ≤ 3, C ≤ 6, L ≤ 3):
- Llamadas máximas ≈ 7^3 = 343 (muy manejable)
- En práctica: ~50-100 llamadas por asignación

#### Espacio (Pila de Llamadas)
- Profundidad máxima: L ≤ 4
- **Espacio**: O(L) = O(1) (constante acotada)

### 4.4 Caso Base y Progreso
| Componente | Implementación |
|------------|----------------|
| **Caso base** | `i >= R || depth >= L` - retorna score 0 |
| **Progreso** | `i + 1` (siguiente robot) y `depth + 1` |
| **Terminación garantizada** | i crece monótonamente hasta R; depth hasta L |

### 4.5 Función de Utilidad
```
U(robot, víctima) = 1000 + 15 × prioridad - 0.5 × (costo_recogida + costo_entrega)
```
- `1000`: Base para priorizar asignaciones sobre no-asignar
- `15 × prioridad`: Víctimas prioridad 3 valen 45 más que prioridad 1
- `-0.5 × costo`: Penaliza rutas largas, favorece eficiencia
- `-1` (inválido): Si energía insuficiente o no hay ruta

---

## 5. Comparación Teórica Dijkstra vs A*

| Aspecto | Dijkstra | A* (Manhattan) |
|---------|----------|----------------|
| **Optimalidad** | ✓ Garantizada | ✓ Garantizada (h admisible) |
| **Reapertura nodos** | No | No (h consistente) |
| **Nodos expandidos** | Todos con dist ≤ óptimo | Subconjunto (guiado por h) |
| **Peor caso tiempo** | O((V+E) log V) | O((V+E) log V) |
| **Caso típico** | Más nodos | Menos nodos (h > 0) |
| **Heurística** | h = 0 | h = Manhattan |

---

## 6. Validación Experimental (Benchmark)

El juego incluye benchmark reproducible:
- **Mapas**: 17×17, 33×33, 65×65, 129×129
- **Semilla**: Fija (configurable en UI)
- **Repeticiones**: 5 × 10 búsquedas por tamaño
- **Mismos**: Orígenes, destinos, grafo, costos para ambos algoritmos
- **Métricas**: ms/búsqueda, nodos expandidos, costo medio, % optimalidad A*

### Resultados Esperados
- A* expande ~30-60% menos nodos que Dijkstra
- Tiempos A* < Dijkstra (salvo overhead heap)
- 100% optimalidad A* (misma ruta que Dijkstra)
- CSV exportable para informe