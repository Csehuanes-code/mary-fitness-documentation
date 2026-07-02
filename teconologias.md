## TECNOLOGÍAS A UTILIZAR

El stack tecnológico seleccionado prioriza el tiempo de desarrollo rápido (enfoque Lean/MVP) y el soporte nativo eficiente para las capacidades del dispositivo.

* **Lenguaje de Programación:** Kotlin (versión estable actual).
* **Framework de Interfaz de Usuario:** Jetpack Compose (Material 3). Permite el desarrollo declarativo rápido de interfaces dinámicas y un rendimiento de renderizado óptimo en hardware limitado.
* **Base de Datos Local:** Room (Abstracción de SQLite oficial de Android). Actuará como la Fuente Única de Verdad (`Single Source of Truth`).
* **Servicio de Nube y Sincronización:** Firebase Firestore. Se debe compilar configurando explícitamente la propiedad de **Persistencia Offline Nativa**. Firestore gestionará la cola de sincronización de manera automática cuando el dispositivo recupere la conectividad a Internet.
* **Tareas en Segundo Plano y Alarmas:** * `WorkManager` para tareas diferibles y de sincronización pesada en segundo plano si fuera necesario.
* `AlarmManager` + `BroadcastReceiver` para el disparo preciso de las notificaciones de vencimiento de planes y toma de medidas sin depender de internet ni de servidores push externos (Firebase Cloud Messaging).

