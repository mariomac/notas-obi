# Propagación de contexto L7 en black-box — paseo por el código (runtimes genéricos)

**Alcance:** la ruta black-box / kprobe para un **lenguaje genérico basado en
thread/fibra/corrutina** (sin uprobes de Go, sin TLS, ignorando la ruta L4 de opciones TCP).
Se recorre el ciclo de vida completo:

1. Se recibe una petición HTTP entrante que trae una cabecera `Traceparent`.
2. Se almacena su contexto de traza.
3. Cuando se hace una llamada HTTP saliente "dentro de ese contexto", se correlaciona el
   thread/fibra/corrutina de la entrante con el de la saliente.
4. Se inyecta el `Traceparent` (L7) en la petición saliente.

> Nota rápida sobre *por qué sockmap/sk_msg y no TC egress*: el `sk_msg` muta el buffer de
> `sendmsg` **antes** de que el kernel segmente el stream y asigne números de secuencia TCP,
> así que puede crecer el payload y dejar que el kernel genere un stream correcto. En TC
> egress el segmento ya tiene sus números de secuencia asignados, así que crecer el payload
> obligaría a reescribir `seq` en cada segmento de salida y traducir el `ack` del peer en
> ingress durante toda la vida de la conexión — un apaño frágil que se abandonó.

## Las 3 piezas que cooperan

En la ruta genérica no hay un único programa, hay **tres** programas BPF cooperando por
conexión:

| Programa | Hook | Rol |
|---|---|---|
| **kprobe recv** | `tcp_recvmsg` | Parsea la petición entrante (y su `Traceparent`) → crea el server span |
| **kprobe send** | `tcp_sendmsg` | Detecta la petición saliente → crea/reporta el client span, coordina vía `outgoing_trace_map` |
| **sk_msg** | sockhash `sock_dir` | **El único que puede crecer el paquete** e inyectar los bytes de la cabecera saliente |

Dos ideas equivocadas frecuentes, corregidas de entrada:

- **La inyección de la cabecera L7 la hace el `sk_msg` (tpinjector), no el kprobe.**
  En modo black-box el kprobe de `tcp_sendmsg` no puede crecer el buffer de envío (eso solo
  es posible con `bpf_probe_write_user` en el caso de uprobe de Go).
- **El almacén de correlación es `server_traces` (indexado por thread), no
  `incoming_trace_map`.** `incoming_trace_map` (clave `connection_info_t`) pertenece a la
  ruta L4 / opciones TCP.

---

## Paso 1 — Llega una petición HTTP con cabecera `Traceparent` (ingress)

1. `tcp_recvmsg` dispara el kprobe en
   `bpf/generictracer/k_tracer.c:815`.
2. El despacho de protocolo en
   `bpf/generictracer/protocol_handler.c:35` detecta HTTP
   (`is_http`) y hace `bpf_tail_call` a `k_tail_protocol_http`.
3. `__obi_continue_protocol_http_tp`
   (`bpf/generictracer/protocol_http.h:444`) escanea la cabecera.
   El scanner byte-a-byte es `bpf_strstr_tp_loop`
   (`bpf/common/trace_util.h:113`), disparado en
   `bpf/generictracer/protocol_http.h:488`. El match del literal
   `Traceparent: ` lo hace `is_traceparent`
   (`bpf/common/trace_util.h:75`).
4. Se decodifica a la struct en
   `bpf/generictracer/protocol_http.h:504-512`:

```c
decode_hex(tp_p->tp.trace_id, t_id, TRACE_ID_CHAR_LEN);
decode_hex((unsigned char *)&tp_p->tp.flags, f_id, FLAGS_CHAR_LEN);
if (meta && meta->type != EVENT_HTTP_CLIENT) {
    decode_hex(tp_p->tp.parent_id, s_id, SPAN_ID_CHAR_LEN);  // el span entrante pasa a ser NUESTRO parent_id
}
```

Heredamos el `trace_id` del caller, su `span_id` se convierte en nuestro `parent_id`, y
generamos un `span_id` propio para el server span (`urand_bytes`,
`bpf/generictracer/protocol_http.h:615`).

> Matiz: el escaneo completo de la cabecera solo corre si `capture_header_buffer` está
> activo; si no, el server span usa IDs correlacionados/generados. Pero cuando hay de verdad
> un `Traceparent` entrante (continuación de traza distribuida), sale de aquí.

---

## Paso 2 — Almacenamiento del contexto (¿dónde exactamente?)

