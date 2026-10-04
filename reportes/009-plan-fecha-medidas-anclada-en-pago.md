# Plan 009: La fecha de toma de medidas se ancla en el pago, no en la última medida

**Fecha:** 2026-10
**Origen:** decisión del propietario sobre la regla de notificación de seguimiento de medidas.
**Documentos que este plan actualiza:** decisión 003 #2 (periodicidad global), `funcionalidades-producto.md` §4, `especificaciones-generales.md` §Notificaciones, `inventario-pantallas.md` §9.2.
**Alcance de código:** `notifications/CalculoAlarmasIdeales.kt`, `workers/DailyCheckWorker.kt`, `data/local/AppSettingsManager.kt`, `ui/ajustes/*`.
**Fuera de alcance:** el esquema de Room. No hay migración.

---

## 1. Por qué cambia

Hoy la fecha de la próxima toma de medidas se calcula desde la última medición:

```kotlin
proximaMedicion = cliente.ultimaMedidaTimestamp + periodicidadDias
```

Eso tiene tres problemas que el propietario detectó al ver el comportamiento real:

1. **Se desincroniza del plan.** La periodicidad es un ajuste global de 15 a 90 días que no sabe nada del plan contratado. Un plan de 21 días con periodicidad de 30 pide medidas 9 días después de que el plan venció.
2. **Es circular.** La fecha depende de una medición que, en principio, la propia notificación debería estar produciendo. Un cliente al que nunca se le midió nunca recibe recordatorios (la condición `ultimaMedidaTimestamp != null` lo bloquea), así que no hay nada que rompa el círculo.
3. **No coincide con la especificación.** `especificaciones-generales.md:15` dice que los recordatorios "dependen enteramente de su plan".

## 2. La regla nueva

```
próximaMedición = fechaPago + plan.duracionDias
```

Sobre esa fecha se programan las 4 alarmas que ya existen, sin cambios:

| Aviso | Momento |
|---|---|
| Aviso general | 1 día antes de la fecha de toma |
| Franja 8:00 | Día de la toma, 08:00 |
| Franja 14:00 | Día de la toma, 14:00 |
| Franja 17:30 | Día de la toma, 17:30 |

`decidirAlertasMedicion` (`PlanificadorAlertas.kt:142-173`) no se toca: ya recibe una fecha y devuelve las 4 decisiones. **Todo el cambio está aguas arriba, en de dónde sale esa fecha.**

### 2.1 Una corrección que el cambio arrastra

El aviso general hoy se dispara **2 días antes** en vez de 1, y no es un error de lectura: es una constante de pagos reutilizada por error.

```kotlin
// PlanificadorAlertas.kt:21 — constante de PAGO
val ANTICIPACION_PAGO = TimeUnit.DAYS.toMillis(2)

// PlanificadorAlertas.kt:153 — usada por MEDIDAS, variable que dice "un día antes"
val diaAntes = fechaProximaMedicion - ANTICIPACION_PAGO
```

`ANTICIPACION_PAGO` es correcta para pagos (`funcionalidades-producto.md:21` dice 2 días). Lo que está mal es que la rama de medidas la comparta. Este plan corrige las dos cosas juntas, porque si se implementa la regla nueva sin corregir esto, el aviso general queda 2 días antes y la especificación sigue siendo falsa.

**Qué hacer:** extraer la constante de su nombre actual y darle a la de medidas su propio valor de 1 día.

```kotlin
val ANTICIPACION_PAGO = TimeUnit.DAYS.toMillis(2)
val ANTICIPACION_MEDICION = TimeUnit.DAYS.toMillis(1)
```

**Por qué ningún test lo detecta hoy:** `NotificationTimingTest.kt:56-114` recalcula `TimeUnit.DAYS.toMillis(1)` en local y asserta sobre su propia aritmética, sin llamar a código de producción; y `CalculoAlarmasIdealesTest.avisarUnDiaAntes` (`:217-222`) construye la expectativa **llamando a `decidirAlertasMedicion`**. Ninguno de los dos puede fallar.

## 3. Decisiones tomadas por el propietario

