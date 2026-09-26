### Inventario de Mensajes Visibles - Nuevas Pantallas y Modificaciones
**Fecha:** 2026-09-21 **Proyecto:** Mary Fitness (Android) **Total de mensajes únicos nuevos:** ~85
**Alcance:** Solo pantallas agregadas en la iteración actual (git diff `HEAD`) y modificaciones con textos nuevos visibles al usuario. No repite los ~220 mensajes ya inventariados en `mensajes-cliente-completo.md`.

Referencia de código: `AjustesSeguridadScreen.kt`, `ClientesOcultosScreen.kt`, `PlanDetailScreen.kt`, `SplashScreen.kt`, `ClienteDetailScreen.kt` (Gobernanza), `PlanesListScreen.kt`, `PlanFormScreen.kt`, `HistorialMedidasScreen.kt`, `ConfiguracionMedidasScreen.kt`, `EjerciciosScreen.kt`, `AjustesScreen.kt`, `MaryFitnessNavGraph.kt`, `Routes.kt`.

---

#### 1. PANTALLAS COMPLETAMENTE NUEVAS

##### 1.1 Splash — Carga inicial
Ubicación: `app/src/main/java/com/maryfitness/app/ui/splash/SplashScreen.kt`
| Mensaje | Contexto | Tipo |
| ------- | -------- | ---- |
| MARY FITNESS | Marca / logo textual en splash | Título de marca |
| Gestión Integral de Gimnasio | Subtítulo descriptivo | Subtítulo |
| Verificando estado de PIN y base local... | Estado mientras `checking == true` (delay 900ms + `checkPin()`) | Estado de carga |
| Iniciando... | Estado cuando `checking == false` antes de `onFinished` | Estado de carga |
| Optimizado para Moto G22 · Helio G37 · Offline-First | Footer técnico | Texto informativo |
| Mary Fitness v0.1.5 · Seguridad local BCrypt | Versión y mecanismo de seguridad | Pie de versión |
| (CircularProgressIndicator sin texto) | Indicador visual | - |

Ruta: No tiene ruta propia, es pantalla raíz efímera que redirige a Login/Onboarding según `tienePin`.

##### 1.2 Seguridad — PIN Administrador (Ajustes > Seguridad)
Ubicación: `app/src/main/java/com/maryfitness/app/ui/ajustes/AjustesSeguridadScreen.kt` — Ruta `ajustes/seguridad` — Título TopBar `Seguridad`
| Mensaje | Contexto / Ubicación en pantalla |
| ------- | -------------------------------- |
| Seguridad - PIN Administrador | Título principal de la sección |
| El PIN protege el acceso a todos los datos. No hay recuperación remota (reporte 007 P2). | Subtítulo explicativo bajo el título |
| Cambiar PIN | Título de GlassCard superior |
| PIN actual (4-6 dígitos) | Label de `MaryFitnessTextField` para `pinActual` |
| PIN nuevo | Label para `pinNuevo` |
| Confirmar nuevo PIN | Label para `confirmarNuevo` |
| Cambiar PIN | Texto de `NeonPrimaryButton` |
| Resetear PIN (emergencia) | Título de GlassCard inferior (zona de peligro, borde `ErrorRed`) |
| Borra el PIN actual y obliga a configurar uno nuevo en el próximo inicio. Usa solo si olvidaste el PIN. | Descripción de zona de reset |
| Resetear PIN | Texto de `NeutralOutlineButton` que abre confirmación |
| El nuevo PIN no coincide con la confirmación. | Mensaje inline tras validar `pinNuevo != confirmarNuevo` |
| PIN actual incorrecto. | Mensaje inline tras `!authManager.validarPin(pinActual)` |
| PIN cambiado con éxito. | Mensaje de éxito (color `SuccessCyan`) tras `configurarPin` ok |
| PIN reseteado. Reinicia la app para crear uno nuevo. | Mensaje tras `resetearPin()` |
| {it.message} | Mensaje de error genérico de `onFailure` de `configurarPin` |
| ¿Resetear PIN? | Título de `AlertDialog` de confirmación |
| Se eliminará el PIN actual. Tendrás que crear uno nuevo al reiniciar la app. | Cuerpo del diálogo |
| Sí, resetear | Botón confirmar (color `ErrorRed`) |
| Cancelar | Botón descartar |

