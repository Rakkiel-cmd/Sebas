# Manual de Conocimiento del Proyecto: Pre-Lab Metalmecánico 3D (Torno CNC)

Este documento resume la arquitectura, lógica de negocio, sistema 3D y estado actual del proyecto para que cualquier agente o desarrollador pueda retomarlo sin necesidad de explorar todo el código desde cero.

---

## 1. Descripción General
**Pre-Lab Metalmecánico 3D · Set-up y Seguridad en Torno CNC** es un simulador educativo web interactivo en 3D. Permite al estudiante practicar la puesta a punto segura de un torno CNC antes de operar una máquina real, evaluando el cumplimiento del protocolo SST/EPP, montaje, sujeción, toma de Cero Pieza (`G54`), selección de RPM y ejecución de un ciclo de desbaste.

- **Stack tecnológico**: HTML5, CSS3, JavaScript (ES6+ en IIFE, sin frameworks ni bundler) y **Three.js `r128`** + **`OrbitControls.js`** cargados vía CDN (`cdn.jsdelivr.net`).
- **Ejecución**: 100% del lado del cliente (estático). No requiere servidor backend ni paso de compilación.

---

## 2. Estructura de Archivos

```text
Pagina sebas/
├── html/
│   └── index.html          # Estructura de la interfaz, panel de pasos, visor 3D y scripts
├── css/
│   └── styles.css          # Tema oscuro industrial, layout Grid/Flex, HUD, modales y responsive
├── js/
│   └── main.js             # Escena Three.js, física básica, máquina de estados y eventos UI
└── Pre-Lab Metalmecánico 3D · Set-up y Seguridad en Torno CNC.html  # Archivo previo en raíz
```

---

## 3. Flujo Funcional y Reglas del Simulador

### 3.1. Estado Global (`S` y `T` en `js/main.js`)
- `S`: `{ epp, mounted, clamped, zero, rpm, mode }`
  - `mode`: `"idle"` | `"busy"` | `"cycle"` | `"crash"` | `"done"` | `"estop"`
- `T`: coordenadas lógicas de la herramienta en milímetros (`{ x: 55, z: 60 }` por defecto).
- **Métrica y Puntaje**:
  - Cada uno de los 4 pasos completados aporta `25%` (máximo `100%`).
  - Penalizaciones (`pen`): `-5` por advertencia (`danger()`), `-8` por colisión en movimiento manual (`jog()`), `-15` por colisión en ciclo (`crash()`).
  - KPIs en cabecera: `#k1` (Puntaje `%`), `#k2` (Errores críticos evitados), `#k3` (Ahorro estimado en USD).

### 3.2. Los 4 Pasos del Protocolo
1. **Paso 1 — Protocolo SST / EPP (`#s1`, `#bEpp`)**:
   - Checkboxes obligatorios (`data-ok="1"`): lentes de seguridad, zapatos dieléctricos, ropa ajustada.
   - Checkboxes prohibidos (`data-ok="0"`): guantes holgados de tela, cadena/anillo/reloj colgante (disparan alerta de atrapamiento).
2. **Paso 2 — Montaje y Sujeción (`#s2`, `#bMount`, `#bClamp`)**:
   - `#bMount`: anima la colocación de una barra de acero (`Ø60 × 160 mm`) en el plato.
   - `#bClamp`: anima el cierre de las 3 mordazas (`setJ()`).
3. **Paso 3 — Parámetros y Cero Pieza (`#s3`, `#rx`, `#rz`, `#bZero`, `[data-r]`)**:
   - Sliders `#rx` (`0–70 mm`) y `#rz` (`-20–100 mm`) mueven el carro en tiempo real (`jog()`).
   - **Detección de contacto (`touch()`)**: válida cuando `S.mounted` es verdadero, `|T.x - 30| <= 1.5` y `|T.z| <= 1.5`.
   - `#bZero`: fija `S.zero` (`G54`) solo si hay EPP, sujeción y contacto en el borde.
   - Botones `[data-r]`: `400`, `800` y `1200 RPM`. Para `Ø60 mm` con `Vc ≈ 80 m/min`, la única velocidad segura en el ciclo es **`400 RPM`**.
