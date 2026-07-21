A continuación, presento el listado de las rutas exactas y especificaciones técnicas para los productos de **AscendAPI** (ExerciseDB y Muscle Visualizer). Cabe destacar que, según la documentación, todos los métodos actuales son de tipo **GET**.

### Configuración General de Cabeceras (Headers)
Para todas las peticiones a través de RapidAPI, el agente debe incluir obligatoriamente:
*   **X-RapidAPI-Key**: Tu clave de API única (obligatoria para autorización).
*   **X-RapidAPI-Host**: Varía según el producto (detallado abajo).
*   **Content-Type**: `application/json`.

---

### 1. ExerciseDB V1 (Dataset Clásico)
**Base URL:** `https://edb-with-gifs-and-images-by-ascendapi.p.rapidapi.com`

#### **GET** `/api/v1/exercises` (Filtrado Avanzado)
*   **Descripción:** Filtra ejercicios por múltiples criterios con soporte de búsqueda difusa y paginación.
*   **Parámetros de Consulta (Query Params):**
    *   `name` (string): Nombre del ejercicio (soporta fuzzy matching).
    *   `bodyParts` (string): Lista separada por comas (ej: "Chest,Shoulders").
    *   `targetMuscles` (string): Músculos primarios (ej: "pectorals").
    *   `equipments` (string): Equipamiento necesario.
    *   `limit` (number): Resultados por página (1-25, default 10).
    *   `after` / `before` (string): Cursores para paginación.
*   **Ejemplo de Respuesta:**
    ```json
    {
      "success": true,
      "meta": { "total": 120, "hasNextPage": true, "nextCursor": "edb_xyz" },
      "data": [{ "exerciseId": "edb_abc", "name": "Bench Press", ... }]
    }
    ```

#### **GET** `/api/v1/exercises/search` (Búsqueda Difusa)
*   **Descripción:** Optimizado para sugerencias de búsqueda mientras el usuario escribe.
*   **Parámetros:** `search` (término), `threshold` (0 a 1 para sensibilidad del emparejamiento).

#### **GET** `/api/v1/exercises/{exerciseId}` (Obtener por ID)
*   **Descripción:** Recupera los detalles completos de un ejercicio específico.
*   **Parámetro de Ruta (Path Param):** `exerciseId` (ej: "edb_T5uXtLj").

---

### 3. Muscle Visualizer (Visualización Anatómica)
**Base URL:** `muscle-visualizer-api.p.rapidapi.com`

#### **GET** `/api/v1/visualization-modes/generate-heatmap-muscle-visualization`
*   **Descripción:** Genera un mapa de calor donde cada músculo puede tener un color único.
*   **Parámetros Comunes:**
    *   `gender` (string): "male" o "female".
    *   `color` (string): Hex o RGB para el resaltado.
    *   `size` (string): Dimensiones (ej: "1080x1080").
    *   `format` (string): png, jpeg o webp.

#### **GET** `/api/v1/visualization-modes/generate-workout-muscle-visualization`
*   **Descripción:** Muestra la activación de músculos **primarios** y **secundarios** en un solo diagrama.
*   **Uso:** Ideal para guías de ejercicios y análisis de entrenamiento.

---

### Endpoints de Utilidad (Liveness y Listas)
*   `GET /api/v1/liveness`: Verifica el estado y disponibilidad del servidor.
*   `GET /api/v1/bodyparts`: Lista todos los nombres de partes del cuerpo disponibles.
*   `GET /api/v1/muscles`: Lista todos los nombres de músculos disponibles.
*   `GET /api/v1/equipments`: Lista todo el equipamiento posible.

### Notas sobre Almacenamiento y Errores
*   **Caché:** Las URLs de imágenes, GIFs y videos **rotan cada lunes a las 00:00 UTC**. Si el plan permite caché, el tiempo de vida (TTL) debe ser menor a 7 días y expirar antes de la rotación semanal para evitar errores 404.
*   **Errores:** Todas las respuestas fallidas incluyen un `requestId`, un código de error (como `BAD_REQUEST`, `UNAUTHORIZED`, `FORBIDDEN`) y un mensaje descriptivo.