# Documentacion Tecnica - Chatbot CCD UNAB

Documento tecnico detallado del Chatbot Inteligente para el Centro de Competencias Digitales de la Universidad Autonoma de Bucaramanga. Complementa al README con las decisiones de diseno, el detalle de la arquitectura, el modelo de datos y la justificacion de cada componente.

---

## 1. Vision general del sistema

El sistema es un asistente conversacional para estudiantes de la UNAB, accesible por Telegram, que responde preguntas sobre la Ruta de Competencias Digitales. Esta construido sobre n8n como motor de orquestacion y emplea un agente ReAct con siete herramientas conectadas a distintas fuentes de datos.

El objetivo de diseno fue construir un asistente que no dependa de comandos rigidos, sino que entienda preguntas en lenguaje natural y resuelva cada una consultando la fuente adecuada: base de datos relacional, hoja de calculo, o base de datos vectorial mediante RAG.

### 1.1 Decisiones de diseno principales

| Decision | Alternativa descartada | Justificacion |
| --- | --- | --- |
| Agente ReAct con Tools | Switch por comandos slash | Permite lenguaje natural, encadenar herramientas y agregar funciones sin reescribir el enrutador |
| PGVector sobre tabla existente | Crear un vector store separado | Reutiliza los datos ya cargados en Postgres y aprovecha el filtrado por metadata |
| Embeddings nv-embed-v1 | text-embedding de OpenAI | Modelo ya usado en la ingesta (replanteado para cargar nuevamente la metadata existente en la tabla documentos_rag); obligatorio mantener consistencia indexar/consultar |
| Telegram como interfaz | App movil nativa | Reduce complejidad de desarrollo y despliegue manteniendo la experiencia tipo chat |
| Limpieza de Markdown en nodo Code | Confiar en el formato del LLM | Telegram en modo texto no tolera Markdown; el nodo garantiza compatibilidad |

---

## 2. Arquitectura por capas

El sistema se organiza logicamente en cinco capas.

![Arquitectura por capas](imagenes/02_capas.png)

La vista de contexto general (qué hace el sistema y con quien interactua) se muestra a continuacion:

![Vista de contexto](imagenes/01_contexto.png)

### 2.1 Capa de presentacion

El unico canal de interaccion es Telegram. El estudiante escribe preguntas en lenguaje natural y recibe respuestas en texto plano. No se usan comandos rigidos; toda la interaccion es conversacional.

### 2.2 Capa de aplicacion (orquestacion)

Implementada en n8n. Toda la lógica del sistema se encuentra consolidada dentro del mismo archivo de exportación (.json) y desplegada en un único lienzo principal, el cual se compone de tres flujos core junto con sus respectivos sub-flujos y componentes auxiliares:

- Workflow principal (Agente conversacional): Recibe el mensaje desde el trigger de Telegram, lo procesa mediante el agente ReAct, gestiona el formato de salida y envía la respuesta final al estudiante.

- Workflow de scraping: Extrae de manera periódica las noticias y novedades del sitio web institucional de la UNAB.

- Workflow de ingesta RAG: Automatiza la segmentación y carga del documento institucional a la base vectorial mediante metadatos, prescindiendo de cargas manuales previas.

- Flujos y componentes auxiliares: Integrados de forma adyacente en el mismo espacio de trabajo para dar soporte a las herramientas del agente.

### 2.3 Capa de inteligencia artificial

- LLM: Anthropic Claude, configurado como Chat Model del agente. Realiza el razonamiento, la seleccion de herramientas y la redaccion de respuestas.
- Embeddings: nvidia/nv-embed-v1, que genera vectores de 4096 dimensiones para la busqueda semantica.
- Memoria: Postgres Chat Memory, que almacena el historial conversacional en la tabla n8n_chat_histories para mantener contexto entre mensajes.

### 2.4 Capa de herramientas

