# Reporte de Estado — Gaps del Sistema de Notificaciones

**Base documental:** `009-gaps-adicionales` · `010-plan-implementacion` · `011-arquitectura-objetivo` · `012-checklist-qa` (+ `008-reporte-notificaciones-problemas`)
**Verificación:** código real (`app/src/main`), manifest, tests, y `documentacion/reportes/`
**Build:** `./gradlew assembleDebug testDebugUnitTest` → **BUILD SUCCESSFUL, 320 tests, 0 fallos** (25 clases) · 3 APKs (arm64-v8a, armeabi-v7a, x86_64)
*(Los 320 tests corresponden al estado de HEAD en `61e46d0`. El árbol de trabajo local suma 13 tests más en `PagoRepositoryTest`, que pertenecen al trabajo de fechas de pago y no forman parte de este sistema de notificaciones.)*

---

## 1. Veredicto por fase

| Fase | Alcance | Estado | % |
|---|---|---|---|
| **Fase 0** | Decisiones de negocio (ADR) | 🟢 Completa | 100% |
| **Fase 1** | Cimientos (requestCode, log, canales, mutex) | 🟢 Completa | 95% |
| **Fase 2** | Disparo confiable (permisos, ciclo de vida SO) | 🟢 Completa | 90% |
| **Fase 3** | Integridad de contenido y reasignación | 🟡 Parcial | 75% |
| **Fase 4** | Auto-reparación y observabilidad | 🟢 Completa en código | 100% |
| **Fase 5** | Volumen, UX y escala | 🟢 Completa en código | 100% |
| **Fase 6** | Migraciones seguras y privacidad | 🟢 Completa en código | 100% |

**Global: ~95% en código.** El patrón que quedaba (fases técnicas pulidas, Fase 4 vacía) se invirtió: la auto-reparación y la observabilidad están implementadas y cubiertas por tests. **Lo que falta ya no es código: es la verificación en el Moto G22 físico**, que es donde los tres estados que deciden si un aviso llega (permiso, alarma exacta, canal) solo se pueden comprobar de verdad.

---

## 2. Lo que YA ESTÁ HECHO (verificado en código)

### Fase 1 — Cimientos ✅
- **Esquema determinista de `requestCode`** — `NotificationScheduler.kt:68-73`: `tipo.ordinal * 10_000_000 + entidadId.toInt()`, con `require(entidadId in 0 until 10_000_000)`. Cubierto por 3 clases de test (`RequestCodeTest`, `RequestCodeCoberturaTest`, `RequestCodeUnicidadTest`) con 100 000 combinaciones. **Gap B1 cerrado.**
- **Tabla `notificacion_log`** — `NotificacionLogEntity.kt`, `NotificacionLogDao.kt`, `MIGRACION_3_4` en `AppDatabase.kt:77-101` (con índice en `requestCode`), DB en v4. **Gap C1 cerrado.**
- **3 canales versionados** — `canal_pagos_v1` (HIGH), `canal_medidas_v1` (HIGH), `canal_sistema_v1` (LOW), con política de no reutilizar ID documentada. **Gaps D3/D4 cerrados.**
- **Mutex** — `ReentrantLock` en `NotificationScheduler.kt:82`, aplicado en los 4 métodos de programa/cancelación. **Gap B2 cerrado.**

### Fase 2 — Disparo confiable ✅
- **`AlarmsRestoreReceiver`** (`AlarmsRestoreReceiver.kt:31-43`) con los 4 intent-filters: `BOOT_COMPLETED`, `MY_PACKAGE_REPLACED`, `TIME_SET`, `TIMEZONE_CHANGED`. No toca Room, encola `RestoreAlarmsWorker` **expedited** con nombre de work propio (`WorkScheduler.kt:53-63`). **Wipe por actualización / boot / TZ cerrados.**
- **Chequeo inmediato post-CRUD** — 9 call-sites: `PagoRepository.kt:142,192,250` y `MedidaRepository.kt:47,67,74,81,157`. **Delay de 24h cerrado.**
- **Permisos vivos en cada apertura** — `DashboardScreen` observa `ON_RESUME` → `NotificationHealthViewModel.refrescar()` → banner persistente `NotificationHealthBanner`. **Gaps A1/A2 cerrados.**
- **Exención de batería opt-in** — `TarjetaOptimizacionBateria` (`AjustesNotificacionesScreen.kt`). Usa `ACTION_IGNORE_BATTERY_OPTIMIZATION_SETTINGS` en vez de `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`, **desviación deliberada y documentada** (evita dependencia de permiso Play). **Gap A3 cerrado con desviación.**
- **Receiver sin I/O** — `NotificationPublisherReceiver.kt` solo lee extras; unificado para pago y medida. La única excepción es el refresco acotado del payload, que va en `goAsync()` con timeout de 3 s. **ANR cerrado.**
- **Sin trampolín** — navegación siempre con `getActivity` + `FLAG_UPDATE_CURRENT or FLAG_IMMUTABLE`. El `getBroadcast` del botón "Okay" es correcto: no navega, solo cierra.

