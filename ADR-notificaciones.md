# ADR — Sistema de Notificaciones

**Fecha:** 2026-09-26
**Estado:** aceptado (defaults aplicados por el agente según `010-plan-implementacion-notificaciones.md` Fase 0, pendiente de ratificación del propietario)
**Alcance:** franja horaria, canales, agrupamiento, privacidad, frecuencia de reconciliación, batería.

## Por qué existe este documento

Los valores que gobiernan el sistema de notificaciones estaban implementados y funcionando, pero **hardcodeados y sin trazabilidad**: 6:00–21:00 en `PlanificadorAlertas`, tres canales `*_v1`, `VISIBILITY_PRIVATE`, ventana de 48 h, ciclo de 24 h. El código era correcto pero ninguna decisión de negocio era rastreable, así que un cambio de criterio posterior no tenía dónde documentarse ni hacia dónde ASAP.

Este ADR fija esas decisiones. Cuando una se cambie, se edita aquí **y** se cambia la constante, en el mismo commit.

## Decisiones

### D-1. Franja horaria de las alertas de pago: 06:00–21:00

* `FRANJA_INICIO_HORA = 6`, `FRANJA_FIN_HORA = 21` (`notifications/PlanificadorAlertas.kt`).
* Todo disparo fuera de la franja se mueve **al inicio de la franja más cercana**: antes de las 06:00 → 06:00 del mismo día; después de las 21:00 → 06:00 del día siguiente, porque las 06:00 de ese día ya pasaron.
* **Por qué no 24 h**: un pago registrado a medianoche genera su aviso a medianoche. El usuario no lo ve, la app parece muda, y el administrador concluye que "las notificaciones no funcionan". La franja no es una preferencia estética: es la diferencia entre un aviso útil y un aviso perdido.
* Aplica a las 4 franjas de medición (8:00 / 14:00 / 17:30 + aviso general) también vía `acotarAFranja`.

### D-2. Tres canales separados desde el día 1

`canal_pagos_v1` (HIGH), `canal_medidas_v1` (HIGH), `canal_sistema_v1` (LOW).

* **Por qué separar**: el administrador puede querer silenciar "Recordatorios de medidas" sin perder "Recordatorios de pago", que es la alerta que sostiene el cobro.
* **Los IDs son inmutables.** La importancia, el sonido y la vibración de un `NotificationChannel` quedan congelados en el primer arranque y después solo el usuario puede cambiarlos. Para cambiar la configuración de un canal hay que crear `*_v2` y migrar; **nunca reutilizar ni reescribir un ID existente**. Esta regla está también en el KDoc de `NotificationScheduler.kt`.

### D-3. Umbral de agrupamiento: N = 6

`UMBRAL_RESUMEN_NOTIFICACIONES = 6`.

* Si en una misma ventana de disparo se generarían **≥ 6** alertas del mismo tipo, se colapsan en **una** notificación resumen con `InboxStyle` en lugar de N individuales. Por debajo del umbral se muestran individuales, porque agrupar unos pocos avisos cuesta más de lo que ahorra.
* **La ventana es el minuto de llegada** de la alarma ya escalonada, y no el tipo de alerta por sí solo. Con 12 pagos que vencen entre hoy y el día 30, un único resumen por tipo los metería a todos en el mismo `setGroup` aunque suenen con semanas de diferencia: la bandeja mostraría un solo resumen que se reescribe cada vez que uno llega, y el administrador vería cómo se borran los nombres que ya estaba leyendo. `planAgrupacionPorVentana` parte por `(tipo, minuto)`.
* **Por qué importa**: 10 notificaciones simultáneas empujan al administrador a silenciar el canal completo, y con él se van los avisos de pago. El agrupamiento no es una mejora estética, es lo que evita perder el canal.
* El resumen navega a la app (Dashboard), donde el detalle de cada cliente ya es accesible. **No** se construye un centro de alertas nuevo: es alcance de negocio, no de salud técnica (ver `008-reporte-notificaciones-problemas.md` §7, pendiente).

### D-4. Privacidad en pantalla bloqueada: `VISIBILITY_PRIVATE` con texto genérico

* `NotificationCompat.VISIBILITY_PRIVATE` en todos los canales.
* El texto público en lockscreen **no** incluye nombre de cliente ni monto: `"Tienes una alerta de Mary Fitness"` (`R.string.notificacion_sistema_generica`).
* **Por qué**: el dispositivo objetivo (Moto G22) no tiene bloqueo por defecto y es manipulado por personal del gimnasio. Un saldo o un nombre en la pantalla bloqueada es una fuga de datos de negocio, no un detalle de UX.