Flujo: Validación síncrona en corrutina `scope.launch` antes de llamar `AdminAuthManager`. Los mensajes de error/éxito se muestran con color condicional `it.contains("éxito")`.

##### 1.3 Clientes ocultos / deshabilitados (Ajustes > Clientes ocultos)
Ubicación: `app/src/main/java/com/maryfitness/app/ui/clientes/ClientesOcultosScreen.kt` — Ruta `clientes/ocultos` — Título TopBar `Clientes Ocultos`
| Mensaje | Contexto |
| ------- | -------- |
| Clientes ocultos / deshabilitados | Título de pantalla |
| Vista administrativa (reporte 007 P2): restaura o purga definitiva. | Subtítulo explicativo |
| No hay clientes ocultos ni deshabilitados. | Empty state cuando ambas listas vacías |
| Ocultos (${N}) - purga automática en 6 meses | Header sección ocultos |
| Deshabilitados (${N}) | Header sección deshabilitados |
| {nombreCompleto} | Nombre del cliente en card |
| {numeroDocumento} | Documento en card |
| Oculto | Chip/label estado oculto (`ErrorRed`) |
| Deshabilitado | Chip/label estado deshabilitado (`WarningOrange`) |
| Restaurar | Botón para ocultos (`SuccessCyan`) |
| Eliminar definitivo | Botón para ocultos (`ErrorRed`) |
| Rehabilitar | Botón para deshabilitados |
| Restaurar cliente | Título de diálogo restaurar |
| Se restaurará y habilitará al cliente. | Cuerpo diálogo restaurar |
| Restaurar | Botón confirmar restaurar |
| Cancelar | Botón descartar |
| Eliminar definitivo | Título de diálogo eliminación definitiva |
| Borrado irreversible (pagos/medidas en cascada). | Cuerpo diálogo eliminación |
| Eliminar | Botón confirmar eliminación (`ErrorRed`) |

ViewModel `ClientesOcultosViewModel` observa `observarClientesOcultos()` y `observarClientesDeshabilitados()` (flows de `ClienteRepository`/`ClienteDao`).

##### 1.4 Detalle de Plan
Ubicación: `app/src/main/java/com/maryfitness/app/ui/planes/PlanDetailScreen.kt` — Ruta `planes/{planId}/detalle` — Título TopBar `Detalle de Plan`
| Mensaje | Contexto |
| ------- | -------- |
| Cargando plan... | Estado cuando `plan == null` |
| {nombre} | Nombre del plan (titleLarge) |
| $${precio} / {duracionDias}d | Precio y duración (titleMedium) |
| Sin beneficios descritos | Fallback cuando `beneficios.isBlank()` |
| ACTIVO | Chip cuando `habilitado == true` (`SuccessCyan`) |
| DESHABILITADO | Chip cuando `habilitado == false` (`Outline`) |
| Tipo: {tipo} (máx {N}) | Info de tipo y cupo |
| Seguimiento medidas: Sí | Cuando `incluyeSeguimientoMedidas == true` |
| Seguimiento medidas: No | Cuando `false` |
| Habilitado | Label del `Switch` |
| Editar | Botón `OutlinedButton` -> navega a `planEditar` |
| Eliminar | Botón `OutlinedButton` (`ErrorRed`) abre diálogo |
| Clientes asociados ({N}) | Header sección clientes |
| Ningún cliente vinculado a este plan. | Empty state clientes |
| {nombreCompleto} / {numeroDocumento} | Fila cliente + `StatusChip` con `estadoCuenta.name` |
| {mensaje} | Texto de `mensaje` del `PlanDetailViewModel` (SuccessCyan/Primary) |
| ¿Eliminar plan? | Título diálogo |
| Si hay clientes vinculados, el borrado será bloqueado. Marca 'Forzar' para eliminar de todas formas (admin override). | Cuerpo diálogo + explicación |
| Forzar eliminación | Label del `Checkbox` |
| Eliminar | Botón confirmar (`ErrorRed`) |
| Cancelar | Botón descartar |

ViewModel: `PlanDetailViewModel` (`plan`, `clientesAsociados` por `observarClientesPorPlan`). Mensajes de toggle: `Estado actualizado.` / errores de `PlanRepository.toggleHabilitado` y `eliminarPlan`/`eliminarPlanForzado`.

---

