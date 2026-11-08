# Handtrack Experimentos — Plan de desarrollo

App local para Windows, que funcione sin conexión, y explore efectos interactivos con
MediaPipe (Hands, Pose, Face y Segmentation) sobre una GPU NVIDIA. Tiene una **ventana de
controles** (operador) y una **ventana de proyección** (salida a pantalla, proyector o Spout).

> Estado: **planificación**. Las decisiones pendientes están marcadas con ❓.
> Este documento es el punto de partida para la sesión de Claude Code en Windows.

---

## 1. Restricción técnica clave: MediaPipe + GPU en Windows

| Vía | ¿Usa la GPU NVIDIA para inferir? | Comentario |
|---|---|---|
| **MediaPipe Python** (`pip install mediapipe`) | ❌ No en Windows: corre en CPU (XNNPACK) | El delegado GPU de Python solo está en Linux. En una CPU moderna igual alcanza ~30 fps con 1–2 modelos. |
| **MediaPipe Tasks Vision Web** (WASM + WebGL) dentro de **Electron** | ✅ Sí (WebGL → ANGLE/D3D11 → NVIDIA) | La vía más directa para tener GPU en Windows. Funciona sin conexión si se empaquetan los `.wasm` y los `.task`. |
| **Modelos MediaPipe convertidos a ONNX** + ONNX Runtime (CUDA/TensorRT/DirectML) | ✅ Sí (CUDA real) | Máximo rendimiento y escalable a varias personas, pero hay que reimplementar la tubería (detector → recorte → landmarks → suavizado). Conviene dejarlo para una fase posterior. |
| WSL2 + MediaPipe Linux GPU | ⚠️ Teóricamente | Acceder a la cámara desde WSL es frágil. No se recomienda. |

**Verificación recomendada (spike de 30 min en la máquina Windows):** correr un hand landmarker
en Python (CPU) y en Electron (GPU), y medir los ms por frame de cada uno.

---

## 2. Opciones de desarrollo

### Opción A — Electron + MediaPipe Web (GPU) + WebGL ⭐ recomendada
- **Stack:** Electron + Vite + TypeScript, `@mediapipe/tasks-vision`, Three.js o regl/WebGL2
  para los shaders, Web Audio (Tone.js) y WebMIDI.
- **Ventanas:** 2 `BrowserWindow`, una de controles y otra de proyección (pantalla completa en
  el segundo monitor). El estado se sincroniza por IPC o `BroadcastChannel`.
- **Offline:** los modelos `.task` y los `wasm/` de MediaPipe van dentro de `assets/`, sin CDN.
- **Empaquetado:** `electron-builder` genera un `.exe` instalable o portable.
- **Pros:** inferencia y render en la GPU, iteración muy rápida de efectos (hot reload),
  audio y MIDI nativos del navegador, UI de controles fácil de armar.
- **Contras:** JS/TS en lugar de Python, el bundle pesa unos 150 MB y hay que cuidar el hilo
  principal (conviene un Web Worker para el tracking).
- **Extra:** salida Spout vía `electron-spout` (Resolume, TouchDesigner, OBS).

### Opción B — Python nativo (MediaPipe CPU + render en GPU)
- **Stack:** Python 3.11/3.12, `mediapipe` (Tasks API), OpenCV (captura y blobs), PySide6
  (controles), ModernGL o pygame-ce (proyección con shaders), `mido` + `python-rtmidi` (MIDI),
  `python-osc`, `pyo` o `sounddevice` (síntesis).
- **Empaquetado:** PyInstaller o Nuitka.
- **Pros:** ecosistema Python (OpenCV, numpy, científico), fácil de leer y modificar, buen
  puente hacia ONNX/CUDA más adelante.
- **Contras:** la inferencia **no** usa la GPU NVIDIA (corre en CPU), y con varios modelos
  simultáneos los fps bajan. Las ventanas OpenGL y Qt juntas requieren algo de cuidado.

### Opción C — Python + ONNX Runtime GPU (CUDA/TensorRT)
- Igual que B, pero los modelos BlazePalm/BlazePose/FaceMesh se convierten a ONNX y se corren
  con `onnxruntime-gpu`.
- **Pros:** GPU NVIDIA real, máximo throughput y posibilidad de varias personas.
- **Contras:** es mucho más trabajo (anchors, ROI, suavizado), requiere instalar
  CUDA/cuDNN, y el empaquetado es pesado.
- **Uso sugerido:** como **fase futura** detrás de una interfaz `Tracker` común, para que los
  efectos no se enteren del cambio.

