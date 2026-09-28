# Gaps Adicionales de Notificaciones — Mary Fitness

**Fecha:** 2026-09-26
**Complementa a:** `reportes/003-decisiones-pendientes.md`, `reportes/008-reporte-notificaciones-problemas.md`, `problems.md` / `solutions.md` (subidos)
**Propósito:** Este documento **no repite** los gaps ya cubiertos en esos archivos (retardo de 24h, permisos POST_NOTIFICATIONS/SCHEDULE_EXACT_ALARM, switches decorativos, notificaciones fantasma, pérdida de persistencia en "Reasignar", clientes sin plan, wipe por actualización, trampolines, flooding por restauración, zona horaria/DST, ANR por I/O). Todos esos siguen vigentes y deben implementarse. Aquí se documentan gaps **nuevos**, típicos de apps Android con `AlarmManager` + `WorkManager` offline-first, que tu diagnóstico aún no cubre.

Cada gap indica: **Contexto**, **Por qué es crítico para el negocio**, **Riesgo si no se corrige**.

---

## A. Ciclo de vida del sistema operativo y permisos "vivos"

### A1. Revocación automática de permisos por inactividad (App Hibernation / Auto-revoke, Android 11+)
**Contexto:** Si el administrador no abre la app durante varios meses (plausible: revisa el gym semanalmente, pero en vacaciones podría no tocar el teléfono), Android revoca automáticamente permisos en runtime (incluye `POST_NOTIFICATIONS` en Android 13+) y puede poner la app en **hibernación** (fuerza el stop del proceso, libera memoria, cierra recursos). Esto ocurre sin ninguna acción explícita del usuario.
**Riesgo:** Los recordatorios de pago y medidas dejan de dispararse silenciosamente, exactamente durante un período de menor supervisión del administrador (ej. vacaciones), que es cuando más se necesita la automatización.
**Cobertura faltante:** Ningún documento actual menciona detección de hibernación ni re-solicitud de permisos revocados por el sistema (no solo por el usuario).

### A2. Revocación de `SCHEDULE_EXACT_ALARM` en cualquier momento (no solo en onboarding)
**Contexto:** El reporte 008 cubre que el permiso no se solicita en Android 14+. Pero incluso si se concede una vez, el usuario puede revocarlo después desde **Ajustes > Apps > Mary Fitness > Alarmas y recordatorios**, en cualquier momento del ciclo de vida de la app, sin que la app lo sepa hasta el siguiente intento de programar una alarma.
**Riesgo:** Degradación silenciosa de exactas a inexactas mucho después de la instalación; el admin cree que todo sigue funcionando igual que el primer día.
**Cobertura faltante:** No hay chequeo periódico (ni al abrir la app) de `canScheduleExactAlarms()`; solo se contempla el flujo inicial.

### A3. Optimización de batería agresiva a nivel de fabricante (Motorola / MediaTek)
**Contexto:** El dispositivo objetivo es un Moto G22. Aunque Motorola es menos agresivo que Xiaomi/Samsung en matar procesos en segundo plano, sigue teniendo un gestor de batería (`Battery Manager`) que puede poner apps en modo "restringido" si el uso de batería en background es alto, deteniendo el proceso y cancelando callbacks (no las alarmas de `AlarmManager` en sí, que sobreviven, pero sí el registro de conectividad de `FirestoreSyncManager` y workers periódicos).
**Riesgo:** Combinado con A1/A2, crea múltiples capas de degradación silenciosa específicas del hardware objetivo, no solo genéricas de Android puro (AOSP).
**Cobertura faltante:** No hay flujo que solicite `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` ni pantalla que explique al admin cómo desactivar la optimización de batería para Mary Fitness específicamente en la capa OEM de Motorola.

### A4. Direct Boot / dispositivo bloqueado tras reinicio
**Contexto:** `BOOT_COMPLETED` en dispositivos con cifrado FBE puede no entregarse hasta que el usuario desbloquee el teléfono por primera vez tras el reinicio (a menos que se use `Direct Boot`/`ComponentCallback` en almacenamiento "device protected"). Si el Moto G22 se reinicia de madrugada (ej. actualización automática del OS) y nadie lo desbloquea hasta la mañana, hay una ventana adicional sin alarmas activas, más allá de la reprogramación por boot ya contemplada.
**Riesgo:** Ventana de "cero alarmas" que se suma a la ya documentada por `BootCompletedReceiver`.
**Cobertura faltante:** No se contempla registrar el receiver como "direct boot aware" ni informar al usuario del estado tras un reinicio.

---

## B. Programación, idempotencia y colisiones de alarmas

