# Reporte de Capacidad del Administrador — Acceso Total y Modificación en Cualquier Momento

**Fecha:** 2026-09-20
**Proyecto:** Mary Fitness — App de Gestión de Gimnasio (Mono-usuario local)
**Solicitante:** El administrador debería tener acceso a todo y estar en capacidad de modificar todo: clientes, planes, medidas, registros. Con modificar se refiere a todo en cualquier momento: creación, eliminación, asignación, modificación activa, etc. Todo.
**Alcance del reporte:** Estado actual en **interfaces visuales** y **código** (Room/Business Logic) respecto al requisito de capacidad total.

---

## 1. Resumen Ejecutivo

**Modelo vigente:** rol único **Administrador** con autenticación local por PIN (`app/src/main/java/com/maryfitness/app/auth/AdminAuthManager.kt:17`). No existe multi-usuario, RBAC ni gating por rol: una vez autenticado, el NavGraph (`app/src/main/java/com/maryfitness/app/ui/navigation/MaryFitnessNavGraph.kt:81`) expone todas las rutas. **Acceso a todo** se cumple a nivel navegación.

**Capacidad de modificación total — no se cumple.** El administrador **no** puede modificar todo en cualquier momento. Existen operaciones CRUD completas solo para **Pagos**; para **Clientes**, **Planes** y **Medidas/Registros** hay bloqueos de negocio, soft-delete sin UI directa y registros inmutables.

| Dominio | Crear (UI) | Leer (UI) | Actualizar (UI) | Eliminar (UI) | Asignar (UI) | Modificar activa (UI) | Código permite más que UI | Veredicto vs "todo en cualquier momento" |
|---|---|---|---|---|---|---|---:|---|
| **Clientes** | ✅ | ✅ | ⚠️ Parcial | ❌ No | ⚠️ Solo vía pago | ⚠️ Solo vía flujo inactividad | Sí (DAO puede purgar) | **INCUMPLE** |
| **Planes** | ✅ | ✅ | ✅ | ❌ No | ⚠️ Solo vía pago | ❌ No | Sí (deshabilitar existe sin UI) | **INCUMPLE** |
| **Config. Medida** | ✅ | ✅ | ✅ | ⚠️ Solo desactivar | N/A | ⚠️ Solo desactivar | Sí (activa=true/false) | **INCUMPLE** |
| **Registros de Medida** | ✅ | ✅ | ❌ No | ❌ No | N/A | N/A | ⚠️ DAO tiene `actualizarRegistro` sin uso | **INCUMPLE** |
| **Pagos/Registros financieros** | ✅ | ✅ | ✅ | ✅ | ✅ | N/A | ✅ | **CUMPLE** |
| **Asignación plan↔cliente** | — | — | — | — | ⚠️ Condicionada | — | Bloqueada por deuda/cupos | **INCUMPLE** |

**Conclusión:** De las 5 familias evaluadas, solo **Pagos** satisface "todo en cualquier momento". Las otras 4 requieren intervención para cerrar la brecha.

---

## 2. Modelo de Administración Actual

### 2.1 Autenticación y rol

- `AdminAuthManager.kt:17-36` — PIN 4-6 dígitos, hash BCrypt con `cost=12`, `DataStore` `admin_prefs`. Métodos: `existePin()`, `configurarPin()`, `validarPin()`, `resetearPin():41`.
- `LoginViewModel.kt:19-56` — estados `RequiereConfiguracion` / `RequierePin` / `Autenticado`. Sin expiración de sesión, sin logout explícito, sin cambio de PIN con UI.
- `LoginScreen.kt:86` — flujo de creación de PIN con solicitud `POST_NOTIFICATIONS` (D6). No hay pantalla de "cambiar PIN" ni "resetear PIN".
- `MainActivity.kt:18-44` — `postLoginDestination` resuelve deep-link de notificación (`ACCION_REASIGNAR_MEDICION`) pero no hay gating por rol: todo es admin.
- **No hay RBAC.** No existen tablas `usuario/rol/permiso`, no hay `isAdmin` ni middleware. `MaryFitnessNavGraph.kt:66-79` mapea 5 pestañas (`TabDestino` en `AppScaffold.kt:34`) visibles para todos los autenticados. Acceso = total por diseño.

### 2.2 Implicación del requisito

El requisito "administrador debería tener acceso a todo" **se interpreta como cumplido** a nivel navegación: no hay pantalla oculta ni feature-flag por rol. La brecha está en **capacidad de modificación**.

---

## 3. Interfaces Visuales — Qué Puede Hacer Hoy el Administrador

### 3.1 Navegación global

Archivo: `app/src/main/java/com/maryfitness/app/ui/navigation/Routes.kt:3` y `MaryFitnessNavGraph.kt:60`

