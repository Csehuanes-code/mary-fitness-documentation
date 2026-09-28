# Arquitectura Objetivo — Sistema de Notificaciones

**Fecha:** 2026-09-26
**Complementa:** `010-plan-implementacion-notificaciones.md`
**Propósito:** especificar los componentes, responsabilidades y flujo de datos que el agente de IA debe implementar. No es un rediseño de toda la app, solo de la capa de notificaciones y sus puntos de integración con `ClienteRepository`/`PagoRepository`/`MedidaRepository`.

---

## 1. Componentes y responsabilidades

### `NotificationScheduler` (única puerta de entrada para programar/cancelar)
- **Responsabilidad única:** es el **único** componente autorizado a llamar `AlarmManager.setExactAndAllowWhileIdle()` / `AlarmManager.cancel()`. Ningún ViewModel, Repository o Receiver llama a `AlarmManager` directamente.
- Expone:
  - `suspend fun scheduleAlertaPago(pagoId: Long, clienteId: Long, fechaDisparo: Instant, payload: NotificacionPayload)`
  - `suspend fun scheduleRecordatorioMedida(clienteId: Long, fechaDisparo: Instant, payload: NotificacionPayload)`
  - `suspend fun cancel(tipoAlerta: TipoAlerta, entidadId: Long)`
  - `suspend fun cancelTodasDeCliente(clienteId: Long)` (para soft-delete/purga)
- Internamente protegido por un `Mutex` único (evita condiciones de carrera — gap B2).
- Cada llamada de programación/cancelación escribe una fila en `NotificacionLogDao` antes de retornar.
- Usa el generador determinista de `requestCode` (ver §3).
- Lee los switches de `SettingsRepository` (`Flow<Boolean>`) antes de programar: si el tipo de alerta está desactivado, no programa y registra `CANCELADA` con motivo "desactivado por usuario" (resuelve el gap de switches decorativos).

### `ReconciliationWorker` (WorkManager, periódico)
- Corre cada N horas (definido en Fase 0, default 8h).
- Calcula el conjunto ideal de alarmas activas a partir del estado actual de Room (clientes visibles, `estadoCuenta`, próxima medición).
- Compara contra `NotificacionLogDao.obtenerProgramadasActivas()`.
- Reprograma lo faltante, cancela lo obsoleto (ej. cliente que ya pagó, cliente ahora oculto).
- Aplica el filtro de caducidad (no reprogramar alertas de más de 48h vencidas — gap de flooding).
- Si detecta más de `X` alarmas a programar en una sola corrida, las programa en lotes escalonados (mitiga saturación de `AlarmManager` — gap B3).
- Emite un resumen de la corrida (cuántas programó, canceló, dejó igual) hacia `notificacion_log` con un registro de tipo `SISTEMA`.

### `AlarmsRestoreReceiver` (BroadcastReceiver universal)
- Intent filters: `BOOT_COMPLETED`, `MY_PACKAGE_REPLACED`, `TIME_SET`, `TIMEZONE_CHANGED`.
- No accede a Room directamente. Su único trabajo es lanzar un `OneTimeWorkRequest` de `RestoreAlarmsWorker` (expedited si es posible) y retornar de inmediato.
- `RestoreAlarmsWorker` internamente delega en la misma lógica que `ReconciliationWorker` (idealmente son el mismo Worker parametrizado con un flag `origen = BOOT | PERIODICO | MANUAL`, para no duplicar lógica de "qué alarmas deberían existir").

### `NotificationPublisherReceiver` (dispara la notificación visual)
- Registrado como el `BroadcastReceiver` destino del `PendingIntent` de `AlarmManager`.
- **No hace I/O pesado ni bloqueante.** Lee `intent.getExtras()` para construir la notificación inmediatamente (gap de ANR).
- Si el extra trae un flag `payloadPotencialmenteDesactualizado = true` (ver §4), intenta una lectura corta a Room con `goAsync()` + timeout de 300ms; si no llega a tiempo, usa el extra tal cual y lo marca en el log como `MOSTRADA_CON_DATOS_POSIBLEMENTE_DESACTUALIZADOS`.
- Verifica `NotificationManagerCompat.areNotificationsEnabled()` y el estado específico del canal antes de intentar mostrar; si está deshabilitado, registra `DESCARTADA_PERMISO` en el log en vez de fallar en silencio.
- Aplica agrupamiento (`setGroup` + notificación resumen `InboxStyle`) cuando corresponde (ver `NotificationGrouper` abajo).
- Construye los botones de acción (`Revisar`/`Reasignar`/`Okay`) como `PendingIntent.getActivity()` con deep link directo — nunca `getBroadcast()` para navegación (evita trampolín).