### D-5. Frecuencia de reconciliación: cada 8 horas

`WorkScheduler.programarChequeoDiario` usa 8 h, no 24 h.

* **Por qué 8 h y no 24 h**: el caso que la reconciliación existe para recuperar es "Android borró la alarma por un reinicio y el usuario no abrió la app". Con 24 h de ciclo, un reinicio un martes deja el dispositivo sin avisos hasta el miércoles, y el administrador no tiene forma de saberlo. 8 h acota la ventana de silencio a un ciclo de trabajo.
* 8 h es también el mínimo que `PeriodicWorkRequest` acepta sin degradar, así que es un límite de la plataforma además de una decisión de negocio.

### D-6. Exención de optimización de batería: opt-in, nunca bloqueante

* Se ofrece en Ajustes > Notificaciones con explicación del impacto, pero **no se bloquea ninguna funcionalidad** si el usuario dice que no. Sin exención la app sigue funcionando; solo pierde precisión en la hora de aviso.
* Se usa `ACTION_IGNORE_BATTERY_OPTIMIZATION_SETTINGS` (la lista del sistema) en vez de `ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` (diálogo directo). **Desviación deliberada**: el diálogo directo exige el permiso `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` en el manifiesto, y Google Play restringe qué apps pueden usarlo salvo justificación. La lista evita depender de una excepción.
* **Por qué no pedirlo de entrada**: pedir un permiso de batería sin explicar el impacto es la vía rápida para que el usuario lo niegue, y para que un usuario razonable sospeche de la app. Un opt-in informado que se acepta vale más que un permiso forzado que se niega.

### D-7. Filtro de caducidad: 48 horas

`VENTANA_MAXIMA_VENCIMIENTO_MS = 48 h`.

* Un pago vencido hace más de 48 h **no genera notificación**: se refleja como badge/contador en el Dashboard.
* **Por qué**: notificar saldos de semanas entrena al usuario a ignorar el canal. Y al abrir la app tras una restauración de backup con 50 pagos históricos, notificar los 50 sería exactamente el flooding que hace silenciar el canal.

### D-8. `requestCode` determinista: `tipo.ordinal * 10_000_000 + entidadId`

Bloques de 10 millones de IDs por tipo de alerta.

* **Por qué determinista y no aleatorio**: para que *cancelar* sea reconstruible. Si el ID de una alarma se pudiera regenerar, no habría forma de encontrar la `PendingIntent` que hay que cancelar, y la cancelación se volvería código muerto (que fue exactamente el defecto encontrado).
* `require(entidadId in 0 until 10_000_000)`: si se excede el rango se lanza excepción en vez de devolver un ID silenciosamente colisionado. El límite está tres órdenes de magnitud por encima de cualquier escala realista de un gimnasio.
* Las 4 franjas de medición de un cliente ocupan IDs contiguos (`clienteId * 4 + franja`) para que no se sobrescriban entre sí.

### D-9. El log de notificaciones se queda local

`notificacion_log` **no** se sube a Firestore.

* **Por qué**: es información técnica de diagnóstico, no de negocio. Subirla a la nube engorda el backup, mezcla datos de diagnóstico con datos de clientes y no aporta nada: el diagnóstico ocurre en el mismo dispositivo donde falla. Se exporta bajo demanda (JSON/CSV) desde Ajustes > Acerca de cuando hace falta mandarlo a soporte.
* Retención: 60 días, purgados por la propia reconciliación. El eMMC del Moto G22 no es infinito y un log que crece sin límite acaba degradando la app.

## Lo que este ADR NO cubre

* **Centro de alertas consolidado** (`008` §7). Es alcance de negocio: requiere una pantalla nueva de bandeja de alertas. Está fuera del sistema de notificaciones *técnico* y sigue pendiente.
* **Direct Boot** (gap A4). No implementado; la ventana de riesgo residual está documentada en `AlarmsRestoreReceiver.kt:20-30`: tras un reinicio con el dispositivo nunca desbloqueado, las alarmas se restauran en el primer desbloqueo, no antes.
* **Notificaciones push (FCM)**. Fuera de alcance por decisión previa: no hay servidor, la app es 100% local (ADR #7 de `decisiones-tecnicas.md`).
