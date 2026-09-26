# Reporte de Problemas en Notificaciones

**Fecha**: 2026-09-25
**Autor**: Análisis basado en revisión de código y documentación
**Proyecto**: Mary Fitness - Aplicación de Gestión de Gym

---

## Resumen Ejecutivo

Este reporte documenta todos los huecos, contradicciones y fallos en la implementación de notificaciones identificados a través del análisis del código fuente y la documentación técnica. El sistema tiene una implementación funcional pero con cadenas de activación frágiles y varios problemas críticos que afectan la experiencia del usuario.

**Hallazgo principal**: Lógica implementada y probada unitariamente, pero cadenas de activación frágiles o desactivadas por configuración (Firebase sin inicializar, ciclos de 24h únicos, permisos especiales no solicitados).

---

## 1. Alarma nunca llega a programarse (Causa raíz más probable)

### 1.1 Dependencia del ciclo diario de 24h
- **Ubicación**: `MaryFitnessApplication.kt:17` → `workers/WorkScheduler.kt`
- **Problema**: Todo depende del ciclo diario de 24h del `WorkScheduler`. Un pago registrado con vencimiento a ≤2 días es descartado silenciosamente (`NotificationScheduler.kt:54`) hasta el próximo chequeo.
- **Impacto**: En instalación fresca pasan hasta 24 h sin ninguna alarma armada. Nadie ejecuta chequeo inmediato al registrar pagos o medidas — solo el boot lo hace (`BootCompletedReceiver.kt:12`).
- **Evidence**: `NotificationScheduler.programarAlertaPago()` descarta si `disparo <= System.currentTimeMillis()`, pero no hay trigger que invoque `ejecutarChequeoInmediato` después de operaciones CRUD de pagos/medidas.

### 1.2 `cancelarAlertaPago` nunca se invoca
- **Ubicación**: `NotificationScheduler.kt:92-99` - función "muerta"
- **Problema**: `cancelarAlertaPago()` nunca se llama desde ningún lugar del código.
- **Impacto**: Notificaciones fantasma tras cancelar plan o eliminar pago. Los alarms siguen activos aunque el plan haya sido cancelado/eliminado.

### 1.3 GAP en DailyCheckWorker
- **Ubicación**: `workers/DailyCheckWorker.kt:38`
- **Problema**: Clientes sin plan asignado nunca generan alertas.
- **Impacto**: Lógica de chequeo diario no considera este caso edge.

---

## 2. POST_NOTIFICATIONS denegado - Receivers silenciosos

### 2.1 Receivers que descartan silenciosamente
- **Ubicación**: `PagoVencimientoReceiver.kt:40-42` y `MedicionReminderReceiver.kt:56-58`
- **Problema**: Si el permiso `POST_NOTIFICATIONS` no está concedido, los receivers retornan temprano sin mostrar ninguna notificación. La alarm puede haber disparado correctamente pero nada aparece en la barra.
- **Impacto**: Usuario ve que la alarm disparó (o no), pero no recibe retroalimentación visual. El silencio total es confundible con "no funcionando".

### 2.2 Sin flujo de onboarding de permisos
- **Hallazgo**: El permiso `POST_NOTIFICATIONS` se solicita durante el setup inicial (`LoginScreen.kt`), pero no hay flujo contextual si el usuario lo deniega.
- **Impacto**: Una vez denegado, el usuario no puede recuperarlo desde dentro de la app (requiere llevar a ajustes del sistema).

---

## 3. Android 14+ (targetSdk 34) - SCHEDULE_EXACT_ALARM denegado por defecto

### 3.1 Sin flujo de solicitud de permiso especial
- **Ubicación**: `NotificationScheduler.kt:124-128`
- **Problema**: `alarmManager.canScheduleExactAlarms()` retorna `false` para apps con targetSdk 34. El código cae back a `set()` inexacto en lugar de solicitar al usuario el permiso mediante `ACTION_REQUEST_SCHEDULE_EXACT_ALARM`.
- **Impacto**: Doze y las capas OEM de ahorro de batería pueden retrasar las alarmas **horas** o incluso días. No existe ningún flujo en la app que solicite esta exención.

