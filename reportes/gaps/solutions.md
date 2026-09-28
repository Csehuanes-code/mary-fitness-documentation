### 1. Retardo de 24 horas y Anulación por Actualizaciones/Zonas Horarias

**Contexto:** Depender de un `DailyCheckWorker` deja ventanas de 24 horas ciegas. Además, reinicios, cambios de hora (`TIME_SET`) y actualizaciones de la app (`MY_PACKAGE_REPLACED`) borran las alarmas en memoria.
**Especificación para el Agente de IA (AI Prompt):**

* **Patrón de Diseño:** Observer / Event-Driven.
* **Implementación:**
1. Crea un `AlarmsRestoreReceiver` que herede de `BroadcastReceiver`. Registra en el `AndroidManifest.xml` los intent filters: `BOOT_COMPLETED`, `MY_PACKAGE_REPLACED`, `TIME_SET` y `TIMEZONE_CHANGED`.
2. En el manejador del Receiver, **no consultes Room directamente**. Lanza un `OneTimeWorkRequest` de WorkManager (ej. `RestoreAlarmsWorker`) para que se ejecute en un hilo de fondo seguro.
3. **Dominio:** Modifica los Casos de Uso de creación/actualización (`CrearPlanUseCase`, `ActualizarPagoUseCase`) para que invoquen inmediatamente a un `NotificationScheduler.schedule()` en lugar de esperar al ciclo diario del worker.



### 2. Caída silenciosa por permisos (Android 13+ y 14+)

**Contexto:** Restricciones modernas de Android bloquean las notificaciones (`POST_NOTIFICATIONS`) y degradan las alarmas exactas a inexactas por el modo Doze (`SCHEDULE_EXACT_ALARM`).
**Especificación para el Agente de IA (AI Prompt):**

* **Capa de Presentación (Compose):** Implementa un flujo preventivo. Crea un Composable `PermissionHandler` utilizando `ActivityResultContracts.RequestPermission` para Android 13+.
* **Capa de Infraestructura:** Antes de llamar a `alarmManager.setExactAndAllowWhileIdle()`, verifica `alarmManager.canScheduleExactAlarms()` (requerido en API 34+).
* **Flujo de Bloqueo:** Si el permiso exacto es falso, el Agente debe generar un `AlertDialog` explicando por qué la app necesita precisión para las alertas de cobro, y proporcionar un botón que lance `Intent(Settings.ACTION_REQUEST_SCHEDULE_EXACT_ALARM)`. Implementa un *fallback* a WorkManager con `setWindow` si el usuario deniega la precisión.

### 3. Desconexión de la interfaz de usuario (Toggles Decorativos)

**Contexto:** Los switches en `AjustesNotificacionesScreen` no hacen nada real a nivel de sistema; las alarmas siguen programándose.
**Especificación para el Agente de IA (AI Prompt):**

* **Persistencia:** Utiliza `Jetpack DataStore` (Preferences) para guardar el estado de los switches (ej. `PREF_ENABLE_PAYMENT_ALERTS`).
* **Reactividad:** Expón un `Flow<Boolean>` desde un `SettingsRepository`. Inyecta este repositorio en el `NotificationScheduler`.
* **Acción:** Cuando el usuario apaga un switch en la UI, el ViewModel debe disparar un Caso de Uso (`CancelAllNotificationsUseCase`) que itere sobre los IDs activos y llame a `alarmManager.cancel(pendingIntent)` inmediatamente, además de actualizar el DataStore.

### 4. Notificaciones fantasma por falta de cancelación

**Contexto:** El método `cancelarAlertaPago` existe pero es código muerto (Dead Code). Los clientes eliminados siguen generando alertas.
**Especificación para el Agente de IA (AI Prompt):**