### Fase 4 — Auto-reparación y observabilidad ✅ (nuevo)
- **B4 — Reconciliación real** — `ReconciliadorAlarmas.kt` compara el conjunto **ideal** (`CalculoAlarmasIdeales.kt`, función pura) contra el que el log dice armado, y devuelve solo las diferencias con motivo escrito: `PROGRAMAR`, `REPROGRAMAR` (perdida), `REALINEAR` (fecha movida), `REARMAR`, `CANCELAR` (huerfana), `MANTENER`. `DailyCheckWorker` ejecuta esas diferencias y escribe un resumen por corrida.
- **Periodividad 8 h** — `WorkScheduler.PERIODICIDAD_RECONCILIACION_HORAS = 8` con `ExistingPeriodicWorkPolicy.UPDATE` (migra las instalaciones que tenían el work de 24 h).
- **`forzarRearmado`** — Android no permite preguntar si una alarma concreta sigue armada. Las corridas `RESTAURACION` y `MANUAL` rearman todo **cuyo último estado sea `PROGRAMADA`**, reusando el mismo `requestCode` (que *reemplaza* en vez de duplicar). Un `MOSTRADA` no se rearma: sería un aviso duplicado en cada reinicio. Cubierto por 3 tests en `ReconciliadorAlarmasTest`.
- **C1 — Log auditable** — los 4 métodos de lectura del DAO ya tienen uso: `NotificacionLogRepository.leerAlarmasRegistradas()` (entrada de la reconciliación), `obtenerUltimoEstadoPorRequestCode`, `observarSaludDelLog` (alimenta la pantalla) y `purgarAntiguos` (retención 60 días, ADR D-9).
- **C2 — Pantalla de diagnóstico** — `DiagnosticoNotificacionesViewModel` + `TarjetaDiagnostico` en Ajustes > Notificaciones: nº de alarmas registradas, antigüedad de la última actividad, avisos de descarte por permiso y de error.
- **C3 — Canario** — `NotificacionPrueba.kt` emite por los 3 canales con las **mismas** comprobaciones que el camino real (permiso → ajuste de la app → canal), y devuelve 4 desenlaces distinguibles (`MOSTRADA`, `SIN_PERMISO`, `CANAL_DESHABILITADO`, `ERROR`). `ERROR` se registra como `ERROR` y no como descarte, para que el diagnóstico no apunte a un permiso que el usuario no tocó.
- **Auditoría exportable** — `NotificacionLogRepository.exportar()` genera JSON y CSV de los últimos 30 días a `filesDir/exportes`, y Acerca de los comparte con `FileProvider` + `FLAG_GRANT_READ_URI_PERMISSION`. No usa Descargas: el archivo lleva nombres de cliente.

### Fase 5 — Volumen y escala ✅ (nuevo)
- **D1 — Agrupamiento** — `NotificationGrouper.kt`: umbral 6, `InboxStyle` con `MAX_LINEAS_RESUMEN = 6` y "y N más". El resumen viaja entero en los extras de cada miembro, así que el receptor sigue sin I/O.
- **Agrupación por ventana de llegada** — `planAgrupacionPorVentana` parte por `(tipo, minuto de llegada ya escalonado)`, no solo por tipo. Sin esto, 12 pagos que vencen entre hoy y el día 30 caerían en un único `setGroup` con semanas de diferencia, y la bandeja mostraría un resumen que se reescribe cada vez que uno llega.
- **B3 — Escalonado** — `escalonarLote` reparte en saltos de 20 s lo que cae en el mismo minuto, con tope de 12 por minuto, y **no mueve** lo que ya tiene hora propia (mover una alarma a la franja de otro día cambiaría el criterio de ADR D-1 para ganar segundos). 500 alarmas en el mismo minuto siguen dentro de la franja.