```
LOGIN → DASHBOARD → CLIENTES → CLIENTE_NUEVO / CLIENTE_DETALLE / CLIENTE_EDITAR / CLIENTE_PAGO / CLIENTE_MEDIDA / CLIENTE_HISTORIAL_MEDIDAS
              → PLANES → PLAN_NUEVO / PLAN_EDITAR → EJERCICIOS / EJERCICIO_DETALLE
              → CONFIGURACION_MEDIDAS
              → AJUSTES → NOTIFICACIONES / VISUAL / ACCIONES / SINCRONIZACION / ACERCA_DE
```

`AppScaffold.kt:48` — barra superior + barra inferior con 5 destinos: INICIO, CLIENTES, PLANES, MEDIDAS, AJUSTES. Sin restricciones.

### 3.2 Clientes

**Pantalla lista — `ClientesListScreen.kt:58`**
- Buscar por `nombreCompleto` o `numeroDocumento` (`ClientesListScreen.kt:86`).
- Filtros: `TODOS / ACTIVOS / DEUDORES / DESHABILITADOS` (`ClientesListScreen.kt:50`). Conteo en chip.
- Card por cliente con `StatusChip` (`ClientesListScreen.kt:167`): Activo / Con Deuda / Vencido / Inactivo / Sin Plan.
- FAB `+` → `CLIENTE_NUEVO` (`ClientesListScreen.kt:132`).

**Formulario — `ClienteFormScreen.kt:32`**
- **Crear (`clienteExistente == null`):** campos `primerNombre*`, `segundoNombre`, `primerApellido*`, `segundoApellido`, `tipoDocumento*` (dropdown `CEDULA_CIUDADANIA/TI/CE/PASAPORTE/NIT:146`), `numeroDocumento*`, `telefono`, `plan` (dropdown de `planesActivos`:99). Botón `Guardar` → `ClienteFormViewModel.crear():48`.
- **Editar (`clienteExistente != null`):** reutiliza mismo Composable pero **no expone selector de plan** (`ClienteFormScreen.kt:128-139`: `if (clienteExistente == null) crear else actualizar`). El VM `actualizar()` en `ClienteViewModel.kt:70` solo reenvía `primerNombre/segundoNombre/.../telefono` a `ClienteRepository.actualizarDatosBasicos():85` — **no permite cambiar `planId`**. Asignación de plan queda fuera del formulario.
- Sin botón Eliminar en lista ni en formulario.

**Detalle — `ClienteDetailScreen.kt:43`**
- Avatar, nombre, documento, chip `EstadoCuenta` (`ClienteDetailScreen.kt:71`).
- Banner deuda/vencido (`ClienteDetailScreen.kt:82`).
- Acciones: `Registrar pago` (`:93`), `Registrar medida` (`:94`), `Editar datos` (`:98`), `Ver historial` (`:99`), `Reasignar toma de medidas` (`:103`), `Cancelar plan actual` (`:111`, con `AlertDialog:178`).
- **Inactivos (>5 meses):** tarjeta `GlassCard` (`:121`) con 3 botones `Eliminar Cliente / Marcar como Inactivo / Mantener en Lista` (`:135-154`). Habilitados solo si `!tieneSaldoPendiente` (`:137`). Mensaje bloqueante si hay deuda: `"Salda la deuda..."` (`:127`).
- **No existe:** eliminar cliente arbitrario, rehabilitar cliente deshabilitado/oculto, cambiar plan directo, editar fecha de registro, reactivar medidas.

### 3.3 Planes

**Lista — `PlanesListScreen.kt:48`**
- Cards con `ACTIVO/DESHABILITADO` (`:108`), categoría `Individual/Grupal (máx X)`, precio, duración, beneficios, botón `Editar` (`:149`).
- FAB `+` → `PLAN_NUEVO` (`:95`). Enlace `Biblioteca de ejercicios →` (`:79`).
- **No hay** botón Eliminar, ni toggle Habilitado, ni contador de clientes por plan, ni detalle con clientes asociados (gap documentado en `006-reporte-comparativo:235`).

**Formulario — `PlanFormScreen.kt:38`**
- **Crear:** `nombre*`, `precio*`, `duracionDias*`, `beneficios`, radio `INDIVIDUAL/GRUPAL` (`:74`), `maxIntegrantes*` si grupal (`:89`), checkbox `incluyeSeguimientoMedidas` (`:97`). → `PlanFormViewModel.crear():41`.
- **Editar:** `copy(nombre, precio, duracionDias, beneficios, tipo, maxIntegrantes, incluyeMedidas)` → `PlanFormViewModel.actualizar():57`. **No expone campo `habilitado`**; no se puede activar/desactivar desde el formulario.
- Validaciones en `PlanRepository.crearPlan():27-35` (nombre no vacío, precio>=0, duración>0, grupal exige max>=2).

**Capacidad activa:** `PlanRepository.deshabilitarPlan():58` y `PlanListViewModel.deshabilitar():26` existen en código pero **nunca se invocan desde UI** (`grep PlanesListScreen` no referencia `deshabilitar`). No hay "Eliminar plan" en ningún `Dao`.

