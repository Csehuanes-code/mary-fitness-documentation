## REGLAS DE CODIFICACIÓN (PARA EL AGENTE DE IA)

Cuando se solicite la generación de código, el agente de IA debe seguir las siguientes directrices estructurales:

1. **Gestión de Estado de la UI:** Utilizar `StateFlow` dentro de los ViewModels para exponer estados inmutables de la pantalla (ej. `UiState` que maneje estados de `Loading`, `Success(data)` y `Error(message)`).
2. **Asincronía Obligatoria:** Toda llamada a los repositorios de Room o Firestore dentro del ViewModel debe ejecutarse mediante Corrutinas de Kotlin bajo el despachador adecuado:
```kotlin
viewModelScope.launch(Dispatchers.IO) { // Operación de datos }

```


3. **Inyección de Dependencias Simplificada:** Para mantener el proyecto liviano y rápido de configurar, utilizar una aproximación manual de inyección de dependencias (Service Locator o contenedor de dependencias simple a nivel de la clase `Application`) o, en su defecto, Hilt configurado de manera directa sin configuraciones redundantes.
4. **Manejo de Errores:** No permitir fallos silenciosos ni caídas de la aplicación (`crashes`). Las excepciones en la capa de datos deben capturarse mediante bloques `try-catch` y transformarse en mensajes legibles para el usuario a través del estado de la UI.
5. **Clean Code:** Mantener las funciones de Compose pequeñas y reutilizables. Separar claramente los componentes de presentación (ej. botones, campos de texto comunes) de las pantallas contenedoras principales.
