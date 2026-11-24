# Protocolo Centinela — Red de Rescate Táctico con IA Agéntica

**Proyecto Final · Algoritmos Computacionales**  
**Fundación Universitaria de Popayán**

---

## 🎮 Descripción del Videojuego

**Protocolo Centinela** es un videojuego interactivo 2D de vista cenital ambientado en un complejo subterráneo afectado por un colapso geotérmico.

A diferencia de una simple simulación pasiva, **tú juegas directamente controlando a la unidad piloto `CMD-0`** (`WASD` / Flechas / cruceta táctil) con habilidades activas de **Sonar (`Espacio`)**, **Despeje de escombros (`E`)** y **Rescate directo / Asistencia energética (`F`)**, mientras coordinas en tiempo real a un escuadrón de **5 agentes autónomos** (`S1`, `S2` Sondas de Reconocimiento y `U1`, `U2`, `U3` Unidades de Extracción).

---

## 🚀 Instrucciones de Ejecución

### Opción 1: Archivo Único Autocontenido (Recomendado)
Abre `index.html` (o `src/index.html`) directamente en cualquier navegador moderno. No requiere compilación, servidores ni dependencias externas.

### Opción 2: Servidor Local
```bash
npx http-server . -p 8000
# Abre http://localhost:8000
```

### Opción 3: GitHub Pages
El archivo principal `index.html` está en la raíz del repositorio para ser desplegado directamente con **GitHub Pages** (`Settings → Pages → Branch: main / root`).

---

## 🕹️ Controles del Jugador (`CMD-0`) y Simulación

### Jugador Activo (`CMD-0`, hexágono dorado)
| Tecla / Botón | Acción |
|---------------|--------|
| **`W A S D` / Flechas** | Mover a `CMD-0` por las galerías del complejo |
| **`Espacio` / Sonar** | Pulso de radar de radio 5 que revela niebla y localiza civiles |
| **`E` / Despejar vía** | Retira escombros o estabiliza riesgo térmico cercano (`10` energía) |
| **`F` / Rescatar o Asistir** | Recoge un civil cercano para evacuarlo tú mismo o transfiere energía a un robot |
| **Clic en robot / pestaña** | Selecciona un agente autónomo para ver su telemetría y ruta |

### Agentes Autónomos
- **Sondas de Reconocimiento (`S1`, `S2`)**: Se dividen el mapa por mitades (Oeste/Este), descubren civiles con radio de visión 3 y recargan en el Núcleo Central cuando su energía baja de 60.
- **Unidades de Extracción (`U1`, `U2`, `U3`)**: Planifican rutas óptimas hasta civiles conocidos y los evacúan a las escotillas seguras. Consumen `1.0` energía/paso sin carga y `1.25` con carga.

### Modos Algorítmicos y Pruebas de Estrés
- **Búsqueda de rutas**: `A*` (heurística Manhattan admisible y consistente) vs `Dijkstra` sobre montículo binario (`Heap`) y grafo de pesos positivos (`1 + riesgo`).
- **Estrategia de asignación**:
  1. **Utilidad Recursiva** (`asignarOptimoRecursivo`): búsqueda combinatoria acotada a profundidad `min(R, 4)` maximizando `1000 + 15·prioridad − 0.5·costo_total`.
  2. **Cercanía (Greedy)**: asignación voraz al civil de menor costo.
- **Pruebas de aceptación en vivo**: Botones **Onda sísmica**, **Colapsar vía** y **Desactivar unidad elegida** con conteo en vivo de **Tareas duplicadas = 0** y **Rutas bloqueadas = 0**.

---

## 📊 Panel de Monitoreo y Benchmark

- **Telemetría del Agente**: Estado, objetivo, último evento, energía, pasos restantes, algoritmo, heurística, costo, nodos expandidos, tiempo (`ms`), llamadas/profundidad de recursión y contadores de aceptación.
- **Comparativa en Vivo**: Tabla acumulada de `Dijkstra` vs `A*` durante la partida.
- **Benchmark Experimental**: Ejecuta 5 repeticiones × 10 búsquedas en mapas `17×17`, `33×33`, `65×65` y `129×129` con exportación directa a **CSV**.

---

## 📁 Estructura del Repositorio

- [`index.html`](index.html): Videojuego completo autocontenido (listo para GitHub Pages).
- [`src/index.html`](src/index.html): Copia sincronizada dentro de `src/`.
- [`docs/arquitectura_agentes.md`](docs/arquitectura_agentes.md): Ciclo del agente, FSM y comunicación multiagente.
- [`docs/grafos_heuristicas_recursividad.md`](docs/grafos_heuristicas_recursividad.md): Modelo de grafo, demostración de admisibilidad/consistencia y recursividad.
- [`docs/analisis_complejidad.md`](docs/analisis_complejidad.md): Análisis teórico y experimental de complejidad.
- [`tests/test_acceptance.html`](tests/test_acceptance.html): Lista de verificación de pruebas de aceptación.