### `NotificationGrouper` (utilidad, no un componente con ciclo de vida propio)
- Dado un lote de notificaciones a mostrar en una ventana corta, decide si se muestran individuales o se colapsan en un resumen, según el umbral de Fase 0.
- Usado tanto por `NotificationPublisherReceiver` (caso de disparo normal) como por `ReconciliationWorker` (caso de recuperación masiva tras restauración).

### `ReasignarMedicionUseCase` / `ReprogramarPagoUseCase`
- Punto único de entrada para la acción "Reasignar".
- Orden estricto: 1) actualizar Room (fuente de verdad) → 2) leer el estado recién guardado → 3) llamar a `NotificationScheduler` con los datos frescos.
- Nunca llama a `AlarmManager` directamente ni construye el `PendingIntent` por su cuenta.

### `PermissionsHealthMonitor`
- Componente ligero (puede ser una clase simple invocada desde `onResume` del Dashboard, no un Worker) que verifica en cada apertura de la app:
  - `POST_NOTIFICATIONS` concedido.
  - `AlarmManager.canScheduleExactAlarms()`.
  - Estado de optimización de batería (`PowerManager.isIgnoringBatteryOptimizations()`), informativo, no bloqueante.
  - Estado habilitado de cada `NotificationChannel` individualmente (no solo el permiso global — gap de canal deshabilitado por separado).
- Expone un `StateFlow<EstadoSaludNotificaciones>` consumido por el banner del Dashboard y por la pantalla "Salud de notificaciones".

---

## 2. Modelo de datos nuevo

### Tabla `notificacion_log`
```
id: Long (PK autoincrement)
tipoAlerta: String  // PAGO_VENCIMIENTO | RECORDATORIO_MEDIDA | SISTEMA
entidadId: Long?     // pagoId o medidaRegistroId/clienteId según tipo
clienteId: Long?
requestCode: Int
estado: String       // PROGRAMADA | DISPARADA | MOSTRADA | MOSTRADA_CON_DATOS_POSIBLEMENTE_DESACTUALIZADOS
                     // | DESCARTADA_PERMISO | CANCELADA | ERROR
fechaProgramada: Long?   // epoch millis, cuándo se planeó disparar
fechaEvento: Long        // epoch millis, cuándo se registró esta fila
detalle: String?         // mensaje de error o motivo de cancelación
origen: String       // CRUD_INMEDIATO | RECONCILIACION | BOOT | REASIGNACION | MANUAL_TEST
pendienteSync: Boolean   // si se decide respaldar el log también en Firestore (opcional, ver nota)
```
> Nota: el log puede mantenerse **solo local** (Room) sin subir a Firestore, dado que es información técnica de diagnóstico, no de negocio. Se recomienda purgarlo también con una ventana de retención (ej. 60 días) vía un job simple, para no crecer indefinidamente en el eMMC del Moto G22.

### Enum `TipoAlerta`
```kotlin
enum class TipoAlerta { PAGO_VENCIMIENTO, RECORDATORIO_MEDIDA, SISTEMA }
```

### `NotificacionPayload` (data class serializada como extras del Intent)
```kotlin
data class NotificacionPayload(
    val clienteId: Long,
    val nombreCliente: String,
    val tipoAlerta: TipoAlerta,
    val detalleCorto: String,     // ej. "$80,000 vence en 2 días" o "Toma de medidas pendiente"
    val payloadPotencialmenteDesactualizado: Boolean = false
)
```

---

## 3. Esquema determinista de `requestCode`

```kotlin
fun generarRequestCode(tipo: TipoAlerta, entidadId: Long): Int {
    // Reserva 3 bloques de 10,000,000 IDs, uno por tipo.
    // Soporta hasta 10 millones de entidades por tipo, muy por encima
    // de cualquier escala realista para un gimnasio.
    require(entidadId in 0 until 10_000_000) { "entidadId fuera de rango soportado" }
    return tipo.ordinal * 10_000_000 + entidadId.toInt()
}
```
Documentar este esquema en el propio código como comentario, y en un test unitario que verifique ausencia de colisiones para el rango completo de `TipoAlerta`.

