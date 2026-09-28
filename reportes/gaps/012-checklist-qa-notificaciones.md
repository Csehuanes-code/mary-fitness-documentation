# Checklist de QA — Sistema de Notificaciones

**Fecha:** 2026-09-26
**Complementa:** `010-plan-implementacion-notificaciones.md`, `011-arquitectura-objetivo-notificaciones.md`
**Regla general:** ninguna prueba de esta lista se considera válida si solo se ejecutó en emulador. El dispositivo objetivo (Moto G22, Android 12/13) debe usarse para todo lo marcado como **[FÍSICO]**.

---

## 1. Pruebas unitarias (ya existen ~118, ampliar con lo siguiente)

- [ ] `generarRequestCode()`: sin colisiones para 100,000 combinaciones simuladas de `tipoAlerta` x `entidadId`.
- [ ] `NotificationScheduler.schedule()` respeta el switch desactivado del `SettingsRepository` (no programa, registra `CANCELADA`).
- [ ] `NotificationScheduler.schedule()` acota la hora de disparo a la franja configurada (Fase 0) cuando el cálculo cae fuera de rango.
- [ ] `ReconciliationWorker`: dado un set de clientes con `estadoCuenta` variado, calcula exactamente el conjunto ideal de alarmas esperado (sin faltantes ni sobrantes).
- [ ] `ReconciliationWorker` es idempotente: ejecutarlo dos veces seguidas sobre el mismo estado no duplica filas en `notificacion_log` ni reprograma innecesariamente.
- [ ] Filtro de caducidad: alertas con más de 48h de vencimiento no se programan (solo se reflejan como badge).
- [ ] `NotificationGrouper`: por debajo del umbral N, genera notificaciones individuales; en o por encima, genera un resumen `InboxStyle`.
- [ ] Cancelación en cascada: `cancelTodasDeCliente()` cancela tanto la alerta de pago como la de medida para el mismo cliente.

## 2. Pruebas de integración (Room in-memory)

- [ ] Migración a `notificacion_log` no rompe datos existentes (correr sobre una copia de `registros-fitness-v2.json` migrada a Room).
- [ ] `PagoRepository.eliminarPago()` invoca la cancelación de notificación (verificar vía mock de `NotificationScheduler`, no vía `AlarmManager` real).
- [ ] `ClienteRepository.aplicarAccionInactividad(ELIMINAR)` y `(DESHABILITAR)` cancelan ambas alarmas del cliente (pago + medida).
- [ ] Editar el monto de un pago con alarma activa reemplaza el `PendingIntent` (no solo la fecha) — verificar que el nuevo payload se use.

## 3. Pruebas instrumentadas / manuales en dispositivo **[FÍSICO — Moto G22]**

### 3.1 Ciclo de vida del sistema operativo
- [ ] Programar una alarma, reiniciar el dispositivo, verificar que se restaura sin abrir la app manualmente (o queda correctamente marcada como pendiente de restaurar tras el primer desbloqueo si no se implementó Direct Boot completo).
- [ ] Simular actualización de la app (`adb install -r`) con alarmas activas: verificar que sobreviven o se restauran automáticamente al primer arranque post-update.
- [ ] Cambiar la hora del sistema manualmente (`TIME_SET`) y verificar reprogramación de alarmas de medida a las franjas correctas.
- [ ] Cambiar la zona horaria del dispositivo y verificar que las 3 franjas diarias de medida se recalculan correctamente.
- [ ] Forzar Doze (`adb shell dumpsys deviceidle force-idle`) y verificar que una alarma exacta programada previamente sigue disparando (`setExactAndAllowWhileIdle`).

### 3.2 Permisos
- [ ] Denegar `POST_NOTIFICATIONS` en el onboarding: verificar que el banner persistente aparece y no puede descartarse sin conceder o reconocer explícitamente el impacto.
- [ ] Conceder el permiso, luego revocarlo manualmente desde Ajustes del sistema, reabrir la app: el banner debe reaparecer sin que el usuario haya tocado ningún cliente.
- [ ] En Android 14+, verificar que `SCHEDULE_EXACT_ALARM` se solicita explícitamente con `ACTION_REQUEST_SCHEDULE_EXACT_ALARM` y que denegarlo produce el fallback documentado (`setWindow` inexacto) sin crash.
- [ ] Revocar el permiso de alarma exacta después de tenerlo concedido; reabrir la app y verificar detección en `PermissionsHealthMonitor` sin necesidad de programar una nueva alarma para descubrirlo.
- [ ] Deshabilitar un canal específico (ej. "Pagos") desde Ajustes del sistema sin tocar el permiso global: verificar que `NotificationPublisherReceiver` lo detecta y registra `DESCARTADA_PERMISO`, no un fallo silencioso.
- [ ] Simular auto-revoke de permisos por inactividad prolongada (`adb shell cmd permission revoke-app-permissions` o dejar la app sin abrir 3+ meses en un dispositivo de prueba con fecha adelantada) y verificar que el sistema lo detecta al reabrir.