Siete herramientas conectadas al agente, cada una con una descripcion semantica que el modelo usa para decidir cuando invocarla. Se detallan en la seccion 5.

### 2.5 Capa de datos y persistencia

- PostgreSQL 16 con la extension pgvector 0.8.2.
- Google Sheets como fuente editable del calendario academico.
- La tabla documentos_rag almacena los chunks vectorizados de las dos fuentes RAG (en un flujo con Scraping inutilizado; en el implementado, el Scraping guarda la metadata en Simple Vector Store, no en una tabla de BD).

---

## 3. Modelo de datos

La base de datos n8n_db contiene las tablas del proyecto y las tablas internas de n8n. La entidad central es estudiante, conectada por clave foranea a las tablas academicas y de sesion (estas ultimas no se incluyeron en el dump (copia de seguridad) dado que se generan automáticamente con la ejecución de n8n).

![Modelo de datos](imagenes/04_modelo_datos.png)

### 3.1 Tablas principales

estudiante: entidad central con id_estudiante (clave primaria), nombre_completo, programa_academico, facultad, semestre_actual, email, telegram_id (unico) y campos de control.

catalogo_materias_ccd: catalogo de materias con codigo_materia (clave primaria), nombre_materia, plan y pilar.

oferta_vigente: cursos ofertados este periodo con campos informativos como id_oferta, codigo_curso, nombre_curso, pilar, aula, docente y cupos_disponibles.

cursos_estudiantes: relacion entre estudiante y materias cursadas, con campos reales indexados como id (clave primaria), id_estudiante (clave foránea), codigo_curso, nombre_materia, semestre, anio y fecha_matricula

registro_nota: notas por estudiante y materia, donde el campo nota se maneja de tipo CHARACTER(1) para almacenar calificaciones cualitativas, e incluye codigo_curso, tipo_registro, fecha_registro y observacion.

documentos_rag: chunks vectorizados para RAG. Es la tabla central del sistema de recuperacion.

n8n_chat_histories: memoria conversacional del agente (tabla gestionada por n8n).

oferta: almacena la oferta global histórica e institucional de cursos del CCD, con id_oferta (clave primaria), nombre, pilar, modalidad, cupos totales y estado activo.

solicitudes_validacion (No utilizada): registra las peticiones de los estudiantes para validar materias de la ruta, con id (clave primaria), vinculada a estudiante por id_estudiante (clave foránea), detallando la materia, fecha de solicitud y el estado (estado_solicitud). No es utilizada

### 3.2 Estructura de la tabla documentos_rag

| Columna | Tipo | Descripcion |
| --- | --- | --- |
| id | SERIAL | Clave primaria |
| nombre_documento | VARCHAR(200) | Nombre del documento origen |
| categoria | VARCHAR(100) | Categoria del chunk (Institucional / Web Scraping) |
| text | TEXT | Contenido textual del fragmento |
| embedding | VECTOR(4096) | Vector del fragmento (nv-embed-v1) |
| metadata | JSONB | Metadatos: category, file_name, title, source_type |
| content_hash | VARCHAR(32) | Hash del contenido para evitar duplicados |
| fecha_ingestion | TIMESTAMP | Fecha de carga |

La columna metadata replica la categoria en la clave category, que es la que usan las herramientas RAG para filtrar. El sistema mantiene tanto la columna nativa categoria como la clave en metadata.

**NOTA: La tabla contiene chunks del documento institucional (activos) y chunks de scraping antiguo (en desuso), y que el scraping actual va a Simple Vector Store en memoria.**

### 3.3 Relaciones

- estudiante 1:N cursos_estudiantes
- estudiante 1:N registro_nota
- estudiante 1:N solicitudes_validacion (Nueva relación)
- estudiante 1:1 (Lógica) progreso_estudiante (Relación plana de consulta de pilares)
- catalogo_materias_ccd 1:N oferta_vigente
- catalogo_materias_ccd 1:N cursos_estudiantes
- oferta 1:1 oferta_vigente (Mapeo de la oferta base hacia el estado vigente del periodo)
---

