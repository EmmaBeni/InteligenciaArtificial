# Arquitectura del sistema

## Agente principal

`run_main_agent(repo_url, repo_path, supervision)` es el punto de entrada y el coordinador de toda la ejecución. 

Sus responsabilidades:
- **Prepara el entorno**: clona el repositorio a analizar (o reutiliza uno ya clonado) y arma el `task_state` inicial con el pedido del usuario.
- **Consulta la memoria persistente**: si el repo ya fue analizado en una sesión anterior, avisa y reutiliza ese contexto (`PROJECT_MEMORY`).
- **Orquesta los subagentes en orden fijo**: Explorer → Researcher → Implementer → Tester → Reviewer. No decide dinámicamente el orden ni salta pasos — es una secuencia lineal donde cada subagente recibe el `state` ya enriquecido por los anteriores.
- **Abre y cierra la traza de Langfuse**: crea un `trace` al inicio y un `span` por subagente, para poder observar la ejecución después.
- **Actualiza la memoria persistente** al terminar (`update_memory`) y **produce el resumen final** en consola (archivos modificados, fuentes consultadas, cantidad de chunks de RAG usados, veredicto del Reviewer).
- **Maneja errores globales**: si algo revienta, lo registra en el progreso y en la traza antes de relanzar la excepción, y asegura el `flush()` de Langfuse en un `finally`.

El agente principal **no ejecuta tools de análisis él mismo** — delega todo el trabajo de exploración, investigación e implementación en los subagentes; su rol es puramente de coordinación y registro.

## Subagentes

Cada subagente corre sobre el mismo mecanismo interno (`inner_loop_with_guards`): recibe un objetivo puntual, puede invocar tools (`list_files`, `read_file`, `write_file`, `run_command`, `web_search`, búsqueda en RAG), y su ejecución está sujeta a los guardrails de permisos y a la detección de loops. Lo que diferencia a cada uno es el **objetivo que se le da** y qué hace con el resultado.

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