#### 2. MODIFICACIONES CON MENSAJES NUEVOS EN PANTALLAS EXISTENTES

##### 2.1 Detalle de cliente — Gobernanza de Cuenta (reporte 007 P0)
Ubicación: `app/src/main/java/com/maryfitness/app/ui/clientes/ClienteDetailScreen.kt` — Sección nueva `GlassCard` con `fullBorderColor = WarningOrange`
| Mensaje | Contexto | Tipo |
| ------- | -------- | ---- |
| Gobernanza de Cuenta | Header de sección administrativa | Título de sección |
| Reporte 007 | Badge lateral | Referencia de spec |
| Estado del Perfil | Label principal del toggle | Etiqueta |
| Cliente Activa (con acceso) | Subtítulo cuando `cliente.habilitado == true` | Estado |
| Cliente Inactiva (bloqueada) | Subtítulo cuando `false` | Estado |
| Cambiar / Reasignar Plan (Admin override) | Botón `NeutralOutlineButton` abre `CambiarPlanDialog` | Botón |
| Restaurar cliente | Botón visible solo si `oculto || !habilitado` (borde `SuccessCyan`) | Botón |
| Eliminar Cliente del Sistema | Botón danger zone (borde `ErrorRed`) abre diálogo eliminación | Botón |
| {accionResultado} | Mensaje inline de `ClienteDetailViewModel` (12dp bajo la sección) | Feedback |
| Cliente habilitado. Acceso restablecido. | Mensaje de éxito tras `toggleHabilitado(true)` | Éxito |
| Cliente deshabilitado. | Tras `toggleHabilitado(false)` | Éxito |
| Cliente eliminado del sistema. | Tras `eliminarClienteDefinitivo()` | Éxito |
| Cliente ocultado (soft delete). Se purgará en 6 meses. | Tras `softDeleteCliente()` | Éxito |
| Cliente restaurado y habilitado. | Tras `restaurarCliente()` | Éxito |
| Plan asignado con éxito. | Tras `asignarPlan` normal | Éxito |
| Plan reasignado en modo administrador (override). | Tras `asignarPlanForzado` | Éxito |

Diálogos agregados en la misma pantalla:

**a) CambiarPlanDialog (`@Composable private fun CambiarPlanDialog`)**
| Campo | Mensaje |
| ----- | ------- |
| Título | Cambiar / Reasignar plan |
| Cuerpo | Selecciona el plan. Con 'Forzar' ignoras bloqueo por deuda/vigencia y cupo grupal. |
| Label dropdown | Plan |
| Opción | Sin plan |
| Opción dinámica | {nombre} - $${precio} (por cada plan de `planRepo.observarPlanes()`) |
| Checkbox | Forzar asignación (admin override) |
| Confirmar | Asignar |
| Cancelar | Cancelar |

**b) Confirmación Eliminar (dos niveles)**
| Campo | Mensaje |
| ----- | ------- |
| Título 1 | ¿Eliminar cliente? |
| Cuerpo 1 | Opción 1: ocultar (soft delete, se conserva en backup y purga en 6 meses). Opción 2: eliminar definitivo e irreversible (borra pagos/medidas en cascada). |
| Botón 1 | Eliminar definitivo (`ErrorRed`) |
| Botón 2 | Ocultar (soft delete) |
| Descartar 1 | Cancelar |
| Título 2 | Confirmar eliminación definitiva |
| Cuerpo 2 | Esta acción borra todos los datos del cliente de forma irreversible. |
| Confirmar 2 | Sí, eliminar (`ErrorRed`) |
| Descartar 2 | Cancelar |

##### 2.2 Lista de Planes — Acciones inline y toggle
Ubicación: `app/src/main/java/com/maryfitness/app/ui/planes/PlanesListScreen.kt` — `PlanCard` modificado
| Mensaje | Contexto |
| ------- | -------- |
| Habilitado | Label junto al `Switch` en cada card |
| Detalle | Botón `NeutralOutlineButton` -> `onDetalleClick` (nuevo, navega a `planDetalle`) |
| Editar | Botón existente, ahora junto a Detalle |
| Eliminar | `TextButton` al pie de la card (`ErrorRed`) |
| {mensaje} | Mensaje del `PlanListViewModel` debajo del listado (`Primary`) |
| Plan habilitado. | Tras `toggleHabilitado(true)` |
| Plan deshabilitado. | Tras `false` |
| Plan eliminado. | Tras `eliminar` success |
| {it.message} | Error de `toggleHabilitado`/`eliminar` (ej: `Plan no encontrado.`, `No se puede eliminar: N cliente(s) aún vinculados...`) |