---

## 4. Flujo de datos: registrar un pago con vencimiento próximo

```
UI (RegistrarPagoScreen)
   │
   ▼
PagoViewModel.registrarPago()
   │
   ▼
PagoRepository.registrarPago()  ── inserta en Room, recalcula estado
   │
   ▼ (dentro de la misma unidad de trabajo, no al final del día)
NotificationScheduler.scheduleAlertaPago(pagoId, clienteId, fechaDisparo, payload)
   │
   ├─► Mutex.withLock { ... }
   ├─► Verifica SettingsRepository (switch activado?)
   ├─► Verifica franja horaria válida (Fase 0) — ajusta fechaDisparo si cae fuera de rango
   ├─► AlarmManager.setExactAndAllowWhileIdle(requestCode=generarRequestCode(...), ...)
   └─► NotificacionLogDao.insertar(estado=PROGRAMADA, origen=CRUD_INMEDIATO)
```

## 5. Flujo de datos: disparo y reconciliación

```
AlarmManager dispara el PendingIntent
   │
   ▼
NotificationPublisherReceiver.onReceive()
   │
   ├─► Lee extras (NotificacionPayload)
   ├─► Verifica canal/permiso habilitado → si no, log DESCARTADA_PERMISO y retorna
   ├─► (opcional) lectura corta a Room con timeout si payload puede estar desactualizado
   ├─► NotificationGrouper decide individual vs resumen
   ├─► NotificationManagerCompat.notify(...)
   └─► NotificacionLogDao.actualizar(estado=MOSTRADA)

--- en paralelo, cada N horas ---

ReconciliationWorker.doWork()
   │
   ├─► Calcula alarmas ideales desde Room (estadoCuenta, próximas mediciones)
   ├─► Compara contra NotificacionLogDao.obtenerProgramadasActivas()
   ├─► Para cada discrepancia: NotificationScheduler.schedule(...) o .cancel(...)
   └─► Log resumen tipo SISTEMA
```

---

## 6. Puntos de integración con código/documentación existente

| Componente objetivo | Reemplaza / integra con |
|---|---|
| `NotificationScheduler` | `notifications/NotificationScheduler.kt` actual — se refactoriza para ser la única puerta de entrada y ganar el Mutex + log |
| `ReconciliationWorker` | Sustituye la dependencia exclusiva de `workers/DailyCheckWorker.kt`; `DailyCheckWorker` puede quedar como disparador adicional de baja frecuencia o fusionarse |
| `AlarmsRestoreReceiver` | Reemplaza/amplía `BootCompletedReceiver.kt` actual (que solo escucha `BOOT_COMPLETED`) |
| `NotificationPublisherReceiver` | Reemplaza la lógica hoy repartida entre `PagoVencimientoReceiver.kt` y `MedicionReminderReceiver.kt` (se recomienda unificar en un solo receiver parametrizado por `TipoAlerta` para no duplicar el manejo de permisos/log) |
| `cancelarAlertaPago` (hoy código muerto) | Se convierte en `NotificationScheduler.cancel(PAGO_VENCIMIENTO, pagoId)`, invocado desde `ClienteRepository.cancelarPlanActual()`, `PagoRepository.eliminarPago()`, `ClienteRepository.aplicarAccionInactividad()` (ELIMINAR y DESHABILITAR) |
| `notificacion_log` | Tabla nueva en `AppDatabase`, requiere su propia migración Room |
| `AjustesNotificacionesScreen.kt` | Los switches deben leerse desde `SettingsRepository` dentro de `NotificationScheduler`, no solo guardarse en DataStore sin efecto |

---

## 7. Qué NO cambia

- Room sigue siendo la única fuente de verdad (ADR #7 de `decisiones-tecnicas.md`).
- No se introduce FCM ni dependencia de servidores push externos; todo sigue siendo local vía `AlarmManager`/`WorkManager`.
- El PIN local sigue siendo el único mecanismo de acceso (ADR #2), sin relación con este sistema.
- La estructura de 3 franjas diarias para medidas (8:00/14:00/17:30) se mantiene; solo se añade acotamiento horario para las alertas de pago.

Ver `012-checklist-qa-notificaciones.md` para el plan de pruebas de esta arquitectura.
