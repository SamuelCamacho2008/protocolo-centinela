# Análisis Teórico y Experimental de Complejidad - Protocolo Centinela

## 1. Análisis Teórico de Complejidad

### 1.1 Notación
- `n`: Tamaño del mapa (n × n celdas)
- `V`: Vértices = celdas de calle ≈ n²/2 (patrón grid 4×4)
- `E`: Aristas ≈ 2V (grafo 4-conexo planar)
- `R`: Robots de transporte activos (≤ 3)
- `C`: Víctimas conocidas candidatas (≤ 6)
- `L`: Límite profundidad recursión = min(R, 4)
- `T`: Ticks totales de simulación = n × 15

---

### 1.2 Búsqueda de Rutas (Dijkstra / A*)

#### Estructuras de Datos
| Estructura | Tipo | Tamaño | Complejidad Espacial |
|------------|------|--------|---------------------|
| `dist` | Float64Array | V | O(V) |
| `prev` | Int32Array | V | O(V) |
| `visited` / `done` | Uint8Array | V | O(V) |
| `Heap` | Array (binary heap) | ≤ V | O(V) |
| **Total por búsqueda** | | | **O(V)** |

#### Tiempo por Búsqueda
```
Dijkstra:  O((V + E) log V) = O(V log V)   (ya que E = O(V))
A*:        O((V + E) log V) = O(V log V)   (peor caso)
A* típico: O(V_h log V)   donde V_h << V (nodos expandidos con h > 0)
```

**Factores constantes**:
- 4 vecinos por vértice → ~4V relajaciones
- Heap push/pop: ~2-3 operaciones por vértice expandido
- Manhattan h: O(1) por evaluación

#### Espacio de Recursividad (Call Stack)
- **Dijkstra/A***: Iterativo (while loop) → **O(1) stack space**
- Solo estructuras de datos en heap

---

### 1.3 Ciclo Principal del Juego (por tick)

#### Percepción (vision)
```
Para cada robot (5): radio 3 → 7×7 = 49 celdas
Tiempo: O(5 × 49) = O(1)   (constante acotada)
```

#### Planificación (search)
```
Llamadas por tick:
- Exploradores (2): 1 búsqueda cada uno si idle
- Transportes (3): 1-2 búsquedas cada uno (recoger + entregar)
- Re-planificaciones: variables (eventos)
Total estimado: 5-15 búsquedas/tick
Tiempo por tick: O(búsquedas × V log V)
```

#### Asignación (assign)
```
Modo Utilidad (recursivo):
- R robots libres, C víctimas candidatas (≤6)
- Llamadas: O((C+1)^R) con L = min(R,4)
- R ≤ 3, C ≤ 6 → máx ~343 llamadas
- Cada llamada: O(1) + acceso matriz utilidad
- Tiempo: O((C+1)^R) = O(1) acotado

Modo Cercanía (greedy):
- R robots × C víctimas
- Tiempo: O(R × C) = O(1) acotado
```

#### Ejecución y Evaluación
```
Por robot: O(longitud_ruta) = O(n) peor caso
Total 5 robots: O(n)
```

#### **Tiempo Total por Tick**
```
O(búsquedas × V log V + n) 
≈ O(V log V)   (domina búsquedas)
```

---

### 1.4 Simulación Completa

#### Ticks Totales
```
T = n × 15
```

#### Búsquedas Totales Estimadas
```
Por tick: ~10 búsquedas promedio
Total: ~10 × n × 15 = 150n búsquedas
```

#### Tiempo Total Teórico
```
O(T × V log V) = O(n × (n²/2) log n) = O(n³ log n)
```
Para n=25: ~25³ × log 25 ≈ 15,625 × 4.6 ≈ 72,000 ops base
Para n=129: ~129³ × log 129 ≈ 2,146,689 × 7 ≈ 15M ops base

#### Espacio Total
```
Grafo: O(V) = O(n²)
Robots: O(1)
Víctimas: O(n) 
Log: O(1) (ventana 40)
Benchmark: O(1) (acumuladores)
**Total: O(n²)**
```

---

### 1.5 Recursividad: Análisis Detallado

#### Función `best(i, used, depth, u, R, C, L)`

**Relación de Recurrencia**:
```
T(i, used, depth) = T(i+1, used, depth)           // no asignar
                  + Σ T(i+1, used∪{j}, depth+1)   // para cada j disponible
```

**Límite Superior**:
- En nivel i: hasta (C - i + 1) llamadas recursivas
- Profundidad máxima: L
- T(R, C, L) ≤ (C+1) × (C) × ... × (C-L+2) ≤ (C+1)^L

**Con parámetros reales**:
- R = 3, C = 6, L = 3
- T ≤ 7³ = 343 llamadas por asignación
- Asignaciones por partida: ~20-50
- Total llamadas/partida: ~7,000-17,000 (trivial)

**Espacio de Pila**:
- Profundidad máxima = L ≤ 4
- Frame por llamada: ~5 variables locales
- **O(L) = O(1)**

---

## 2. Análisis Experimental (Benchmark)

### 2.1 Metodología
- **Mapas**: 17×17, 33×33, 65×65, 129×129
- **Semilla**: Fija (2026 por defecto, configurable)
- **Repeticiones**: 5 repeticiones × 10 búsquedas = 50 búsquedas por algoritmo por tamaño
- **Condiciones idénticas**: Mismos orígenes, destinos, grafo, costos
- **Algoritmos**: Dijkstra vs A* (Manhattan)