##### 2.3 Formulario de Plan — Switch habilitado
Ubicación: `app/src/main/java/com/maryfitness/app/ui/planes/PlanFormScreen.kt`
| Mensaje | Contexto |
| ------- | -------- |
| Plan habilitado | Label del `Switch` agregado (junto al checkbox de seguimiento) |

El valor `habilitado` se persiste al crear/actualizar (`habilitado = habilitado`). No cambia etiquetas existentes pero agrega un estado visible nuevo.

##### 2.4 Historial y comparación de medidas — Corrección y eliminación
Ubicación: `app/src/main/java/com/maryfitness/app/ui/medidas/HistorialMedidasScreen.kt` (`HistorialMedidasViewModel`)
| Mensaje | Contexto |
| ------- | -------- |
| Medida / Base / Anterior / Actual + ✎ | Header de tabla (nuevo icono de acción en header) |
| ✎ | Botón `TextButton` por fila (NeonPink) para corregir valor |
| Registros ({N}) - eliminar si hay error de digitación | Header nueva sección de listado de registros |
| Base - dd/MM/yyyy | Etiqueta para registro `esBase == true` (formateado con `SimpleDateFormat`) |
| Registro dd/MM/yyyy HH:mm | Etiqueta para registro normal |
| id:{id} | Subtexto con ID del registro |
| 🗑 | Botón eliminar registro (`ErrorRed`) |
| {mensaje} | Mensaje del ViewModel (`Primary`, bodySmall) |
| Registro eliminado. | Tras `eliminarRegistro` success + recarga de comparación |
| Valor corregido. | Tras `actualizarValorMedida` success |
| Corregir valor | Título diálogo edición |
| Nuevo valor | Label de `OutlinedTextField` |
| Guardar | Confirmar diálogo corregir |
| Cancelar | Descartar |
| ¿Eliminar registro? | Título diálogo eliminar |
| Se eliminará el registro y sus valores asociados de forma irreversible. | Cuerpo |
| Eliminar | Confirmar (`ErrorRed`) |

Nota: La sección de comparación ahora se renderiza solo si `comparacion != null`, pero la lista de registros se muestra siempre si `registros.isNotEmpty()`.

Validaciones / errores propagados (mensajes técnicos del repository):
| Mensaje | Origen |
| ------- | ------ |
| Registro no encontrado. | `MedidaRepository.eliminarRegistro` / `actualizarRegistroTimestamp` |
| Valor no encontrado. | `MedidaRepository.actualizarValorMedida` |

##### 2.5 Configuración de variables de medida — Reactivación
Ubicación: `app/src/main/java/com/maryfitness/app/ui/medidas/ConfiguracionMedidasScreen.kt` + `ConfiguracionMedidaViewModel.kt`
| Mensaje | Contexto |
| ------- | -------- |
| Medidas desactivadas (puedes reactivar) | Título de sección amarilla (`WarningOrange`) visible solo si `configuracionesInactivas.isNotEmpty()` |
| {nombreCompleto} ({nombreAbreviado}) | Nombre en card de inactivas |
| Desactivada | Subtexto de estado |
| Reactivar | Botón `TextButton` (`SuccessCyan`) llama `viewModel.reactivar(id)` |

ViewModel expone dos nuevos flows: `configuracionesTodas` y `configuracionesInactivas`.

##### 2.6 Biblioteca de ejercicios — Filtro local por bodyPart
Ubicación: `app/src/main/java/com/maryfitness/app/ui/ejercicios/EjerciciosScreen.kt`
| Mensaje | Contexto |
| ------- | -------- |
| Todos | `FilterChip` para limpiar filtro (`filtroBodyPart == null`) |
| {bodyPart} | Chips dinámicos por cada `bodyPart` distinto en `state.data.flatMap { bodyParts }.distinct().sorted()` |
| Sin resultados para el filtro. | Empty state cuando `filtrados.isEmpty()` |

Comportamiento: Filtro es local (no llama a API), se aplica sobre `current.data`.

