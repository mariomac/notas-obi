# AGENTS.md — Guía para agentes en `.notas`

## Qué es esta carpeta

Esta carpeta **no contiene código propio**. Es un cuaderno personal de notas
sobre [OBI (OpenTelemetry eBPF Instrumentation)](https://github.com/open-telemetry/opentelemetry-ebpf-instrumentation),
un proyecto complejo que estoy tratando de entender a fondo. El objetivo es ir
acumulando en ella documentos en Markdown que expliquen partes del código, flujos,
decisiones de diseño, etc., a medida que los voy investigando.

`.notas/` vive **dentro del propio repositorio OBI**, como una carpeta más en su
raíz. No es un submódulo ni un repo vendorizado: es el repo `notas-obi` clonado
como subcarpeta. **No se mezcla con el Git de OBI** porque `.notas` está en el
`.gitignore_global` del usuario, así que OBI nunca la ve como cambios. Las notas
tienen su propio historial Git (dentro de `.notas/.git`).

## Estructura

- **El código objeto de análisis es el repositorio OBI que contiene esta carpeta**,
  es decir, el **directorio padre** de `.notas/`. Desde una nota, el código vive
  un nivel por encima (`../bpf/…`, `../pkg/…`, etc.). No se modifica desde aquí.
- `devdocs/` (en la raíz de OBI, o sea `../devdocs/` desde una nota) — documentación
  oficial del proyecto OBI. **Fuente primaria** para entender internals: pipeline,
  propagación de contexto, correlación trace-log, integración con runtimes (Go,
  Node, Python, Java…), formato de eventos BPF, etc. Consultar aquí antes de
  deducir cosas desde el código.
- `*.md` en `.notas/` — **las notas**. Cada archivo es un walkthrough o análisis
  sobre un tema concreto. Ejemplo actual:
  [l7-context-propagation-blackbox-walkthrough.md](./l7-context-propagation-blackbox-walkthrough.md)
  (paseo por el código de la ruta L7 black-box de propagación de contexto).

## Cómo debe trabajar un agente en este repo

### Al leer / investigar

1. **Empieza por `devdocs/`** (en la raíz de OBI, `../devdocs/` desde una nota)
   para el tema que se te pida. Suele haber un documento que ya explica la parte
   a alto nivel; ahorra horas de lectura de C y de código Go.
2. **Después baja al código** en la raíz de OBI (Go en `pkg/`, `cmd/`,
   `internal/`; BPF/C en `bpf/`). Desde `.notas/` esas rutas son `../pkg/…`,
   `../bpf/…`, etc.
3. **Nunca inventes rutas o funciones.** Si citas un símbolo, verifica que
   existe (`grep`/`Read`) — el código de OBI cambia rápido y las notas pueden
   quedar obsoletas.

### Al escribir notas

- **Idioma: español.** Todas las notas existentes están en español; mantener
  coherencia.
- **Formato de referencias al código:** `` `ruta/relativa/a/la/raíz/de/OBI/archivo.ext:línea` ``.
  Las rutas se dan **relativas a la raíz del repositorio OBI** (p.ej.
  `bpf/generictracer/k_tracer.c:815`), **sin** el prefijo `../` y **sin** `obi/`,
  porque las notas leen mejor así y esa es la ruta canónica dentro del proyecto.
  (Para *abrir* el archivo desde `.notas/` recuerda anteponer `../`; en el texto
  de la nota no se escribe.) Ver el walkthrough existente como referencia de estilo.
- **Cita líneas concretas** cuando hables de un sitio específico del código.
  Facilita volver a comprobarlo cuando el código de OBI se actualice.
- **Estructura recomendada** (no obligatoria):
  - Alcance / qué cubre y qué NO
  - Piezas que intervienen (tabla si son varias)
  - Recorrido paso a paso del flujo
  - Estructuras de datos clave al final
  - Resumen visual (bloque de código en ASCII) si ayuda
- **Corrige ideas equivocadas al principio** ("dos ideas equivocadas frecuentes,
  corregidas de entrada") cuando el tema tenga trampas conocidas. El objetivo de
  las notas es que la próxima lectura no vuelva a caer en ellas.
- **No añadas comentarios ni explicaciones al código de OBI.** No es nuestro
  código y no lo mantenemos. Todo lo que aprendas va a las notas Markdown.

### Al crear un nuevo documento

- Nombre en kebab-case, descriptivo del tema: `<tema>-<subtema>-<forma>.md`
  (ej. `l7-context-propagation-blackbox-walkthrough.md`,
  `go-uprobes-injection-walkthrough.md`).
- Deja el archivo dentro de `.notas/`.
- Si la nota complementa/contrasta a otra existente, enlázala explícitamente.

### Lo que NO hay que hacer

- **No modificar el código de OBI** (el directorio padre de `.notas/`) como parte
  del trabajo de notas. Cualquier cambio en OBI debería ir vía PR al repo original.
  Recuerda: OBI **no ve** `.notas/` (está en `.gitignore_global`), pero sí ve
  cualquier edición que hagas en `../bpf/`, `../pkg/`, etc.
- **No hacer commits de código de OBI** desde una sesión de notas. Los commits de
  las notas se hacen dentro de `.notas/` (su propio repo Git).
- **No crear scripts, Makefiles, ni infraestructura de build** dentro de `.notas/`.
  Es puramente documental.
- **No generar resúmenes automáticos de directorios** ("archivos en pkg/…").
  Las notas deben tener un objetivo concreto — un flujo, un mecanismo, una
  decisión — no ser un índice del código.

## Contexto útil para agentes

- OBI mezcla **Go (userspace)** y **C con CO-RE / libbpf (kernel eBPF)**. La
  frontera está en `pkg/internal/ebpf/` (Go) ↔ `bpf/` (C).
- Términos que aparecen mucho y conviene tener claros antes de escribir:
  *server span*, *client span*, *sockmap / sockhash*, *sk_msg*, *kprobe*,
  *uprobe*, *tail call*, *traceparent* (W3C), *tpinjector*, *generictracer*,
  *black-box vs uprobe path*.
- Runtimes con tratamiento especial: **Go (uprobes)**, **Node.js (event loop)**,
  **Python asyncio**, **Java virtual threads**, **nginx / puma** (correlación
  por conexión).

## Estado actual

> **Para agentes:** léelo al empezar una sesión; actualiza el "foco / hilos
> abiertos" al terminar algo relevante. Solo estado accionable, no historial.

- **Commit de OBI de referencia:** las notas existentes se escribieron contra
  `c1ed7d9b` (2026-07-02). Ahora el código es el checkout vivo del repo OBI que
  contiene `.notas/` (ya no hay pin de submódulo), así que puede haber avanzado:
  **verificar los símbolos citados** (`grep`/`Read`) si el `HEAD` de OBI difiere.
- **Foco actual / hilos abiertos:** *(vacío — apuntar aquí el próximo tema en
  investigación y las dudas pendientes)*