| # | Pregunta | Decisión | Consecuencia técnica |
|---|---|---|---|
| 1 | ¿Y si la fecha pasa y el cliente no renueva? | **Se detiene hasta nuevo pago** | Cuando `próximaMedición` queda en el pasado, las 4 alarmas caen en `VENTANA_ANTICIPACION_VENCIDA` y no se programan. El cliente reaparece en el ciclo al registrar un pago nuevo. No hace falta ningún código de "reintento". |
| 2 | El ajuste "Periodicidad (días)" | **Eliminarlo de Ajustes** | Se borran el campo de texto, el método del ViewModel y las claves de DataStore. Sin configuración que prometa una cosa y el código haga otra. |
| 3 | Cliente con plan pagado y nunca medido | **Sí, empieza a notificar** | Se elimina la condición `ultimaMedidaTimestamp != null` como requisito. Es el efecto buscado: la fecha ya no sale de las medidas. |
| 4 | ¿Cuándo se invalida la reasignación manual? | **Con un pago nuevo O con medidas nuevas** | Se conservan las dosTrigger actuales y se añade el pago. Ver §5.2. |

### 3.1 El plan diario no necesita código nuevo

`TipoPlan` solo distingue `INDIVIDUAL` y `GRUPAL` (`Enums.kt:11-14`): **no existe un tipo "diario"**. Un plan diario es simplemente un plan con `duracionDias = 1`, y el de semilla ya viene con el seguimiento apagado:

```kotlin
// DevDataSeeder.kt:117-125
planMap["plan_diario"] = planRepo.crearPlan(
    nombre = "Diario",
    duracionDias = 1,
    beneficios = "Pase de un dia, sin seguimiento de medidas",
    incluyeSeguimientoMedidas = false)
```

La regla del propietario —"a un cliente con plan diario no se le toman medidas"— ya se cumple por la bandera `incluyeSeguimientoMedidas`. Pero se cumple **por casualidad de los datos**, no por una garantía: hoy nada impide que un administrador cree un plan de 1 día con la casilla de seguimiento marcada, y ese plan sí generaría recordatorios. Por eso el plan agrega la validación de §5.1.

## 4. Superficie de cambio

`calcularAlarmasIdeales` tiene un solo punto de entrada en producción (`DailyCheckWorker.kt:182`) y una sola construcción de su proyección (`DailyCheckWorker.kt:408-435`). El embudo es angosto, que es la mejor noticia de este plan.

```
AppSettingsManager.remedicionPeriodicidadDias        <- se elimina
        │
DailyCheckWorker.reconciliar()                       DailyCheckWorker.kt:165-166, 182-188
        │
calcularAlarmasIdeales(clientes, ahora, …)           CalculoAlarmasIdeales.kt:53-111
        │  ← CAMBIO PRINCIPAL: la fórmula de :81-94
        ▼
decidirAlertasMedicion(...)                          PlanificadorAlertas.kt:142-173  (sin cambios)
        │
DailyCheckWorker.ejecutar() → ReconciliadorAlarmas   DailyCheckWorker.kt:200-205, 268-357
```

**No se toca:** `PlanEntity`, `PagoEntity`, `PlanRepository`, `PagoRepository`, `ClienteRepository`, `PlanificadorAlertas`, `NotificationScheduler`, los receptores, `AppDatabase`.

### 4.1 `fechaVencimiento` ya es exactamente lo que se necesita

`PagoRepository.kt:138-142` ya deriva el vencimiento como `fechaPago + duracionDias`, con una salvedad deliberada (`Hallazgo 1.d`): continuar un período con saldo pendiente **no** renueva la vigencia.

```kotlin
val fechaVencimiento = if (saldoPeriodoVigente != null) {
    ordenadosDesc.first().fechaVencimiento
} else {
    fechaPago + TimeUnit.DAYS.toMillis(plan.duracionDias.toLong())
}
```

Esto significa que **la regla se implementa moviendo el ancla, no escribiendo aritmética nueva**: el pago más reciente no cancelado ya lleva la fecha correcta, y además es la misma columna que alimenta la alarma de vencimiento de pago. Reutilizarla evita que las dos alarmas se calculen con dos fórmulas que puedan divergir.

## 5. Cambios de código

### 5.1 `notifications/CalculoAlarmasIdeales.kt`

**(a) Proyección `ClienteParaAlertas` (`:15-28`)** — reemplazar el campo `ultimaMedidaTimestamp` por el ancla de pago:

```kotlin
/** `fechaPago` + `duracionDias` del plan pagado. `null` si el cliente no tiene pagos válidos. */
val proximaMedicionPorPago: Long? = null
```

