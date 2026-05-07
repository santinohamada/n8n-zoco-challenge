# Event Automation Engine: Agenda Tucumán

Este repositorio contiene dos flujos de trabajo avanzados desarrollados en **n8n** diseñados para la gestión inteligente de eventos. El ecosistema integra web scraping, procesamiento de lenguaje natural (NLP) mediante modelos de lenguaje de gran escala (LLM) y una base de datos vectorial/relacional en Supabase.

## 🚀 Workflows

### 1. Extracción y Sincronización de Agenda (ETL)
Este flujo automatiza la recolección de eventos desde fuentes externas, los normaliza y los persiste de forma eficiente.

* **Trigger:** Programado cronológicamente (`Schedule Trigger`).
* **Extracción:** Realiza un `GET` a `agenda.eltucumano.com` y utiliza selectores CSS para capturar títulos, lugares y fechas crudas.
* **Procesamiento:** * **Limpieza:** Un nodo de JavaScript limpia saltos de línea y normaliza el texto.
    * **Fingerprinting:** Genera un hash único basado en `nombre-lugar-fecha` para evitar colisiones lógicas antes de la inserción.
* **Enriquecimiento AI:** Utiliza **Groq Chat Model** para estructurar descripciones complejas o fechas ambiguas a un formato JSON estricto.
* **Persistencia:** Realiza un `POST` a **Supabase** utilizando lógica de `on_conflict` sobre el fingerprint para asegurar que no existan duplicados en la base de datos.

### 2. Validador Inteligente de Duplicados (API)
Un servicio tipo "Check-before-post" que utiliza IA para determinar si un evento que se intenta cargar ya existe, incluso si el texto no es idéntico.

* **Entrada:** Webhook que recibe los datos del evento (nombre, fecha, lugar, ID opcional).
* **Lógica de Negocio:**
    1.  **Consulta de proximidad:** Busca eventos existentes en Supabase para la misma fecha.
    2.  **Bypass Optimizado:** Si no hay eventos para esa fecha, el flujo salta el procesamiento de IA (`Edit Fields`) para ahorrar latencia y tokens, marcando el evento como "no duplicado".
    3.  **Comparación Semántica (LLM):** Si existen eventos, **Groq** compara el nuevo evento contra los registros encontrados, calculando un porcentaje de coincidencia y proporcionando una razón lógica.
* **Respuesta de la API:**
    ```json
    {
      "isDuplicate": true,
      "matchScore": 0.85,
      "reason": "El evento 'Circo Ánima' ya existe en la misma ubicación con un nombre similar.",
      "duplicateEventId": "594c58be-..."
    }
    ```

---

## 🛠️ Stack Tecnológico

* **Orquestador:** [n8n](https://n8n.io/)
* **Inteligencia Artificial:** [Groq Cloud](https://groq.com/) (LLM de alta velocidad)
* **Base de Datos:** [Supabase](https://supabase.com/) (PostgreSQL + PostgREST)
* **Lenguajes:** JavaScript (para transformación de datos complejos)
* **Fuente de datos:** Scraping dinámico sobre portales de cultura locales.

## 📋 Requisitos de Configuración

1.  **Variables de Entorno en n8n:**
    * `SUPABASE_URL`: Endpoint de tu API de Supabase.
    * `SUPABASE_KEY`: Service Role Key o Anon Key según permisos.
    * `GROQ_API_KEY`: Para la conexión con el nodo de Groq.
2.  **Base de Datos:**
    * Tener creada la tabla `eventos` con una restricción de unicidad (unique constraint) sobre la columna `fingerprint`.
