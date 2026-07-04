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

1) Aceder a Agent_entregable_final.ipynb en Google Colab
2) Se cargan las credenciales como Secrets de Colab:

    - OPEN_API_KEY
    - LANGFUSE_PK
    - LANGFUSE_SK

    Se activa el toggle "Notebook access" para que el notebook pueda leerlos. Caso contrario, los pedirá como input al ejecutar el código.
3) Run all. La primera celda se encargará de instalar las dependencias necesarias. 

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
Una vez corridas todas las celdas de setup, se ejecuta el agente con:

pythonREPO_URL = "https://github.com/facebook/create-react-app"
state = run_main_agent(repo_url=REPO_URL, supervision=False)

### Parámetros:
repo_url: URL de un repositorio público de GitHub. El agente lo clona (--depth 1, trae sólo el útlimo commit de cada archivo en el árbol) dentro del workspace.
repo_path: alternativa a repo_url si el repo ya está clonado localmente o en el workspace.
supervision: si es True, pide confirmación (s/n) antes de ejecutar tools destructivas (write_file, run_command). En False, corre de punta a punta sin pausas.


### Output de cada corrida:
ARCHITECTURE_REPORT.md dentro del repo clonado, con el reporte de arquitectura producido por el Implementer.
Traza en Langfuse (https://cloud.langfuse.com, proyecto asociado a tus keys) con un span por subagente — ahí se ven prompts, tool calls, tokens y latencia de la ejecución.
Actualización de project_memory.json con un resumen de la sesión y la arquitectura detectada del repo.
Resumen en consola al final, con archivos modificados, fuentes consultadas, cantidad de chunks de RAG usados y el veredicto del Reviewer (APROBADO/RECHAZADO).


### Consultar traza en Langfuse
cloud.langfuse.com → pestaña "Traces" → buscar la traza react-architecture-agent con la fecha/hora de la corrida en cuestión

### Notas
El notebook no usa frameworks de orquestación. El harness, el manejo de tools y la coordinación de subagentes están implementados a mano.
Si el modelo entra en un loop (repite la misma tool con los mismos argumentos), el agente lo detecta y lo notifica en el log ([LOOP DETECTADO]) 
antes de forzarlo a cambiar de estrategia.



