# Decisiones Pendientes y Preguntas por Resolver

**Fecha**: 20 de agosto de 2026  
**Origen**: Resultados de pruebas integrales (reporte 002) y análisis de código  
**Propósito**: Documentar las decisiones comprometedoras que requieren input del negocio antes de avanzar

---

## 1. Acumulación de Pagos Parciales

**Prioridad**: CRÍTICA  
**Afecta**: `PagoRepository.kt`, `ClienteRepository.kt`, UI del Dashboard, lógica de estados

### Pregunta
¿Cómo debe el sistema determinar el estado de cuenta de un cliente que realizó múltiples abonos parciales?

### Contexto
Actualmente, `recalcularEstado()` solo examina el **último pago** registrado. Un cliente con 3 abonos de $50K para un plan de $150K aparece como DEUDA aunque haya pagado el total acumulado.

### Opciones

| Opción | Descripción | Ventajas | Desventajas |
|---|---|---|---|
| **A) Acumular pagos** | `recalcularEstado()` suma todos los pagos del cliente y compara contra `montoTotal` | El estado refleja la realidad financiera | Cambia significativamente la lógica; cada pago individual ya no tiene sentido como snapshot |
| **B) Pago acumulativo único** | Un solo registro de `PagoEntity` cuyo `montoPagado` se incrementa con cada abono | Simple, un registro = un cliente | Pierde historial de transacciones individuales |
| **C) Mantener actual + banner** | Mantener la lógica actual pero mostrar en UI "Total pagado: $150K de $150K" | Mínimo cambio de código | El estado DEUDA sigue siendo confuso para el admin |
| **D) Último pago + acumulado** | El estado se basa en el último pago, pero la UI muestra ambos (estado + total acumulado) | Transparencia sin romper la lógica actual | Complejidad en UI |

### Decisión requerida
¿Qué opción se implementa? Si es A o B, ¿hay reglas de negocio específicas sobre cómo se manejan los reembolsos o pagos duplicados?

---

## 2. Periodicidad de Re-mediciones

**Prioridad**: ALTA  
**Afecta**: `DailyCheckWorker.kt`, `AppSettingsManager.kt`, notificaciones de medidas

### Pregunta
¿La periodicidad de las re-mediciones debe basarse en la duración del plan o en un valor configurable independiente?

### Contexto
Existe un setting `remedicionPeriodicidadDias` (default: 30) en `AppSettingsManager`, pero `DailyCheckWorker` ignora este valor y usa `plan.duracionDias` para calcular la próxima medición.

### Opciones

| Opción | Descripción | Escenario |
|---|---|---|
| **A) Usar plan.duracionDias** | La periodicidad viene dada por el plan (30 días para mensual, 90 para trimestral) | Simple, coherente con la vigencia del plan |
| **B) Usar remedicionPeriodicidadDias** | El admin configura cada cuántos días se piden medidas, sin importar el plan | Flexibilidad para planes largos con mediciones frecuentes |
| **C) Mínimo entre ambos** | Se usa el menor valor entre `plan.duracionDias` y `remedicionPeriodicidadDias` | Garantiza mediciones frecuentes pero respeta el plan |
| **D) Configurable por plan** | Agregar campo `periodicidadMedidas` al `PlanEntity` | Máxima flexibilidad pero más complejidad en UI de creación de planes |

### Decisión requerida
¿Cuál lógica aplica? Si es B o D, ¿cuál es el valor por defecto y el rango permitido?

---

## 3. Comportamiento del Botón "Reasignar"

**Prioridad**: ALTA  
**Afecta**: `MedicionReminderReceiver.kt`, navegación, UI de detalle de cliente

### Pregunta
¿Qué debe suceder cuando el administrador toca "Reasignar" en la notificación de toma de medidas?

### Contexto
La decisión técnica #6 en `decisiones-tecnicas.md` define que se abre la pantalla de detalle del cliente con un diálogo de selección de fecha/hora. Pero este flujo no está implementado.

### Preguntas específicas

