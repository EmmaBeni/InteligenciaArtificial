# Arquitectura del sistema

## 1. Agente principal (`run_agent`)

Es el punto de entrada conversacional. Corre en un loop de chat (`while True`
esperando `input()`), mantiene su propio historial de mensajes (`messages`)
y decide, turno a turno, si:

- responde directo en modo chat (`inner_loop_unificado(messages, supervision)`,
  sin subagentes ni guards), o
- delega el pedido al pipeline de subagentes (`subagent_pipeline(...)`) cuando
  el usuario escribe `/analyze`.

Antes de arrancar el chat, `init_repo()` clona (o reutiliza) el repositorio una
única vez. El agente principal crea y reutiliza durante toda la sesión:
- un `state` (`new_task_state`) compartido con los subagentes,
- un `trace` de Langfuse (`lf.trace(...)`) donde cuelgan los spans de cada
  subagente.

Comandos que expone: `/help`, `/analyze`, `/plan`, `/supervision`, `/config`,
`/status`, `/reset`, `/exit`.

## 2. Subagentes (orquestados por `subagent_pipeline`)

Cada subagente corre `inner_loop_unificado` con guards activados (porque se
le pasa `state`) y su propio `span` de Langfuse. `subagent_pipeline` los
ejecuta siempre en este orden: explorer → researcher → implementer → tester
→ reviewer.

| Subagente | Responsabilidad | Función |
|---|---|---|
| Explorer | Recorre el repo con `list_files`/`read_file`, identifica stack, estructura de carpetas y componentes clave. No modifica archivos. | `run_explorer` |
| Researcher | Consulta el RAG (`rag_search`) sobre buenas prácticas de arquitectura React y, si hace falta, `web_search`. Registra fuentes en `state["sources_consulted"]`. | `run_researcher` |
| Implementer | Con los hallazgos de explorer + researcher, redacta `ARCHITECTURE_REPORT.md` del repo analizado y lo guarda con `write_file`. | `run_implementer` |
| Tester | Verifica con `read_file` que el reporte exista y tenga contenido; no lo modifica. | `run_tester` |
| Reviewer | Revisa el trabajo de los cuatro anteriores y dictamina `APROBADO`/`RECHAZADO` con justificación. | `run_reviewer` |

No todos comparten las mismas tools en la práctica: explorer/tester solo
leen, researcher agrega RAG/web, implementer es el único que escribe el
reporte. Todos pasan igual por `validate_tool_call` (políticas del YAML).

## 3. Estado compartido (`task_state`, ver `new_task_state`)

Vive en memoria durante la sesión y se pasa por referencia a cada subagente:

- `original_request`: pedido original del usuario.
- `repo_path`: repo sobre el que se está trabajando.
- `progress`: lista de eventos (`log_progress`), incluye inicio/fin de cada
  subagente y detecciones de loop.
- `subagent_results`: output de texto de cada subagente (explorer, researcher,
  implementer, tester, reviewer).
- `sources_consulted`: qué se consultó y de dónde (RAG, web_search),
  diferenciado de lo que viene del repo (tools `read_file`/`list_files`,
  visible en `progress`) o de la memoria persistente (aviso `[MEMORIA]`).
- `rag_chunks_used`: fragmentos concretos recuperados del RAG.
- `files_modified`: archivos escritos (ej. `ARCHITECTURE_REPORT.md`).
- `observations`: notas relevantes.
- `tool_call_log`: historial de llamadas a tools, insumo de `detect_loop`.

## 4. Memoria persistente entre sesiones (`project_memory.json`)

Se carga con `load_memory()` al iniciar y se actualiza con `update_memory()`
al terminar cada corrida del pipeline: guarda un resumen por sesión
(`sessions`) y la arquitectura detectada por repo (`architecture`), para que
`subagent_pipeline` avise `[MEMORIA] Ya analicé este repo antes` si vuelve a
verlo.

## 5. RAG

Documentación de React descargada (`REACT_DOCS_SOURCES`) → chunking por
palabras con overlap (`chunk_text`) → embeddings (`text-embedding-3-small`,
`get_embedding`) → almacenamiento vectorial persistente en ChromaDB
(`chroma_client.PersistentClient`, colección `react_docs`). `rag_search`
devuelve texto, título, url y score por chunk recuperado. El researcher
consulta el RAG primero y usa `web_search` como fallback/complemento.

## 6. Config y guardrails (`agent.config.yaml`, `guardrails.json`)

`validate_tool_call` corre en cada llamada a tool y valida, en este orden:
guardrails de paths (`blocked_paths`, `allowed_directories`), políticas de
lectura/escritura del YAML (`permissions.read.deny` / `write.deny`) y
políticas de comandos (`commands.deny` bloquea, `commands.require_approval`
pide confirmación por `input()`).

## 7. Manejo de contexto y loops

`inner_loop_unificado` resume el historial (`summarize_history`) cuando
supera `MAX_CONTEXT_TOKENS`, solo en modo subagente (`state is not None`).
`detect_loop` compara la clave `(subagente, tool, args)` contra
`state["tool_call_log"]`; si se repite `threshold` veces, corta la tool call,
lo deja registrado en `progress` como `LOOP DETECTADO ... — cambiando
estrategia` y el subagente sigue sin ejecutar esa acción de nuevo.

## 8. Observabilidad (Langfuse)

Un `trace` por sesión de chat (`react-architecture-agent-chat`), un `span`
por subagente dentro de `subagent_pipeline`, y una `generation` por llamada
al LLM dentro de `inner_loop_unificado` (prompt, modelo, tokens, latencia)
cuando se le pasa el `span` correspondiente.

## 9. Plugins (extra opcional)

`TOOL_REGISTRY` + `@register_tool(...)` permiten agregar tools nuevas
(ver `find_todos` como ejemplo) sin tocar `execute_tool` ni
`inner_loop_unificado`: alcanza con decorar la función y llamar a
`refresh_tools()` para reconstruir `TOOLS_SCHEMA`/`TOOLS_MAP`/`DESTRUCTIVE_TOOLS`.
