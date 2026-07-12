# Análisis de Resultados y Evidencia de Ejecución

Este documento recopila la evidencia de ejecución y el análisis de resultados obtenidos al correr el Coding Agent sobre el repositorio front-end [pickndrive-ia](https://github.com/emmabeni27/pickndrive-ia), correspondiente al caso de uso establecido. Se presentan tres tareas experimentales que demuestran la robustez, el uso de RAG, el mecanismo de memoria y el cumplimiento de las restricciones del sistema.

---

## Tarea 1: RAG + Caso de Uso Completo (Primera ejecución)

### Objetivo
Analizar la arquitectura completa de la base de código de `pickndrive-ia` y generar un informe detallado (`ARCHITECTURE_REPORT.md`) que describa el stack tecnológico, dependencias, convención de carpetas, riesgos arquitectónicos y comandos útiles. Esta prueba se ejecuta en una sesión limpia sin memoria previa del repositorio, forzando la ejecución lineal del pipeline completo de subagentes (`/analyze`).

### Evidencia del Output
Al ejecutar la tarea mediante el comando interactivo `/analyze`, se observa la siguiente secuencia de logs en consola:

```text
Prompt: /analyze

[PLAN] Generando plan...
Plan propuesto:
- Identificar la raíz y stack de pickndrive-ia usando list_files.
- Analizar dependencias clave inspeccionando package.json.
- Consultar en el RAG buenas prácticas de React 19 y enrutamiento.
- Generar el informe ARCHITECTURE_REPORT.md con recomendaciones y riesgos.
- Validar el informe con el Tester y someter a evaluación del Reviewer.

¿Aprobás este plan? (s/n/modificar): s

Agente: [MAIN] Pedido recibido: /analyze
[EXPLORER] iniciando
  [EXPLORER] iteración 1
[EXPLORER] tool: list_files ['.']
  [EXPLORER] iteración 2
[EXPLORER] tool: list_files ['pickndrive-ia']
  [EXPLORER] iteración 3
[EXPLORER] tool: read_file ['pickndrive-ia/package.json']
  [EXPLORER] iteración 4
[EXPLORER] tool: read_file ['pickndrive-ia/README.md']
...
  [EXPLORER] iteración 19
  [LOOP DETECTADO] 'read_file' repetido 3 veces sin avance.
[EXPLORER] LOOP DETECTADO en read_file — cambiando estrategia
  [EXPLORER] iteración 20
[EXPLORER] tool: list_files ['root']
...
[EXPLORER] terminado
[RESEARCHER] iniciando
  [RESEARCHER] iteración 1
  ...
[RESEARCHER] terminado
[IMPLEMENTER] iniciando
  [IMPLEMENTER] iteración 1
[IMPLEMENTER] tool: write_file ['/content/workspace/pickndrive-ia/ARCHITECTURE_REPORT.md']

[SUPERVISIÓN] El agente quiere ejecutar: write_file
¿Permitir? (s/n): s
  [IMPLEMENTER] iteración 2
[IMPLEMENTER] terminado
[TESTER] iniciando
  [TESTER] iteración 1
[TESTER] tool: read_file ['/content/workspace/pickndrive-ia/ARCHITECTURE_REPORT.md']
  [TESTER] iteración 2
[TESTER] terminado
[REVIEWER] iniciando
  [REVIEWER] iteración 1
[REVIEWER] terminado
✓ Memoria guardada en /content/workspace/project_memory.json

Agente: [PIPELINE COMPLETADO]
Archivos modificados: ['/content/workspace/pickndrive-ia/ARCHITECTURE_REPORT.md']
Veredicto del reviewer: APROBADO
```

### Fuentes Recuperadas (RAG)
Durante la fase del **Researcher**, el agente consultó la base vectorial en ChromaDB para fundamentar las recomendaciones sobre React. Las fuentes y fragmentos recuperados en esta ejecución incluyeron:

1. **Documento**: `Reducer and Context`
   - **URL**: `https://raw.githubusercontent.com/reactjs/react.dev/main/src/content/learn/scaling-up-with-reducer-and-context.md`
   - **Contexto**: Lógica para centralizar estados complejos y propagación de datos mediante contexto, utilizada para justificar la mitigación de lecturas no reactivas del storage en el reporte de arquitectura.
2. **Documento**: `Managing State`
   - **URL**: `https://raw.githubusercontent.com/reactjs/react.dev/main/src/content/learn/managing-state.md`
   - **Contexto**: Principios de estructuración de estado y sincronización.

### Explicación de lo Observado
1. **Coordinación Secuencial**: El agente principal coordinó de forma estricta el paso del estado `task_state` entre los 5 subagentes. Cada uno aportó información incremental (el Explorer mapeó el disco, el Researcher inyectó el contexto de buenas prácticas, el Implementer redactó el Markdown, el Tester leyó el archivo final y el Reviewer analizó todo para dar el veredicto).
2. **Detección de Loops**: Se observa un evento de seguridad crítico en la **iteración 19 del Explorer**: el modelo LLM intentó leer repetidamente el mismo archivo en un ciclo de razonamiento redundante. El detector `detect_loop()` identificó que la llamada `read_file` con la misma ruta se había repetido 3 veces, bloqueando la ejecución de esa tool-call específica y forzando al subagente a cambiar su estrategia (pasando a listar la carpeta `"root"` y utilizar `find_todos`).

---

## Tarea 2: Memoria Persistente (Misma sesión, segunda ejecución)

### Objetivo
Volver a ejecutar el pipeline de análisis (`/analyze`) sobre el mismo repositorio, inmediatamente después de haber concluido la Tarea 1, sin reiniciar el chat interactivo ni borrar el disco. Esto verifica que el agente reconozca el trabajo previamente realizado a través del sistema de memoria persistente.

### Evidencia del Output
Al ingresar nuevamente el comando `/analyze` en la consola interactiva:

```text
Prompt: /analyze

[PLAN] Generando plan...
Plan propuesto:
- Comprobar si ya existe un informe de arquitectura en pickndrive-ia.
- Reutilizar la memoria de arquitectura persistida en project_memory.json.

¿Aprobás este plan? (s/n/modificar): s

Agente: [MAIN] Pedido recibido: /analyze
[MEMORIA] Ya analicé este repo antes. Usando contexto previo.
[EXPLORER] iniciando
  ...
✓ Memoria guardada en /content/workspace/project_memory.json

Agente: [PIPELINE COMPLETADO]
Archivos modificados: ['/content/workspace/pickndrive-ia/ARCHITECTURE_REPORT.md']
Veredicto del reviewer: APROBADO
```

Al verificar el archivo de almacenamiento local `project_memory.json`, se constata la persistencia de ambas sesiones:

```json
{
  "sessions": [
    {
      "date": "2026-07-11T21:37:29.389450",
      "request": "Análisis conversacional del repositorio",
      "repo_path": "/content/workspace/pickndrive-ia",
      "files_modified": [
        "/content/workspace/pickndrive-ia/ARCHITECTURE_REPORT.md"
      ],
      "observations": [],
      "sources": [...]
    },
    {
      "date": "2026-07-11T21:59:57.005148",
      "request": "Análisis conversacional del repositorio",
      "repo_path": "/content/workspace/pickndrive-ia",
      "files_modified": [
        "/content/workspace/pickndrive-ia/ARCHITECTURE_REPORT.md"
      ],
      "observations": [],
      "sources": [...]
    }
  ],
  "architecture": {
    "/content/workspace/pickndrive-ia": "A continuación un resumen conciso de la arquitectura observada en el repositorio..."
  }
}
```

### Explicación de lo Observado
El sistema detectó la clave de ruta `/content/workspace/pickndrive-ia` en la sección `architecture` de `project_memory.json`. El pipeline de subagentes inyectó este contexto preexistente directo en la memoria de la sesión actual (`task_state`), imprimiendo el aviso `[MEMORIA] Ya analicé este repo antes. Usando contexto previo.`. Esto evita el reprocesamiento innecesario del repositorio completo, ahorrando tiempo de cómputo y consumo de tokens de API.

---

## Tarea 3: Restricción de Permisos y Cambio de Estrategia

### Objetivo
Forzar una falla en los permisos de lectura del agente para evaluar su resiliencia. Se reinicia el entorno y se modifica el archivo `agent.config.yaml` añadiendo `package.json` a la lista de bloqueos en lectura:

```yaml
permissions:
  read:
    deny:
      - ".env"
      - "**/*.pem"
      - "**/*.key"
      - "secrets/**"
      - "package.json"
```

Al iniciar un `/analyze`, el Explorer intentará inspeccionar `package.json` (el archivo estándar donde se declaran las dependencias en Node/React) y se topará con el bloqueo. Se observa cómo reacciona ante esta denegación.

### Evidencia del Output
Durante el análisis, cuando el Explorer intenta leer el archivo bloqueado, el validador actúa:

```text
[EXPLORER] iteración 3
[BLOQUEADO] read_file: 'pickndrive-ia/package.json' denegado por config.
  [EXPLORER] iteración 4
[EXPLORER] tool: read_file ['pickndrive-ia/README.md']
```

El subagente Explorer, al recibir la denegación de lectura de `package.json`, reacciona de la siguiente forma en su historial de razonamiento registrado en la traza:

> *"El acceso a package.json ha sido denegado por la configuración de seguridad del agente. Cambiaré de estrategia: leeré el archivo README.md del repositorio e inspeccionaré la carpeta `src/` para identificar el stack tecnológico, dependencias y estructura a partir del código fuente."*

### Explicación de lo Observado
1. **Seguridad Activa**: La función `validate_tool_call()` interceptó correctamente el intento de `read_file` sobre `pickndrive-ia/package.json`, debido a que coincidía con el patrón de bloqueo del YAML. En lugar de propagar una excepción de Python o romper el pipeline, devolvió un mensaje de error controlado al LLM (`[BLOQUEADO]...`).
2. **Capacidad de Adaptación**: El modelo de lenguaje interpretó el bloqueo como una restricción del entorno y, de forma autónoma, cambió su estrategia: en lugar de detenerse o entrar en un bucle repetitivo de error, recurrió a fuentes alternativas de información (`README.md`, listados de carpetas bajo `src/`, etc.) logrando identificar que el proyecto utilizaba React, Vite, TypeScript y Tailwind CSS. Esto demuestra el cumplimiento del requerimiento del TP de adaptación de estrategia frente a bloqueos o errores de ejecución.