1. **¿El diálogo de reasignación es un DatePicker + TimePicker nativo o un componente custom?**
2. **¿La reasignación aplica solo a la siguiente medición o a todas las futuras?**
3. **¿Se puede reasignar a una fecha en el pasado?** (útil si el admin olvidó registrar medidas)
4. **¿La reasignación afecta la periodicidad del plan o solo aplaza el recordatorio?**
5. **¿Qué pasa si el admin reasigna y luego el plan vence antes de la nueva fecha?**

### Decisión requerida
Definir el comportamiento exacto del diálogo y las reglas de negocio de reasignación.

---

## 4. Planes Grupales: Validación de Cupo

**Prioridad**: MEDIA  
**Afecta**: `PagoRepository.kt`, UI de selección de planes, `PlanEntity`

### Pregunta
¿Se debe validar el cupo disponible en planes grupales al momento de asignar un cliente?

### Contexto
`PlanEntity` tiene `maxIntegrantes` para planes grupales, pero no se valida al registrar un pago. Dos clientes pueden ser asignados al mismo plan grupal sin importar el límite.

### Opciones

| Opción | Descripción |
|---|---|
| **A) Validar en pago** | `PagoRepository.registrarPago()` verifica cuántos clientes tienen el mismo `planId` y rechaza si supera `maxIntegrantes` |
| **B) Validar en UI** | El dropdown de planes muestra "Lleno" si no hay cupo, pero no bloquea la inserción directa |
| **C) Sin validación** | `maxIntegrantes` es solo informativo para el admin |

### Decisión requerida
¿Se valida y bloquea, solo se informa, o se ignora? Si se valida, ¿cuál es el comportamiento cuando el plan está lleno?

---

## 5. Overlapping de Planes en un Cliente

**Prioridad**: MEDIA  
**Afecta**: `PagoRepository.kt`, `ClienteRepository.kt`, modelo de datos

### Pregunta
¿Un cliente puede tener pagos de múltiples planes diferentes al mismo tiempo?

### Contexto
`ClienteEntity.planId` es un FK simple (un solo plan). Pero un cliente podría querer cambiar de plan (ej: de Mensual a Trimestral) mientras tiene un plan activo. Actualmente, el sistema actualiza `planId` al registrar un nuevo pago con un plan diferente, pero no cancela ni registra el plan anterior.

### Preguntas específicas

1. **¿El plan anterior se cancela automáticamente?**
2. **¿Se genera un reembolso o ajuste proporcional?**
3. **¿El cliente puede tener historial de múltiples planes activos en diferentes periodos?**
4. **¿El `planId` en `ClienteEntity` refleja el plan ACTUAL o el ÚLTIMO pagado?**

### Decisión requerida
Definir la política de cambio de plan y si se requiere un modelo de "suscripción" con fechas de inicio/fin.

---

## 6. Permisos de Notificación en Android 13+

**Prioridad**: MEDIA  
**Afecta**: `PagoVencimientoReceiver.kt`, `MedicionReminderReceiver.kt`, UI de onboarding

### Pregunta
¿Cómo se maneja el permiso `POST_NOTIFICATIONS` que es requerido en Android 13+?

### Contexto
Las notificaciones se programan correctamente pero se silencian silenciosamente si el permiso no está concedido. No hay instrucción al usuario.

### Opciones

| Opción | Descripción |
|---|---|
| **A) Solicitar en onboarding** | Pedir permiso durante la configuración inicial del PIN |
| **B) Solicitar al primer pago** | Pedir permiso cuando se registra el primer pago (momento más relevante) |
| **C) Banner informativo** | Mostrar un banner en Dashboard si el permiso no está concedido |
| **D) Solicitar siempre** | Usar `shouldShowRequestPermissionRationale()` y reintentar |

### Decisión requerida
¿En qué momento se solicita el permiso? ¿Se bloquea la funcionalidad si no se concede?

---

## 7. Persistencia de Datos tras Soft Delete

**Prioridad**: BAJA  
**Afecta**: `ClienteRepository.kt`, queries de Firestore, reglas de retención

### Pregunta
¿Cuánto tiempo se conservan los datos de un cliente "oculto" (soft delete)?

### Contexto
Cuando un cliente se marca como `oculto = true`, sus datos permanecen en Room y Firestore pero se excluyen de consultas. No hay política de purga.

### Opciones