* **Principio de Responsabilidad Única (SOLID):** Modifica los Casos de Uso transaccionales (`DeletePlanUseCase`, `DeletePagoUseCase`, `DeleteClienteUseCase`).
* **Implementación:** Inyecta la interfaz del `NotificationScheduler` en estos Casos de Uso. Antes o inmediatamente después de eliminar el registro en Room, debes invocar `scheduler.cancelNotification(entityId)`. Garantiza que el algoritmo de generación de IDs para los `PendingIntent` sea determinista (ej. `entityId.hashCode()`) para poder reconstruir el ID exacto al cancelar.

### 5. Pérdida de persistencia al "Reasignar"

**Contexto:** Tocar "Reasignar" en la notificación pospone la alarma local, pero no actualiza la base de datos. El próximo worker sobrescribe el cambio leyendo los datos antiguos.
**Especificación para el Agente de IA (AI Prompt):**

* **Arquitectura Limpia:** La acción de la notificación **nunca** debe hablar directamente con el `AlarmManager`.
* **Flujo Correcto:**
1. El botón "Reasignar" (PendingIntent) lanza un `BroadcastReceiver` o Servicio.
2. Este invoca a un `ReprogramarPagoUseCase(pagoId, nuevaFecha)`.
3. El Caso de Uso actualiza el campo `fecha_notificacion` o `fecha_vencimiento` en el `PagoRepository` (Room) primero.
4. El repositorio emite el cambio y el Caso de Uso delega al `NotificationScheduler` reprogramar la alarma basándose en el nuevo estado de verdad (Single Source of Truth).



### 6. Bloqueo por trampolines de notificación (Android 12+)

**Contexto:** Usar un `BroadcastReceiver` intermedio para manejar el click de "Revisar" y luego lanzar la Activity es bloqueado por el OS.
**Especificación para el Agente de IA (AI Prompt):**

* **Enrutamiento Directo:** Para botones que abren la UI, prohíbe el uso de `PendingIntent.getBroadcast()`.
* **Implementación:** Genera un `PendingIntent.getActivity()` con un `Intent` explícito que contenga un **Deep Link** o una ruta de Jetpack Compose Navigation hacia `ClienteDetailScreen`. Asegúrate de usar los flags `PendingIntent.FLAG_UPDATE_CURRENT` y `PendingIntent.FLAG_IMMUTABLE` obligatorios en Android 12+.

### 7. Colapso (Flooding) post-restauración en Firebase

**Contexto:** Al restaurar un backup desde la nube, Room se llenará de datos antiguos. El sistema agendará alertas para todos los pagos vencidos históricamente, lanzando decenas de notificaciones a la vez.
**Especificación para el Agente de IA (AI Prompt):**

* **Regla de Caducidad (Filtro):** En el `NotificationScheduler` y el `DailyCheckWorker`, implementa una política de filtrado temporal.
* **Condición:** `if (fechaVencimiento.isBefore(LocalDate.now().minusDays(2))) { return; }`. No programar alarmas para eventos que ya pasaron su umbral de relevancia (ej. más de 48 horas en el pasado). Considerarlos "vencidos en silencio" y gestionarlos solo con badges en la UI, no con alertas push sonoras.

### 8. Riesgo de ANR por consultas síncronas en Receivers (Hardware eMMC)

**Contexto:** El Moto G22 sufre de cuellos de botella en I/O. Consultar Room dentro del `onReceive` de una alarma para armar el texto de la notificación provocará un ANR.
**Especificación para el Agente de IA (AI Prompt):**

* **Transferencia de Carga Util (Payload):** Elimina la necesidad de consultar la base de datos en el momento en que se dispara la alarma.
* **Implementación:** Cuando se programa la alarma mediante `alarmManager.setExact(...)`, el agente debe serializar los datos mínimos necesarios (Nombre del Cliente, Monto, Tipo de Alerta) como `Extras` dentro del `Intent` encapsulado en el `PendingIntent`.
* **Ejecución:** En el `onReceive` del `NotificationPublisherReceiver`, extrae directamente los datos de `intent.getExtras()` y lanza la notificación al instante (complejidad O(1)), evitando cualquier I/O contra el disco o Room.
