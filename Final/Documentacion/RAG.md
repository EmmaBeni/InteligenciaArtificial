# Base RAG

## Fuentes Utilizadas
* Documentación oficial de React descargada del repositorio de Github reactjs/react.dev (branch main) en formato .md vía raw.githubsercontent.com
* Algunos documentos puntuales:
    1) React-Learn (learn/index.md)
    2) Thinking in React
    3) Props (passing-props-to-a-component.md)
    4) Managing State
    5) Hooks Reference
    6) Reducer and COntext (scaling-up-with-reducer-and-context.md)
* En cada corrida del notebook se descargan con request.get() porque no se cachean en disco. 
Si react.dev cambiara los archivos, el índice se arma distinto la próxima vez. 
De todos modos, el documentación conceptual y no adaptada al repositorio analizado. 

## Estrategia de Chunking
Está a cargo de chunk_text(). Realiza el chunking por palabras y no por oraciones o estrucutras markdown (siguiendo headers/secciones).
El tamaño del chunk es de 500 palabras. El overlap enrte chunks consecutivos es de 50 palabras, lo que evita cortar ideas jsuto en el borde. 
La información contenida en estos es id (hash MD5 de url:posicion), text, title, url, chunk_index.
No cuenta el algorimto con una limpieza del markdown, la división se realiza sobre el .md crudo. 

## Embeddings
text-embedding-3-small de OpenAI es el modelo utilizado. Genera un embedding por chunk, vía client.embeddings.create()
Se estableció como límite de seguridad un truncamiento del texto a 8000 caracteres antes de mandarlo a embedding, en caso de que algún chunk sea demasiado largo. 
Este modelo se usa tanto para indexar los chunks así como para hacer el embedding de la query de búsqueda (es necesario para que la comparación tenga sentido).

## Almacenamiento y Búsqueda
Decidimos recrear el almacenamiento vecotrial en memoria utilizando 3 listas paralelas: rag_texts, rag_embeddings, rag_metadata (título + url), más una matriz numpy (rag_matrix, shape (n_chunks, 1536).
Se arman a partir de los embeddings. Dado que son unas pocas docenas de chunks, se puede aplicar fuerza bruta en la búsqueda. 
Devuelve, por defecto, los 3 chunks más similares, cada uno etiquetado con "source": "RAG", más su score de similitud, título y URL de origen.
Estos resultados son los que consume el Researcher: se agregan a state["rag_chunks_used"] y a state["sources_consulted"], y el texto de los chunks (primeros 400 caracteres de cada uno) se le pasa como contexto extra en el prompt.

## Almacenamiento vectorial: numpy en memoria vs. base vectorial dedicada

| Aspecto | Numpy en memoria (elegido) | Base vectorial dedicada (ej. Chroma) |
|---|---|---|
| **Persistencia** | No persiste entre corridas — se re-embeddean los chunks cada vez que se corre el notebook | Persiste en disco, evita re-embeddear en corridas sucesivas |
| **Performance a esta escala** | Búsqueda por fuerza bruta, instantánea con ~40-60 chunks | Indexación (HNSW/IVF) pensada para volúmenes mucho mayores; sin ventaja perceptible acá |
| **Complejidad de setup** | Cero dependencias extra, funciona con lo que ya usa el proyecto (numpy) | Dependencia adicional, con riesgos de compatibilidad de entorno (ej. versión de SQLite en Colab) |
| **Escalabilidad** | No escala a corpus grandes (miles de documentos) | Pensada para escalar sin degradar tiempos de búsqueda |
| **Costo en llamadas a la API** | Se re-embeddea todo el corpus en cada sesión | Embeddings calculados una sola vez, reutilizados entre sesiones |
| **Adecuación al caso de uso** | Corpus fijo y acotado (6 documentos de referencia de React), sin necesidad de updates incrementales ni filtros de metadata complejos | Aporta valor cuando el corpus crece, cambia con frecuencia, o se necesita filtrado avanzado |

**Decisión**: se optó por almacenamiento vectorial en memoria (matriz numpy + similitud coseno) dado el volumen acotado y estático del corpus RAG. El mecanismo es funcionalmente equivalente al de una base vectorial dedicada (embeddings + búsqueda por distancia), sin la complejidad ni las dependencias adicionales que no se justifican a esta escala.