| Opción | Descripción |
|---|---|
| **A) Conservar indefinidamente** | Los datos nunca se eliminan físicamente |
| **B) Purgar después de X meses** | Eliminar físicamente después de 6/12/24 meses |
| **C) Exportar antes de purgar** | Generar un reporte PDF/CSV antes de eliminar |
| **D) Confirmación manual** | El admin decide cuándo purgar desde Ajustes |

### Decisión requerida
¿Hay requisitos legales o de negocio para conservar historial de clientes eliminados?

---

## 8. Sincronización Firestore: Estrategia de Merge

**Prioridad**: BAJA  
**Afecta**: `FirestoreSyncManager.kt`, `SyncManager.kt`

### Pregunta
¿Se requiere descarga (download) de datos desde Firestore, o solo subida (upload)?

### Contexto
Actualmente la sincronización es upload-only. Si la app se usa en múltiples dispositivos, los cambios nunca se propagan.

### Preguntas

1. **¿La app se usará en más de un dispositivo?** Si no, upload-only es suficiente.
2. **Si se necesita multi-dispositivo, ¿cuál es la estrategia de merge?** (última escritura gana, merge por campos, etc.)
3. **¿Se necesita modo offline completo o se acepta connectividad intermitente?**

### Decisión requerida
Definir si Firestore es backup o sincronización real.

---

## 9. Manejo de Planes Vencidos sin Acción

**Prioridad**: BAJA  
**Afecta**: UI de Dashboard, `DailyCheckWorker.kt`

### Pregunta
¿Qué pasa con los clientes en estado VENCIDO que no realizan ninguna acción?

### Contexto
Un cliente puede estar VENCIDO indefinidamente. No hay flujo automático que lo mueva a otra estado ni que solicite al admin una decisión.

### Opciones

| Opción | Descripción |
|---|---|
| **A) No hacer nada** | El cliente permanece VENCIDO hasta que pague o se inactivite |
| **B) Notificación recurrente** | Recordar al admin cada X días sobre clientes vencidos |
| **C) Migración automática a inactivo** | Después de Y días vencido, marcar como candidato a inactividad |
| **D) Banner persistente** | Mostrar contador de clientes vencidos en Dashboard sin fecha de caducidad |

### Decisión requerida
¿Hay un tiempo límite para que un cliente vencido pueda reactivarse?

---

## 10. Requisitos de Pruebas Automatizadas

**Prioridad**: MEDIA  
**Afecta**: CI/CD, calidad del código

### Pregunta
¿Qué nivel de cobertura de pruebas se requiere antes de cada release?

### Contexto
Se crearon 188 pruebas unitarias (118 exitosas en la primera ejecución). No existen pruebas de instrumentación (androidTest) ni tests de UI.

### Opciones

| Opción | Descripción |
|---|---|
| **A) Solo unit tests** | Mantener la cobertura actual (~80% lógica de negocio) |
| **B) + Room integration tests** | Agregar tests con base de datos in-memory para validar DAOs |
| **C) + UI tests** | Agregar Espresso/Compose tests para flujos críticos |
| **D) + E2E tests** | Tests completos de flujos de usuario con Maestro/Appium |

### Decisión requerida
¿Cuál es el标准 de calidad mínimo para cada release?

---

## Resumen de Prioridad

| # | Decisión | Prioridad | Impacto |
|---|---|---|---|
| 1 | Acumulación de pagos parciales | **CRÍTICA** | Integridad financiera |
| 2 | Periodicidad de re-mediciones | **ALTA** | Notificaciones incorrectas |
| 3 | Comportamiento "Reasignar" | **ALTA** | Funcionalidad incompleta |
| 4 | Validación de cupo grupal | **MEDIA** | Regla de negocio |
| 5 | Overlapping de planes | **MEDIA** | Modelo de datos |
| 6 | Permisos notificación | **MEDIA** | Experiencia usuario |
| 7 | Persistencia tras delete | **BAJA** | Cumplimiento |
| 8 | Estrategia Firestore | **BAJA** | Multi-dispositivo |
| 9 | Planes vencidos sin acción | **BAJA** | Flujo admin |
| 10 | Estándar de pruebas | **MEDIA** | Calidad |