### Opción D — Híbrido: tracker separado + motor visual externo
- Un proceso de tracking (A o B) envía landmarks por **OSC/WebSocket**, y el render se hace en
  TouchDesigner, Unity, Godot o el mismo Electron.
- **Pros:** desacopla, y permite usar herramientas de VJ existentes.
- **Contras:** hay dos aplicaciones que mantener. Como base para "una app" no conviene, pero
  sí como **feature**: salida OSC de landmarks.

> Referencia rápida: si la meta fuera solo la estética, TouchDesigner con el plugin
> MediaPipe de la comunidad ya hace mucho de esto, pero la licencia gratuita limita la
> resolución y no sería "nuestra" app.

### Recomendación
**Opción A** como base, con la arquitectura preparada para:
- exponer los landmarks por **OSC** (opción D) y
- poder enchufar un tracker **ONNX/CUDA** externo más adelante (opción C).

---

## 3. Arquitectura propuesta (para la opción A, que se traslada casi igual a B)

```
┌─────────────── Proceso principal (Electron main) ───────────────┐
│ Ventanas, menús, presets en disco (JSON), salida Spout/OSC, MIDI │
└───────────────┬──────────────────────────────────┬──────────────┘
                │ IPC                              │ IPC
┌───────────────▼──────────────┐     ┌─────────────▼───────────────┐
│ Ventana CONTROLES            │     │ Ventana PROYECCIÓN          │
│ - selección de cámara        │     │ - cámara (getUserMedia)     │
│ - lista de efectos / escenas │◄───►│ - TrackerManager (Worker)   │
│ - parámetros (UI generada)   │state│ - EffectHost → render WebGL │
│ - preview + overlay debug    │     │ - audio (Web Audio/MIDI)    │
│ - fps / latencia / modelos   │     └─────────────────────────────┘
└──────────────────────────────┘
```

### Módulos
- **`CameraSource`**: elige el dispositivo, la resolución y los fps, y permite espejar y
  recortar (ROI).
- **`TrackerManager`**: activa **solo** los modelos que piden los efectos activos (hands,
  pose, face, segmentación y gestures). Corre en un Worker, devuelve un `TrackingFrame`
  normalizado y aplica un **filtro One Euro** para suavizar.
- **`TrackingFrame`** (contrato común para todos los trackers, sean web, Python u ONNX):
  ```ts
  { t: number, hands: Hand[], poses: Pose[], faces: Face[], mask?: Texture, gestures?: ... }
  ```
- **`Effect`**: un plugin con la siguiente interfaz:
  ```ts
  interface Effect {
    id: string; name: string;
    requires: ('hands'|'pose'|'face'|'segmentation'|'gestures')[];
    params: ParamSchema;           // la UI de controles se genera sola a partir de esto
    init(ctx): void;
    update(frame: TrackingFrame, dt: number, params): void;
    render(gl, params): void;
    dispose(): void;
  }
  ```
- **`Mapping/Calibration`**: transforma de cámara a proyección (homografía de 4 esquinas o
  keystone), lo que importa si se proyecta sobre la persona o sobre una pared.
- **`Presets/Escenas`**: guardan el efecto, sus parámetros y la calibración en un JSON, con
  teclas rápidas para cambiar de escena en vivo.
- **`Outputs`**: ventana a pantalla completa, Spout (fase 3), OSC de landmarks (fase 3) y
  grabación (fase 3).

### Estructura de carpetas sugerida
```
app/
  main/            # electron main: ventanas, ipc, presets, spout, osc
  renderer/
    control/       # UI de controles
    projection/    # host de efectos + render
    shared/        # store, tipos, TrackingFrame
  tracking/        # worker MediaPipe, filtros, normalización
  effects/
    debug-skeleton/
    hand-paint/
    blob-tracking/
    hole-in-wall/
    body-music/
    face-fx/
assets/
  models/          # *.task (hand_landmarker, pose_landmarker_{lite,full}, face_landmarker, selfie_segmenter)
  wasm/            # archivos de @mediapipe/tasks-vision/wasm
presets/
```

---

## 4. Catálogo de efectos