##### 2.7 Ajustes (menú) — Nuevos ítems
Ubicación: `app/src/main/java/com/maryfitness/app/ui/ajustes/AjustesScreen.kt` — `ITEMS` ampliado
| Mensaje | Destino | Icono |
| ------- | ------- | ----- |
| Seguridad (PIN) | `Routes.SEGURIDAD` (`ajustes/seguridad`) | `Icons.Filled.Lock` |
| Clientes ocultos / deshabilitados | `Routes.CLIENTES_OCULTOS` (`clientes/ocultos`) | `Icons.Filled.VisibilityOff` |

Navegación agregada en `MaryFitnessNavGraph.kt`:
| Pantalla | TopBar title |
| -------- | ------------ |
| `AjustesSeguridadScreen` | Seguridad |
| `ClientesOcultosScreen` | Clientes Ocultos |
| `PlanDetailScreen` | Detalle de Plan |

##### 2.8 Navegación — Nuevas rutas
Ubicación: `app/src/main/java/com/maryfitness/app/ui/navigation/Routes.kt`
| Constante | Valor |
| --------- | ----- |
| `PLAN_DETALLE` | `planes/{planId}/detalle` |
| `CLIENTES_OCULTOS` | `clientes/ocultos` |
| `SEGURIDAD` | `ajustes/seguridad` |
| `planDetalle(planId)` | Helper `planes/{id}/detalle` |

---

#### 3. MENSAJES DE ERROR / VALIDACIÓN NUEVOS (REPOSITORIOS)

| Mensaje | Ubicación | Cuándo se muestra |
| ------- | --------- | ----------------- |
| Cliente no encontrado. | `ClienteRepository.eliminarClienteDefinitivo`, `softDeleteCliente`, `restaurarCliente`, `setHabilitado`, `asignarPlan` | `clienteDao.obtenerCliente(id) == null` |
| Configuración no encontrada. | `MedidaRepository.reactivarConfiguracion` / `desactivarConfiguracion` | `observarConfiguracionPorId == null` |
| Plan no encontrado. | `PlanRepository.toggleHabilitado`, `eliminarPlan`, `eliminarPlanForzado` | `planDao.obtenerPlan == null` |
| No se puede eliminar: {N} cliente(s) aún vinculados a este plan. Reasigna o cancela primero. | `PlanRepository.eliminarPlan` (con `clienteDao`) | Bloqueo si `contarVisiblesPorPlan > 0` (usado en `PlanDetailScreen` con conteo local) |
| No se puede eliminar: {N} cliente(s) aún vinculados. | `PlanDetailViewModel.eliminar` | Validación en UI antes de llamar repo (mensaje local) |

Todos usan `Result.failure(IllegalStateException(...))` y se propagan a `ViewModel.mensaje` / `accionResultado` y se renderizan como `Text(it.message, color=Primary/ErrorRed)`.

---

#### 4. RESUMEN POR CATEGORÍA (SOLO LO NUEVO)

| Categoría | Cantidad | Ejemplos nuevos |
| --------- | -------- | --------------- |
| Navegación y etiquetas | ~8 | Detalle de Plan, Seguridad, Clientes Ocultos, MARY FITNESS |
| Autenticación y seguridad | ~10 | PIN actual/nuevo, Resetear PIN, PIN cambiado con éxito |
| Estados de pantalla (empty/loading) | ~6 | Cargando plan..., No hay clientes ocultos..., Sin resultados para el filtro |
| Mensajes de éxito | ~12 | Cliente restaurado y habilitado., Plan habilitado., Valor corregido. |
| Mensajes de error / validación | ~5 | Cliente no encontrado., No se puede eliminar: N vinculados |
| Diálogos de confirmación | ~20 | ¿Eliminar plan?, Forzar eliminación, ¿Resetear PIN? |
| Etiquetas de campo y filtros | ~12 | PIN nuevo, Plan, Todos, Plan habilitado, Nuevo valor |
| Acciones de botón | ~15 | Restaurar, Eliminar definitivo, Rehabilitar, Reactivar, Asignar |

**Nota de trazabilidad:** Estos mensajes complementan los ~220 de `mensajes-cliente-completo.md`. La fuente de verdad son los archivos `.kt` listados arriba; cualquier cambio en los literales de `Text(...)`, `label =`, `title =`, o `Result.failure(...)` debe reflejarse aquí.
