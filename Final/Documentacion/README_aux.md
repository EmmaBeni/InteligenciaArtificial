A continuación se presenta una explicación detallada del código celda por celda de [Agente_final.ipynb](file:///C:/Users/laris/Downloads/InteligenciaArtificial/Final/Agente_final.ipynb).

**Celda 1**:
Instalación de las dependencias requeridas en el entorno (generalmente Google Colab o Jupyter): `openai` para los modelos GPT, `chromadb` como base de datos vectorial persistente, `langfuse` (versión < 3 para soporte de `.trace()`), `pyyaml`, `tiktoken` y `requests`.

**Celda 2**:
Importación de módulos estándar (`os`, `json`, `subprocess`, `requests`, `yaml`, `re`, `hashlib`), configuración del cliente `OpenAI` (leyendo las credenciales desde `userdata` de Colab o por input por consola si no están cargadas), definición de los modelos a utilizar (`MODEL = "gpt-5-nano"` y `EMBED_MODEL = "text-embedding-3-small"`) y establecimiento del directorio de trabajo (`/content/workspace`).

**Celda 3**:
Inicialización de la conexión con Langfuse Cloud mediante `Langfuse(public_key, secret_key, host)`. Realiza un chequeo de autenticación (`lf.auth_check()`) para asegurar el correcto flujo de observabilidad y trazabilidad de la ejecución del agente.

**Celda 4**:
Definición de las fuentes de documentación de React (`REACT_DOCS_SOURCES`) descargadas directamente desde los repositorios de GitHub en formato `.md` crudo. Se define la función `fetch_doc(url)` para realizar la petición HTTP y descargar la documentación en memoria.

**Celda 5**:
Definición de la función de fraccionamiento de texto (`chunk_text`) que divide el texto markdown en bloques de palabras (tamaño por defecto de 500 palabras y solapamiento de 50 palabras) y genera un hash MD5 único como identificador para cada fragmento. Asimismo, se define la función `get_embedding(text)` que realiza la llamada a la API de OpenAI para obtener la representación vectorial de cada bloque.

**Celda 6**:
Inicialización de la base de datos vectorial persistente **ChromaDB** (`chromadb.PersistentClient`) en el workspace (`/content/workspace/chroma_db`). Crea la colección `"react_docs"`. Si la colección está vacía (`collection.count() == 0`), indexa los bloques de documentación de React generando y subiendo sus embeddings. Si la colección ya existe, la reutiliza evitando costos adicionales de API. También define la función `rag_search(query, n_results)` para recuperar de manera semántica los fragmentos más relevantes.

**Celda 7**:
Creación física del archivo de configuración `agent.config.yaml` en el workspace. Este archivo define la estructura de permisos y políticas aplicables al agente: rutas de lectura denegadas (archivos secretos, claves, etc.), rutas de escritura denegadas (carpetas de configuración o control de versiones) y comandos del sistema restringidos o que requieren supervisión del usuario.

**Celda 8**:
Manejo de la memoria del agente. Se implementa el estado temporal de la tarea actual (`new_task_state`) en memoria para compartir el progreso, archivos modificados y chunks consultados entre los subagentes. También se define la memoria persistente del proyecto (`load_memory()`, `save_memory()`, `update_memory()`) en el archivo `project_memory.json` para dar continuidad y recordar qué repositorios ya fueron analizados en ejecuciones previas del agente.

**Celda 9**:
Control de contexto y loops de ejecución.
- `count_tokens()`: Mide el consumo de tokens en el historial.
- `summarize_history()`: Resume el contexto intermedio del historial de chat cuando excede el límite (`MAX_CONTEXT_TOKENS`) para evitar overflow en el modelo.
- `detect_loop()` y `make_call_key()`: Implementan la detección de bucles infinitos basándose en una clave compuesta que incluye el subagente (`subagent_name:tool_name:args`). Si se repite una acción la cantidad de veces configurada en `threshold`, el agente corta la llamada a la herramienta y busca una estrategia alternativa.
- `inner_loop_unificado()`: Orquesta el loop principal de interacción LLM-herramientas. Opera en **Modo Simple** (chat directo, sin guards ni resúmenes) y en **Modo con Guards** (para subagentes, aplicando la seguridad y el registro detallado en el estado y Langfuse).

**Celda 10**:
Definición e inicialización de la lógica para los subagentes especializados mediante `_run_subagent()`. Cada subagente se configura con un prompt de sistema, un objetivo concreto y un span de Langfuse:
- `run_explorer()`: Analiza la estructura del proyecto y su stack sin modificar archivos.
- `run_researcher()`: Investiga vulnerabilidades, obsolescencias y dependencias consultando el RAG y web_search.
- `run_implementer()`: Redacta y escribe el reporte `ARCHITECTURE_REPORT.md` en el repositorio.
- `run_tester()`: Comprueba la existencia y validez del reporte generado.
- `run_reviewer()`: Dictamina si el reporte cumple con la solicitud original (APROBADO/RECHAZADO).

**Celda 11**:
Implementación de `init_repo()`, que se encarga de clonar (o reutilizar) el repositorio git de interés, y del pipeline de subagentes (`subagent_pipeline()`), que coordina la ejecución secuencial de Explorer → Researcher → Implementer → Tester → Reviewer. Al finalizar la corrida exitosa, actualiza la memoria persistente.

**Celda 12**:
Sistema de registro de herramientas en runtime (Plugins). Implementa el decorador `@register_tool` para registrar dinámicamente herramientas en el objeto `TOOL_REGISTRY` y define las herramientas base que el agente puede usar:
- `read_file`: Lee archivos.
- `write_file`: Escribe o sobrescribe archivos (marcada como destructiva).
- `run_command`: Ejecuta comandos de consola (marcada como destructiva).
- `list_files`: Lista contenidos de directorios.
- `web_search`: Realiza búsquedas mediante el modelo LLM.

La función `refresh_tools()` reconstruye el esquema de herramientas (`TOOLS_SCHEMA`, `TOOLS_MAP` y `DESTRUCTIVE_TOOLS`) de forma automática.

![img.png](../../img.png)
![img_1.png](../../img_1.png)

**Celda 13**:
Configuración de la política de comandos seguros. Define `ALLOWED_COMMANDS` (comandos de consola no dañinos como `ls`, `dir`, `pwd`, `grep`, `echo`) e implementa la función `is_safe_command(command)`. Si el agente intenta ejecutar un comando que coincide con esta lista y no tiene operadores de encadenamiento de consola, se le permite correr sin necesidad de aprobación humana, mejorando la autonomía del agente para consultas del sistema básicas.

**Celda 14**:
Generación del archivo de seguridad `guardrails.json` que contiene restricciones críticas a nivel del sistema (directorios permitidos, rutas bloqueadas absolutas como `/etc` y comandos completamente prohibidos).

**Celda 15**:
Función `load_guardrails()` para cargar en memoria las restricciones duras del sistema definidas en `guardrails.json`.

**Celda 16**:
Carga del YAML y validación de las herramientas mediante `validate_tool_call()`. Esta función es el guardián del sistema: intercepta cada llamada de herramienta propuesta por el modelo y verifica que no acceda a directorios restringidos por los guardrails locales, que cumpla las políticas de lectura/escritura del YAML, y que no use comandos peligrosos bloqueados.

**Celda 17**:
Generación programmaticamente del archivo explicativo `ARQUITECTURA.md` directamente en el workspace del proyecto. Este archivo contiene la especificación estática del diseño del sistema, roles de agentes y flujo de datos.

**Celda 18**:
Capa de ejecución de herramientas y seguridad interactiva.
- `execute_tool()`: Valida guardrails, comprueba supervisión interactiva para herramientas destructivas, y atrapa excepciones para que no se propague un crash en la API.
- `sanitize_messages()`: Red de seguridad extra que elimina pares incompletos de `assistant(tool_calls)` y `tool(response)` en el historial del chat para evitar fallos de petición HTTP.
- `plan_mode_flow()`: Si está activo, genera una propuesta de plan en pasos detallados mediante el LLM y pide aprobación al usuario por consola (`s/n/modificar`).

**Celda 19**:
Definición de `run_agent()`. Implementa la interfaz conversacional de consola CLI con el menú interactivo, la inicialización del repositorio remoto (`REPO_URL`) y el control dinámico mediante comandos de usuario como `/exit`, `/reset`, `/plan`, `/supervision`, `/config` y `/analyze`.

**Celda 20**:
Funciones de pruebas y utilidades para simular entornos restringidos en testing (ejemplo: creación de directorios bloqueados y archivos con permisos completos para evaluar si el agente respeta los límites de chmod).

**Celda 21**:
Demostración práctica del sistema de plugins. Registra la herramienta `find_todos` decorada con `@register_tool` y ejecuta `refresh_tools()`, haciéndola disponible en el esquema y mapa del agente en tiempo de ejecución sin haber alterado el núcleo de su código.

**Celda 22**:
Punto de ejecución real del agente interactivo llamando a `run_agent()`.

**Celda 23**:
Verificación y lectura en consola de los registros históricos de ejecución guardados en `project_memory.json`.

---

### Detección de bucles resuelta (Disclaimer actualizado) 
En versiones previas del agente, la clave utilizada para registrar llamadas a herramientas era global y no distinguía al emisor. Dado que el objeto `state` es compartido por referencia entre todos los subagentes, la ejecución sucesiva de herramientas legítimas similares (ej. que el Explorer y el Researcher listen o lean archivos en secuencia) generaba falsos positivos bloqueando al agente por "bucle detectado".

En esta versión final, la clave generada por `make_call_key` incorpora el nombre del subagente:
```python
def make_call_key(subagent_name: str, tool_name: str, args: dict) -> str:
    return f"{subagent_name}:{tool_name}:{json.dumps(args, sort_keys=True)}"
```
De este modo, aunque continúen operando sobre el historial unificado `state["tool_call_log"]`, el detector evalúa individualmente el comportamiento de cada subagente en su fase, resolviendo los falsos positivos y garantizando fluidez en la orquestación.
