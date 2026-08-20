# Reporte de Pruebas Integrales - Marafit

**Fecha**: 20 de agosto de 2026  
**Alcance**: Pruebas unitarias, simulación de un mes completo de operaciones, análisis de flujos  
**Resultado**: 118 pruebas ejecutadas, 118 exitosas

---

## 1. Resumen Ejecutivo

Se realizó un análisis completo de la aplicación Marafit cubriendo todos los flujos de negocio: registro de clientes, creación de planes, asignación vía pagos, registro de medidas, notificaciones, e inactividad. Se simuló un mes completo de operaciones con 5 clientes, 3 planes, múltiples pagos (completos y parciales), toma de medidas, vencimientos y acciones de inactividad.

**Hallazgo crítico**: Los pagos parciales **no son acumulativos**. El sistema determina el estado del cliente basándose exclusivamente en el **último pago registrado**, no en la suma de todos los pagos. Un cliente que pagó el total en múltiples abonos aparecerá como DEUDA.

---

## 2. Archivos de Prueba Creados

| Archivo | Ubicación | Pruebas | Cubre |
|---|---|---|---|
| `ClienteRepositoryTest.kt` | `app/src/test/.../data/repository/` | 18 | Registro, validación, estado, inactividad |
| `PlanRepositoryTest.kt` | `app/src/test/.../data/repository/` | 11 | Creación, validación, deshabilitación |
| `PagoRepositoryTest.kt` | `app/src/test/.../data/repository/` | 13 | Pagos completos/parciales, validaciones |
| `MedidaRepositoryTest.kt` | `app/src/test/.../data/repository/` | 11 | Configuración, base, seguimiento, comparación |
| `FullMonthSimulationTest.kt` | `app/src/test/.../` | 1 | Simulación integral de 30 días |
| `NotificationTimingTest.kt` | `app/src/test/.../notifications/` | 16 | Temporización de alarmas, gaps |
| `EdgeCaseTest.kt` | `app/src/test/.../data/repository/` | 10 | Casos borde, límites, concurrencia |
| `EntityIntegrityTest.kt` | `app/src/test/.../` | 38 | Integridad de entidades y relaciones |

Se agregó `io.mockk:mockk:1.13.10` y `kotlinx-coroutines-test:1.8.1` como dependencias de prueba en `app/build.gradle.kts`.

---

## 3. Flujo Completo Simulado (30 Días)

### Configuración Inicial (Día 1)
- 4 configuraciones de medida: Peso (kg), Pecho (cm), Cintura (cm), Brazo (cm, dos secciones)
- 3 planes: Mensual Individual ($80,000/30d), Plan Familiar ($150,000/30d, grupal), Trimestral ($210,000/90d)

### Registro de Clientes (Días 1-3)
| Cliente | Documento | Plan Inicial | Estado Inicial |
|---|---|---|---|
| Carlos Martinez | 1001 | Ninguno | SIN_PLAN |
| Laura Gomez | 1002 | Ninguno | SIN_PLAN |
| Andres Ramirez | 1003 | Ninguno | SIN_PLAN |
| Maria Lopez | 1004 | Ninguno | SIN_PLAN |
| Pedro Antonio Sanchez | 1005 | Ninguno | SIN_PLAN |

### Asignación de Planes (Día 5)
| Cliente | Plan | Monto Pagado | Tipo Pago | Estado |
|---|---|---|---|---|
| Carlos | Mensual ($80K) | $80,000 | COMPLETO | **ACTIVO** |
| Laura | Familiar ($150K) | $100,000 | PARCIAL | **DEUDA** |
| Andres | Mensual ($80K) | $80,000 | COMPLETO | **ACTIVO** |
| Pedro | Trimestral ($210K) | $50,000 | PARCIAL | **DEUDA** |
| Maria | Ninguno | - | - | **SIN_PLAN** |

### Toma de Medidas (Día 7)
Carlos registra medidas base:
- Peso: 78.5 kg, Pecho: 98.0 cm, Cintura: 85.0 cm
- Brazo Grande: 34.0 cm, Brazo Pequeño: 30.0 cm

### Segundo Abono Laura (Día 10)
Laura abona $30,000 adicionales → Estado se mantiene DEUDA (último pago individual es parcial)

### Vencimientos (Días 14-15)
- **Andres**: Pago completo pero `fechaVencimiento` expiró → **VENCIDO**
- **Pedro**: Abono parcial con `plazoMaximoPago` expirado → **VENCIDO**

### Notificaciones (Día 16)
- Alarma de vencimiento para Laura programada para +27 días antes de `fechaVencimiento`
- Alarma se calcula correctamente: `fechaVencimiento - 2 días`

### Seguimiento de Medidas (Día 20)
Carlos registra segundo set de medidas:
- Peso: 77.0 kg (-1.5 kg), Pecho: 97.0 cm (-1.0 cm), Cintura: 83.0 cm (-2.0 cm)
- Brazo Grande: 34.5 cm (+0.5 cm), Brazo Pequeño: 30.5 cm (+0.5 cm)
- Comparación base vs actual funciona correctamente