## 4. Workflows en n8n

### 4.1 Workflow principal (agente)

Secuencia: Telegram Trigger, ReAct Agent con Tools, Code (limpiador de Markdown), Send a text message.

El Telegram Trigger captura el mensaje entrante. El ReAct Agent recibe el texto, razona sobre la intencion, selecciona y ejecuta las herramientas necesarias, y compone la respuesta. El nodo Code elimina la sintaxis Markdown problematica antes del envio, porque Telegram en modo texto simple no la interpreta correctamente.

![Flujo del agente ReAct](imagenes/05_flujo_agente.png)

El despliegue fisico de estos componentes en el servidor se muestra en la vista de despliegue:

![Vista de despliegue](imagenes/03_despliegue.png)

El agente tiene conectados como sub-nodos:
- Chat Model: Anthropic Claude.
- Memory: Postgres Chat Memory (tabla n8n_chat_histories).
- Siete herramientas (seccion 5).

### 4.2 Workflow de scraping periodico

Secuencia: Schedule Trigger, HTTP Request al indice de noticias, HTML para extraer enlaces, Loop Over Items, HTTP Request a cada noticia, HTML para extraer contenido, Embeddings, Vector Store.

Es un scraping de dos niveles. El primer nivel recorre la pagina indice de noticias y extrae los enlaces a las noticias individuales. El segundo nivel entra a cada noticia y extrae su contenido completo, no solo el titulo. El contenido se vectoriza y se almacena en el vector store con categoria Web Scraping. El Schedule Trigger garantiza la ejecucion periodica.

### 4.3 Workflow de ingesta RAG

Secuencia: Trigger, carga del documento institucional, division en chunks, generacion de embeddings con nv-embed-v1, almacenamiento en documentos_rag.

Carga el documento oficial del CCD, lo fragmenta, genera los vectores y los guarda con categoria Institucional. Este workflow se ejecuta de forma puntual cuando se actualiza el documento fuente.

### 4.4 Mapeo a los workflows propuestos

El enunciado del proyecto plantea cuatro workflows separados (consulta general, calendario, cursos realizados y noticias). En esta implementacion las cuatro funciones existen, pero estan unificadas bajo un solo agente ReAct en lugar de cuatro flujos independientes. La siguiente tabla muestra la correspondencia:

| Workflow propuesto | Implementacion en este proyecto |
| --- | --- |
| Workflow 1 - Consulta general (RAG) | Herramienta consultar_documento_institucional |
| Workflow 2 - Consulta calendario | Herramienta get_calendar |
| Workflow 3 - Cursos realizados | Herramientas get_student_id_tool y get_student_courses_tool |
| Workflow 4 - Noticias y nuevos cursos | Workflow de scraping + herramienta vector store web |

La eleccion de unificar las funciones en un agente, en lugar de cuatro flujos con enrutamiento por reglas, se explica en la seccion 6.

---

## 5. Herramientas del agente

Cada herramienta tiene una descripcion en lenguaje natural que el agente lee para decidir cuando usarla. Las descripciones incluyen reglas de exclusion explicitas (cuando NO usar la herramienta) para evitar confusiones entre herramientas con temas similares.

| Herramienta | Tipo | Fuente | Cuando se usa |
| --- | --- | --- | --- |
| get_calendar | Sub-workflow | Google Sheets | Fechas del calendario academico |
| oferta_tool | Sub-workflow | PostgreSQL | Cursos vigentes con NRC, modalidad y precio |
| get_contacto | Sub-workflow | PostgreSQL | Datos de contacto del CCD |
| get_student_id_tool | Sub-workflow | PostgreSQL | Busqueda de id_estudiante por nombre |
| get_student_courses_tool | Sub-workflow | PostgreSQL | Cursos cursados por un estudiante |
| consultar_documento_institucional | PGVector Store | pgvector | Informacion teorica e institucional del CCD |
| vector store web | Vector Store | Simple Vector Store (Memoria n8n) | Noticias y novedades del sitio web |

