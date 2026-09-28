Para garantizar la confianza usuario-aplicación y consolidar el sistema de notificaciones analizado en el archivo todo_el_repo.txt y la documentación adjunta, a continuación se detallan los fallos (gaps) existentes y los riesgos sistémicos no documentados que el agente de IA debe resolver.

**Gaps Documentados en el Código Actual**

* **Retardo de 24 horas en la programación (`DailyCheckWorker`)**: Las alarmas dependen de un worker diario, por lo que un evento registrado en el transcurso del día no programará su notificación hasta el siguiente ciclo, existiendo una ventana de hasta 24 horas sin alarmas activas, agravada tras un reinicio del dispositivo.


* **Caída silenciosa por permisos en Android 13+ y 14+**: Si el usuario no concede el permiso `POST_NOTIFICATIONS` en Android 13+, los receptores descartan la ejecución silenciosamente sin informar. En Android 14+, el permiso `SCHEDULE_EXACT_ALARM` viene denegado por defecto, provocando que las alarmas exactas caigan a un estado inexacto y se retrasen por horas a causa del modo Doze, sin existir un flujo en la interfaz que solicite este permiso especial.


* **Desconexión de la interfaz de usuario**: Los interruptores de la pantalla de configuración (`AjustesNotificacionesScreen`) no detienen la programación de las alarmas en el `NotificationScheduler`, actuando de forma puramente decorativa. De igual forma, la configuración de periodicidad de días es ignorada en favor de la duración total del plan.


* **Notificaciones fantasma por falta de cancelación**: La función `cancelarAlertaPago` existe pero nunca es invocada en el código. Al cancelar un plan o eliminar un pago, las notificaciones programadas seguirán disparándose.


* **Pérdida de persistencia al reasignar**: La opción de "Reasignar" reprograma la alarma en el sistema para el ciclo actual, pero no actualiza la fecha en la base de datos, por lo que el próximo ciclo del worker sobreescribirá el cambio.


* **Silencio por falta de asignación**: Los clientes que se encuentran en estado sin plan asignado nunca generan alertas.



**Nuevos Gaps Estructurales y de Ecosistema (No Documentados)**

* **Anulación total por actualizaciones In-App**: El sistema cuenta con un actualizador (`ActualizacionManager`) que descarga el APK desde Cloudinary y lanza el instalador del sistema. Al reemplazar el paquete de la aplicación, el sistema operativo Android elimina automáticamente todas las alarmas de `AlarmManager`. Sin un receptor del evento `ACTION_MY_PACKAGE_REPLACED`, la aplicación quedará silenciada tras cada actualización.


* **Bloqueo por trampolines de notificación (Android 12+)**: La aplicación requiere interacciones rápidas como "Revisar" y "Reasignar" desde la notificación. Si estos botones llaman a un `BroadcastReceiver` que posteriormente lanza la actividad, el sistema Android 12+ bloqueará la apertura de la app (restricción de Notification Trampolines).


* **Colapso (Flooding) post-restauración en Firebase**: La Fase 3 implementará la restauración de la base de datos de Room desde la nube. Al descargar el historial, el `DailyCheckWorker` detectará instantáneamente múltiples pagos vencidos y medidas atrasadas del pasado, disparando decenas de notificaciones de golpe si no existe un filtro de caducidad.


* **Desfase por zonas horarias y horarios de verano**: `AlarmManager` evalúa tiempos absolutos. Si el usuario cambia de zona horaria o el dispositivo aplica un cambio de horario de verano, las notificaciones exigidas a las 8:00 AM, 2:00 PM y 5:30 PM se dispararán en horas desfasadas si el sistema no escucha activamente los cambios del reloj del sistema.


* **Riesgo de ANR por hardware eMMC**: Los receptores de transmisión (`BroadcastReceiver`) tienen un límite de ejecución muy corto. Dado que el objetivo es un Moto G22 con memoria eMMC 5.1 y procesador Helio G37, ejecutar consultas síncronas hacia la base de datos Room dentro del receptor para estructurar la notificación provocará bloqueos ANR (Application Not Responding).



**Directrices de Cobertura para el Agente de IA**

* **Centralizar la reprogramación (Receiver Universal)**: El agente debe crear un único receptor que escuche `BOOT_COMPLETED`, `MY_PACKAGE_REPLACED`, `TIME_SET` y `TIMEZONE_CHANGED`, responsable de limpiar y reconstruir todo el árbol de alarmas de `AlarmManager` de forma segura.


* **Integración profunda del flujo "Reasignar"**: Instruir al agente para que la acción de reprogramar actualice los valores directamente en las tablas de Room (`MedidaRepository` y `ClienteRepository`), no solo en el administrador de alarmas.


* **Enrutamiento directo y seguro**: El agente debe estructurar los botones de acción usando `PendingIntent.getActivity()` con una ruta profunda (Deep Link) directa a `ClienteDetailScreen`, evadiendo el bloqueo de trampolines de Android 12+.


* **Gestión asíncrona en Receptores**: Obligar al agente a usar bloqueos asíncronos (`goAsync()`) o lanzar un `OneTimeWorkRequest` acelerado si el receptor de notificación requiere leer entidades desde Room, evitando saturar el hilo principal del Moto G22.


* **Flujo de permisos preventivo**: El agente debe programar diálogos bloqueantes (o informativos) que disparen explícitamente la acción del sistema `ACTION_REQUEST_SCHEDULE_EXACT_ALARM` para Android 14+ antes de delegar la tarea al planificador de alarmas.
