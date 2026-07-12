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
El sistema utiliza **ChromaDB** como base vectorial persistente (`chromadb.PersistentClient`) alojada localmente en `/content/workspace/chroma_db`. 

El proceso de almacenamiento y búsqueda funciona así:
1. **Inicialización**: Se inicializa el cliente persistente y se crea (o recupera) la colección `react_docs`.
2. **Indexado condicional**: Si la colección no contiene documentos (`collection.count() == 0`), se generan los embeddings de los chunks y se indexan en ChromaDB junto a su respectiva metadata (título y URL original). Si ya existen documentos indexados, se omiten la generación y la inserción para ahorrar costos de API y tiempo de cómputo.
3. **Búsqueda**: La función `rag_search()` calcula el embedding de la consulta del usuario mediante `get_embedding()` y consulta la colección usando `collection.query()`, retornando los 3 chunks más similares.

Los resultados devueltos incluyen el texto, el título, la URL, la similitud semántica (calculada a partir de la distancia devuelta por Chroma) y la etiqueta `"source": "RAG"`. Estos fragmentos son consumidos por el **Researcher**, que los registra en `state["sources_consulted"]` y los inyecta en su contexto de prompt.

## Almacenamiento vectorial: numpy en memoria vs. base vectorial dedicada (ChromaDB)

| Aspecto | NumPy en memoria (descartado) | Base vectorial dedicada (ChromaDB) (elegido) |
|---|---|---|
| **Persistencia** | No persiste entre corridas: se re-embeddean los chunks cada vez que se corre el notebook. | Persiste en disco (`/content/workspace/chroma_db`), evitando re-indexar en ejecuciones sucesivas si ya hay datos. |
| **Performance a esta escala** | Búsqueda por fuerza bruta rápida en memoria (~40-60 chunks). | Indexación optimizada (HNSW); tiempo de respuesta instantáneo e ideal para escalar a futuro. |
| **Complejidad de setup** | Cero dependencias extra, solo NumPy. | Requiere instalar e importar la biblioteca `chromadb`. |
| **Escalabilidad** | Limitado. Degradación del rendimiento de búsqueda al crecer el número de documentos. | Totalmente escalable a miles o millones de documentos sin pérdida de rendimiento. |
| **Costo en llamadas a la API** | Alto en el largo plazo, ya que requiere llamar a la API de embeddings por cada chunk en cada inicio de sesión. | Bajo y controlado: los embeddings se generan una única vez al poblar la base por primera vez. |
| **Adecuación al caso de uso** | Adecuado solo para pruebas efímeras. | Excelente para mantener una base de conocimiento persistente, permitiendo consultas rápidas y ampliación de documentos sin coste adicional recurrente. |

**Decisión**: Se optó por **ChromaDB** como base vectorial persistente debido a que reduce significativamente el número de llamadas a la API de OpenAI al evitar re-generar embeddings en cada ejecución del notebook. Además, sienta una base de desarrollo profesional que permite escalar el corpus de conocimiento sin comprometer el rendimiento.