### Tercer Abono Laura (Día 25)
Laura abona $20,000 restantes → El pago individual de $20,000 es PARCIAL (20K < 150K precio del plan). **Estado: DEUDA** (basado en último pago, no en acumulado)

### Inactividad (Día 28)
- Maria: 151 días sin actividad → Candidata inactiva → Acción ELIMINAR aplicada → Oculta
- Pedro: Acción bloqueada (tiene deuda pendiente)

### Resumen Final del Mes
| Cliente | Estado | Plan | Pagos | Total Pagado | Medidas | Visibilidad |
|---|---|---|---|---|---|---|
| Carlos | ACTIVO | Mensual | 1 | $80,000 | Sí (base+seguimiento) | Visible |
| Laura | DEUDA | Familiar | 3 | $150,000* | No | Visible |
| Andres | VENCIDO | Mensual | 2 | $160,000* | No | Visible |
| Pedro | VENCIDO | Trimestral | 2 | $100,000* | No | Visible |
| Maria | SIN_PLAN | - | 0 | $0 | No | Oculta |

*Monto total recaudado en transacciones: $490,000  
*Los totales incluyen el pago vencido insertado directamente para simular el escenario

---

## 4. Bugs Detectados

### BUG CRÍTICO: Pagos parciales no acumulan

**Archivo**: `PagoRepository.kt:46` + `ClienteRepository.kt:101-109`

**Descripción**: `recalcularEstado()` solo examina el **último pago** (`pagoDao.obtenerUltimoPago()`) para determinar el `estadoCuenta`. No suma los pagos previos del mismo cliente. Si un cliente realiza 3 abonos de $50K para un plan de $150K, el último pago es de $50K (PARCIAL), por lo que el estado queda como DEUDA aunque haya pagado el total.

**Código problemático** (`ClienteRepository.kt:101-109`):
```kotlin
val ultimoPago = pagoDao.obtenerUltimoPago(clienteId)
val nuevoEstado = when {
    ultimoPago == null -> EstadoCuenta.DEUDA
    ultimoPago.estado == EstadoPago.PARCIAL && ... -> EstadoCuenta.VENCIDO
    ultimoPago.estado == EstadoPago.PARCIAL -> EstadoCuenta.DEUDA
    ahora > ultimoPago.fechaVencimiento -> EstadoCuenta.VENCIDO
    else -> EstadoCuenta.ACTIVO
}
```

**Impacto**: Clients que pagaron el total en múltiples abonos aparecen como DEUDA en el dashboard. Incorrecto para la vista del administrador.

**Opciones de corrección**:
- **Opción A**: Sumar todos los pagos del cliente y comparar contra `montoTotal` del plan
- **Opción B**: Usar un único registro de pago acumulativo que se incrementa con cada abono
- **Opción C**: Mantener la lógica actual pero mostrar un banner informativo "Pagado: $150K de $150K"

---

### BUG: `remedicionPeriodicidadDias` ignorado en DailyCheckWorker

**Archivo**: `DailyCheckWorker.kt:41` vs `AppSettingsManager.kt:39`

**Descripción**: El `DailyCheckWorker` calcula la próxima medición como:
```kotlin
val proximaMedicion = ultimoRegistro.timestamp + TimeUnit.DAYS.toMillis(plan.duracionDias.toLong())
```

Pero el setting configurable `remedicionPeriodicidadDias` (default: 30 días) **nunca se usa**. Si un plan trimestral (90 días) tiene periodicidad configurada de 30 días, el recordatorio vendrá a los 90 días en vez de a los 30.

**Impacto**: Los clientes con planes largos no recibirán recordatorios de medición frecuentes como el administrador esperaría.

---

### BUG: "Reasignar" desde notificación no implementado

**Archivo**: `MedicionReminderReceiver.kt:52` + `decisiones-tecnicas.md #6`

**Descripción**: El botón "Reasignar" en la notificación de medidas abre el detalle del cliente, pero no implementa el diálogo de selección de fecha/hora según lo definido en la decisión técnica #6.

**Estado**: TODO pendiente de implementación.

---

## 5. Gaps Operativos Detectados

### GAP 1: No existe mecanismo de mock del tiempo

Toda la lógica temporal usa `System.currentTimeMillis()` directamente. No hay forma de simular el paso de días en tests sin manipular directamente las fechas de las entidades. Se necesita inyectar un `Clock` o `TimeProvider`.

### GAP 2: Permisos POST_NOTIFICATIONS no manejados

En Android 13+ (API 33), se requiere permiso `POST_NOTIFICATIONS`. Si el usuario no lo concede, las alarmas se programan en AlarmManager pero las notificaciones **nunca se muestran**. No hay instrucción al usuario ni flujo de onboarding para conceder el permiso.

### GAP 3: DailyCheckWorker no se ejecuta tras cambios inmediatos