### 2.2 Métricas Recogidas
| Métrica | Descripción |
|---------|-------------|
| `ms_medio` | Tiempo promedio por búsqueda (ms) |
| `nodos_medios` | Nodos expandidos promedio |
| `costo_medio` | Costo de ruta promedio |
| `A*_igual_óptimo` | % búsquedas donde A* = costo Dijkstra |

### 2.3 Formato CSV Exportado
```csv
equipo,semilla,tamano,repeticion,algoritmo,ms_medio,nodos_medios,costo_medio
"8 hilos / Win32",2026,17,1,Dijkstra,0.1234,45,12.50
"8 hilos / Win32",2026,17,1,A*,0.0892,28,12.50
...
```

### 2.4 Resultados Esperados (Referencia)

#### Tamaño 17×17 (V ≈ 144)
| Algoritmo | ms/búsqueda | Nodos | Costo | Optimalidad A* |
|-----------|-------------|-------|-------|----------------|
| Dijkstra  | ~0.1-0.3    | ~120  | ~15   | 100%           |
| A*        | ~0.05-0.2   | ~60   | ~15   | 100%           |

#### Tamaño 33×33 (V ≈ 544)
| Algoritmo | ms/búsqueda | Nodos | Costo | Optimalidad A* |
|-----------|-------------|-------|-------|----------------|
| Dijkstra  | ~0.3-0.8    | ~450  | ~35   | 100%           |
| A*        | ~0.15-0.4   | ~200  | ~35   | 100%           |

#### Tamaño 65×65 (V ≈ 2112)
| Algoritmo | ms/búsqueda | Nodos | Costo | Optimalidad A* |
|-----------|-------------|-------|-------|----------------|
| Dijkstra  | ~1-3        | ~1800 | ~70   | 100%           |
| A*        | ~0.5-1.5    | ~700  | ~70   | 100%           |

#### Tamaño 129×129 (V ≈ 8320)
| Algoritmo | ms/búsqueda | Nodos | Costo | Optimalidad A* |
|-----------|-------------|-------|-------|----------------|
| Dijkstra  | ~5-15       | ~7000 | ~140  | 100%           |
| A*        | ~2-8        | ~2500 | ~140  | 100%           |

### 2.5 Análisis de Escalabilidad

**Ratio nodos A*/Dijkstra esperado**: 0.3 - 0.5 (A* expande 30-50%)

**Ratio tiempos A*/Dijkstra esperado**: 0.4 - 0.7 (overhead heap similar)

**Complejidad empírica**: Debería seguir ~O(V log V) ≈ O(n² log n)

---

## 3. Prueba de Aceptación (Requisitos Guía)

### 3.1 Escenario de Prueba
1. Iniciar partida (tamaño 25, semilla 2026)
2. Esperar a que se detecten víctimas
3. **Bloquear una vía**: Botón "Bloquear vía" + click en calle transitada
4. **Desactivar un robot**: Click en robot transporte + "Desactivar robot elegido"
5. Verificar en panel:
   - `Tareas duplicadas` = 0
   - `Rutas con celdas bloqueadas` = 0
   - Re-asignación automática de víctima del robot desactivado
   - Re-planificación de rutas afectadas por bloqueo

### 3.2 Verificación de Algoritmos
- Cambiar selector "Algoritmo" a Dijkstra → jugar
- Cambiar a A* → jugar
- Comparar panel "Dijkstra frente a A*": nodos A* ≤ Dijkstra
- Ejecutar benchmark → verificar 100% optimalidad A*

### 3.3 Verificación de Recursividad
- Modo "Asignación por utilidad" (usa recursión)
- Modo "Asignación por cercanía" (no usa recursión)
- Panel muestra: `Recursión: X llamadas, profundidad Y`
- Profundidad nunca > 4 (límite L = min(R,4))

---

## 4. Resumen de Complejidad por Componente

| Componente | Tiempo | Espacio | Notas |
|------------|--------|---------|-------|
| **Dijkstra** | O(V log V) | O(V) | Montículo binario |
| **A*** | O(V log V) peor, O(V_h log V) típico | O(V) | h admisible + consistente |
| **Asignación Utilidad** | O((C+1)^R) acotado | O(L) stack | Recursividad con poda, L≤4 |
| **Asignación Cercanía** | O(R×C) | O(1) | Greedy |
| **Percepción** | O(1) por tick | O(V) global | Radio 3 fijo |
| **Simulación completa** | O(n³ log n) | O(n²) | T = 15n ticks |
| **Benchmark** | O(tamaños × reps × búsquedas × V log V) | O(1) extra | Offline, no afecta juego |

---

## 5. Conclusiones

1. **Dijkstra y A*** cumplen complejidad teórica O((V+E) log V) con montículo binario
2. **A* con Manhattan** es admisible y consistente → optimalidad garantizada, sin reapertura
3. **Recursividad acotada** (L ≤ 4) garantiza terminación y espacio O(1) en pila
4. **Benchmark reproducible** permite validación experimental con CSV exportable
5. **Prueba de aceptación** verificable en UI: contadores de duplicados y rutas bloqueadas en 0