### 3.3 Volumen y agrupamiento
- [ ] Crear 10 clientes con vencimiento el mismo día: verificar que se muestra 1 notificación resumen, no 10 individuales.
- [ ] Restaurar un backup simulado con 50 pagos vencidos históricos: verificar que no se genera un aluvión de notificaciones (filtro de caducidad + agrupamiento + escalonado).
- [ ] Verificar que ninguna alerta de pago se dispara fuera de la franja horaria configurada, incluso si el pago original se registró a medianoche.

### 3.4 Contenido y reasignación
- [ ] Programar alerta de pago, editar el monto del pago antes de que dispare, verificar que la notificación mostrada refleja el monto actualizado.
- [ ] Tocar "Reasignar" desde la notificación, matar el proceso de la app inmediatamente (`adb shell am force-stop`), reabrir la app y verificar que la nueva fecha persiste en Room (no solo en la alarma en memoria).
- [ ] Ocultar un cliente con medición pendiente: verificar que no llega ninguna notificación posterior de "Revisar"/"Reasignar" para ese cliente, y que tocar una notificación previa ya en la bandeja (si existiera) no crashea al navegar a un cliente oculto.

### 3.5 Observabilidad
- [ ] Cancelar manualmente una alarma vía `adb shell dumpsys alarm | grep maryfitness` y forzar la ejecución del `ReconciliationWorker`: verificar que la alarma se restaura y queda registrada en `notificacion_log`.
- [ ] Verificar que la pantalla "Salud de notificaciones" refleja en tiempo real el estado real de permisos tras cambiarlos desde Ajustes del sistema (sin reiniciar la app).
- [ ] Usar el botón de notificación de prueba y verificar recepción visual en los 3 canales en menos de 5 segundos.
- [ ] Exportar el log de notificaciones desde Ajustes > Acerca de y verificar que el archivo generado es legible y contiene las últimas alarmas programadas/disparadas.

### 3.6 Privacidad
- [ ] Con el teléfono bloqueado, verificar que el contenido visible de la notificación no incluye nombre de cliente ni monto (`VISIBILITY_PRIVATE`).
- [ ] Al desbloquear, verificar que sí se muestra el detalle completo al desplegar/abrir la notificación.

### 3.7 Trampolines y navegación
- [ ] Tocar "Revisar" y "Reasignar" desde una notificación con la app **cerrada por completo** (no solo en background) en Android 12+: verificar que navega directamente al detalle del cliente sin bloqueo del sistema ni retraso perceptible.

---

## 4. Matriz de escenarios de estrés (previo a cada release)

| Escenario | Resultado esperado |
|---|---|
| App nunca abierta en 6 meses (simulado con fecha adelantada) | Al reabrir, se detecta hibernación/revocación y se re-solicitan permisos con explicación clara |
| Batería en modo ahorro extremo activado por el usuario | Alarmas exactas siguen disparando (`setExactAndAllowWhileIdle`); worker de reconciliación puede demorarse pero no se pierde permanentemente |
| Reinstalación completa (no update) con backup en Firestore | Restauración no genera flooding de notificaciones para el historial completo |
| Cliente con pago y medida venciendo el mismo día | Ambas alertas llegan, en canales/horarios diferenciados, sin colisión de `requestCode` |
| Edición rápida y repetida de un mismo pago (3 ediciones en 10s) | Solo queda una alarma activa final con el dato correcto, sin duplicados ni condiciones de carrera |
| Cambio de zona horaria en pleno viaje del administrador | Las 3 franjas diarias de medida se recalculan sin duplicar ni perder el recordatorio del día |

---

## 5. Criterios de salida (Definition of Done a nivel release)

Un release que toque el módulo de notificaciones **no se publica** hasta que:

1. Las pruebas unitarias e instrumentadas de las secciones 1-2 pasan al 100%.
2. Al menos las pruebas 3.1, 3.2 y 3.4 completas se ejecutaron en el Moto G22 físico (no solo emulador).
3. El `notificacion_log` de una sesión de prueba completa no muestra ningún estado `ERROR` no explicado.
4. La pantalla "Salud de notificaciones" reporta estado correcto tras la sesión de pruebas manuales.
5. Se documentó explícitamente cualquier limitación conocida (ej. Direct Boot no implementado) en el reporte de release, no se asume tácitamente.

---

## 6. Plantilla de reporte de incidente (para cuando el propietario reporte "no me llegó la notificación")

Al recibir un reporte de este tipo, el flujo de diagnóstico debe ser:

1. Pedir/exportar el `notificacion_log` del dispositivo del propietario (vía Ajustes > Acerca de > Exportar log).
2. Buscar la entidad afectada (`clienteId`/`pagoId`) en el log: ¿existe una fila `PROGRAMADA`? ¿Llegó a `DISPARADA`/`MOSTRADA`? ¿Se marcó `DESCARTADA_PERMISO`?
3. Si no existe ninguna fila para esa entidad, el fallo está en la programación (revisar `ReconciliationWorker` y el chequeo inmediato post-CRUD).
4. Si existe `PROGRAMADA` pero nunca `DISPARADA`, el fallo está en `AlarmManager`/OEM (revisar optimización de batería, Doze, exención).
5. Si existe `DISPARADA` pero no `MOSTRADA`, el fallo está en permisos de notificación o canal deshabilitado.
6. Documentar el hallazgo como nuevo gap si no encaja en ninguna categoría ya cubierta por `009-gaps-adicionales-notificaciones.md`.