### Fase 6 — Migraciones seguras ✅ (nuevo)
- **E3 — Flag "esquema listo"** — `EsquemaNotificaciones.asegurarListo()` verifica con una consulta mínima que la base abre completa, persiste el flag en DataStore (que no depende de Room) y descarta la instancia envenenada de Room. `DailyCheckWorker` lo ejecuta **primero**, antes de cualquier lectura, y reintenta hasta 3 veces. `EsquemaNotificaciones.estaListo()` permite a las rutas que no deben abrir Room consultarlo sin hacerlo.
- **F1 — Privacidad** — `VISIBILITY_PRIVATE`, y el resumen sin importes (se lee en pantalla bloqueada).

### Fase 0 — Formalización ✅ (nuevo)
- `documentacion/ADR-notificaciones.md` con D-1 a D-9: franja, concurrencia, umbral, versionado de canales, idempotencia por `requestCode`, compatibilidad de `requestCode`, periodicidad, retención, y estado terminal.

### Gaps de `008` (previos) — todos resueltos
`cancelarAlertaPago` ya no es código muerto: se invoca desde `ClienteRepository` (softDelete, eliminarDefinitivo, setHabilitado, aplicarAccionInactividad), `PagoRepository.eliminarPago` y `DailyCheckWorker`. Los switches **sí** se leen y la rama `else` cancela. Cliente sin plan entra al ciclo. Fantasmas canceladas.

---

## 3. Lo que FALTA

### 🔴 FASE 4/5/6 — Verificación en dispositivo físico
Todo el código de las fases 4, 5 y 6 está implementado, compilado y cubierto por tests unitarios. **Lo que no está verificado es que funcione en el hardware objetivo**, y esa verificación no se puede hacer desde aquí.

| Ítem | Por qué los tests no lo cubren |
|---|---|
| `dumpsys alarm` tras reinicio, actualización de APK y cambio de zona | `AlarmManager` real y política de OEM |
| Doze y ahorro agresivo del Moto G22 | Comportamiento del fabricante, no reproducible en emulador |
| Alarma exacta vs. inexacta según permiso | `canScheduleExactAlarms()` es del sistema |
| Silenciar un canal concreto sin tocar el permiso | Ajustes del sistema |
| Canary en los 3 canales en el dispositivo | `NotificationManager` real |
| Actualización de la app (desinstalar el work de 24 h) | Ciclo de vida de `WorkManager` |

### 🟡 FASE 3 — Integridad de contenido
- **E1 (payload congelado)**: resuelto por refresco en segundo plano tras publicar (`EXTRA_PAYLOAD_DESACTUALIZADO` + `goAsync()` con timeout de 3 s), no por reemplazo directo del `PendingIntent` al editar un pago. El aviso se publica con el texto congelado y se corrige en segundo plano, lo que garantiza que el aviso exista aunque la lectura no llegue a tiempo. **El criterio de aceptación de `012 §3.4` sigue sin test instrumentada**: la corrección de un nombre solo se puede ver en pantalla.
- **"Reasignar"** persiste vía override en DataStore, no en entidad Room. `DailyCheckWorker` lo respeta y `limpiarReasignacionesObsoletas` lo borra cuando ya se registraron medidas nuevas. Funcional; no ideal.

### ⏸️ Fuera de alcance
- **A4 — Direct Boot**: no implementado, y documentado. El receiver no es `directBootAware`, así que tras un reinicio sin desbloqueo las alarmas no se restauran hasta el primer desbloqueo. Es acotado y conocido; la alternativa (ser direct boot aware) crashea al abrir Room en almacenamiento bloqueado.
- **008 §7 — Centro de alertas consolidado**: es una pieza de negocio, no de salud técnica.
- **Vencimientos > 48 h**: siguen entrando por la vía del badge, no de la notificación. Decisión de negocio.

---

## 4. Estado del QA (`012`)