`ultimaMedidaTimestamp` **se conserva**, pero deja de ser el ancla y pasa a servir solo para §5.2. Sin este campo, la invalidación de la reasignación por "medidas nuevas" quedaría imposible de evaluar.

**(b) Fórmula (`:81-94`)**:

```kotlin
if (cliente.planIncluyeSeguimientoMedidas) {
    // Sin pago no hay fecha de la que partir: el plan se contrató junto con el pago.
    val porPago = cliente.proximaMedicionPorPago
    if (porPago != null) {
        var proximaMedicion = porPago
        cliente.medicionReasignada?.let { (definidaEn, nuevaFecha) ->
            val obsoleta = (cliente.ultimaMedidaTimestamp ?: Long.MIN_VALUE) >= definidaEn ||
                cliente.fechaPagoAncla >= definidaEn
            if (!obsoleta) {
                proximaMedicion = nuevaFecha
            }
        }
        decidirAlertasMedicion(remedicionHabilitada, ahora, proximaMedicion)...
    }
}
```

Dos diferencias respecto a hoy, y las dos son deliberadas:

- Se **quitó** el requisito `ultimaMedidaTimestamp != null` (decisión 3 del propietario).
- Se **quitó** el uso de `periodicidadDias`. El parámetro y su clamp (`CalculoAlarmasIdeales.kt:60-63`) quedan sin uso y se eliminan de la firma.

`ultimaMedidaTimestamp` **no se borra**: pasa de ser el ancla a ser solo una de las dos señales que invalidan una reasignación. Por eso el `?: Long.MIN_VALUE` —sin registro previo, la señal es "nunca se invalidó por medidas", no "se invalidó".

**(c) Invalidación de la reasignación (decisión 4).** Hoy la condición es `ultimaMedidaTimestamp < definidaEn`. Pasa a ser un OR de dos condiciones, porque bajo la regla nueva hay dos hechos que pueden volver obsoleta una reasignación:

| Hecho | Por qué invalida | Cómo se detecta |
|---|---|---|
| El cliente registró medidas nuevas | Ya tomó medidas, la fecha pospuesta perdió su objeto | `ultimaMedidaTimestamp >= definidaEn` |
| El cliente registró un pago nuevo | El ancla se recalculó desde el pago nuevo | `fechaPagoDelAncla >= definidaEn` |

Esto exige que la proyección lleve **también** el `timestamp` del pago que sirvió de ancla, no solo la fecha calculada. Sin eso no hay forma de distinguir "el ancla no cambió" de "el ancla se movió".

> **Nota de diseño:** la segunda condición es la que hace que una reasignación no sobreviva indefinidamente. Sin ella, un cliente que renueva tres veces acumularía reasignaciones emanadas de renovaciones distintas que se pisan entre sí, y el comportamiento sería imposible de explicar.

### 5.2 `workers/DailyCheckWorker.kt`

**(a) Proyección (`:408-435`)** — cambiar el cálculo del ancla. De paso se corrige un bug latente que la fórmula nueva vuelve visible (§5.3):

```kotlin
val pagosVigentes = pagoRepository.observarPagosDeCliente(id).first()
    .filter { it.estado != EstadoPago.CANCELADO }
val pagoAncla = pagosVigentes.maxByOrNull { it.fechaPago }
val planPagado = pagoAncala?.let { planRepository.obtenerPlan(it.planId) }
```

`duracionDias` se lee del plan **del pago** (`pagoAncla.planId`), no del plan actual del cliente. Ver §6.1.

**(b) Firma y lectura del ajuste (`:165-166`, `:182-188`)** — quitar la lectura de `remedicionPeriodicidadDias` y el argumento.

**(c) `limpiarReasignacionesObsoletas` (`:364-384`)** — aplicar el mismo OR de §5.1, para que la limpieza del DataStore y la decisión del cálculo no discrepen.

**(d) KDoc (`:97-101`)** — el comentario "la periodicidad de re-mediciones viene del setting global (15-90 días, default 30), no de la duración del plan" queda directamente invertido por este plan y debe reescribirse.

### 5.3 Bug latente que se corrige de paso

`DailyCheckWorker.kt:416-419` toma el pago más reciente y **después** filtra los cancelados:

```kotlin
.maxByOrNull { it.fechaPago }
?.takeIf { it.estado != EstadoPago.CANCELADO }
```