### 3.4 Medidas — Configuraciones

**Pantalla — `ConfiguracionMedidasScreen.kt:40`**
- Lista de `configuraciones` activas (`:58`) con `GlassCard` clickeable → `viewModel.iniciarEdicion(config)` (`:60`). Botón tacho → `configuracionAEliminar` (`:80`) → `AlertDialog Desactivar medida` (`:150`).
- Formulario inferior "Agregar nueva medida": `nombreCompleto*`, `nombreAbreviado*`, `unidad*`, checkboxes `dosSecciones`, `obligatoria` → `viewModel.crear():42` (`:127`).
- Dialog edición `EditarConfiguracionDialog:173` con los mismos 5 campos → `viewModel.editar():48`.
- **Solo "Desactivar"** (soft `activa=false` en `MedidaRepository.desactivarConfiguracion():38`). No hay Eliminar físico, ni Reactivar (requiere `actualizarConfiguracion(... activa=true)` pero sin UI para listar inactivas). `MedidaDao.observarConfiguracionesActivas():17` filtra `activa=1`, las desactivadas desaparecen sin retorno.

### 3.5 Medidas — Registros (tomas por cliente)

**Registrar — `RegistrarMedidaScreen.kt:50`**
- Header con cliente (`:75`), `InfoBanner` referencia base (`:88`), selector fecha `DatePicker` (`:91-104`, permite fecha pasada — D3.3 `:61`), campos dinámicos según `configuraciones` (`:105-136`: `tieneDosSecciones` genera 2 inputs `MAS_GRANDE/MAS_PEQUENA`, sino `UNICA`), botón `Guardar Medidas Corporales` → `RegistrarMedidaViewModel.registrar():46`.
- Validación en `MedidaRepository.registrarMedida():75`: no vacío, no futuro (+60s tolerancia `:23`), y si existe base, `setNuevo == setBase` (`:92-98`) o error.

**Historial — `HistorialMedidasScreen.kt:27`**
- Solo lectura: tabla `Medida | Base | Anterior | Actual` (`:42`). Datos de `MedidaRepository.obtenerComparacion():116` (`base / anterior / actual`). Mensaje si `comparacion==null` (`:38`).
- **No hay** Editar registro, Eliminar registro, ni Editar valor individual. `MedidaDao` sí tiene `actualizarRegistro():63` y `actualizarValor():66` pero `MedidaRepository` no los expone y ningún ViewModel/Screen los usa.

### 3.6 Pagos — Registros financieros

**Pantalla — `RegistrarPagoScreen.kt:54`**
- Dropdown plan con `precio` y `ocupación grupal` (`:93-128`, bloquea si `lleno && planId != planActualId:109` — D4).
- Resumen financiero `Total abonado: $X de $Y / Saldo pendiente` (`:132`, `PeriodoFinanciero` de `ClienteRepository.calcularPeriodoActual():50`).
- Campo `Monto pagado*` (`:151`), botón `Pagar saldo restante` si hay periodo vigente (`:164`), lógica `esAbono = monto in 1 until saldoReferencia` (`:174`) con campo `Plazo máximo (días)*` (`:192`).
- Botón `Registrar` → `PagoViewModel.registrarPago():72`.
- **Historial (`:219`):** por cada `PagoEntity` fila con `fecha · $monto · estado` (`COMPLETO/CANCELADO/PARCIAL`) + 3 iconos: `+ Abonar` si `abonable` (`:252`, Hallazgo 1.a), `✎ Editar` (`:257`), `🗑 Eliminar` (`:260`). Diálogos `DialogoAbonarPago:308`, `DialogoEditarPago:359`, `AlertDialog Eliminar:290`.
- Edición: `modificarPago()` con techo `techo = montoTotal - (totalPagado - montoActual)` si en periodo vigente (`PagoRepository.modificarPago():187-199`). `abonarSobrePago()` incrementa monto sin crear fila nueva (`PagoRepository.abonarSobrePago():137`). `eliminarPago()` borra física (`PagoRepository.eliminarPago():232` → `PagoDao.eliminar():37`) y recalcula estado (`clienteRepository.recalcularEstado`).
- **Es la única entidad con CRUD completo y asignación flexible vía UI.**

---

## 4. Código — Qué Permite Realmente la Capa de Datos

### 4.1 Entidades