### B1. Colisión de `PendingIntent` por IDs no verdaderamente únicos
**Contexto:** Si el `requestCode` del `PendingIntent` se genera con `entityId.hashCode()` (como sugiere `solutions.md #4`) sin combinar también el **tipo de alarma** (pago vs medida) y la **fecha objetivo**, dos alarmas distintas para el mismo cliente (ej. alerta de pago y recordatorio de medida) pueden colisionar y sobreescribirse mutuamente, o un "Reasignar" puede cancelar la alarma equivocada.
**Riesgo:** Notificaciones perdidas o duplicadas de forma intermitente y difícil de reproducir en QA porque depende del hash específico de los IDs.
**Cobertura faltante:** Ningún documento define el **esquema exacto** de generación de `requestCode` (debe ser determinista, documentado y con espacio de colisión probado, ej. `(tipoAlerta.ordinal * 100_000_000) + clienteId` en vez de `hashCode()`).

### B2. Falta de idempotencia ante reprogramaciones concurrentes
**Contexto:** Si `DailyCheckWorker`, el chequeo inmediato post-CRUD (solución #1 propuesta) y el `AlarmsRestoreReceiver` se disparan casi al mismo tiempo (ej. justo después de un reinicio con un pago recién editado), pueden todos intentar reprogramar la misma alarma simultáneamente sin ningún lock, dejando el sistema en un estado no determinista (¿cuál de las tres escrituras "gana"?).
**Riesgo:** Condiciones de carrera silenciosas; alarmas con fecha desactualizada ganando sobre la fecha correcta.
**Cobertura faltante:** No hay mención de un mutex/lock (`Mutex` de coroutines o `WorkManager` con `ExistingWorkPolicy.REPLACE`/`KEEP` bien elegido) que serialice todas las escrituras al `NotificationScheduler`.

### B3. Límite de frecuencia y agrupamiento (batching) de `AlarmManager`
**Contexto:** Con muchos clientes (planes grupales, varios vencimientos el mismo día), se pueden programar decenas de `setExactAndAllowWhileIdle` para horarios muy cercanos. Android puede agrupar/retrasar alarmas no urgentes bajo ciertas condiciones de ahorro de energía incluso siendo "exactas", y postear muchas notificaciones en un corto lapso puede activar el **rate-limiting de notificaciones** del sistema (relevante sobre todo si en el futuro se usa FCM, pero también aplica a ráfagas locales).
**Riesgo:** El día que más importa (muchos vencimientos simultáneos) es el día con más probabilidad de fallos de entrega o de que el usuario ignore/silencie el aluvión.
**Cobertura faltante:** No existe estrategia de **agrupamiento con notificación resumen** (`NotificationCompat.InboxStyle` / `setGroup`) para picos de alertas del mismo tipo el mismo día.

### B4. Falta de reconciliación / auto-reparación periódica
**Contexto:** Todos los mecanismos actuales son "dispara y olvida": se programa la alarma y se asume que sigue viva. No hay ningún proceso que **verifique** periódicamente que, para cada cliente que debería tener una alerta activa (según su `estadoCuenta`/próxima medición), exista efectivamente un `PendingIntent` registrado en `AlarmManager`.
**Riesgo:** Es el gap más peligroso de todos porque es la causa raíz común detrás de casi todos los demás: sin reconciliación, cualquier falla puntual (crash, condición de carrera, OEM kill) se vuelve permanente hasta el próximo evento manual sobre ese cliente.
**Cobertura faltante:** No hay mención de un `ReconciliationWorker` (job periódico, ej. cada 6-12h) que recalcule el conjunto ideal de alarmas y lo compare/corrija contra el estado real.

---

## C. Observabilidad y confianza (nadie audita si las notificaciones realmente llegaron)

### C1. No existe registro de auditoría de notificaciones
**Contexto:** Hoy no hay ninguna tabla ni log persistente que registre "se programó la alarma X para la fecha Y" ni "se disparó/mostró la notificación Z a las W". Crashlytics (fase 5, aún no implementada) solo capturaría excepciones, no fallos silenciosos (que es exactamente el patrón dominante identificado en el reporte 008: "todo falla en silencio").
**Riesgo:** Es imposible diagnosticar a posteriori si un cliente no fue notificado por bug, por permiso denegado, por OEM, o porque nunca se programó la alarma. El propietario reportará "no me llegó la notificación" sin ninguna forma de saber por qué.
**Cobertura faltante:** No existe especificación de una tabla `notificacion_log` (o similar) con estados `PROGRAMADA / DISPARADA / MOSTRADA / DESCARTADA_POR_PERMISO / CANCELADA`.

### C2. Sin pantalla de diagnóstico para el propio administrador
**Contexto:** El admin no tiene forma de verificar, sin salir de la app, si sus notificaciones críticas están realmente activas (permiso concedido, alarmas exactas habilitadas, cuántas alarmas hay programadas ahora mismo).
**Riesgo:** Pérdida de confianza en el producto ("¿de verdad me va a avisar?") sin manera de auto-verificar.
**Cobertura faltante:** No existe una pantalla tipo "Salud del sistema de notificaciones" (independiente del Centro de Alertas del inventario 7.3, que es sobre *contenido de negocio*, no sobre *salud técnica del sistema*).

### C3. Ausencia de "canary" o notificación de prueba
**Contexto:** No hay forma de que el admin dispare una notificación de prueba inmediata para confirmar visualmente que su dispositivo específico (con su configuración OEM particular) efectivamente muestra las notificaciones de Mary Fitness.
**Riesgo:** El primer indicio de que algo falla es un cliente que dejó de pagar sin que nadie lo note a tiempo — es decir, el fallo se descubre por daño real al negocio, no proactivamente.
**Cobertura faltante:** No documentado en ningún reporte.

---

## D. Volumen, relevancia y fatiga de notificaciones (UX de confiabilidad percibida)

### D1. Fatiga de notificaciones / "alert fatigue"
**Contexto:** Con el esquema actual (3 franjas diarias por recordatorio de medida + alertas de pago + posible centro de alertas), un gym con 20-30 clientes activos puede generar muchas notificaciones diarias. Ya existe el riesgo mencionado en `solutions.md #7` (flooding tras restauración), pero el mismo riesgo existe en **operación normal** sin necesidad de una restauración: varios clientes venciendo la misma semana producen ráfagas repetidas.
**Riesgo:** Un admin saturado de notificaciones tiende a silenciar la app completa (deshabilitar notificaciones a nivel de sistema), lo cual anula *todas* las alertas, incluidas las críticas de cobro — el peor escenario posible para el negocio.
**Cobertura faltante:** No hay política de agrupamiento/resumen diario ("Tienes 5 alertas de pago y 3 de medidas hoy") como alternativa a notificaciones individuales sueltas.

### D2. Franja horaria de la alerta de pago no acotada a horas razonables
**Contexto:** La alerta de vencimiento de pago se calcula como `fechaVencimiento - 2 días`, pero `fechaVencimiento` hereda la **hora exacta** del momento en que se registró el pago original (ej. si el pago se registró a las 23:40, la alerta de vencimiento sonará a las 23:38 dos días antes). Las alertas de medición sí tienen franjas fijas (8:00/14:00/17:30); las de pago no.
**Riesgo:** Notificaciones de cobro sonando de madrugada, generando molestia y motivando a silenciar el canal completo (ver D1).
**Cobertura faltante:** No documentado; `especificaciones-generales.md` y `funcionalidades-producto.md` no fijan una franja horaria para la alerta de pago.

### D3. Falta de distinción de importancia/canal para "no molestar"
**Contexto:** Si todos los tipos de alerta usan el mismo `NotificationChannel` con la misma importancia, el usuario no puede silenciar selectivamente (ej. medidas) sin perder también las de cobro (críticas para el negocio), o viceversa.
**Riesgo:** Decisión de UX que compromete justamente el tipo de alerta más importante para el negocio (cobro) al empaquetarla junto a la menos crítica (medidas).
**Cobertura faltante:** No se especifican canales de notificación separados por tipo con importancia diferenciada (`IMPORTANCE_HIGH` para pagos vencidos, `IMPORTANCE_DEFAULT` para medidas).

### D4. Inmutabilidad de `NotificationChannel` tras la primera creación
**Contexto:** Una vez que una app crea un `NotificationChannel` con un ID dado, **no puede cambiar su importancia, sonido ni vibración programáticamente después** (solo el usuario puede desde Ajustes del sistema). Si en la v0.1 se crea un canal de "Pagos" con importancia baja por error, ningún update futuro del APK lo podrá corregir para usuarios que ya lo tengan instalado.
**Riesgo:** Un error de configuración inicial en los canales se vuelve **permanente** para instalaciones existentes salvo que el usuario reconfigure manualmente.
**Cobertura faltante:** No hay estrategia de versionado de IDs de canal (ej. `pagos_v2`) para poder "recrear" canales corregidos en actualizaciones futuras.

---

## E. Datos, migración y consistencia con el resto del sistema

### E1. Notificación con datos obsoletos por payload "congelado" en el `Intent`
**Contexto:** La solución #8 (correcta para evitar ANR) propone serializar los datos mínimos como `Extras` al programar la alarma. Pero si el pago se **edita** después de programada la alarma y antes de que se dispare (ej. el admin corrige el monto), la notificación mostrará el monto viejo porque el payload ya quedó "congelado" en el `PendingIntent`.
**Riesgo:** Información incorrecta mostrada al admin justo en el momento de cobrar a un cliente — puede generar disputas de monto.
**Cobertura faltante:** No se define una estrategia de invalidación: al editar un pago, además de reprogramar la fecha, debe **reemplazarse** el `PendingIntent` (no solo la fecha) para refrescar el payload, o el receiver debe hacer una lectura mínima y rápida (ej. `Room` con `Dispatchers.IO` + timeout corto) en vez de confiar 100% en extras.

### E2. Notificaciones para clientes ocultos/eliminados con alarmas ya programadas
**Contexto:** Relacionado con el gap ya documentado de `cancelarAlertaPago` nunca invocado, pero específicamente: al aplicar **soft delete** (ocultar cliente) o **purga física** a los 6 meses, ¿qué pasa con las alarmas de medición (no solo las de pago, que ya están cubiertas por el hallazgo 008 §1.2)? El `MedicionReminderReceiver` también debe cancelarse.
**Riesgo:** Un cliente oculto/purgado sigue generando una notificación de "Reasignar toma de medidas" que, al tocarla, navega a un cliente que ya no existe o está oculto (crash o pantalla vacía).
**Cobertura faltante:** El hallazgo 008 solo menciona `cancelarAlertaPago`; falta el equivalente para medidas y para el flujo completo de soft-delete/purga (no solo cancelación de plan).

### E3. Migraciones de esquema Room que cambian el significado de columnas usadas por el scheduler
**Contexto:** El plan de mejoras Firebase (fase 3-4) añade columnas y una futura migración `MIGRACION_3_4`. Si una migración de Room falla o corre parcialmente en un dispositivo con la app abierta en background, el `NotificationScheduler` podría leer un esquema a medio migrar.
**Riesgo:** Crashes o lecturas corruptas específicamente en el flujo de notificaciones durante ventanas de actualización de la app.
**Cobertura faltante:** No se documenta orden de arranque seguro (verificar migración completa antes de que cualquier receiver/worker acceda a Room).

---

## F. Seguridad y privacidad (aplica directamente a "confianza usuario-aplicación")

### F1. Contenido sensible visible en pantalla de bloqueo
**Contexto:** Las notificaciones de deuda/pago pueden mostrar nombre del cliente y montos adeudados. Por defecto, Android puede mostrar el contenido completo en la pantalla de bloqueo (`VISIBILITY_PUBLIC`).
**Riesgo:** Filtración de información financiera de terceros (clientes del gym) si el teléfono del administrador queda a la vista de otras personas — relevante en un contexto de gimnasio donde el teléfono puede quedar en el mostrador.
**Cobertura faltante:** No se especifica `setVisibility(NotificationCompat.VISIBILITY_PRIVATE)` con contenido resumido en el lockscreen y detalle completo solo al desbloquear.

---

## Resumen priorizado (a incorporar al backlog junto con 003/008)

| # | Gap | Prioridad sugerida | Por qué |
|---|---|---|---|
| B4 | Falta de reconciliación/auto-reparación | **CRÍTICA** | Es la red de seguridad que cubre todos los demás fallos silenciosos |
| C1 | Sin auditoría de notificaciones | **CRÍTICA** | Sin esto, ningún otro gap es diagnosticable en producción |
| A1 | Auto-revoke / hibernación | ALTA | Ocurre exactamente cuando menos supervisión hay |
| A2 | Revocación tardía de alarma exacta | ALTA | Degradación invisible post-instalación |
| B1 | Colisión de IDs de PendingIntent | ALTA | Causa bugs intermitentes difíciles de reproducir |
| D1/D2/D3 | Fatiga y horarios de notificación | ALTA | Riesgo de que el usuario desactive todo el canal |
| E1 | Payload congelado desactualizado | MEDIA | Afecta confianza en montos mostrados |
| E2 | Alarmas huérfanas de medidas en clientes ocultos | MEDIA | Extiende un bug ya conocido a otro flujo |
| C2/C3 | Diagnóstico y canary | MEDIA | Mejora confianza percibida, no bloquea operación |
| A3/A4 | OEM battery / Direct Boot | MEDIA | Específico del hardware objetivo (Moto G22) |
| D4 | Inmutabilidad de canal | BAJA (pero urgente antes del primer release) | Un error aquí es irreversible para usuarios ya instalados |
| B2/B3 | Concurrencia y batching de alarmas | BAJA-MEDIA | Se agrava con escala (más clientes) |
| F1 | Privacidad en lockscreen | BAJA | No rompe funcionalidad, pero es exposición de datos de terceros |
| E3 | Migraciones a medio camino | BAJA | Bajo riesgo dado el tamaño de la BD, pero fácil de prevenir |

Ver `010-plan-implementacion-notificaciones.md` para cómo estos gaps (combinados con los ya documentados) se traducen en fases de trabajo concretas.
