# Reporte de Diagnóstico - Estado de Funcionalidades Reportadas

**Fecha**: 22 de agosto de 2026
**Alcance**: Análisis estático de código fuente y documentación (sin ejecución ni modificaciones)
**Origen**: Observaciones del propietario sobre tres funcionalidades
**Resultado**: Diagnóstico confirmado con causas raíz identificadas y rutas exactas

---

## 1. Resumen Ejecutivo

Se investigaron las tres observaciones reportadas por el propietario:

| # | Observación | Veredicto |
|---|---|---|
| 1 | No permite pagar saldo pendiente o abonar en el mismo registro de pago parcial | **Parcialmente incorrecta**: la capacidad existe, pero faltan la acción "abonar sobre el registro" y el flujo "pagar saldo restante" |
| 2 | La sincronización web no funciona, sobre todo al iniciar | **Confirmada**: la sync está muerta en runtime porque Firebase nunca se inicializa |
| 3 | No hay notificaciones | **Confirmada**: el diseño es correcto pero la cadena de programación tiene eslabones rotos |

---

## 2. Hallazgo 1 — Pagos: saldo pendiente y pagos parciales

### Lo que SÍ existe hoy

- **Pagos parciales (abonos)**: estado `PARCIAL` con `plazoMaximoPago` (`app/src/main/java/com/marafit/app/data/local/entity/PagoEntity.kt:33-37`).
- **Múltiples pagos sobre el mismo plan**: `registrarPago` nunca impide un segundo pago sobre el plan actual (`data/repository/PagoRepository.kt:52-61`, la comprobación cross-plan solo aplica si el plan difiere).
- **Acumulación de abonos por período**: `ClienteRepository.calcularPeriodoActual()` suma los abonos consecutivos hasta cubrir `montoTotal` (`data/repository/ClienteRepository.kt:249-283`). Diseño documentado en `documentacion/Pendientes/003-respuesta-decisiones-pendientes.md` (decisión #1: "A. Acumular pagos").
- Cubrir un saldo pendiente registrando un segundo abono **sí liquida el período**.

### Lo que NO existe (núcleo válido de la queja)

1. **No hay acción "abonar sobre el mismo registro"**: `registrarPago` siempre inserta una fila nueva de `PagoEntity` (`PagoRepository.kt:79-90`). El historial solo ofrece Editar-monto-exacto o Eliminar (`ui/pagos/RegistrarPagoScreen.kt:216-221`). No existe operación incremental que sume al `montoPagado` del registro parcial existente.
2. **No existe botón/flujo "pagar saldo restante"** que precargue el faltante calculado (`resumenFinanciero.saldoPendiente`) y liquide el período.
3. **UX engañosa al liquidar saldos**: el aviso de abono calcula `precioPlan - monto` ignorando lo ya abonado (`RegistrarPagoScreen.kt:153-159`). Ejemplo: debe $50 de un plan de $100; al teclear $50 la UI anuncia "Saldo pendiente: $50" tras guardar (falso) y **exige nuevamente plazo máximo**, porque la fila nueva nace como PARCIAL contra su propio montoTotal aunque liquide el período.
4. **Renovación implícita no deseada**: todo `registrarPago` fija `fechaVencimiento = hoy + duracionDias` (`PagoRepository.kt:77-78`); liquidar un saldo antiguo extiende la vigencia desde hoy.
5. **Tope duro por fila en edición**: `modificarPago` valida contra el montoTotal del registro individual, no contra el acumulado del período (`PagoRepository.kt:109-111`).

### Corrección mínima sugerida

(a) Acción "Abonar/Pagar saldo" sobre registros PARCIALES que incremente `montoPagado` y recalcule el estado del período; (b) precarga del saldo real en la pantalla de pago y aviso basado en acumulado; (c) eximir de plazo cuando el abono liquida el período; (d) decidir si liquidar saldo debe o no renovar vigencia.

---

## 3. Hallazgo 2 — Sincronización web

### Causa raíz: Firebase nunca se inicializa

La lógica de sync está completa e incluso probada (cola `pendienteSync`, subida ordenada por dependencias, tests en `FirestoreSyncManagerTest.kt`), pero está **inactiva en runtime**:

| Evidencia | Ubicación |
|---|---|
| Plugin google-services comentado | `app/build.gradle.kts` y raíz `build.gradle.kts` ("descomentar cuando exista un google-services.json real") |
| No existe `app/google-services.json` | Verificado en disco |
| Ninguna llamada a `FirebaseApp.initializeApp()` en el código | Solo `FirebaseApp.getInstance()` en `sync/FirestoreSyncManager.kt:55` |
| Guardia que aborta silenciosamente | `FirestoreSyncManager.kt:79`: `if (!firestoreDisponible() || callbackRegistrado) return` |

Consecuencia: en `MarafitApplication.onCreate` (`MarafitApplication.kt:16`) el único disparador de arranque (`iniciarObservacionConectividad()`) sale sin registrar el callback de red. **Ni al iniciar ni ante cambios de conectividad se sincroniza jamás.** La UI lo refleja: chip "No configurada" (`ui/sync/AjustesSincronizacionScreen.kt:39`), botón manual deshabilitado (`:66`), "Última sincronización: Nunca" mientras los pendientes se acumulan. No hay mensaje de error para el usuario.

### Defectos latentes (si se activara Firebase)

1. **Crash potencial**: `FirebaseFirestore.getInstance()` está fuera del try en `sincronizarPendientes()` (`FirestoreSyncManager.kt:124`).
2. **Sin reintento temporal**: fallas con red activa no se reintentan hasta el próximo cambio de conectividad (no hay WorkManager con constraint de red).
3. **Disparos concurrentes sin mutex**: WiFi+datos pueden lanzar dos subidas en paralelo (`FirestoreSyncManager.kt:84-86`).

### Nota adicional de configuración

El `local.properties` real solo contiene `sdk.dir`: faltan `ASCEND_API_KEY` (búsqueda de ejercicios devolverá 401/403) y `MARAFIT_UPDATE_INFO_URL` (actualizaciones en estado `SinConfigurar`). No afecta la sync, pero son dependencias externas igualmente inactivas.

### Corrección mínima sugerida

(1) Añadir `app/google-services.json` real; (2) descomentar el plugin en ambos `build.gradle.kts`; (3) mover el `getInstance()` dentro del try; (4) añadir Worker periódico con `NetworkType.CONNECTED`.

---

## 4. Hallazgo 3 — Notificaciones

### Balance documental vs implementación

La especificación documentada coincide casi al 100% con el motor implementado (cálculo -2 días, franjas 8:00/14:00/17:30, acciones Revisar/Reasignar/Okay, canales). Fuentes contrastadas: `documentacion/funcionalidades-producto.md:21-23`, `documentacion/especificaciones-generales.md:12-18`, `documentacion/reportes/002-reporte-pruebas-integrales.md`. Las divergencias documentales pendientes son: Centro de alertas consolidado (#7.3 de `inventario-pantallas.md`) inexistente y switches on/off (#9.2) decorativos.

### Por qué no llegan notificaciones (causas ordenadas por verosimilitud)

1. **La alarma nunca llega a programarse** (causa raíz más probable). Todo depende del ciclo diario de 24 h (`MarafitApplication.kt:17` → `workers/WorkScheduler.kt`). Un pago registrado con vencimiento a ≤2 días es descartado silenciosamente (`notifications/NotificationScheduler.kt:54`) hasta el próximo chequeo. En instalación fresca pasan hasta 24 h sin ninguna alarma armada. Nadie ejecuta chequeo inmediato al registrar pagos o medidas — solo el boot lo hace (`BootCompletedReceiver.kt:12`).
2. **POST_NOTIFICATIONS denegado**: los receivers descartan silenciosamente (`PagoVencimientoReceiver.kt:40-42`, `MedicionReminderReceiver.kt:56-58`). La alarma dispara pero nada aparece.
3. **Android 14+ (targetSdk 34)**: `SCHEDULE_EXACT_ALARM` viene denegado por defecto para apps nuevas; cae a `set()` inexacto (`NotificationScheduler.kt:124-128`) retrasable horas por Doze, y **no existe ningún flujo** que solicite el permiso especial (`ACTION_REQUEST_SCHEDULE_EXACT_ALARM`) ni exención de batería.
4. **Switches decorativos**: `notifPagoVencimientoEnabled`/`notifRemedicionEnabled` de `AjustesNotificacionesScreen.kt:56-81` no se leen al programar alarmas; apagarlos no cancela nada.
5. **Bugs asociados**: `cancelarAlertaPago` nunca se invoca desde ningún lugar (`NotificationScheduler.kt:92-99` muerto) → notificaciones fantasma tras cancelar plan o eliminar pago; clientes sin plan asignado nunca generan alertas (`DailyCheckWorker.kt:38`); GAP 3 del propio `002-reporte-pruebas-integrales.md:165-167` sigue abierto.

### Corrección mínima sugerida

Invocar chequeo inmediato tras registrar/editar/eliminar pagos y medidas; leer los switches en `DailyCheckWorker` antes de programar; añadir flujo de solicitud de alarmas exactas; usar `cancelarAlertaPago` al cancelar planes/eliminar pagos; ejecutar chequeo inmediato en el primer arranque; documentar el requisito "Alarmas y recordatorios" de Android 14.

---

## 5. Consideraciones Transversales

- Los tres hallazgos comparten un patrón: **lógica implementada y probada unitariamente, pero cadenas de activación frágiles o desactivadas por configuración** (Firebase sin inicializar, ciclos de 24 h únicos, permisos especiales no solicitados).
- El dispositivo objetivo es un Moto G22 (Android 12/13) donde las capas OEM de ahorro de batería agravan tanto la sync como las alarmas exactas.
- Ninguno de los defectos produce errores visibles al usuario: todos fallan en silencio.

---

## 6. Recomendaciones de Priorización

1. **Notificaciones** — impacto directo en operación diaria (cobranza y re-medición); correcciones de código puro, sin dependencias externas.
2. **Pagos** — corrección de UX crítica para la facturación; requiere decisión de negocio sobre renovación al liquidar saldo (punto d).
3. **Sincronización** — bloqueada por una dependencia externa (`google-services.json` que solo el propietario puede generar en su consola de Firebase); el resto son mejoras de robustez.