- `ClienteEntity.kt:22` — `id, primerNombre, segundoNombre, primerApellido, segundoApellido, tipoDocumento (enum), numeroDocumento (unique index:20), telefono, planId (FK SET_NULL:13), fechaRegistro, fechaUltimaActividad, estadoCuenta (cache derivado), habilitado=true, oculto=false, fechaOcultado, fechaProximaRevisionInactividad, pendienteSync`. Sin `fotoPerfil`, sin `estado` adicional.
- `PlanEntity.kt:8` — `id, nombre, precio, duracionDias, beneficios, tipo (INDIVIDUAL/GRUPAL), maxIntegrantes, incluyeSeguimientoMedidas, habilitado, pendienteSync`.
- `ConfiguracionMedidaEntity` (Room) — `id, nombreCompleto, nombreAbreviado, unidadMedida, tieneDosSecciones, obligatoria, activa, pendienteSync` (vía `MedidaDao:17,26`).
- `MedidaRegistroEntity.kt:20` — `id, clienteId (FK CASCADE), timestamp, esBase, pendienteSync`.
- `MedidaValorEntity` — `id, registroId (FK CASCADE), configuracionMedidaId, seccion (UNICA/MAS_GRANDE/MAS_PEQUENA), valor`.
- `PagoEntity.kt:27` — `id, clienteId (FK CASCADE), planId (FK CASCADE), montoTotal, montoPagado, estado (COMPLETO/PARCIAL/CANCELADO), fechaPago, fechaVencimiento, plazoMaximoPago, pendienteSync`.

### 4.2 DAOs — Capacidad física

| DAO | Métodos relevantes | Gaps |
|---|---|---|
| `ClienteDao.kt:17` | `observarClientesVisibles:20` (`oculto=0`), `observarCliente:24`, `obtenerCliente:27`, `obtenerPendientesSync:34`, `obtenerCandidatosInactivos:36`, `insertar:46`, `actualizar:49`, `contarPorDocumento:52`, `contarTodos:55`, `contarVisiblesPorPlan:59`, `observarCuposPorPlan:63`, `purgarOcultos:78` (`DELETE WHERE oculto=1 AND fechaOcultado < limite`) | **No tiene `eliminar(cliente)`**. No hay `@Delete`, no hay `SELECT ocultos`, no hay `actualizar habilitado` dedicado (se hace vía `actualizar` genérico). |
| `PlanDao.kt:12` | `observarPlanes:14`, `observarPlanesActivos:17`, `observarPlan:20`, `obtenerPlan:23`, `obtenerPendientesSync:27`, `insertar:30`, `actualizar:33` | **No tiene `eliminar` ni `deshabilitar` dedicado**; no expone `observarPlanesInactivos`. |
| `MedidaDao.kt:14` | `observarConfiguracionesActivas:17`, `observarConfiguracionPorId:20`, `insertarConfiguracion:23`, `actualizarConfiguracion:26`, `observarRegistrosDeCliente:29`, `obtenerRegistroBase:32`, `obtenerUltimoRegistro:35`, `obtenerRegistroAnterior:38`, `obtenerValoresDeRegistro:41`, `obtenerRegistrosPendientesSync:44`, `obtenerValoresPendientesSync:48`, `obtenerConfiguracionesPendientesSync:52`, `insertarRegistro:56`, `insertarValores:59`, `actualizarRegistro:62`, `actualizarValor:65`, `insertarRegistroConValores:68` (transaccional) | **No tiene `eliminarRegistro` ni `eliminarValor` ni `eliminarConfiguracion`**. Tiene `actualizar*` pero sin uso en Repository para registros. |
| `PagoDao.kt:12` | `observarPagosDeCliente:15`, `obtenerPagosDeCliente:18`, `obtenerUltimoPago:21`, `obtenerPago:24`, `obtenerPendientesSync:28`, `insertar:31`, `actualizar:34`, `eliminar:37` (`@Delete`) | **CRUD completo** (única con `@Delete`). |

### 4.3 Repositories — Reglas de negocio que limitan "en cualquier momento"

**`ClienteRepository.kt:41`**
- `crearCliente():52` — valida no vacío, `contarPorDocumento>0` → error, asigna `estadoCuenta = DEUDA si planId!=null else SIN_PLAN`.
- `actualizarDatosBasicos():85` — valida nombres/documento, hace `copy(... pendienteSync=true)` sin tocar `planId/habilitado/oculto`.
- `cancelarPlanActual():184` — marca pagos `CANCELADO`, `planId=null`, recalcula. Es la **única vía para desasignar plan** sin pagar deuda.
- `obtenerPlanActualBloqueante():168` — bloquea asignación si `conDeuda || vigenteReciente (7 días gracia GRACIA_PLAN_RECIENTE_MS:29)`. Impide "asignar cualquier plan en cualquier momento".
- `aplicarAccionInactividad():225` — bloqueada si `tieneSaldoPendiente():157` (`saldoPendiente>0`). Acciones: `ELIMINAR → oculto=true + fechaOcultado`, `DESHABILITAR → habilitado=false`, `IGNORAR → fechaProximaRevision +30d`. **Sin vía para rehabilitar** (no hay `restaurar`/`habilitar`).
- `purgarClientesOcultos():204` — `DELETE` tras 180 días (`RETENCION_OCULTOS_MS:22`). Automático vía `DailyCheckWorker`.
- `tieneSaldoPendiente / calcularPeriodoActual:249` — acumula abonos del periodo vigente (abonos parciales suman, frontera es `COMPLETO` previo).