**No** en `incoming_trace_map`. Ese mapa (clave `connection_info_t`) solo lo escribe la ruta
L4 de opciones TCP en
`bpf/tpinjector/tpinjector.c:507` y lo consume
`find_trace_for_server_request`.

Para L7, todo pasa por `server_or_client_trace`
(`bpf/common/trace_lifecycle.h:104`), invocado desde
`bpf/generictracer/protocol_http.h:527-543`. En la rama
`EVENT_HTTP_REQUEST` (server) escribe en **cuatro** almacenes, pero el que importa para la
correlación es:

```c
// trace_lifecycle.h:145 — LA CLAVE de la correlación genérica
bpf_map_update_elem(&server_traces, &t_key, tp_p, BPF_ANY);
```

Los cuatro almacenes:

- **`server_traces`** — clave `trace_key_t` (= **thread**). El vínculo "este thread está
  sirviendo esta traza". ← el importante.
- `server_traces_aux` — clave `connection_info_part_t` (puerto efímero + addr). Para
  runtimes que correlacionan por conexión (nginx, puma).
- `trace_map` — clave `{conn, type}`
  (`bpf/common/tracing.h:61`). Para la ruta de correlación por conexión.
- `obi_ctx` (indexado por `pid_tgid`) — superficie externa para trace-log correlation.

---

## Paso 3 — Correlación entrante → saliente (identidad de thread)

La clave de correlación **no** es la conexión ni el `pid_tgid` crudo, sino la **identidad de
thread namespaced**, `trace_key_t`
(`bpf/common/trace_key.h:10-14`):

```c
typedef struct trace_key {
    u64 extra_id;    // namespace de pids / desambiguador de runtime
    pid_key_t p_key; // {tid, pid, ns} tal y como se ve DENTRO del contenedor
    u8 _pad[4];
} trace_key_t;
```

Se construye desde el `task_struct` de la tarea que corre ahora mismo (`task_tid`, no
`bpf_get_current_pid_tgid`, para que funcione dentro de contenedores) en
`trace_key_from_pid_tid`
(`bpf/common/trace_parent.h:33`).

Cuando el mismo thread hace la **llamada saliente**, el kprobe de `tcp_sendmsg` la detecta
como `EVENT_HTTP_CLIENT` y llama `find_trace_for_client_request`
(`bpf/generictracer/protocol_http.h:623`) → reconstruye el mismo
`trace_key_t` y lo busca en `server_traces`. El corazón está en
`find_parent_process_trace`
(`bpf/common/trace_parent.h:114`):

```c
tp_info_pid_t *server_tp = bpf_map_lookup_elem(&server_traces, t_key);  // ¿este thread tiene traza activa?
if (server_tp) return server_tp;
// si no, el worker != accept thread: sube por la cadena de clones
const pid_key_t *p_tid = bpf_map_lookup_elem(&clone_map, &t_key->p_key); // hasta 5 niveles
t_key->p_key = *p_tid;
```

Al acertar, el hijo hereda
(`bpf/common/trace_parent.h:358-359`):

```c
__builtin_memcpy(tp->trace_id, server_tp->tp.trace_id, ...);  // misma traza
__builtin_memcpy(tp->parent_id, server_tp->tp.span_id, ...);  // parent = span del server
```

Dos sutilezas importantes del modelo "thread/fibra/corrutina":

- **Thread pool / worker != accept thread:** por eso existe `clone_map`
  (`bpf/maps/clone_map.h`, poblado en el kretprobe de clone en
  `bpf/generictracer/k_tracer.c:1356`) que puentea child→parent thread
  hasta 5 niveles.
- **Fibras/corrutinas != thread del SO:** cuando el modelo de concurrencia no es 1 request =
  1 thread, `find_parent_trace`
  (`bpf/common/trace_parent.h:276`) prueba lookups específicos por
  runtime **antes** del thread-walk genérico:
  - Node.js (event loop): `nodejs_fd_map` → fd del servidor
  - Python asyncio: `python_thread_state` / `python_context_task` (la corrutina activa)
  - Java virtual threads: `java_tasks` (análogo a `clone_map`) + remapeo del carrier tid
  - nginx / puma (Ruby): correlación por conexión vía `server_traces_aux`

---

## Paso 4 — Inyección del `Traceparent` en la saliente (L7)

La inyección de bytes la hace el **`sk_msg` del tpinjector**, no el kprobe. En black-box el
kprobe de `tcp_sendmsg` no puede crecer el buffer (eso solo es `bpf_probe_write_user` en el
caso de uprobe de Go). El reparto de trabajo:

**El `sk_msg` (`obi_packet_extender`,
`bpf/tpinjector/tpinjector.c:896`) hace el trabajo pesado**, y corre en
contexto del thread que envía (durante el `sendmsg`), así que la correlación por thread
también le funciona a él:

1. Detecta HTTP (`protocol_detector`) y hace **su propia búsqueda de padre** en
   `init_tp_ctx_parent_tp` (`bpf/tpinjector/tpinjector.c:352`):
   ```c
   t_ctx->has_parent_tp = find_parent_trace_for_client_request(
       &t_ctx->p_conn, ..., k_lw_thread_none, &t_ctx->parent_tp);  // mismo server_traces indexado por thread
   ```
   (Pasa `k_lw_thread_none` → salta el lookup de goroutines de Go, coherente con no-Go.)
2. `create_trace_info` (`bpf/tpinjector/tpinjector.c:360`) decide: si hay
   padre, hereda `trace_id` + `parent_id`; si no, mint nuevo `trace_id`. Siempre genera un
   `span_id` fresco.
3. El crecimiento del paquete ocurre en `extend_and_write_tp`
   (`bpf/tpinjector/tpinjector.c:748`):
   ```c
   bpf_msg_push_data(msg, offset, TP_SIZE, 0);   // crece el buffer ANTES de la numeración de secuencia TCP
   ...
   make_tp_string_skb(ptr, tp, msg->data_end);   // escribe "Traceparent: 00-<trace>-<span>-0X\r\n"
   ```
   El offset es justo tras la primera línea de la request
   (`bpf/tpinjector/tpinjector.c:791`).
4. Escribe el resultado en `outgoing_trace_map` (clave `egress_key_t`) con `written=1`.

**El kprobe de `tcp_sendmsg` (después)** ve `written=1` en `outgoing_trace_map`
(`bpf/generictracer/protocol_http.h:588-598`), reutiliza ese contexto
para emitir el **client span** a userspace, y borra la entrada. Este es el mecanismo de
exclusión mutua del flag `written`: el `sk_msg` lo pone a 1 al inyectar, y el kprobe lo
comprueba para no volver a inyectar/duplicar por la misma conexión (solo un método de
inyección por conexión).

Para que el `sk_msg` corra sobre ese socket, el socket tuvo que entrar en el sockhash
`sock_dir` — vía `sockops` en connect (`bpf/tpinjector/tpinjector.c:394`,
la llamada a `bpf_sock_hash_update`) o vía el iterador de arranque `sock_iter.c` para
conexiones preexistentes.

---

## Resumen de principio a fin

```
INGRESS (thread T, conexión C_in)
  kprobe tcp_recvmsg → parsea "Traceparent" (trace_util.h)
    → hereda trace_id, parent_id = span entrante; genera span propio
    → server_or_client_trace: server_traces[trace_key_t{T}] = ctx   ← almacén indexado por THREAD
                              (+ server_traces_aux[C_in], trace_map, obi_ctx)

  [el mismo thread T ejecuta la lógica de negocio → llama a un servicio downstream]

EGRESS (thread T, conexión C_out)
  sk_msg tpinjector (durante sendmsg, ANTES de segmentar):
    → find_parent_trace_for_client_request → server_traces[trace_key_t{T}]  ← MISMA clave
       (con clone_map / mapas por-runtime si worker != accept, o fibra/corrutina)
    → create_trace_info: trace_id heredado, parent_id = span del server, span nuevo
    → bpf_msg_push_data + make_tp_string_skb → inyecta la cabecera  ← INYECCIÓN L7
    → outgoing_trace_map[C_out].written = 1

  kprobe tcp_sendmsg (después): ve written=1, emite el client span, limpia
```

### Estructuras de datos clave

- `tp_info_t` / `tp_info_pid_t` — `bpf/common/tp_info.h:16-32`
  (`trace_id`, `span_id`, `parent_id`, `ts`, `flags`; más `pid`, `valid`, `written`,
  `req_type`).
- `trace_key_t` (clave de thread) — `bpf/common/trace_key.h`;
  mapa `server_traces` — `bpf/maps/server_traces.h`.
- `trace_map_key_t` (clave de conexión) — `bpf/common/trace_map_key.h`;
  `trace_map` — `bpf/maps/trace_map.h`.
- `outgoing_trace_map` (clave de egress) — `bpf/maps/outgoing_trace_map.h`.
- `incoming_trace_map` (solo L4) — `bpf/maps/incoming_trace_map.h`.
- `clone_map` (child→parent thread) — `bpf/maps/clone_map.h`.