| Sección | Ítems | Estado |
|---|---|---|
| §1 Unitarias (8) | 8 | ✅ Cubiertas: requestCode ×3, cascada, timing, franjas, **idempotencia de reconciliación**, **filtro 48 h**, **agrupador y escalonado** |
| §2 Integración Room (4) | 4 | 🟡 1 parcial (`MaryFitnessDatabaseIntegrationTest` en androidTest). **Faltan 3**: `eliminarPago` → cancelación, `aplicarAccionInactividad` cascada, edición de monto → nuevo payload |
| §3 Instrumentadas **[FÍSICO]** | 18 | 🔴 **0%**. Sin carpeta de evidencias, sin sesión de dispositivo documentada |
| §4 Matriz de estrés (6) | 6 | 🔴 0%. La lógica de escalonado y agrupación está testeada, pero no la ráfaga real |
| §5 Definition of Done (5) | 5 | 🔴 **1 de 5 verificable** (las unitarias). Las otras 4 exigen el dispositivo |

**El punto crítico del plan era explícito** (`010`): *"ninguna fase se considera terminada sin sus criterios de aceptación verificados en el Moto G22 físico"*. Los 320 tests pasan, pero son **lógica pura sin Android**: no cubren un solo `AlarmManager`, `NotificationManager` ni `PendingIntent` real. **El gap que queda no es de código, es de validación.**

---

## 5. Tabla de trazabilidad gap → estado

| Gap | Descripción | Estado |
|---|---|---|
| B1 | Colisión de `requestCode` | ✅ Cerrado |
| B2 | Concurrencia / mutex | ✅ Cerrado |
| B3 | Saturación / batching | ✅ Cerrado (escalonado 20 s, tope 12/min) |
| B4 | **Reconciliación / auto-reparación** | ✅ Cerrado (8 h, con `forzarRearmado` en restore y manual) |
| C1 | Auditoría de notificaciones | ✅ Cerrado (lectura + salud + purga + export) |
| C2 | Pantalla de diagnóstico | ✅ Cerrado (4 de 4 indicadores) |
| C3 | Canario / prueba | ✅ Cerrado (3 canales, 4 desenlaces) |
| D1 | Fatiga de notificaciones | ✅ Cerrado (umbral 6, `InboxStyle`, por ventana) |
| D2 | Franja horaria de pago | ✅ Cerrado |
| D3 | Canales por tipo | ✅ Cerrado |
| D4 | Inmutabilidad de canal | ✅ Cerrado (política `_v1`) |
| A1 | Auto-revoke / hibernación | ✅ Banner en cada apertura |
| A2 | Revocación tardía de alarma exacta | ✅ `PermissionsHealthMonitor` en `ON_RESUME` |
| A3 | OEM battery (Moto G22) | ✅ Opt-in documentado |
| A4 | Direct Boot | ⏸️ **No implementado, documentado** |
| E1 | Payload congelado | 🟡 Resuelto por refresco en segundo plano; sin test instrumentada |
| E2 | Alarmas huérfanas en soft-delete | ✅ `cancelarTodasDeCliente` en cascada + `CANCELAR` de la reconciliación |
| E3 | Migraciones a medio camino | ✅ Cerrado (flag en DataStore, 3 reintentos) |
| F1 | Privacidad en lockscreen | ✅ `VISIBILITY_PRIVATE`, resumen sin importes |
| 008 §7 | Centro de alertas consolidado | 🔴 Falta (es de negocio, no de salud técnica) |

---

## 6. Recomendación de orden

1. **QA físico en el Moto G22** — es lo único que separa "código completo" de "funciona en el dispositivo objetivo". Empezar por: `dumpsys alarm` tras reinicio y tras actualizar el APK, canario en los 3 canales, y un ciclo de Doze con la app en segundo plano.
2. **QA de integración Room** (`012 §2`, 3 ítems) — los tres casos que faltan son los que borran datos, y son los que más superficie tienen para dejar una alarma colgada.
3. **Sesión instrumentada de estrés** (`012 §4`) — 50 vencimientos el mismo día, con captura de la bandeja.
4. **E1 en instrumentada** — editar el nombre de un cliente con la alarma ya armada, y comprobar que el texto corregido llega.
5. **008 §7 (centro de alertas)** — solo cuando el anterior esté verde: es funcionalidad de negocio y reutiliza todo lo anterior.


---