**`PlanRepository.kt:8`**
- `crearPlan():18` — valida nombre, precio/duración, grupal exige `max>=2`.
- `actualizarPlan():50` — valida nombre, `copy(pendienteSync=true)`.
- `deshabilitarPlan():58` — `copy(habilitado=false)`. **No hay `habilitar` ni `eliminar`**.

**`MedidaRepository.kt:25`**
- `crearConfiguracion():44`, `actualizarConfiguracion():30`, `desactivarConfiguracion():38` (`activa=false`).
- `registrarMedida():75` — obliga a mismo `set(configId,seccion)` que base si existe; tolera fecha pasada. **No hay `actualizarRegistro`/`eliminarRegistro`/`actualizarValor`/`eliminarValor`** aunque el DAO los soporta.

**`PagoRepository.kt:12`** — **CRUD total del admin (D1, Hallazgo 1)**
- `registrarPago():37` — valida plan existe, monto 0< `monto <= precio`, saldo periodo vigente, `esCompleto`, plazo si parcial, `obtenerPlanActualBloqueante` (bloquea solapamiento), valida cupo grupal (D4 `contarVisiblesPorPlan`), calcula `fechaVencimiento` (si saldo vigente → respeta vencimiento prometido, sino `ahora+duracionDias:106`), inserta, `cliente.planId=planId` si cambió, `registrarInteraccion + recalcularEstado`.
- `abonarSobrePago():137` — solo `PARCIAL` del periodo vigente, `montoAbono <= saldoPendiente`, si liquida → `COMPLETO` y `plazo=null`.
- `modificarPago():187` — techo dinámico `techo = montoTotal - (totalPagado - montoActual)` si en periodo vigente, sino `montoTotal`; exige plazo si queda parcial.
- `eliminarPago():232` — `@Delete` física + `recalcularEstado`.

### 4.4 ViewModels — Lo que la UI puede invocar

- `ClienteFormViewModel:37` — `crear()` / `actualizar()` (sin plan). `ClienteListViewModel:24` solo observa. `ClienteDetailViewModel:97` — `cliente`, `esCandidatoInactivo:109`, `tieneSaldoPendiente`, `aplicarAccionInactividad:126`, `cancelarPlanActual:141` (cancela alertas `NotificationScheduler` + `alCambiarPlan`), `reasignarProximaMedicion:160`.
- `PlanListViewModel:19` — `uiState` + `deshabilitar()` (sin uso). `PlanFormViewModel:36` — `crear()` / `actualizar()`.
- `ConfiguracionMedidaViewModel:15` — `configuraciones`, `configuracionEditando`, `crear()`, `editar()`, `desactivar()`. Sin `eliminar` ni `reactivar`.
- `RegistrarMedidaViewModel:24` — `configuraciones`, `cliente`, `registrar()` + callback `alRegistrarMedidas` (rearma alarmas). `HistorialMedidasViewModel:62` solo lectura de `comparacion`.
- `PagoViewModel:29` — `historialPagos`, `resumenFinanciero`, `plazoDefault`, `planesActivos`, `planActualId`, `cuposPorPlan`; métodos `registrarPago:72`, `modificarPago:87`, `abonarSobrePago:99`, `eliminarPago:110` — único VM con CRUD completo.

---

## 5. Matriz de Capacidad "Todo en Cualquier Momento" — Evidencia por Operación

### 5.1 Clientes

| Operación | UI actual | Código actual | ¿En cualquier momento? | Archivo:línea evidencia |
|---|---|---|---|---|
| Crear | ✅ FAB + formulario | ✅ `ClienteRepository.crearCliente:52` | Sí | `ClientesListScreen.kt:132`, `ClienteFormScreen.kt:128` |
| Ver / Buscar / Filtrar | ✅ | ✅ `observarClientesVisibles:20` | Sí | `ClientesListScreen.kt:69-86` |
| Editar datos personales | ✅ (sin plan) | ✅ `actualizarDatosBasicos:85` | Parcial — no puede cambiar plan ni documento si colisiona | `ClienteFormScreen.kt:134`, `ClienteViewModel.kt:70` |
| Asignar plan | ⚠️ Solo al crear o al registrar pago | `registrarPago:122` + `cancelarPlanActual:184` | No — requiere cancelar plan bloqueante o pagar saldo | `ClienteFormScreen.kt:99`, `PagoRepository.kt:78-87` |
| Eliminar | ❌ Solo vía flujo inactivo "Eliminar Cliente" | ⚠️ Solo soft `oculto=true` + purga a 6 meses | No — condicionado a `!tieneSaldoPendiente && >150d inactivo` | `ClienteDetailScreen.kt:135`, `ClienteRepository.kt:228` |
| Deshabilitar / Reactivar | ⚠️ Solo deshabilitar vía inactivo | ⚠️ `habilitado=false` sin `habilitar=true` | No | `ClienteDetailScreen.kt:143`, `ClienteRepository.kt:239` |
| Modificar activa (habilitado) libre | ❌ No | ❌ No hay `habilitar` | No | — |

