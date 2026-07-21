Para configurar e integrar adecuadamente un agente con la infraestructura de **AscendAPI**, aquí tienes las especificaciones técnicas detalladas basadas en la documentación oficial:

### 1. Infraestructura y Autenticación
*   **Plataforma de Acceso:** La API se gestiona exclusivamente a través de **RapidAPI**.
*   **Protocolo:** Sigue un diseño **RESTful** con respuestas en formato **JSON**.
*   **Headers Obligatorios:** Cada petición debe incluir los siguientes encabezados de autenticación:
    *   `X-RapidAPI-Key`: Tu clave de API única (debe mantenerse en secreto y no exponerse en el lado del cliente).
    *   `X-RapidAPI-Host`: Dependiendo del producto, por ejemplo: `edb-with-gifs-and-images-by-ascendapi.p.rapidapi.com` para ExerciseDB V1.

### 2. Especificaciones de Productos y Datos
*   **ExerciseDB V2 (Flagship):** Contiene más de **11,000 ejercicios** validados por expertos, con videos HD, imágenes y metadatos multilingües.
*   **ExerciseDB V1:** Versión clásica con más de **2,000 ejercicios** y animaciones GIF.
*   **Muscle Visualizer:** API para generar diagramas anatómicos dinámicos (mapas de calor y resaltado de músculos) en modelos masculinos y femeninos.

### 3. Lógica de Peticiones y Filtrado
*   **Filtrado Avanzado:** Permite combinar múltiples criterios como grupo muscular, equipamiento, dificultad y tipo de ejercicio.
*   **Búsqueda Difusa (Fuzzy Matching):** El endpoint de búsqueda admite un parámetro `threshold` (0 para coincidencia exacta, 1 para búsqueda muy laxa) para sugerir nombres de ejercicios mientras el usuario escribe.
*   **Paginación Basada en Cursores:** A diferencia del sistema tradicional de offset, AscendAPI utiliza cursores (`after`, `before`) para garantizar el rendimiento en grandes conjuntos de datos.
    *   **Meta campos:** El objeto `meta` en la respuesta indica `hasNextPage` y proporciona el `nextCursor`.

### 4. Guías de Codificación e Integración
*   **Rotación de URLs de Medios:** Las URLs de imágenes, GIFs y videos **rotan semanalmente todos los lunes a las 00:00 UTC**.
    *   **Regla Crítica:** No almacenes URLs de medios de forma permanente. Si tu plan permite caché, el TTL debe expirar antes de la rotación del lunes para evitar enlaces rotos (errores 404).
*   **Caché:** Solo está permitido si tu plan lo especifica. Los **IDs de ejercicio son estables** y pueden tratarse como identificadores permanentes.
*   **Manejo de Medios:** Se recomienda usar `videoUrl` para pantallas de detalle y `imageUrls` (en resoluciones como 360p o 480p) para vistas de lista para optimizar tiempos de carga.

### 5. Límites y Errores
*   **Rate Limiting:**
    *   **Planes Gratuitos:** Limitados a **1,000 peticiones por hora**.
    *   **Llamadas a Medios:** Generalmente son ilimitadas ya que se sirven a través de un CDN global.
*   **Gestión de Errores:** Todas las respuestas de error tienen una estructura consistente: código de error, mensaje legible, link a la documentación y un `requestId` (esencial para soporte técnico).
    *   **Códigos comunes:** `401 Unauthorized` (falta clave API), `403 Forbidden` (el plan no cubre el recurso), `429 Too Many Requests` (límite excedido).

### 6. Casos de Uso para el Agente
El agente puede utilizar estos datos para construir:
*   Generadores de planes de entrenamiento inteligentes.
*   Bibliotecas de ejercicios con instrucciones paso a paso.
*   Visualizaciones de activación muscular para evitar el sobreentrenamiento.
*   Integraciones con flujos de IA (LLMs) mediante futuros servidores MCP.