Si el pago más reciente está cancelado, el resultado es `null` aunque exista un pago válido más antiguo. Hoy eso solo afecta a la alarma de pago; bajo la regla nueva el cliente perdería **también** las 4 alarmas de medidas. El patrón correcto ya se usa en `ClienteRepository.kt:152`:

```kotlin
.filter { it.estado != EstadoPago.CANCELADO }.maxByOrNull { it.fechaPago }
```

## 6. Riesgos

### 6.1 `cliente.planId` puede divergir del plan del pago

`modificarPago` (`PagoRepository.kt:234-302`) puede mover la `fechaPago` de un pago sin reasignar el plan — la guarda que lo impide (`:159-161`) solo existe en `registrarPago`. Si `duracionDias` se leyera de `cliente.planId`, se calcularía el vencimiento con la duración del plan equivocado.

**Mitigación:** `duracionDias` viene de `pagoAncla.planId`. El gate `planIncluyeSeguimientoMedidas` sigue viniendo de `cliente.planId` porque es el plan que el cliente tiene activo; se acepta la asimetría a propósito y se documenta.

### 6.2 Un pago retroactivo puede apuntar a otro plan

`registrarPago` acepta pagos con fecha pasada (`esRetroactivo`). Si ese pago es del plan A y el cliente tiene el plan B, el ancla queda anclada en el plan A. Es el comportamiento correcto —el pago se hizo por el plan A— pero conviene que el test lo cubra explícitamente para que no se lea como bug.

### 6.3 La alarma de pago y la de medidas comparten fecha