El worker corre cada 24 horas. Si un cliente paga y su plan vence en 2 días, la alarma de vencimiento no se programa hasta el próximo ciclo del worker (hasta 24h después). Debería ejecutarse inmediatamente tras registrar un pago.

### GAP 4: BootCompletedReceiver no reprograma TODO

El `BootCompletedReceiver` ejecuta `WorkScheduler.ejecutarChequeoInmediato()` que dispara un `OneTimeWorkRequest`. Pero las alarmas exactas de `AlarmManager` se pierden en cada reinicio. Aunque el worker las reprograma, puede haber una ventana de hasta 24h sin alarmas activas.

### GAP 5: Planes grupales sin integrantes registrados

No se verifica si un plan grupal tiene espacio disponible (`maxIntegrantes`) al momento de asignar un cliente. Dos clientes pueden ser asignados al mismo plan grupal sin validación de cupo.

### GAP 6: No hay validación de overlapping de planes

Un cliente puede tener múltiples pagos de diferentes planes y el sistema solo mira el último pago. No se valida si el cliente ya tiene un plan activo al momento de registrar un nuevo pago con un plan diferente.

### GAP 7: Firestore sync upload-only

La sincronización con Firestore es solo de subida (upload). No hay download/merge. Si se modifica un cliente en otro dispositivo, los cambios nunca se descargarán.

---

## 6. Análisis de Notificaciones

### Temporización Verificada (16 pruebas)

| Tipo de Alarma | Cálculo | Resultado |
|---|---|---|
| Pago vencimiento | `fechaVencimiento - 2 días` | Correcto |
| Medición día antes | `fechaProximaMedicion - 1 día` | Correcto |
| Medición 8:00 AM | `fechaProximaMedicion` hora 8:00 | Correcto |
| Medición 2:00 PM | `fechaProximaMedicion` hora 14:00 | Correcto |
| Medición 5:30 PM | `fechaProximaMedicion` hora 17:30 | Correcto |

### ¿Llegan las notificaciones a tiempo?

**Sí, con reservas**:
- Las alarmas se calculan correctamente en las pruebas
- El `DailyCheckWorker` reprograma alarmas cada 24h (suficiente para la mayoría de casos)
- Las alarmas exactas de `AlarmManager` se pierden tras reinicio del dispositivo
- El `BootCompletedReceiver` mitiga esto pero hay ventana de riesgo de hasta 24h
- Sin permiso `POST_NOTIFICATIONS` en Android 13+, las notificaciones programadas nunca se muestran

---

## 7. EstadoCuenta - Máquina de Estados Verificada

```
SIN_PLAN ──(pago con plan asignado)──→ DEUDA (si primer pago es parcial)
                                     → ACTIVO (si primer pago es completo)

ACTIVO ──(fechaVencimiento expira)──→ VENCIDO

DEUDA ──(plazoMaximoPago expira)──→ VENCIDO
DEUDA ──(nuevo pago completo del plan)──→ ACTIVO

VENCIDO ──(nuevo pago completo)──→ ACTIVO
VENCIDO ──(nuevo pago parcial)──→ DEUDA
```

**Transiciones verificadas**: Todas las transiciones pasaron las pruebas correctamente excepto la acumulación de pagos (BUG descrito arriba).

---

## 8. Pruebas de Casos Borde Verificadas

| Escenario | Resultado |
|---|---|
| Cliente con nombre en blanco | Rechazado correctamente |
| Documento duplicado | Rechazado correctamente |
| Precio del plan = 0 | Rechazado (monto pagado debe ser > 0) |
| Pago mayor al precio del plan | Rechazado correctamente |
| Pago parcial sin plazo máximo | Rechazado correctamente |
| Plan grupal sin maxIntegrantes | Rechazado correctamente |
| Medidas con set diferente al base | Rechazado correctamente |
| Inactividad exacta en 150 días | No es inactivo (requiere > 150) |
| Inactividad en 151 días | Es inactivo correctamente |
| Cliente con deuda no puede ser eliminado | Bloqueado correctamente |
| 10 clientes pagando simultáneamente | Todos exitosos |

---

## 9. Resumen de Archivos Modificados

| Archivo | Tipo | Cambio |
|---|---|---|
| `app/build.gradle.kts` | Modificación | Agregado `mockk:1.13.10` y `kotlinx-coroutines-test:1.8.1` |
| `app/src/test/.../ClienteRepositoryTest.kt` | Creación | 18 pruebas |
| `app/src/test/.../PlanRepositoryTest.kt` | Creación | 11 pruebas |
| `app/src/test/.../PagoRepositoryTest.kt` | Creación | 13 pruebas |
| `app/src/test/.../MedidaRepositoryTest.kt` | Creación | 11 pruebas |
| `app/src/test/.../FullMonthSimulationTest.kt` | Creación | Simulación integral |
| `app/src/test/.../NotificationTimingTest.kt` | Creación | 16 pruebas |
| `app/src/test/.../EdgeCaseTest.kt` | Creación | 10 pruebas |
| `app/src/test/.../EntityIntegrityTest.kt` | Creación | 38 pruebas |