### 5.1 Diseño de descripciones excluyentes

Cuatro herramientas tocan temas potencialmente cercanos (calendario, oferta, documento institucional, noticias). Para que el agente no las confunda, cada descripcion indica tanto su ambito como sus exclusiones. Por ejemplo, la herramienta de noticias indica explicitamente que NO debe usarse para fechas del calendario (eso corresponde a get_calendar) ni para la oferta con precios (eso corresponde a oferta_tool). Esta tecnica de descripciones excluyentes mejora notablemente la precision en la seleccion de herramientas.

### 5.2 Control de privacidad

Las herramientas que acceden a datos personales (get_student_courses_tool) estan gobernadas por reglas en el System Message del agente: el bot debe pedir identificacion antes de mostrar datos, no comparte informacion de un estudiante con otro, y agrega un aviso de privacidad al final de cada consulta personal.

---

## 6. Deteccion de intencion en la arquitectura

El enunciado propone un Workflow 2 con deteccion de intencion explicita (Pregunta, deteccion intencion, Google Sheets, respuesta). Este es el patron clasico de chatbots: un componente separado, basado en reglas o en un clasificador, asigna una intencion al mensaje y lo enruta a un sub-flujo.

En esta implementacion se adopto un enfoque diferente: la deteccion de intencion esta embebida en el razonamiento del agente ReAct. El modelo Claude analiza la pregunta y, leyendo las descripciones semanticas de las herramientas, decide cual invocar. No existe un nodo Switch ni un clasificador separado.

Ventajas de este enfoque frente al enrutamiento clasico por reglas:

- Maneja preguntas ambiguas o formuladas de multiples maneras sin necesidad de listas de palabras clave.
- Permite encadenar varias herramientas en una sola consulta (por ejemplo, buscar el id de un estudiante y luego sus cursos).
- Agregar una nueva intencion solo requiere agregar una herramienta con su descripcion, sin modificar un enrutador central.
- Es el patron utilizado actualmente en la industria para construir agentes (function calling / tool use).

En terminos conceptuales, el conjunto de descripciones de herramientas cumple la misma funcion que un clasificador de intenciones, pero de forma mas expresiva. La siguiente tabla resume el mapeo intencion - herramienta:

| Intencion del estudiante | Herramienta seleccionada |
| --- | --- |
| Consultar fechas del calendario | get_calendar |
| Consultar oferta de cursos | oferta_tool |
| Consultar contacto | get_contacto |
| Consultar identificacion | get_student_id_tool |
| Consultar historial de cursos | get_student_courses_tool |
| Consultar informacion institucional | consultar_documento_institucional |
| Consultar noticias y novedades | vector store web |

---

## 7. Sistema RAG en detalle

### 7.1 Fuentes y separacion por categoria

El sistema mantiene una colección RAG dentro de la misma tabla documentos_rag, diferenciada por la clave category en metadata:

- Institucional: chunks del documento oficial del CCD.
- Web Scraping (categoría en desuso en esta tabla): el flujo de scraping actual guarda el contenido en un Simple Vector Store en memoria de n8n, no en documentos_rag.


### 7.2 Pipeline de recuperacion

El siguiente diagrama de secuencia muestra una consulta RAG completa al documento institucional:

![Flujo de consulta RAG](imagenes/06_flujo_rag.png)

1. La pregunta del estudiante se vectoriza con nv-embed-v1.
2. Se ejecuta una busqueda por similitud coseno en pgvector, filtrando por la categoria correspondiente.
3. Se recuperan los fragmentos mas relevantes (por defecto los 5 mas cercanos).
4. Los fragmentos se entregan al agente como contexto.
5. Claude redacta la respuesta apoyandose en esos fragmentos.