**Brecha crítica clientes:** No existe "Eliminar cliente" ni "Cambiar plan" directo. `ClienteFormScreen.kt:128` hardcodea la rama edición sin plan; `ClienteRepository` no expone `eliminarCliente(id)` ni `asignarPlan(clienteId, planId)`. Un admin que necesite reasignar plan a un cliente con deuda debe primero saldar o cancelar, no puede forzar asignación.

### 5.2 Planes

| Operación | UI | Código | ¿En cualquier momento? | Evidencia |
|---|---|---|---|---|
| Crear | ✅ | ✅ | Sí | `PlanFormScreen.kt:119`, `PlanRepository.kt:18` |
| Listar con estado | ✅ | ✅ | Sí | `PlanesListScreen.kt:87`, `PlanDao.kt:14` |
| Editar nombre/precio/duración/beneficios/tipo/max/incluyeMedidas | ✅ | ✅ | Sí | `PlanFormScreen.kt:121`, `PlanRepository.kt:50` |
| Deshabilitar | ❌ (VM existe sin botón) | ✅ `deshabilitarPlan:58` | No en UI | `PlanListViewModel.kt:26` sin uso en `PlanesListScreen.kt` |
| Habilitar (reactivar) | ❌ No | ❌ No existe `habilitarPlan` | No | — |
| Eliminar físico | ❌ No | ❌ No existe `@Delete` | No | `PlanDao.kt` sin `eliminar` |
| Ver clientes asignados | ❌ No | ⚠️ `ClienteDao.contarVisiblesPorPlan:59` existe pero sin UI | No | `006-reporte:237` |

**Brecha planes:** El admin no puede eliminar un plan erróneo ni reactivar uno deshabilitado desde la UI. `PlanDao` no tiene `eliminar`, y `PlanFormScreen` no expone `habilitado` como switch.

### 5.3 Medidas — Configuraciones

| Operación | UI | Código | ¿En cualquier momento? | Evidencia |
|---|---|---|---|---|
| Crear configuración | ✅ | ✅ | Sí | `ConfiguracionMedidasScreen.kt:127`, `MedidaRepository.kt:44` |
| Editar (nombre/abreviatura/unidad/doble/obligatoria) | ✅ Dialog | ✅ | Sí | `ConfiguracionMedidasScreen.kt:140`, `ConfiguracionMedidaViewModel.kt:48` |
| Desactivar (ocultar de futuras tomas) | ✅ Tacho + confirm | ✅ | Sí | `ConfiguracionMedidasScreen.kt:80`, `MedidaRepository.kt:38` |
| Eliminar físico | ❌ No | ❌ No existe | No | `MedidaDao` sin `@Delete` para config |
| Reactivar | ❌ No (desaparece de lista) | ⚠️ Podría vía `actualizarConfiguracion(activa=true)` pero sin UI | No | `MedidaDao.kt:17` filtra `activa=1` |
| Modificar activa libre | ⚠️ Solo desactivar | ⚠️ Solo `activa=false` | Parcial | — |

### 5.4 Medidas — Registros (historial por cliente)

| Operación | UI | Código | ¿En cualquier momento? | Evidencia |
|---|---|---|---|---|
| Registrar toma | ✅ con fecha retroactiva | ✅ `registrarMedida:75` | Sí | `RegistrarMedidaScreen.kt:143`, `MedidaRepository.kt:75` |
| Ver historial comparativo | ✅ `Base/Anterior/Actual` | ✅ `obtenerComparacion:116` | Sí | `HistorialMedidasScreen.kt:42` |
| Editar registro/valor | ❌ No | ❌ Repo no expone `actualizarRegistro/Valor` (DAO sí) | No | `MedidaDao.kt:62` vs `MedidaRepository.kt` |
| Eliminar registro | ❌ No | ❌ No existe | No | — |
| Reprogramar próxima toma | ✅ `Reasignar` Date+Time | ✅ `ClienteDetailViewModel.reasignarProximaMedicion:160` + `NotificationScheduler.reasignarProximaMedicion` | Sí (solo futura) | `ClienteDetailScreen.kt:103`, `ClienteViewModel.kt:160` |

**Brecha registros:** El admin no puede corregir una toma errónea (ej. peso mal digitado). Debe dejar el dato incorrecto o insertar una nueva toma que quedará como "Actual" distorsionando el progreso. La infraestructura `MedidaDao.actualizarRegistro:62 / actualizarValor:65` existe pero no hay Repository/ViewModel/UI que la consuma.

### 5.5 Pagos — Referencia de cumplimiento

