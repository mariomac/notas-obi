# AGENTS.md — Guía para agentes en `notas-obi`

## Qué es este repositorio

Este repositorio **no contiene código propio**. Es un cuaderno personal de notas
sobre [OBI (OpenTelemetry eBPF Instrumentation)](https://github.com/open-telemetry/opentelemetry-ebpf-instrumentation),
un proyecto complejo que estoy tratando de entender a fondo. El objetivo es ir
acumulando en él documentos en Markdown que expliquen partes del código, flujos,
decisiones de diseño, etc., a medida que los voy investigando.

## Estructura

- `obi/` — **submódulo git** apuntando a `open-telemetry/opentelemetry-ebpf-instrumentation`.
  Es el **código fuente objeto de análisis**. No se modifica desde este repo.
- `obi/devdocs/` — documentación oficial del proyecto OBI. **Fuente primaria**
  para entender internals: pipeline, propagación de contexto, correlación
  trace-log, integración con runtimes (Go, Node, Python, Java…), formato de
  eventos BPF, etc. Consultar aquí antes de deducir cosas desde el código.
- `*.md` en la raíz — **las notas**. Cada archivo es un walkthrough o análisis
  sobre un tema concreto. Ejemplo actual:
  [l7-context-propagation-blackbox-walkthrough.md](l7-context-propagation-blackbox-walkthrough.md)
  (paseo por el código de la ruta L7 black-box de propagación de contexto).

## Cómo debe trabajar un agente en este repo

### Al leer / investigar

1. **Empieza por `obi/devdocs/`** para el tema que se te pida. Suele haber un
   documento que ya explica la parte a alto nivel; ahorra horas de lectura de C
   y de código Go.
2. **Después baja al código** en `obi/` (Go en `obi/pkg/`, `obi/cmd/`,
   `obi/internal/`; BPF/C en `obi/bpf/`).
3. **Nunca inventes rutas o funciones.** Si citas un símbolo, verifica que
   existe (`grep`/`Read`) — el código de OBI cambia rápido y las notas pueden
   quedar obsoletas.

### Al escribir notas

- **Idioma: español.** Todas las notas existentes están en español; mantener
  coherencia.
- **Formato de referencias al código:** `` `ruta/relativa/al/submódulo/archivo.ext:línea` ``.
  Las rutas se dan **relativas a `obi/`** (p.ej. `bpf/generictracer/k_tracer.c:815`),
  no como `obi/bpf/...`, porque las notas leen mejor así y el lector sabe que el
  código vive en el submódulo. Ver el walkthrough existente como referencia de
  estilo.
- **Cita líneas concretas** cuando hables de un sitio específico del código.
  Facilita volver a comprobarlo cuando el submódulo se actualice.
- **Estructura recomendada** (no obligatoria):
  - Alcance / qué cubre y qué NO
  - Piezas que intervienen (tabla si son varias)
  - Recorrido paso a paso del flujo
  - Estructuras de datos clave al final
  - Resumen visual (bloque de código en ASCII) si ayuda
- **Corrige ideas equivocadas al principio** ("dos ideas equivocadas frecuentes,
  corregidas de entrada") cuando el tema tenga trampas conocidas. El objetivo de
  las notas es que la próxima lectura no vuelva a caer en ellas.
- **No añadas comentarios ni explicaciones al código de `obi/`.** No es nuestro
  código y no lo mantenemos. Todo lo que aprendas va a las notas Markdown.

### Al crear un nuevo documento

- Nombre en kebab-case, descriptivo del tema: `<tema>-<subtema>-<forma>.md`
  (ej. `l7-context-propagation-blackbox-walkthrough.md`,
  `go-uprobes-injection-walkthrough.md`).
- Deja el archivo en la raíz del repo.
- Si la nota complementa/contrasta a otra existente, enlázala explícitamente.

### Lo que NO hay que hacer

- **No hacer commits dentro de `obi/`**. Es un submódulo upstream; cualquier
  cambio en él debería ir vía PR al repo original, no aquí.
- **No modificar el pin del submódulo** (`git -C obi checkout ...`) salvo que se
  pida explícitamente. Las notas se escriben contra un commit concreto de OBI.
- **No crear scripts, Makefiles, ni infraestructura de build.** Este repo es
  puramente documental.
- **No generar resúmenes automáticos de directorios** ("archivos en obi/pkg…").
  Las notas deben tener un objetivo concreto — un flujo, un mecanismo, una
  decisión — no ser un índice del código.

## Contexto útil para agentes

- OBI mezcla **Go (userspace)** y **C con CO-RE / libbpf (kernel eBPF)**. La
  frontera está en `obi/pkg/internal/ebpf/` (Go) ↔ `obi/bpf/` (C).
- Términos que aparecen mucho y conviene tener claros antes de escribir:
  *server span*, *client span*, *sockmap / sockhash*, *sk_msg*, *kprobe*,
  *uprobe*, *tail call*, *traceparent* (W3C), *tpinjector*, *generictracer*,
  *black-box vs uprobe path*.
- Runtimes con tratamiento especial: **Go (uprobes)**, **Node.js (event loop)**,
  **Python asyncio**, **Java virtual threads**, **nginx / puma** (correlación
  por conexión).