4. **Paso 4 — Prueba de Mecanizado (`#s4`, `#bGo`, `stepCycle()`)**:
   - Si falta `S.zero` (`kind: "nozero"`): el carro avanza en diagonal y colisiona contra la pieza.
   - Si `S.rpm === 800`: rompe inserto y portaherramienta en la fase de corte (`$450 USD`).
   - Si `S.rpm === 1200`: fuerza centrífuga suelta la pieza (eyección física `ej`) y destruye el inserto (`$1,450 USD`).
   - Si `S.zero` existe y `S.rpm === 400`: ejecuta 5 fases (`ph 0` a `4`): acercamiento a `G54` → penetración de `4 mm` (`Ø60` a `Ø52 mm`) → desbaste longitudinal de `120 mm` con virutas → retirada en `X` → retirada en `Z` → `finish()`.

### 3.3. Seguridad Adicional y Controles
- **Parada de emergencia (`#estop`)**: detiene el husillo y bloquea la operación hasta liberarse.
- **Colisiones en manual (`hits()`)**: detecta choque contra la barra (`T.x < 29 && T.z < -0.5`) o contra el contrapunto (`T.x < 12 && T.z > 35 && T.z < 96`), devolviendo el carro a `T.x = 55`.
- **Demo de error de novato (`#bNov`)**: omite EPP, sujeción y G54, programa `1200 RPM` y ejecuta el ciclo para demostrar una eyección y colisión total.

---

## 4. Sistema 3D y Mapeo de Coordenadas (`js/main.js`)
- **Escala**: `U = 40` (`40 mm` por cada unidad 3D).
- **Constantes geométricas**: `AY = 2.8` (altura del eje del husillo), `FACE = 1.3` (cara frontal de la pieza en X), `CH = -2.7` (posición del plato), `R0 = 0.75` (radio base equivalente a `30 mm`).
- **Conversión lógico → 3D (`apply()`)**:
  - `car.position.x = FACE + T.z / U` (eje axial Z del torno mapeado al eje X de Three.js).
  - `cs.position.z = T.x / U` (eje radial X del torno mapeado al eje Z de Three.js).
- **Mecanizado visual (`setCut(l)`)**: la barra (`wp`) se compone de dos cilindros (`rough` opaco y `cut` brillante) que ajustan su escala y posición dinámicamente conforme avanza el desbaste.
- **Efectos**: sistema de 200 partículas (`pts`, `chip()`, `chips()`), luz puntual de corte (`lamp`), halo de `G54` (`zm`), marcador de colisión (`cm` + `redL`), destello rojo (`#flash`) y sacudida de cámara (`shake`).

---

## 5. Estado Actual del Avance
### Completado
- Estructura modular separada en `html/index.html`, `css/styles.css` y `js/main.js`.
- Interfaz industrial oscura completa, responsiva (`@media(max-width:860px)`) y accesible (`focus-visible`, `prefers-reduced-motion`).
- Simulación 3D funcional con Three.js, OrbitControls, iluminación, sombras y texturas procedimentales.
- Flujo completo de 4 pasos con validaciones, penalizaciones, KPIs, HUD en vivo, registro de eventos (`#log`) y modales de resultado/alarma.
- Comentarios cortos en español añadidos en los tres archivos principales (`html/index.html`, `css/styles.css`, `js/main.js`).

### Pendiente / Posibles Mejoras Futuras
- **Persistencia**: actualmente no guarda progreso ni puntajes en `localStorage` ni en servidor; al recargar la página se reinicia la sesión.
- **Modo offline**: `three.min.js` y `OrbitControls.js` dependen de CDN externo; si se requiere uso sin internet, conviene descargarlos localmente.
- **Ampliación de operaciones**: hoy solo simula una operación fija de cilindrado/desbaste exterior (`4 mm × 120 mm`). Se podría extender a refrentado, ranurado o múltiples pasadas programables en código G.
- **Audio**: no cuenta aún con efectos de sonido (motor del husillo, alarma de colisión, corte).
