# Documentación de Arquitectura de Agentes - Protocolo Centinela

## 1. Visión General

El sistema implementa una arquitectura de agentes inteligentes autónomos basada en el ciclo:
**Entorno → Percepción → Memoria y Contexto → Planificación → Decisión → Ejecución → Evaluación → Actualización**

Cada unidad autónoma (`S1`, `S2`, `U1`, `U2`, `U3`) opera de forma independiente, percibiendo su entorno local, manteniendo memoria de eventos, planificando rutas, tomando decisiones basadas en objetivos y restricciones, ejecutando acciones válidas y re-planificando ante cambios del entorno. Adicionalmente, el jugador controla directamente a la unidad piloto **`CMD-0`** (movimiento `WASD`/flechas, pulso de sonar, despeje de vías y rescate directo).

---

## 2. Tipos de Agentes

### 2.1 Sondas de Reconocimiento (S1, S2)
- **Rol**: Descubrir civiles atrapados y mapear el complejo subterráneo
- **Comportamiento**: Se asignan sectores de exploración (mitades Oeste/Este del mapa)
- **Percepción**: Radio de visión = 3 celdas
- **Estados**: `explorar`, `recarga`, `carga`, `libre`
- **Energía inicial**: 150 unidades
- **Consumo**: 1 unidad por paso

### 2.2 Unidades de Extracción (U1, U2, U3)
- **Rol**: Recoger civiles descubiertos y evacuarlos a escotillas seguras
- **Comportamiento**: Se asignan civiles según estrategia (utilidad recursiva o cercanía)
- **Percepción**: Radio de visión = 3 celdas (compartida)
- **Estados**: `recoger`, `entregar`, `recarga`, `carga`, `libre`
- **Energía inicial**: 150 unidades
- **Consumo**: 1.25 unidades por paso (llevando civil), 1 unidad (vacío)

### 2.3 Comandante de Campo (CMD-0 - Jugador)
- **Rol**: Unidad interactiva operada por el jugador
- **Acciones**: Movimiento libre (`WASD`/flechas), Pulso de Sonar (`Espacio`, radio 5), Despeje de escombros (`E`), Rescate directo o recarga aliada (`F`)

---

## 3. Ciclo del Agente

### 3.1 Percepción
```javascript
// Cada tick, cada robot actualiza celdas vistas en radio 3
for (dy = -3 to 3) for (dx = -3 to 3) 
    seen[robot.y + dy][robot.x + dx] = true
```
- Observaciones locales (radio 3 celdas)
- Eventos autorizados: hallazgo de víctimas, colapso de vías, réplicas
- Detección de energía crítica

### 3.2 Memoria y Contexto
Estructuras de memoria por agente:
- `seen`: Uint8Array - celdas exploradas globalmente
- `victims`: Array - víctimas conocidas con estado (hidden/known/claimed/carry/ok)
- `claims`: Asignaciones actuales robot-víctima
- `path`: Ruta planificada actual
- `energy`: Nivel de energía restante
- `replans`: Contador de re-planificaciones

### 3.3 Planificación
**Algoritmos implementados**:
1. **Dijkstra**: Búsqueda de costo mínimo con montículo binario
2. **A***: Búsqueda guiada con heurística Manhattan

**Grafo**:
- Vértices: Celdas de calle libres (`road[i] = 1`)
- Aristas: 4 vecinos (arriba, abajo, izquierda, derecha)
- Costo de arista: `1 + risk[v]` donde `risk ≥ 0`
- Vías bloqueadas: Eliminadas del grafo (`blocked[i] = 1`)

### 3.4 Decisión
**Modo Utilidad** (recursivo con poda):
```
utilidad = 1000 + 15 × prioridad_víctima − 0.5 × (costo_recogida + costo_entrega)
```
- Función recursiva `best(i, used, depth, ...)` con caso base: `i >= R || depth >= L`
- Límite de profundidad: `min(R, 4)` donde R = robots libres

**Modo Cercanía** (greedy):
- Asigna cada robot a la víctima conocida más cercana disponible
- Sin recursividad, complejidad O(R × V)

### 3.5 Ejecución
- Un paso por tick del juego
- Verificación de celdas bloqueadas antes de moverse
- Consumo de energía según carga
- Llegada a objetivo → transición de estado

### 3.6 Evaluación y Re-planificación
Triggers de re-planificación:
- Celda de la ruta bloqueada (`blocked[next] = 1`)
- Riesgo crítico en ruta (`risk ≥ 4`)
- Energía insuficiente para completar misión
- Objetivo ya no válido (víctima rescatada por otro)

---

## 4. Comunicación Multiagente

**Mensajes implementados** (registro en panel):
- `Msg S1→Escuadrón: libero civil #X (vía inaccesible)`
- `Msg U1: reclamo civil #X (P2, costo Z, utilidad)`
- `U2 solicita recarga: batería crítica.`
- `Msg emergencia: civil #X requiere reasignación inmediata.`

No hay canal de comunicación explícito; la comunicación es **estigmergia** a través del estado compartido del entorno (memoria global `g`).

---

## 5. Estados de Víctimas

| Estado | Descripción |
|--------|-------------|
| `hidden` | No descubierta (fuera de radio de visión) |
| `known` | Detectada, sin asignar |
| `claimed` | Asignada a un robot de transporte |
| `carry` | Siendo transportada |
| `ok` | Entregada en zona segura |

---

## 6. Eventos del Entorno

### 6.1 Réplicas Programadas
- 2 réplicas en ticks `n*3` y `n*7` (n = tamaño del mapa)
- Colapsan una vía aleatoria + zona de riesgo radio 2 (costo +4)

### 6.2 Réplica Provocada
- Botón "Provocar réplica" - misma lógica

### 6.3 Bloqueo Manual
- Botón "Bloquear vía" + click en calle
- Elimina la celda del grafo

### 6.4 Desactivación de Robot
- Click en robot + botón "Desactivar robot elegido"
- Si llevaba víctima → víctima vuelve a `known` en esa posición

---

## 7. Condiciones de Victoria/Derrota

**Victoria**: `rescued >= goal` (goal = 70% de víctimas totales)

**Derrota**:
- `t >= limit` (límite = `n * 14` ticks)
- `robots_transporte_activos == 0`