Ambas se anclan en el mismo pago, así que disparan sobre la misma fecha, pero con reglas distintas: `decidirAlertaPago` descarta pagos vencidos hace más de 48 h (`VENTANA_MAXIMA_VENCIMIENTO_MS`), `decidirAlertasMedicion` no tiene tope de antigüedad. Es intencional —las medidas se piden aunque el pago esté atrasado (decisión 003 #3.5)— pero debe quedar escrito.

### 6.4 Borrar un plan en cascada deja al cliente sin alarmas

`pagos.planId` es `ON DELETE CASCADE` y `clientes.planId` es `ON DELETE SET NULL`: borrar un plan borra sus pagos y deja al cliente sin plan → conjunto ideal vacío → alarmas canceladas. Comportamiento preexistente, no introducido aquí, pero ahora más fácil de alcanzar.

## 7. Trabajo de pruebas

`CalculoAlarmasIdealesTest.kt` es el archivo principal. Re-escribir o eliminar:

| Test actual | Acción |
|---|---|
| `sin una toma previa no hay fecha de la que partir` (`:94`) | **Se elimina.** Invertido por decisión 3. Su reemplazo afirma que un cliente sin toma previa **sí** recibe las 4 franjas. |
| `un cliente con seguimiento tiene las cuatro franjas` (`:84`) | Re-escribir con el ancla de pago. |
| `al que le toca medirse hoy recibe las tres franjas pero no el aviso previo` (`:115`) | Re-escribir: el ancla se fija explícitamente, no se simula "medido hace 30 días". |
| `la reasignacion manual mueve la proxima medicion` (`:153`) | Re-escribir. |
| `una reasignacion obsoleta se ignora` (`:168`) | **Se duplica en dos**: obsoleta por medidas nuevas, y obsoleta por pago nuevo. |
| `una periodicidad fuera de rango se acota en vez de aceptarse` (`:186`) | **Se elimina.** No hay periodicidad que acotar. |
| `sin pago no hay alarma de pago` (`:70`) | Se conserva y se le añade el gemelo: sin pago tampoco hay alarma de medidas. |

Tests nuevos que hay que escribir:

| Test | Qué fija |
|---|---|
| `la fecha de toma es el pago mas los dias del plan` | La fórmula central. Caso del propietario: pago hoy + plan de 30 → dentro de 30 días. |
| `un plan de 21 dias tiene su propia fecha` | Que la duración sale del plan, no de un valor fijo. |
| `un plan diario no genera franjas aunque se marque el seguimiento` | La garantía explícita de §3.1, para que no dependa de la semilla. |
| `un plan sin seguimiento no genera franjas` | Ya existe como `un cliente sin seguimiento de medidas no genera franjas` (`:108`); se conserva. |
| `pagar una deuda no extiende la fecha de toma` | Fija el comportamiento de `saldoPeriodoVigente` (`PagoRepository.kt:138-142`) desde el lado de las alarmas. |
| `un pago cancelado no borra el ancla de un pago valido anterior` | El bug de §5.3. |
| `el aviso general es un dia antes` | **Regresión de §2.1**, y esta vez calculando la expectativa a mano, sin llamar a la función de producción. |
| `las 4 franjas caen dentro de 06:00 a 21:00` | Ya existe en `PlanificadorAlertasTest.kt:295-309`. |

`NotificationTimingTest.kt:185` (`DailyCheckWorker usa la periodicidad configurada por el admin`) **se elimina**: asserta exactamente el comportamiento que este plan borra.

## 8. Documentación a actualizar

| Documento | Línea | Qué dice hoy | Qué debe decir |
|---|---|---|---|
| `Pendientes/003-respuesta-decisiones-pendientes.md` | `:8-9` | Decisión 003 #2: periodicidad en días, 15–90, default 30, "sin importar el plan" | Supersedida. Registrar la regla nueva y que el ajuste se elimina. |
| `funcionalidades-producto.md` | `:23` | "recordatorio periódico configurable … tomando como referencia la fecha del último registro" | Referencia a `fechaPago + duracionDias` del plan. |
| `especificaciones-generales.md` | `:15,17` | "1 día antes, y en 3 secciones del día" | Correcto tal cual, pero depende de §2.1 para que el código lo cumpla. Añadir que la fecha de toma es el vencimiento del plan. |
| `pantallas/inventario-pantallas.md` | `:169-170` | "ajustar la periodicidad del recordatorio de re-medición" | El ajuste ya no existe; describir la derivación automática. |
| `pantallas/dialogos-confirmacion.md` | `:108-122` | Tabla de tokens | Ya actualizada con `WarningYellow` y la tabla de `EstadoCuentaVisual`. Sin cambios adicionales. |
| `ADR-notificaciones.md` | — | Sin regla de ancla de fecha | **ADR nuevo**: documentar que el ancla es el pago, no la última medida, y por qué. |
| `reportes/gaps/problems.md` | `:11` | "periodicidad ignorada en favor de la duración total del plan" — el bug ya resuelto | Ya no aplica; la duración del plan ahora manda por diseño. |
| `reportes/002-reporte-pruebas-integrales.md` | `:130-141` | Reporte histórico del bug `ultimoRegistro + duracionDias` | **No tocar**: es el registro histórico de un hallazgo ya resuelto. |

## 9. Fases

| Fase | Alcance | Dependencias | Tamaño |
|---|---|---|---|
| 1 | Guardas de plan: duración mínima con seguimiento, y filtro de pagos cancelados (§5.1, §5.3) | — | Pequeña |
| 2 | Ancla de pago en `CalculoAlarmasIdeales` + proyección en `DailyCheckWorker` (§5.1, §5.2) | Fase 1 | Mediana |
| 3 | Corrección del aviso general a 1 día (§2.1) + regresión | — | Pequeña |
| 4 | Reasignación: nuevo criterio de invalidación (§5.1c) | Fase 2 | Mediana |
| 5 | Retiro del ajuste de periodicidad de Ajustes y DataStore | Fase 2 | Pequeña |
| 6 | Re-escritura y ampliación de pruebas (§7) | Fases 1–5 | Mediana |
| 7 | Documentación (§8) + ADR | Fase 6 | Pequeña |

Las fases 1 y 3 son independientes y pueden ir en cualquier orden. La 5 depende de la 2 porque el ajuste no se puede quitar antes de que la regla nueva esté funcionando: si se quita antes, los clientes quedan sin fecha de toma en el intervalo.

## 10. Verificación

- `./gradlew :app:testDebugUnitTest` — verde.
- `./gradlew :app:compileDebugKotlin` — sin warnings nuevos.
- Prueba manual del caso del propietario: cliente con plan mensual de 30 días pagado hoy → las 4 alarmas caen dentro de 30 días, con el aviso general el día 29.
- Prueba manual: mismo cliente, plan diario pagado hoy → 0 alarmas de medidas.
- Prueba manual: plan de 15 días pagado hace 7 días → las alarmas caen dentro de 8 días.
- Reiniciar la app y comprobar que la reconciliación no reprograma nada (idempotencia).