### 7.3 Consistencia de embeddings

Un requisito critico del RAG es que el modelo de embeddings usado para indexar el documento sea identico al usado para consultar. Ambos usan nv-embed-v1 con vectores de 4096 dimensiones. Usar modelos distintos produciria vectores no comparables y resultados sin sentido.

### 7.4 Mantenimiento de la base vectorial

Durante el desarrollo se detecto que el documento institucional habia sido cargado dos veces con capitalizaciones distintas en la categoria, generando 103 chunks duplicados. Se resolvio conservando una sola version (categoria Institucional) y normalizando la columna categoria. El campo content_hash y la consistencia en la capitalizacion de la categoria previenen futuras duplicaciones.

---

## 8. Despliegue e infraestructura

| Elemento | Detalle |
| --- | --- |
| Proveedor | DigitalOcean Droplet |
| Sistema operativo | Ubuntu 24 LTS |
| Contenedores | Docker (n8n, PostgreSQL + pgvector, Nginx Proxy Manager) |
| Proxy inverso | Nginx Proxy Manager (con Let's Encrypt; HTTPS) |
| DNS | DuckDNS (dominio dinamico) |
| Base de datos | PostgreSQL 16 con pgvector 0.8.2 |

El acceso web a n8n esta protegido por autenticacion basica. Los webhooks de Telegram se exponen sin esa autenticacion para permitir las llamadas entrantes del bot.

### 8.1 Consideraciones de seguridad

- Las credenciales (contrasenas de base de datos, tokens, API keys) deben gestionarse mediante variables de entorno y nunca incluirse en el repositorio.
- Se recomienda cambiar todas las contrasenas por defecto por contrasenas fuertes.
- El archivo .env debe estar listado en .gitignore.
- Las credenciales de los nodos de n8n no se exportan en los JSON de los workflows; deben reconfigurarse al importar.

---

## 9. Limitaciones y trabajo futuro

### 9.1 Limitaciones

- El web scraping depende de la estructura HTML del sitio de la UNAB; cambios en el diseno del sitio requieren actualizar los selectores.
- La base de datos de estudiantes es sintetica y no esta conectada al sistema academico real.
- La extraccion de algunos fragmentos del PDF con tablas o imagenes complejas puede ser imperfecta.
- El scraping no se guarda directamente en BD, sino en la memoria de n8n (con Simple Vector Store; esto implica que las noticias se pierden cada vez que n8n se reinicia).

### 9.2 Trabajo futuro

- Reemplazar el scraping por una API oficial de la UNAB si estuviera disponible.
- Mejorar el extractor del PDF para manejar mejor tablas y figuras.
- Implementar un panel de monitoreo de las consultas y las herramientas mas usadas.
- Anadir reranking de resultados RAG para mejorar la precision de la recuperacion.
- Añadir memoria en BD, para elementos referentes a Web Scraping.


---

## 10. Glosario

| Termino | Definicion |
| --- | --- |
| RAG | Generacion aumentada por recuperacion: el modelo responde apoyado en fragmentos recuperados de una fuente |
| Agente ReAct | Patron de agente que combina razonamiento y accion para seleccionar y ejecutar herramientas |
| Embedding | Representacion vectorial de un texto que captura su significado |
| pgvector | Extension de PostgreSQL para almacenar y buscar vectores |
| Chunk | Fragmento de texto en que se divide un documento para vectorizarlo |
| Similitud coseno | Medida de cercania entre dos vectores usada en la busqueda semantica |
| Tool / Herramienta | Funcion conectada al agente que consulta una fuente de datos especifica |
| System Message | Instrucciones de comportamiento que recibe el agente |

---

Universidad Autonoma de Bucaramanga (UNAB) - Centro de Competencias Digitales - Proyecto de Ciencia de Datos.