| Operación | UI | Código | Evidencia |
|---|---|---|---|
| Registrar pago / abono | ✅ | ✅ | `RegistrarPagoScreen.kt:206`, `PagoRepository.kt:37` |
| Abonar sobre parcial vigente | ✅ Dialogo "Abonar" | ✅ | `RegistrarPagoScreen.kt:281`, `PagoRepository.kt:137` |
| Modificar monto/plazo | ✅ Dialogo "Modificar" | ✅ techo acumulado | `RegistrarPagoScreen.kt:270`, `PagoRepository.kt:187` |
| Eliminar | ✅ Confirm | ✅ `@Delete` | `RegistrarPagoScreen.kt:290`, `PagoRepository.kt:232` |
| Ver historial + saldo periodo | ✅ | ✅ | `RegistrarPagoScreen.kt:219`, `PagoRepository.kt:146` |

**Pagos es el único dominio que cumple "todo en cualquier momento".** Debe usarse como patrón para los demás.

---

## 6. Asignación y Modificación Activa — Detalle Transversal

### 6.1 Asignación plan ↔ cliente

Flujo actual:
1. **Al crear cliente:** `ClienteFormScreen.kt:99` dropdown `planesActivos` → `ClienteRepository.crearCliente(planId:60)`.
2. **Al registrar pago:** `PagoRepository.registrarPago:122` hace `cliente.copy(planId=planId)` si cambió.
3. **Al cancelar:** `ClienteDetailScreen.kt:111` → `ClienteDetailViewModel.cancelarPlanActual:141` → `ClienteRepository.cancelarPlanActual:184` (marca pagos CANCELADO, `planId=null`).

Limitaciones:
- No hay `Asignar plan` directo sin pago. `ClienteFormViewModel.actualizar()` no acepta `planId`.
- Bloqueo por deuda o vigencia reciente: `obtenerPlanActualBloqueante:168` (`saldoPendiente>0 || ahora <= vencimiento+7d`) → error `"Debe cancelarlo antes..."` (`PagoRepository.kt:82`).
- Bloqueo por cupo grupal: `contarVisiblesPorPlan >= maxIntegrantes` (`PagoRepository.kt:92-98`).
- Todo correcto como regla de negocio, pero **contradice "en cualquier momento"** si el admin espera forzar asignación ignorando deuda/cupos. No hay modo `forzarAsignacion` para admin.

### 6.2 Modificación activa (habilitado / oculto / activa)

| Entidad | Campo | Toggle UI | Código | Reversible | Evidencia |
|---|---|---|---|---|---|
| Cliente `habilitado` | `ClienteEntity.kt:36` | ❌ Solo `DESHABILITAR` vía inactivo | `aplicarAccionInactividad:239` (`habilitado=false`) | No | `ClienteDetailScreen.kt:143` |
| Cliente `oculto` | `ClienteEntity.kt:37` | ❌ Solo `ELIMINAR` vía inactivo | `aplicarAccionInactividad:234` (`oculto=true`) | No (purga a 180d) | `ClienteDao.purgarOcultos:78` |
| Plan `habilitado` | `PlanEntity.kt:17` | ❌ Ninguno | `deshabilitarPlan:58` | No | `PlanFormScreen.kt` sin switch |
| Config `activa` | `ConfiguracionMedidaEntity` | ⚠️ Solo `Desactivar` | `desactivarConfiguracion:38` | No en UI | `ConfiguracionMedidasScreen.kt:80` |

Ningún toggle es libre "en cualquier momento" ni reversible desde UI.

---

## 7. Brechas vs Requisito y Riesgos

1. **Clientes no eliminables a demanda.** Solo el flujo de inactividad (>150d `ClienteRepository.UMBRAL_INACTIVIDAD_MS:15`) permite soft-delete y exige `saldoPendiente==0`. Un cliente con datos erróneos o duplicado por `numeroDocumento` no puede borrarse si tiene deuda o es reciente. `ClienteDao` no tiene `@Delete` físico a demanda.
2. **Sin reasignación forzada de plan.** El admin no puede corregir una asignación errónea sin cancelar o saldar. No existe `forzarAsignacionPlan` ni UI de "Cambiar plan" directa.
3. **Planes sin eliminación ni reactivación.** No hay forma de borrar un plan de prueba ni de reactivar uno deshabilitado. `PlanDao` carece de `@Delete` y `habilitar`.
4. **Registros de medida inmutables.** Error de digitación queda permanente. `HistorialMedidasScreen.kt:27` es solo lectura; `MedidaRepository` no expone `actualizar/eliminar` aunque `MedidaDao` sí tiene los `@Update`.
5. **Configuraciones sin reactivación.** Al desactivar, desaparecen de `observarConfiguracionesActivas:17` sin pantalla de "Desactivadas". `AppSettings`/`Sync` no listan inactivas.
6. **Sin gestión de PIN para admin.** `AdminAuthManager.resetearPin:38` existe pero sin UI en `AjustesScreen.kt:15` (solo Notificaciones/Visual/Acciones/Sync/Acerca). Si el admin olvida el PIN, debe borrar `DataStore` (reinstalar) y pierde acceso.

