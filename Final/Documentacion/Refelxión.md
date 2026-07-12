# Reflexión final

## ¿Qué funcionó bien?

- La arquitectura multiagente quedó completa y con responsabilidades bien separadas: Explorer, Researcher, Implementer, Tester y Reviewer, cada uno con su propio objetivo y, en la práctica, con distinto acceso a tools (Explorer y Tester solo leen, Researcher agrega RAG/web, Implementer es el único que escribe el reporte).
- El RAG quedó atado a un almacenamiento vectorial persistente real (ChromaDB), no a un array en memoria como en versiones anteriores del proyecto, y en cada corrida se pudo ver y documentar qué chunks y fuentes se recuperaron.
- El estado compartido (`task_state`) registró todo lo que pedía la consigna: pedido original, progreso, resultados por subagente, fuentes consultadas, archivos modificados y observaciones.
- La memoria persistente entre sesiones funcionó de forma verificable: al analizar el mismo repositorio una segunda vez en la misma sesión, apareció el aviso `[MEMORIA] Ya analicé este repo antes` y `project_memory.json` quedó con dos entradas en `sessions`.
- La configuración de permisos (`agent.config.yaml`) se validó en cada tool call, incluyendo un caso forzado de restricción de lectura para observar la reacción del agente.
- Se registró al menos un loop real, no forzado: el Explorer, en una corrida completa, repitió la lectura de `ARQUITECTURA.md` y el sistema lo detectó y registró el cambio de estrategia.
- La observabilidad con Langfuse quedó funcionando de punta a punta: una traza por sesión de chat, un span por subagente y una generación por cada llamada al LLM, con prompt, modelo, tokens y latencia.
- El extra opcional (sistema de plugins) quedó implementado con un registro de tools desacoplado del núcleo (`TOOL_REGISTRY` + `@register_tool`), y se usó en una corrida real: el Researcher llamó al plugin `find_todos` sobre el repositorio y obtuvo un resultado concreto.
- Se logró, en la versión final, unificar el chat conversacional del agente principal con el pipeline de subagentes en un solo sistema, algo que en versiones anteriores del proyecto quedaba como dos caminos separados.

## ¿Qué falló / limitaciones encontramos?

- El subagente Researcher llegó a necesitar 46 turnos en una sola corrida. El motivo: sigue "cambiando de estrategia" —probando nuevas queries de búsqueda— sin llegar nunca a una respuesta final, y la detección de loops no lo frena porque solo identifica repetición **exacta** de una tool con los mismos argumentos, no un patrón de iteración excesiva con variaciones menores.
- El dashboard de Langfuse mostró un delay notable entre que el agente ejecutaba algo y que aparecía en la interfaz. Esto se debe a que el SDK batchea los eventos internamente y solo se fuerza el envío completo (`flush()`) al terminar todo el pipeline, sumado a la latencia propia de ingesta del lado del servidor.
- El pipeline de subagentes siempre ejecuta el mismo objetivo fijo por subagente, sin importar el texto que el usuario escriba antes de `/analyze`: no se puede pedir una tarea distinta (por ejemplo, buscar vulnerabilidades en vez de analizar arquitectura) sin cambiar código.
- El resumen de contexto (`summarize_history`) solo se activa en el modo de ejecución con subagentes; el chat directo con el agente principal puede acumular historial sin límite.
- La memoria persistente resultó débil como prueba de un cambio de comportamiento real: el aviso `[MEMORIA]` es solo informativo. El sistema no usa la arquitectura ya guardada para evitar que el Explorer vuelva a recorrer todo el repositorio desde cero, ni para que el Researcher deje de repetir la misma búsqueda, ni para que el Implementer actualice el reporte existente en vez de reescribirlo por completo. En otras palabras: el agente "sabe" que ya vio el repo, pero no actúa distinto por eso.
- `ARCHITECTURE_REPORT.md` se sobreescribe íntegramente en cada corrida, sin versionado ni historial de reportes previos.
- Dentro de una misma sesión de chat, el registro de llamadas a tools (`tool_call_log`) y los chunks de RAG usados (`rag_chunks_used`) se acumulan entre distintas ejecuciones de `/analyze` en vez de reiniciarse, lo que puede alterar cuándo se dispara la detección de loops en corridas sucesivas.
- Si el usuario escribe una tarea en lenguaje natural en vez del comando `/analyze`, el sistema cae en modo chat simple: sin RAG, sin memoria, sin detección de loops y sin trazas en Langfuse. Esto ocurrió en una corrida real del proyecto y produjo 59 iteraciones sin ninguno de los guardrails esperados.
- Durante el desarrollo se identificaron y corrigieron varios bugs que vale la pena dejar documentados como parte del proceso: spans de Langfuse anidados bajo el subagente incorrecto en una versión temprana, pérdida completa del logging de llamadas al LLM en una versión intermedia, una condición de supervisión (`allow_command`) que terminaba dejando pasar sin aprobación cualquier tool destructiva distinta de `run_command` (incluyendo `write_file`), y una celda duplicada con código viejo que, de haberse ejecutado, hubiera revivido bugs ya resueltos.

## ¿Cuándo se detectaron loops o falta de evidencia?

- Loop real, no forzado: el Explorer repitió la lectura de `ARQUITECTURA.md` durante el análisis del repositorio y el sistema lo registró correctamente, cambiando de estrategia.
- Loop no detectado por el guard (falso negativo real, dejado como hallazgo): el Researcher usó 46 turnos variando sus búsquedas sin que el mecanismo de detección lo frenara, porque nunca repitió una llamada con argumentos idénticos.
- Corrida dirigida para observar el comportamiento ante restricciones: se bloqueó temporalmente la lectura de `package.json` en la configuración de permisos para verificar que el Explorer reconozca la restricción y continúe con otra fuente o cambie de estrategia.
- Falta de evidencia observada: la corrida en modo chat directo (sin `/analyze`) no generó ninguna traza en Langfuse, ninguna fuente de RAG ni ninguna entrada de memoria, lo que deja en evidencia que buena parte de las capacidades del sistema dependen de que el usuario use el comando correcto.

## ¿Qué mejoraríamos?

- Agregar un tope duro de iteraciones por subagente, además de la detección de repetición exacta, para cortar patrones de iteración excesiva sin convergencia aunque los argumentos varíen.
- Hacer que el Researcher arme la query de RAG a partir del pedido real del usuario y de los hallazgos del Explorer, en vez de usar siempre una query fija.
- Extender el resumen de contexto también al modo de chat directo con el agente principal.
- Hacer que la memoria persistente cambie el comportamiento real del sistema: pasarle al Explorer la arquitectura ya guardada de un repositorio conocido para evitar repetir toda la exploración desde cero.
- Versionar `ARCHITECTURE_REPORT.md` en vez de sobreescribirlo en cada corrida.
- Llamar a `flush()` de Langfuse al finalizar cada subagente, no solo al final del pipeline completo, para reducir el delay de visualización en el dashboard.
- Habilitar `/subagents` como una decisión autónoma del agente principal, para que decida por sí mismo cuándo delegar en el pipeline de subagentes en vez de depender de que el usuario escriba el comando exacto.