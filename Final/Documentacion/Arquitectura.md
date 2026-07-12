# Arquitectura del sistema

## Agente principal

`run_agent()` es el punto de entrada principal del sistema, implementado como un chat loop interactivo por consola. 

Sus responsabilidades:
- **Prepara el entorno**: Al iniciar la sesión, clona el repositorio a analizar (o reutiliza uno ya clonado) a través de `init_repo()`.
- **Mapeo de comandos interactivos**: Procesa comandos de control y configuración tales como `/help`, `/commands`, `/analyze`, `/plan`, `/supervision`, `/config`, `/status`, `/reset` y `/exit`.
- **Paso a paso guiado (Plan Mode)**: En caso de estar activo (`plan_mode=True`), genera y propone una lista detallada de pasos a seguir para el prompt ingresado antes de ejecutar herramientas, requiriendo confirmación del usuario (`plan_mode_flow()`).
- **Enrutamiento selectivo**: 
  - Si el usuario ejecuta `/analyze`, el prompt se delega al pipeline completo de subagentes (`subagent_pipeline`).
  - En caso contrario, se responde de manera directa mediante el chat en modo simple usando el loop unificado.
- **Creación de Trazas e Historial**: Crea y mantiene el `state` (`new_task_state`) compartido y la traza de Langfuse (`react-architecture-agent-chat`) durante toda la sesión interactiva.
- **Orquestación de Subagentes (`subagent_pipeline`)**: Si se activa el análisis, orquesta a los subagentes en su orden lineal preestablecido (Explorer → Researcher → Implementer → Tester → Reviewer).
- **Actualización de Memoria y Resumen**: Al finalizar la ejecución del pipeline, actualiza la memoria en disco (`update_memory()`) y genera el resumen con los veredictos en consola.

El agente principal coordina y expone la interfaz de usuario, delegando las tareas de exploración e implementación profundas al pipeline de subagentes.

## Subagentes

Cada subagente corre sobre el mismo mecanismo unificado de control (`inner_loop_unificado`): recibe su prompt de sistema y un objetivo puntual, e interactúa con las herramientas disponibles bajo supervisión y guardrails estrictos. Lo que diferencia a cada subagente es el **objetivo específico asignado** y los datos que consolida en el estado compartido.

| Subagente | Rol | Qué recibe | Qué produce |
|---|---|---|---|
| **Explorer** | Entiende el repositorio: estructura de carpetas, stack tecnológico, dependencias declaradas, convenciones. Explora con `list_files`/`read_file`, nunca escribe. | El `repo_path` y el `task_state` vacío | `state["subagent_results"]["explorer"]`: resumen de la arquitectura detectada |
| **Researcher** | Busca contexto adicional sobre el stack detectado: primero en el RAG (documentación indexada), y si no hay evidencia suficiente, en la web. | El resultado del Explorer | `state["subagent_results"]["researcher"]`, más entradas en `state["sources_consulted"]` y `state["rag_chunks_used"]` |
| **Implementer** | Redacta el reporte de arquitectura (`ARCHITECTURE_REPORT.md`) a partir de lo que encontraron Explorer y Researcher. Es el único que escribe archivos. | Resultados de Explorer + Researcher | `state["files_modified"]` actualizado, contenido del reporte |
| **Tester** | Valida el resultado con checks definidos (por ejemplo, que el archivo del reporte exista y tenga contenido, o correr algún comando de verificación). | El archivo generado por el Implementer | `state["subagent_results"]["tester"]`: resultado de la validación |
| **Reviewer** | Revisa el reporte final contra el pedido original (`state["original_request"]`) y da un veredicto. | Todo el `state` acumulado | `state["subagent_results"]["reviewer"]`: veredicto (aprobado/rechazado) y observaciones |

Ningún subagente tiene acceso a exactamente las mismas tools ni a los mismos permisos — por ejemplo, el Explorer no puede escribir archivos aunque la tool `write_file` exista en el sistema, porque su objetivo se lo prohíbe explícitamente y los guardrails validan cada `tool_call` antes de ejecutarla.

## Estructura del estado compartido

Todos los subagentes reciben **el mismo objeto `state`** (no copias), creado una sola vez por `new_task_state()` al inicio de la tarea. Esto es lo que permite que cada subagente vea el trabajo de los anteriores sin tener que pasarse mensajes entre sí:

```python
state = {
    "original_request": str,       # el pedido original del usuario, nunca se pisa
    "repo_path": str,               # path local al repo clonado

    "progress": [str, ...],         # log legible de todo lo que hizo cada subagente,
                                     # usado para mostrarle al usuario qué pasó

    "subagent_results": {           # el resultado (texto) que devolvió cada subagente
        "explorer": str | None,
        "researcher": str | None,
        "implementer": str | None,
        "tester": str | None,
        "reviewer": str | None
    },

    "sources_consulted": [          # fuentes usadas por el Researcher, con su origen
        {"source": "rag" | "web", "title": str, "url": str, ...}, ...
    ],

    "files_modified": [str, ...],   # paths de archivos que el Implementer efectivamente tocó

    "observations": [str, ...],     # notas relevantes que cualquier subagente puede dejar

    "rag_chunks_used": [...],       # chunks concretos recuperados del RAG durante la tarea
    "tool_call_log": [str, ...]     # historial de llamadas (tool + args) usado exclusivamente
                                     # para detectar loops, no se le muestra al usuario
}
```

## Loop unificado e inner loop

El sistema unifica el comportamiento de ejecución mediante la función `inner_loop_unificado(messages, supervision, state=None, span=None, subagent_name="main")`:
- **Modo Simple** (cuando `state` es `None`): Se utiliza en el chat directo del agente principal. No realiza detección de loops ni resúmenes automáticos de contexto largo.
- **Modo con Guards** (cuando se provee `state`): Utilizado por los subagentes. Habilita la detección de loops mediante `detect_loop()`, la reducción de contexto largo (a través de `summarize_history()` al superar `MAX_CONTEXT_TOKENS`) y el logging de progreso en el `state` compartido. Si se pasa un `span`, genera la traza correspondiente en Langfuse.

### Detección de loops mejorada
El detector de loops (`detect_loop()`) compara la firma de la llamada actual contra `state["tool_call_log"]`. La clave compuesta se genera como `f"{subagent_name}:{tool_name}:{args_json}"`, lo que encapsula el historial por subagente y evita que llamadas repetidas legítimas entre distintos agentes (por ejemplo, `list_files` consecutivas por Explorer y Researcher) sean erróneamente interpretadas como loops infinitos.

Separado del `task_state` (que vive solo durante una ejecución) está `PROJECT_MEMORY`, cargado desde `project_memory.json` y persistido entre sesiones distintas del notebook:

```python
PROJECT_MEMORY = {
    "sessions": [...],       # resumen de cada corrida pasada (fecha, pedido, archivos tocados)
    "architecture": {        # arquitectura detectada, indexada por repo_path
        "<repo_path>": str
    }
}
```

La diferencia clave entre ambos: `task_state` es el estado de **una tarea puntual** y se descarta al terminar (salvo lo que `update_memory` decide volcar a `PROJECT_MEMORY`); `PROJECT_MEMORY` es lo único que sobrevive entre ejecuciones distintas del notebook, y es lo que le permite al agente decir "ya analicé este repo antes" en una corrida futura.