### 3.2 Sin manejo de exención de batería
- **Problema**: El app no solicita la exención "Alarmas y recordatorios" (SCHEDULE_EXACT_ALARM) de Android 14+.
- **Impacto** en dispositivo objetivo (Moto G22): Las capas OEM agresivas de ahorro de batería detienen las alarmas exactas después de cierto tiempo en segundo plano.

---

## 4. Switches decorativos en Ajustes - No afectan programación

### 4.1 `notifPagoVencimientoEnabled` / `notifRemedicionEnabled` no se leen
- **Ubicación**: `AjustesNotificacionesScreen.kt:56-81` - switches de toggle
- **Problema**: Los switches permiten activar/desactivar notificaciones en la UI, pero **estos valores no se leen** al programar alarms en `NotificationScheduler`. La lógica de programación ignora completamente el estado de estos switches.
- **Impacto**: El usuario cree que al apagar el switch las alarms se cancelan, pero siguen programándose cada ciclo de 24h. Apagar el switch es efectivamente decorativo.

### 4.2 Sin conexión entre UI y scheduler
- **Problema**: No existe una capa que lea `viewModel.pagoVencimientoEnabled` / `viewModel.remedicionEnabled` y le pase la información a `NotificationScheduler`.
- **Impacto**: Los switches son solo indicadores visuales, no controladores funcionales.

---

## 5. Configuración de periodicidad no refleja alarms

### 5.1 Periodicidad de re-medición ignorada al reprogramar
- **Ubicación**: `AjustesNotificacionesScreen.kt:97-111` - campo de texto de periodicidad (15-90 días)
- **Problema**: El usuario puede configurar la periodicidad (ej. cada 30 días), pero este valor **no se usa** al llamar a `programarAlertaMedicion()`. Siempre se usa la fecha de próxima medición del plan.
- **Impacto**: El usuario no tiene control efectivo sobre la frecuencia de los recordatorios de medidas a través de la UI.

---

## 6. Flujo "Reasignar" desde notificación sin actualizar system

### 6.1 `reasignarProximaMedicion` no conecta con business logic
- **Ubicación**: `NotificationScheduler.kt:118-121`
- **Problema**: La función cancela y reprograma alarms con nueva fecha, pero **no actualiza la fecha de próxima medición en la base de datos**. El cambio es efímero para el ciclo actual pero no persiste en la configuración del plan.
- **Impacto**: Si el administrador reasigna desde la notification, la próxima medición volverá a la fecha original al siguiente ciclo a menos que también se edite el plan/cliente en la base de datos.

### 6.2 Decisión técnica #4 (decisiones-tecnicas.md:39-41)
- **Contexto**: Tocar "Reasignar" desde notification abre detalle del cliente con datepicker/timepicker, pero no hay implementación completa para persistir el cambio y re-programar system-wide.

---

## 7. Falta centro de alertas consolidado

### 7.1 Documentado pero no implementado
- **Fuente**: `inventario-pantallas.md 7.3`, `006-reporte-comparativo-documentacion-codigo-interfaz.md:110`, `007-reporte-capacidad-administrador-acceso-total.md`
- **Problema**: La documentación menciona un "Centro de alertas consolidado" que debería mostrar vencidos, deudores, medidas pendientes e inactivos en una vista unificada.
- **Estado actual**: No existe pantalla consolidada. El dashboard muestra elementos separados (tarjetas de conteo, banner de vencidos).
- **Impacto**: El administrador debe navegar multiple screens para tener una visión completa del estado de notificaciones.

---

## 8. Bugs asociados y edge cases

### 8.1 Cliente sin plan asignado
- **Ubicación**: `DailyCheckWorker.kt:38`
- **Problema**: Clients sin plan asignado nunca generan alertas de vencimiento.
- **Impacto**: Usuarios "hueco negro" donde clientes órfanos no producen ninguna notificación.

### 8.2 GAP en reporte-pruebas-integrales.md:165-167
- **Referencia**: `002-reporte-pruebas-integrales.md`
- **Problema**: Se menciona una brecha abierta en las pruebas de timing de notificaciones.

---

## 9. Priorización y recomendaciones

### 9.1 Alta Prioridad (Impacto directo en operación diaria)

1. **Invocar chequeo inmediato tras registrar/editar/eliminar** pagos y medidas
   - Llamar a `WorkScheduler.ejecutarChequeoInmediato()` después de operaciones CRUD
   - Evita el delay de 24h en instalación fresca

