# Plan de Implementación — Sistema de Notificaciones Confiable

**Fecha:** 2026-09-26
**Objetivo de negocio:** El administrador debe poder confiar ciegamente en que recibirá cada alerta de cobro y cada recordatorio de medición, sin ventanas silenciosas ni fallos invisibles.
**Insumos:** `003-decisiones-pendientes.md`, `008-reporte-notificaciones-problemas.md`, `problems.md`/`solutions.md` (gaps ya documentados), `009-gaps-adicionales-notificaciones.md` (gaps nuevos).
**Principio rector:** ninguna fase se considera "terminada" sin sus criterios de aceptación verificados en el Moto G22 físico, no solo en emulador.

---

## Fase 0 — Fundaciones y decisiones de negocio previas (bloqueantes)

Antes de tocar código, se requieren decisiones explícitas del propietario (si no se toman, el agente debe usar el default indicado y dejarlo documentado):

| Decisión | Default si no se responde |
|---|---|
| ¿Franja horaria válida para alertas de pago (evitar madrugada)? | 6:00–21:00; si el cálculo cae fuera, mover al inicio de la franja más cercana |
| ¿Canales de notificación separados por tipo (pago/medida/sistema)? | Sí, 3 canales desde el día 1 (ver D4 de 009) |
| ¿Notificación de resumen diario si hay más de N alertas? | Sí, umbral N=6 |
| ¿Visibilidad de contenido en pantalla bloqueada? | `VISIBILITY_PRIVATE` con texto genérico ("Tienes una alerta de Mary Fitness") |
| ¿Frecuencia del job de reconciliación? | Cada 8 horas |
| ¿Se solicita exención de optimización de batería? | Sí, de forma opcional y explicada (no bloqueante) |