| Efecto | Modelos | Idea | Dificultad |
|---|---|---|---|
| **Debug / esqueleto** | todos | Overlay de landmarks, fps y latencia. Es la herramienta base. | Baja |
| **Pintura a mano** | hands (+gestures) | Pellizco = pincel; palma abierta = borrar; el color depende de la mano o la altura; trazos con trail o shader de tinta, y desvanecimiento opcional. | Baja |
| **Blob tracking** | segmentación o OpenCV-like | Cajas, IDs, líneas que conectan blobs y etiquetas con coordenadas (estética "TouchDesigner"). En la web: máscara → componentes conexas en el Worker. | Media |
| **Partículas y campos** | hands/pose | Las manos atraen o repelen partículas en la GPU; el cuerpo genera flujo. | Media |
| **Atravesar el hueco** (hole in the wall) | segmentación + pose | Una pared con una silueta se acerca; se mide qué parte del cuerpo queda dentro del hueco y se calcula el puntaje. Usa niveles y un modo juego. | Media |
| **Música con el cuerpo** | hands/pose | Theremin (altura y volumen con cada mano), pads espaciales (zonas que disparan notas), escalas cuantizadas, salida **MIDI** (loopMIDI → DAW) o synth interno. | Media |
| **Face FX** | face (blendshapes) | Boca abierta o cejas como triggers, máscaras, distorsión, control de parámetros con la expresión. | Media |
| **Espejo / slit-scan / feedback** | segmentación | Silueta con ecos temporales, slit-scan y feedback de shader. | Baja/Media |
| **Multi-persona** | pose `numPoses>1` | Interacciones entre personas (líneas, colisiones). | Media/Alta |

---

## 5. Fases

### Fase 0 — Spike (0,5 días)
- [ ] Medir MediaPipe Web (GPU) frente a Python (CPU) en la máquina real.
- [ ] Confirmar la decisión de stack.

### Fase 1 — MVP funcional (objetivo prioritario)
- [ ] Shell de Electron con 2 ventanas (controles y proyección a pantalla completa en el
      monitor elegido).
- [ ] Selección de cámara, espejo y resolución.
- [ ] `TrackerManager` con hands y pose (GPU), y carga de modelos local (offline verificado
      desconectando la red).
- [ ] Sistema de efectos con UI de parámetros autogenerada.
- [ ] Efectos: **Debug**, **Pintura a mano**, **Partículas**.
- [ ] Panel de rendimiento (fps de cámara, inferencia y render).
- [ ] Build `.exe` portable.

### Fase 2 — Más efectos
- [ ] Face landmarker y segmentación.
- [ ] **Blob tracking**, **Atravesar el hueco**, **Música con el cuerpo** (Web Audio y MIDI
      out), **Face FX**.
- [ ] Presets y escenas en JSON, con hotkeys.
- [ ] Calibración cámara↔proyección (4 esquinas).

### Fase 3 — Show y extensibilidad
- [ ] Salida **Spout** y **OSC** de landmarks (para TouchDesigner, Resolume y Max).
- [ ] Grabación de la salida y de las sesiones de tracking (replay sin cámara para
      desarrollar).
- [ ] Multi-persona, gesture recognizer y transiciones entre escenas.
- [ ] Control remoto desde el celular (UI web por LAN, opcional).

### Fase 4 — Futuro
- [ ] Tracker ONNX Runtime CUDA/TensorRT (opción C) detrás del mismo `TrackingFrame`.
- [ ] Varias cámaras y cámara de profundidad.
- [ ] Editor de efectos y shaders en vivo.

---

## 6. Decisiones pendientes ❓
1. ¿Stack A (Electron/TS, GPU) o B (Python, CPU)? ¿Qué lenguaje te resulta más cómodo para
   modificar los efectos vos mismo?
2. ¿Hardware? Modelo de GPU, de cámara (¿webcam 720p/1080p a 60 fps?) y CPU.
3. Salida de proyección: ¿segundo monitor o proyector solamente, o también Spout/NDI hacia
   otro software?
4. Música: ¿synth interno, MIDI hacia un DAW (Ableton, etc.), o ambos?
5. ¿Una persona o varias a la vez frente a la cámara?
6. ¿Se proyecta **sobre** la persona o pared (requiere calibración) o en una pantalla
   enfrentada, tipo espejo?

---

## 7. Prompt sugerido para arrancar en la sesión de Windows

> Leé `PLAN.md`. Vamos con la **Opción [A/B]**. Implementá la **Fase 0 y la Fase 1**:
> estructura de carpetas, 2 ventanas (controles y proyección), selección de cámara,
> TrackerManager con hands y pose cargando los modelos desde `assets/` (sin internet),
> sistema de efectos con parámetros autogenerados y los efectos Debug, Pintura a mano y
> Partículas. Agregá un script para descargar los modelos `.task` una sola vez y verificá
> que la app funcione con la red desconectada. Mi GPU es ___ y mi cámara es ___.