2. **Leer switches de notificaciones en DailyCheckWorker antes de programar**
   - `DailyCheckWorker` debe verificar `viewModel.pagoVencimientoEnabled` y `viewModel.remedicionEnabled`
   - Si están desactivados, cancelar alarms existentes en lugar de programar nuevas

3. **Añadir flujo de solicitud de alarmas exactas (SCHEDULE_EXACT_ALARM)**
   - Agregar pantalla/flow que solicite `ACTION_REQUEST_SCHEDULE_EXACT_ALARM` en Android 14+
   - Mostrar explicación del impacto de no conceder el permiso

4. **Usar `cancelarAlertaPago` al cancelar planes/eliminar pagos**
   - Conectar la función "cancelar plan" con `NotificationScheduler.cancelarAlertaPago()`
   - Evita notificaciones fantasma

### 9.2 Media Prioridad (Mejoras funcionales)

5. **Conectar switches de AjustesNotificacionesScreen con la lógica de programación**
   - Leer `pagoVencimientoEnabled` / `remedicionEnabled` y pasar a `NotificationScheduler`
   - Permitir al usuario realmente controlar si las alarms se programan

6. **Persistir cambios de reasignación en base de datos**
   - `reasignarProximaMedicion()` debe actualizar la fecha de próxima medición en cliente/entity/plan

7. **Usar periodicidad configurada en Ajustes al programar alarms**
   - Leer `remedicionPeriodicidadDias` y usarla en `programarAlertaMedicion()` en lugar de fechas fijas

### 9.3 Baja Prioridad (Mejoras de experiencia)

8. **Implementar centro de alertas consolidado**
   - Pantalla unificada con estado de todas las notificaciones activas/vencidas/pendientes

9. **Mejorar manejo de edge cases** (cliente sin plan, GAP en pruebas, etc.)

---

## 10. Matriz de síntomas y causas

| Síntoma | Causa raíz | Ubicación |
|---------|-----------|-----------|
| No hay notificaciones después de registrar pago/medida | Sin chequeo inmediato; ciclo de 24h | `MaryFitnessApplication.kt:17`, `WorkScheduler.kt` |
| Alarms disparan pero no aparecen | POST_NOTIFICATIONS denegado | `PagoVencimientoReceiver.kt:40-42`, `MedicionReminderReceiver.kt:56-58` |
| Alarms retrasadas horas | Android 14+ sin SCHEDULE_EXACT_ALARM | `NotificationScheduler.kt:124-128` |
| Switch de ajustes no hace nada | Switches no leídos por scheduler | `AjustesNotificacionesScreen.kt:56-81`, `NotificationScheduler` |
| Notificaciones fantasma tras cancelar plan | `cancelarAlertaPago` nunca llamado | `NotificationScheduler.kt:92-99` |
| Periodicidad configurada no aplica | Periodicidad ignorada en programación | `AjustesNotificacionesScreen.kt:97-111`, `NotificationScheduler.kt:71-90` |
| Centro de alertas no existe | Pantalla no implementada | `inventario-pantallas.md 7.3` |

---

## 11. Estado actual por funcionalidad

| Funcionalidad | Código | Interfaz | Comentario |
|--------------|--------|----------|------------|
| Alerta vencimiento pago (2d ant.) | ✅ Implementado | ✅ Toggle en Ajustes | ⚠️ No se lee switch; ❌ No se re-programa al cambiar plan |
| Recordatorio medida (1d + 3 franjas) | ✅ Implementado | ✅ Toggle + periodicidad en Ajustes | ⚠️ Periodicidad no refleja; ⚠️ `reasignarProximaMedicion` no persiste |
| Acciones rápidas (Revisar/Reasignar/Okay) | ✅ Implementado | ✅ En notification | ✅ Funciona correctamente |
| Ajustes de notificación (toggle/periodicidad) | ✅ Implementado | ✅ Screen completa | ❌ Switches son decorativos; ❌ Periodicidad no aplica |
| Centro de alertas consolidado | ❌ No implementado | ❌ No existe | Documentado pero pendiente |
| Cancelación de alarms | ⚠️ Parcial | N/A | `cancelarAlertaPago` existe pero nunca llamado |

---

**Fin del reporte**