**Entregable de la fase:** documento corto `ADR-notificaciones.md` con estas respuestas fijadas (puede fusionarse con `decisiones-tecnicas.md` como ADR #8).

---

## Fase 1 — Cimientos: modelo de datos y esquema de IDs (bloqueante para todo lo demás)

**Por qué va primero:** casi todos los gaps (B1, B2, B4, C1, E1) dependen de tener un esquema de identificación y persistencia correcto desde el inicio; construir encima de un esquema ambiguo obliga a rehacer trabajo después.

### Tareas
1. **Esquema determinista de `requestCode` de `PendingIntent`** (gap B1):
   - Definir función pura `generarRequestCode(tipoAlerta: TipoAlerta, entidadId: Long): Int`.
   - Ejemplo: `tipoAlerta.ordinal * 10_000_000 + entidadId.toInt()` (documentar el límite de `entidadId` soportado y qué pasa si se excede).
   - Escribir tests unitarios que prueben ausencia de colisión para rangos realistas de IDs.
2. **Tabla `notificacion_log`** (gap C1):
   - Columnas sugeridas: `id, tipoAlerta, entidadId, clienteId, estado (PROGRAMADA/DISPARADA/MOSTRADA/DESCARTADA_PERMISO/CANCELADA/ERROR), fechaProgramada, fechaEvento, detalleError, requestCode`.
   - Cada punto del pipeline (`NotificationScheduler.schedule/cancel`, el `Receiver` al disparar, al mostrar o al fallar) debe escribir una fila.
   - Migración Room correspondiente (ver Fase 5 para cuidado de migraciones).
3. **Canales de notificación versionados** (gap D3/D4):
   - Crear 3 canales desde el inicio: `pagos_v1`, `medidas_v1`, `sistema_v1`.
   - Documentar en código un comentario explícito: "Si se necesita cambiar importancia/sonido, crear `_v2` y migrar, nunca reutilizar el ID".
4. **Mutex único para escritura de alarmas** (gap B2):
   - Un solo punto de entrada (`NotificationScheduler`) protegido por `Mutex` de coroutines o `synchronized` para evitar reprogramaciones concurrentes.

### Criterios de aceptación
- [ ] Ningún `requestCode` colisiona entre tipos de alerta para al menos 100,000 clientes simulados.
- [ ] Toda llamada a `schedule()`/`cancel()` deja rastro en `notificacion_log`.
- [ ] Los 3 canales existen desde el primer arranque con importancia correcta y documentada como inmutable.

---

## Fase 2 — Disparo confiable: permisos, exactitud y ciclo de vida del SO

Cubre los gaps ya documentados en 008/problems.md (24h delay, permisos 13+/14+, wipe por update, trampolines, ANR) **más** los nuevos de la sección A de `009`.

### Tareas
1. **`AlarmsRestoreReceiver` universal** (ya propuesto en `solutions.md #1`, ampliar):
   - Intent filters: `BOOT_COMPLETED`, `MY_PACKAGE_REPLACED`, `TIME_SET`, `TIMEZONE_CHANGED`, y añadir manejo consciente de **Direct Boot** (gap A4): registrar también un receiver "device protected storage aware" si aplica, o documentar explícitamente la ventana de riesgo residual si no se implementa Direct Boot completo.
   - Lanza `RestoreAlarmsWorker` (WorkManager, no lógica directa en el receiver).
2. **Chequeo inmediato post-CRUD** (ya documentado en 008 §9.1):
   - Todo caso de uso de creación/edición/eliminación de pagos y medidas invoca `NotificationScheduler` de forma síncrona (dentro de la misma transacción lógica), no espera al worker diario.
3. **Flujo de permisos vivo, no solo de onboarding** (gaps A1, A2 nuevos):
   - Verificar `POST_NOTIFICATIONS` y `canScheduleExactAlarms()` en **cada** apertura de la app (`onResume` del Dashboard), no solo en el onboarding.
   - Si alguno falta, mostrar banner persistente y no descartable (similar al de deuda) con acción directa a Ajustes.
   - Detectar señales de hibernación/auto-revoke: si el permiso estaba `true` y ahora es `false` sin que el usuario lo haya tocado desde la app, registrar evento en `notificacion_log` como `DESCARTADA_PERMISO` para diagnóstico.
4. **Exención de optimización de batería (opt-in)** (gap A3):
   - Pantalla explicativa en Ajustes > Seguridad (o nueva sección "Notificaciones confiables") con botón que lanza `ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`.
   - No bloquear el uso de la app si se rechaza, pero dejar indicador visible del estado.
5. **`NotificationPublisherReceiver` sin I/O pesado** (ya documentado, `solutions.md #8`):
   - Extras mínimos en el `Intent`: `clienteId, nombreCliente, tipoAlerta, montoODetalle` — pero ver Fase 3 para el problema de payload congelado.
6. **Enrutamiento sin trampolín** (ya documentado, `solutions.md #6`):
   - Todas las acciones de notificación usan `PendingIntent.getActivity()` con deep link directo, flags `FLAG_UPDATE_CURRENT or FLAG_IMMUTABLE`.

### Criterios de aceptación
- [ ] Tras forzar reinicio del Moto G22 con alarmas programadas, todas se restauran antes de la primera interacción del usuario (o quedan registradas como pendientes de restaurar hasta el primer desbloqueo, con log explícito).
- [ ] Editar un pago/medida dispara verificación de notificación en <2 segundos, sin esperar al ciclo diario.
- [ ] Revocar manualmente el permiso de alarmas exactas desde Ajustes del sistema y reabrir la app muestra el banner en la siguiente apertura (sin necesidad de tocar ningún cliente).
- [ ] Actualizar la app vía `ActualizacionManager` no deja ningún cliente sin sus alarmas reprogramadas tras el primer arranque post-update.

---

## Fase 3 — Integridad del contenido y de la reasignación

Cubre gaps E1, E2 y el ya documentado de persistencia en "Reasignar" (`solutions.md #5`).

### Tareas
1. **"Reasignar" como fuente única de verdad** (ya documentado):
   - El flujo actualiza primero `MedidaRepository`/`ClienteRepository` (Room), y solo después reprograma la alarma leyendo el nuevo estado — nunca al revés.
2. **Invalidar payload al editar una entidad con alarma activa** (gap E1, nuevo):
   - Al editar un pago o medida con alarma pendiente, **reemplazar** el `PendingIntent` completo (cancelar + reprogramar con extras nuevos), no solo la fecha.
   - Alternativa aceptada: el receiver hace una lectura corta a Room con `withTimeoutOrNull(300ms)` y usa el extra como *fallback* si la lectura falla o tarda; documentar la decisión tomada.
3. **Cancelación completa en soft-delete y purga** (gap E2, nuevo — extiende el hallazgo de `cancelarAlertaPago`):
   - Al ocultar (`oculto=true`), deshabilitar (`habilitado=false`) o purgar un cliente: cancelar **tanto** la alarma de pago **como** la de medición asociadas.
   - Al purgar físicamente (6 meses), verificar adicionalmente que no quede ningún `PendingIntent` huérfano (auditar contra `notificacion_log`).

### Criterios de aceptación
- [ ] Editar el monto de un pago con alerta programada y esperar a que dispare muestra el monto **nuevo**, no el que existía al momento de programar.
- [ ] Ocultar un cliente con medición pendiente no genera ninguna notificación posterior para ese cliente.
- [ ] Reasignar una medición y matar el proceso de la app inmediatamente después conserva el cambio (verificar en Room, no solo en `AlarmManager`).

---

## Fase 4 — Auto-reparación y observabilidad (la red de seguridad)

Cubre B4 y C1/C2/C3 — los gaps más críticos según `009`.

### Tareas
1. **`ReconciliationWorker` periódico**:
   - `WorkManager` periódico (cada 8h, configurable en Fase 0) que:
     a. Calcula el conjunto ideal de alarmas que deberían existir (según `estadoCuenta`, `plazoMaximoPago`, próxima medición de cada cliente visible).
     b. Compara contra lo realmente registrado en `notificacion_log` con estado `PROGRAMADA`.
     c. Reprograma lo faltante y cancela lo que ya no debería existir (ej. cliente que pagó).
   - Debe ser **idempotente**: correr dos veces seguidas no debe duplicar alarmas (usa el mismo `requestCode` determinista de la Fase 1, así que reprogramar = reemplazar).
2. **Pantalla "Salud de notificaciones"** (gap C2, nueva, en Ajustes):
   - Muestra: estado de `POST_NOTIFICATIONS`, estado de alarma exacta, estado de optimización de batería, cantidad de alarmas activas, última corrida de `ReconciliationWorker` y cuántas discrepancias corrigió.
3. **Botón de notificación de prueba** (gap C3, nuevo):
   - Dispara una notificación inmediata de cada canal (pago/medida/sistema) para que el admin verifique visualmente que su dispositivo específico las muestra.
4. **Auditoría accesible para soporte**:
   - Exportar `notificacion_log` (últimos 30 días) a JSON/CSV desde Ajustes > Acerca de, para diagnóstico remoto si el propietario reporta un fallo.

### Criterios de aceptación
- [ ] Cancelar manualmente una alarma vía ADB (`adb shell dumpsys alarm`) y esperar a la siguiente corrida de reconciliación restaura la alarma sin intervención del usuario.
- [ ] La pantalla de salud refleja en tiempo real cambios de permisos hechos desde Ajustes del sistema.
- [ ] El botón de prueba muestra notificación en menos de 5 segundos en los 3 canales.

---

## Fase 5 — Volumen, UX y casos de escala

Cubre D1, D2, D3, B3 y el flooding de restauración (`solutions.md #7`).

### Tareas
1. **Filtro de caducidad** (ya documentado, `solutions.md #7`): no programar/mostrar alertas push para eventos vencidos hace más de 48h; gestionarlos solo como badge/contador en Dashboard.
2. **Notificación resumen** (gap D1, nuevo): si en una misma corrida se generarían ≥N alertas del mismo tipo el mismo día (N definido en Fase 0), agrupar en una sola notificación con `InboxStyle` en vez de N notificaciones individuales.
3. **Franja horaria acotada para alertas de pago** (gap D2, nuevo): normalizar la hora de disparo de la alerta de vencimiento a la franja definida en Fase 0, independientemente de la hora exacta en que se registró el pago original.
4. **Prevención de saturación de `AlarmManager`** (gap B3, nuevo): si el `ReconciliationWorker` detecta más de X alarmas a programar en una sola corrida (ej. tras una restauración masiva), programarlas en lotes escalonados por pocos segundos en vez de todas en la misma llamada, y usar la notificación resumen del punto 2 para el resultado visible al usuario.

### Criterios de aceptación
- [ ] Restaurar un backup con 50 pagos vencidos históricos no genera más de 1 notificación de alerta de pago (resumen), no 50.
- [ ] Ninguna alerta de pago se dispara antes de las 6:00 ni después de las 21:00 (o la franja definida).
- [ ] Un día con 10 vencimientos simultáneos produce 1 notificación resumen navegable al detalle, no 10 notificaciones sueltas.

---

## Fase 6 — Migraciones seguras y privacidad

Cubre E3 y F1.

### Tareas
1. **Orden de arranque seguro** (gap E3, nuevo): ningún receiver/worker de notificaciones accede a Room antes de confirmar que todas las migraciones (incluida `MIGRACION_3_4` del plan Firebase) completaron exitosamente; usar un flag de "esquema listo" en `DataStore` verificado al inicio de `MaryFitnessApplication`.
2. **Privacidad en lockscreen** (gap F1, nuevo): `setVisibility(VISIBILITY_PRIVATE)` en todos los canales; el texto público genérico no debe incluir nombre de cliente ni monto.

### Criterios de aceptación
- [ ] Con el teléfono bloqueado, la notificación visible no revela nombre de cliente ni monto; al desbloquear, sí se ve el detalle completo.
- [ ] Forzar un fallo de migración en un dispositivo de prueba no provoca crash del `ReconciliationWorker` (falla de forma controlada y reintenta tras confirmar migración).

---

## Orden recomendado y dependencias

```
Fase 0 (decisiones) ──► Fase 1 (cimientos: IDs, log, canales)
                              │
                              ├──► Fase 2 (disparo confiable / permisos / ciclo de vida)
                              │         │
                              │         └──► Fase 3 (integridad de contenido / reasignar)
                              │
                              └──► Fase 4 (reconciliación / observabilidad)  ← depende de Fase 1
                                        │
                                        └──► Fase 5 (volumen / UX)
                                                  │
                                                  └──► Fase 6 (migraciones / privacidad)
```

Fase 1 es estrictamente bloqueante para Fase 4 (necesita el log y el esquema de IDs). Fases 2 y 3 pueden avanzar en paralelo una vez completada la Fase 1. Fase 4 (reconciliación) es la de mayor prioridad de negocio después de los cimientos: es la única que convierte "notificaciones que probablemente funcionan" en "notificaciones que se auto-corrigen si fallan".

---

## Definición de "hecho" para todo el sistema (no solo por fase)

El sistema de notificaciones se considera confiable cuando, simultáneamente:

1. Existe un log auditable de cada alarma programada/disparada/mostrada/cancelada (Fase 1/4).
2. Un job de reconciliación corrige automáticamente cualquier discrepancia sin intervención humana (Fase 4).
3. El admin tiene una forma de auto-verificar la salud del sistema sin salir de la app (Fase 4).
4. Ninguna acción de negocio (pago, medida, cancelación, ocultar cliente) puede dejar una alarma huérfana o desactualizada (Fase 2/3).
5. El volumen de notificaciones no incentiva al admin a desactivar el canal completo (Fase 5).
6. Los datos sensibles no quedan expuestos en pantalla de bloqueo (Fase 6).

Ver `011-arquitectura-objetivo-notificaciones.md` para el diseño técnico de los componentes mencionados y `012-checklist-qa-notificaciones.md` para el plan de pruebas que verifica estos criterios.
