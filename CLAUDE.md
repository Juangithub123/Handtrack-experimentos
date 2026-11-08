# CLAUDE.md

App de escritorio para **Windows**, que funcione **sin conexión**, con efectos interactivos de
manos, cuerpo y cara usando MediaPipe. El plan completo, con decisiones, arquitectura, catálogo
de efectos y fases, está en **`PLAN.md`**. Leelo antes de trabajar.

## Stack decidido
- Electron + Vite + TypeScript.
- `@mediapipe/tasks-vision` con `delegate: "GPU"`, ejecutado en un Web Worker.
- Render en WebGL2 (Three.js o regl). Audio con Web Audio / Tone.js (synth interno).
- Empaquetado con electron-builder (`.exe` portable).

## Hardware objetivo
GTX 1060, CPU AMD quad core, webcam USB 1080p. La salida es un proyector o segunda pantalla
sobre una pared, sin calibración.

## Reglas del proyecto
- **Sin conexión siempre**: nada de CDNs en runtime. Los modelos `.task` van en
  `assets/models/` y los `wasm` de MediaPipe en `assets/wasm/`. Se descargan una sola vez con
  `npm run fetch-models`.
- **Varias personas**: los efectos iteran sobre `frame.hands[]`, `frame.poses[]` y
  `frame.faces[]`. Nunca suponer una sola persona.
- Cargar solo los modelos que piden (`requires`) los efectos activos.
- Cada efecto vive en `effects/<id>/` e implementa la interfaz `Effect` (ver `PLAN.md`
  sección 3). La UI de controles se genera a partir de `params`.
- Todo el texto de la UI va en español.
- Al terminar una tarea del plan, marcar el checkbox correspondiente en `PLAN.md`.