---

## 8. Recomendaciones Priorizadas (para cumplir "todo en cualquier momento")

### P0 — Cierre de brecha funcional directa

- **Clientes: CRUD total.** Añadir `ClienteDao.eliminar(id)` / `ClienteRepository.eliminarCliente(id)` con confirmación y cascada (FK ya es `CASCADE` para pagos/medidas), y exponer `habilitar/ocultar` libres sin condición de 150d. Permitir `actualizar` con `planId` opcional. UI: botón `Eliminar cliente` y `Cambiar plan` en `ClienteDetailScreen.kt:92` (junto a `Cancelar plan actual`).
- **Planes: eliminación y toggle.** Añadir `PlanDao.eliminar` (bloquear si `contarVisiblesPorPlan>0` o forzar con reasignación), `habilitarPlan`, y switch `Habilitado` en `PlanFormScreen.kt:96` + botón `Eliminar` en `PlanesListScreen.kt:148`.
- **Registros de medida: edición/eliminación.** Exponer en `MedidaRepository` `actualizarRegistro/actualizarValores/eliminarRegistro` envolviendo `MedidaDao.actualizarRegistro:62`/`actualizarValor:65` + `@Delete` nuevo, y UI en `HistorialMedidasScreen.kt:44` (icono editar/eliminar por fila, dialog de corrección).
- **Configuraciones: reactivación.** Añadir `MedidaDao.observarTodasConfiguraciones` y `observarInactivas`, y UI de "Medidas desactivadas" con botón `Reactivar` (`activa=true`).

### P1 — Asignación sin fricción para admin

- Añadir `ClienteRepository.asignarPlanForzado(clienteId, planId, forzar: Boolean)` que, si `forzar==true` y el caller es admin (siempre), salte `obtenerPlanActualBloqueante` y `cupo` o pida confirmación explícita. UI: dialog "Forzar asignación (ignorar deuda/cupos)".

### P2 — Operacional

- Añadir `Ajustes > Seguridad > Cambiar PIN / Resetear PIN` consumiendo `AdminAuthManager.configurarPin:24` y `resetearPin:38`.
- Añadir `Ajustes > Clientes ocultos/deshabilitados` listando `SELECT * WHERE oculto=1 OR habilitado=0` con acciones `Restaurar`.

### Patrón a replicar

Tomar `RegistrarPagoScreen.kt:219-304` (historial con `✎/🗑/+` + dialogs + recálculo) como patrón para `HistorialMedidasScreen` y `ClienteDetailScreen`.

---

## 9. Archivos Clave Revisados

**Interfaces:** `AppScaffold.kt:34`, `MaryFitnessNavGraph.kt:60`, `Routes.kt:3`, `LoginScreen.kt:43`, `ClientesListScreen.kt:58`, `ClienteFormScreen.kt:32`, `ClienteDetailScreen.kt:43`, `PlanesListScreen.kt:48`, `PlanFormScreen.kt:38`, `ConfiguracionMedidasScreen.kt:40`, `RegistrarMedidaScreen.kt:50`, `HistorialMedidasScreen.kt:27`, `RegistrarPagoScreen.kt:54`, `AjustesScreen.kt:15`, `DashboardScreen.kt`.

**Código:** `AdminAuthManager.kt:17`, `ClienteEntity.kt:22`, `PlanEntity.kt:8`, `MedidaRegistroEntity.kt:20`, `PagoEntity.kt:27`, `ClienteDao.kt:17`, `PlanDao.kt:12`, `MedidaDao.kt:14`, `PagoDao.kt:12`, `ClienteRepository.kt:41`, `PlanRepository.kt:8`, `MedidaRepository.kt:25`, `PagoRepository.kt:12`, `ClienteViewModel.kt:24`, `PlanViewModel.kt:19`, `ConfiguracionMedidaViewModel.kt:15`, `MedidaViewModel.kt:24`, `PagoViewModel.kt:29`.

**Documentación de referencia:** `documentacion/reportes/006-reporte-comparativo-documentacion-codigo-interfaz.md`, `documentacion/funcionalidades-producto.md`, `documentacion/pantallas/inventario-pantallas.md`.

---

## 10. Veredicto Final

- **Acceso a todo:** ✅ Cumple. El admin autenticado navega a todas las pantallas sin restricción por rol.
- **Modificar todo en cualquier momento:** ❌ **No cumple.** Solo **Pagos** tiene CRUD total (crear, abonar, modificar, eliminar, reasignar plan vía pago, en cualquier momento). **Clientes, Planes, Configuraciones y Registros de medida** están limitados a flujos condicionados, soft-delete sin retorno y registros inmutables.

Para satisfacer el requisito, ejecutar el **P0** arriba. Con esos 4 cambios, la matriz del §1 pasaría de 1/5 a 5/5 dominios con capacidad total tanto en código como en interfaz.

