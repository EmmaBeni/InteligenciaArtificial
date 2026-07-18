# Coding Agent - Final
## Análisis de Repositorios React

Este sistema multi-agente fue construído sobre el coding agent de clase para analizar repositorios de React.
Aquí se combinan un agente principal, 5 subagentes especializados (Explorer, Researcher, Implementer, Tester, Reviewer), RAG sobre documentación de React, memoria persistente por proyecto, 
políticas de permisos configurables y observabilidad con Langfuse. 

## Requisitos

* Cuenta de Google (para poder usarlo en Google Colab)
* API key de OpenAI con acceso a gpt-5-nano y text-embedding-3-small
* Cuenta de Langfuse Cloud con public y private key
* Acceso al repositorio y documentación de React (lo busca el RAG)

Sin embargo, todo corre en Google Colab por lo que no es necesario hacer isntalaciones localmente. 

## Instalación

1) Acceder a [Agente_final.ipynb](file:///C:/Users/laris/Downloads/InteligenciaArtificial/Final/Agente_final.ipynb) o a la versión con perfilado de herramientas programático [Agente_final_feedback.ipynb](file:///C:/Users/laris/Downloads/InteligenciaArtificial/Final/Agente_final_feedback.ipynb) en Google Colab (o entorno local Jupyter).
2) Se cargan las credenciales como Secrets de Colab (o variables de entorno si se corre en local):

    - `OPENAI_API_KEY`
    - `LANGFUSE_PK`
    - `LANGFUSE_SK`

    Si usas Colab, activa el toggle "Notebook access" para que el notebook pueda leerlos. Caso contrario, se solicitarán por consola al ejecutar.
3) Run all. La primera celda se encargará de instalar las dependencias necesarias (OpenAI, ChromaDB, Langfuse, PyYAML, Tiktoken, etc.). 

## Configuración

### Worksapce
El agente trabaja en /content/workspace (variable WORKSPACE) donde clona el repositorio a analizar y guarda archivos de configuración y memoria. 

### Políticas del agente (agent.config.yaml)
Se generan automáticamente al correr el notebook, siendo los valores por defecto los siguientes:

workspace: /content/workspace

````
permissions:
  read:
    deny:
      - ".env"
      - "**/*.pem"
      - "**/*.key"
      - "secrets/**"
  write:
    deny:
      - ".github/**"
      - "package-lock.json"
      - "yarn.lock"
commands:
  deny:
    - "rm -rf"
    - "git push"
    - "sudo"
    - "chmod"
    - "curl | bash"
    - "wget | bash"
  require_approval:
    - "npm install"
    - "pip install"
    - "git commit" 
````
Si se quisiera alterar lo que el agetne puede leer, escribir o ejecutar se puede cambiar la configuración
antes de correrla en la celda correspondiente (load_config).
Para los comadnos de tipo require_approval pide confirmación por consola antes de su ejecución (s/n).

### RAG
La base de conocimiento se construye indexando documentación oficial de React (páginas de react.dev) definida en REACT_DOCS_SOURCES. 
Para analizar un ecosistema distinto de React, hay que:

* Cambiar las URLs en REACT_DOCS_SOURCES por documentación del framework/lenguaje que corresponda
* Volver a correr las celdas de chunking e indexado (generan embeddings nuevos con text-embedding-3-small)


### Memoria persistente
Se almacena en project_memory.json dentro del workspace. Al correr el notebook varias veces sobre el mismo repositorio, 
el agente avisará que ya fue analizado antes y reutilizará ese contexto. Es necesario borrar el archivo para resetear el contexto.

### Ejecución
Una vez corridas todas las celdas de setup, se inicia el bucle de chat interactivo del agente principal ejecutando:

```python
run_agent()
```

Esto arrancará una interfaz interactiva de consola (CLI) donde el agente estará esperando instrucciones del usuario.

### Comandos de control disponibles en la consola:
Al ingresar un prompt, puedes utilizar comandos especiales para configurar o arrancar la ejecución:
- `/help` o `/commands`: Muestra los comandos disponibles y la ayuda.
- `/analyze`: Ejecuta el pipeline completo de subagentes (Explorer → Researcher → Implementer → Tester → Reviewer) para analizar el repositorio.
- `/plan`: Activa o desactiva la generación interactiva del plan de pasos a seguir (`ON` / `OFF`) antes de que el agente ejecute herramientas.
- `/supervision`: Activa o desactiva la supervisión humana (`ON` / `OFF`) sobre las herramientas destructivas (`write_file`, `run_command`).
- `/config` o `/status`: Muestra el estado de la configuración y variables activas.
- `/reset`: Limpia el historial de mensajes de la conversación.
- `/exit`: Finaliza la sesión y sale de la consola interactiva.

### Parámetros configurables en el código:
- **`REPO_URL`**: URL del repositorio público en GitHub a clonar y analizar (por defecto configurado como `"https://github.com/emmabeni27/pickndrive-ia"`).
- **`supervision`**: Controla si el agente requiere confirmación interactiva (`s/n`) para ejecutar operaciones destructivas. Se puede cambiar dinámicamente mediante el comando `/supervision`.
- **`plan_mode`**: Si está activo, el agente presentará un plan detallado antes de proceder a usar herramientas y esperará la confirmación del usuario (`s/n/modificar`).

### Output de cada corrida de análisis:
- **`ARCHITECTURE_REPORT.md`**: Creado dentro de la carpeta del repositorio clonado, con el informe de arquitectura detallado redactado por el subagente Implementer.
- **Traza en Langfuse**: Visible en [cloud.langfuse.com](https://cloud.langfuse.com) bajo el nombre de traza `react-architecture-agent-chat` (contiene la observabilidad del chat global y spans detallados por subagente).
- **Actualización de `project_memory.json`**: Guarda el historial de resúmenes de las sesiones del repositorio trabajado y los datos de arquitectura.
- **Resumen en consola**: Al finalizar el análisis, el pipeline imprime el estado de la corrida, los archivos modificados y el veredicto del Reviewer (`APROBADO`/`RECHAZADO`).

### Consultar traza en Langfuse
Accede a cloud.langfuse.com → pestaña "Traces" → busca la traza `react-architecture-agent-chat` correspondiente a la corrida realizada.

### Notas
El notebook no usa frameworks de orquestación. El harness, el manejo de tools y la coordinación de subagentes están implementados a mano.
Si el modelo entra en un loop (repite la misma tool con los mismos argumentos), el agente lo detecta y lo notifica en el log ([LOOP DETECTADO]) 
antes de forzarlo a cambiar de estrategia.



