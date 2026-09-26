# Diálogos de Confirmación — Inventario Visual y Estructura

**Proyecto:** Mary Fitness (Android)  
**Fecha:** 2026-09-21 (actualizado: 2026-09-21)  
**Ubicación de referencia:** `app/src/main/java/com/maryfitness/app/ui/`  
**Tema:** `MaryFitnessTheme` (`ui/theme/Theme.kt:12`, `ui/theme/Color.kt`) — exclusivamente oscuro (dark), sin variante clara.

> Este documento describe **visualmente cómo están estructurados** los cuadros de confirmación tipo `AlertDialog` / `DatePickerDialog` de la app. Es la especificación para la próxima iteración de rediseño visual. Para el inventario de textos, ver `mensajes-nuevas-pantallas.md` y `mejora-mensajes-nuevas-pantallas.md`.

---

## 1. Inventario de diálogos actuales

| # | Disparador (acción del usuario) | Pantalla / archivo | Título del diálogo | Tipo |
|---|----------------------------------|--------------------|--------------------|------|
| 1 | `TextButton "Eliminar"` en tarjeta de plan (lista) | `ui/planes/PlanesListScreen.kt:89-96` → `PlanCard` + diálogo `planAEliminar` (nuevo) | `¿Eliminar este plan?` | Confirmación destructiva con consecuencias + `Checkbox` forzar |
| 2 | `OutlinedButton "Eliminar"` en detalle de plan | `ui/planes/PlanDetailScreen.kt:75-79` → diálogo `confirmarEliminar` | `¿Eliminar este plan?` | Confirmación destructiva con consecuencias + `Checkbox` forzar (dinámico según `clientes.size`) |
| 3 | `OutlinedButton "Eliminar Cliente..."` en detalle de cliente | `ui/clientes/ClienteDetailScreen.kt:209-214` → `confirmarEliminar` (nivel 1) | `¿Cómo deseas eliminar al cliente?` | Elección binaria (Archivar vs Eliminar definitivo) — 2 botones en `confirmButton` Column |
| 4 | Segundo nivel tras elegir "Eliminar Definitivamente" | `ui/clientes/ClienteDetailScreen.kt:281-289` → `confirmarEliminarDefinitivo` | `¿Confirmas la eliminación permanente?` | Confirmación destructiva final |
| 5 | `NeutralOutlineButton "Cambiar o Reasignar Plan"` | `ui/clientes/ClienteDetailScreen.kt:256-265` → `CambiarPlanDialog` | `Reasignar Plan de Cliente` | Formulario embebido (dropdown + `Checkbox` + helper text) |
| 6 | `OutlinedButton "Cancelar plan actual"` | `ui/clientes/ClienteDetailScreen.kt:121-127` → `confirmarCancelarPlan` | `¿Deseas cancelar el plan actual?` | Confirmación con consecuencia (queda sin plan) |
| 7 | Click en tarjeta de medida (toda la GlassCard) | `ui/medidas/ConfiguracionMedidasScreen.kt:60-62` → `configuracionEditando` → `EditarConfiguracionDialog` | `Editar configuración` | Formulario de edición (`MaryFitnessTextField` x3 + 2 `Checkbox`) |
| 8 | Click en icono `🗑` (Delete) en tarjeta de medida | `ui/medidas/ConfiguracionMedidasScreen.kt:81-88` → `configuracionAEliminar` | `Desactivar medida` | Confirmación reversible (no borra, desactiva) |
| 9 | `TextButton "Restaurar"` / `"Reactivar cuenta"` en clientes ocultos | `ui/clientes/ClientesOcultosScreen.kt:68-69,81` → `targetRestaurar` | `¿Restaurar cliente?` | Confirmación positiva (SuccessCyan) |
| 10 | `TextButton "Eliminar definitivamente"` en clientes ocultos | `ui/clientes/ClientesOcultosScreen.kt:94-97` → `targetEliminar` | `¿Eliminar cliente definitivamente?` | Confirmación destructiva irreversible (cascada pagos/medidas) |
| 11 | `TextButton "Editar"` por fila en historial de medidas | `ui/medidas/HistorialMedidasScreen.kt:74-76` → `valorAEditar` | `Editar Valor de Medida` | Edición inline (`OutlinedTextField` con label `Nuevo valor (cm / kg)`) |
| 12 | `TextButton "Eliminar"` por registro en historial | `ui/medidas/HistorialMedidasScreen.kt:94` → `registroAEliminar` | `¿Eliminar este registro de medidas?` | Confirmación destructiva irreversible |
| 13 | `NeutralOutlineButton "Restablecer PIN..."` (zona peligro) | `ui/ajustes/AjustesSeguridadScreen.kt:64` → `confirmarReset` | `¿Restablecer el PIN de acceso?` | Confirmación destructiva de seguridad |
| 14 | `NeutralOutlineButton "Reasignar toma de medidas"` | `ui/clientes/ReasignarMedicionDialog.kt:38` | `DatePickerDialog` → `Hora del recordatorio` (2 pasos) | Flujo en 2 pasos: `DatePickerDialog` + `AlertDialog` con `TimePicker` |

Todos usan `androidx.compose.material3.AlertDialog` (salvo el paso 1 de #14 que es `DatePickerDialog`). No hay `Dialog` custom ni `BottomSheet`.

---

## 2. Anatomía visual común (estructura base)

Todos los diálogos comparten la misma anatomía Material 3, renderizada sobre el tema oscuro `MaryFitnessDarkColorScheme` (`Theme.kt:12-26`).

### 2.1 Overlay / Scrim
- **Scrim:** capa semitransparente negra (`~scrim 0.6`) que cubre toda la pantalla, bloquea interacción por debajo. `onDismissRequest` cierra al tocar fuera (excepto `DatePickerDialog` paso tiempo, que requiere acción explícita).
- **Posición:** centrado vertical y horizontalmente, con `padding` exterior de 32dp aprox. No es full-screen.
- **Elevación:** `AlertDialog` eleva el contenedor sobre el scrim (tonal elevation + shadow). Fondo del contenedor = `colorScheme.surfaceContainerHigh` en dark (`#2A2A2A` aprox, ver `Color.kt:19`), con shape `RoundedCornerShape(28dp)` por defecto de M3 (no `GlassCard` de 8dp).

```
┌─────────────────────────────────────────────┐  ← Scrim (negro 60%, bloquea fondo)
│                                             │
│   ┌───────────────────────────────────┐     │
│   │  ┌─────────────────────────────┐  │     │  ← Contenedor AlertDialog
│   │  │ Título (titleLarge)         │  │     │     - ancho máx ~560dp, mín 280dp
│   │  │                             │  │     │     - padding interno 24dp
│   │  │ Cuerpo (bodyMedium/small)   │  │     │     - esquinas 28dp, elevación M3
│   │  │  • lista consecuencias      │  │     │
│   │  │  • checkbox opcional        │  │     │
│   │  │  • campos si aplica         │  │     │
│   │  │                             │  │     │
│   │  │  [Cancelar]  [Confirmar]    │  │     │  ← Row de botones, alineados End
│   │  └─────────────────────────────┘  │     │
│   └───────────────────────────────────┘     │
│                                             │
└─────────────────────────────────────────────┘
```

### 2.2 Contenedor
- **Componente:** `AlertDialog` de M3, no `GlassCard`. No tiene barra de acento lateral de 4dp ni borde `fullBorderColor` de `GlassCard.kt:43`.
- **Fondo:** `MaryFitnessColors.SurfaceContainerHigh` (`#2A2A2A`) vía `colorScheme.surface`. En diálogos destructivos no cambia el fondo; solo los botones usan color de acento.
- **Forma:** `RoundedCornerShape(28dp)` (default M3). No es 8dp como las tarjetas.
- **Ancho:** fluido, `wrapContent` hasta 560dp máx en phone, centrado. En tablet, max 560dp.
- **Padding interno:** M3 default: título 24dp top/sides, cuerpo 24dp sides, botones 8dp bottom + 8dp sides, separación vertical título-cuerpo 16dp.

### 2.3 Título (`title = { Text(...) }`)
- **Tipografía:** `MaterialTheme.typography.titleLarge` o `titleMedium` según pantalla. Color `MaryFitnessColors.OnSurface` (`#E5E2E1`) por defecto. No usa `MaryFitnessMono`.
- **Contenido:** siempre pregunta directa con `¿...?` (ej: `¿Eliminar este plan?`, `¿Restaurar cliente?`, `Desactivar medida` sin `¿` en caso reversible).
- **Alineación:** start (izquierda), no centrado. Una línea, excepcionalmente dos si incluye nombre de plan/cliente.
- **Ejemplo con dato dinámico:** `¿Estás seguro de que deseas eliminar el plan "Plan Premium"?` — nombre interpolado en `PlanesListScreen.kt:150`.

### 2.4 Cuerpo (`text = { Column/Text/... }`)
Estructura interna en 3 capas, de arriba a abajo:

**a) Pregunta principal (bodyMedium, OnSurface)**
- Ej: `¿Estás seguro de que deseas eliminar el plan "X"?` (`PlanesListScreen.kt:151-155`)
- Color `OnSurface`, weight regular.

**b) Bloque "Consecuencias:" (titleSmall + lista bodySmall)**
- **Encabezado:** `Text("Consecuencias:", style=titleSmall, color=OnSurface)` + `Spacer(4.dp)` — presente en diálogos destructivos nuevos (#1, #2).
- **Lista de viñetas:** cada `Text("• ...", style=bodySmall, color=OnSurfaceVariant (#DDBED2))`. 2–3 ítems:
  - `• El plan se eliminará de forma permanente y no se podrá recuperar.` (siempre)
  - `• Los clientes vinculados quedarán sin plan y deberán ser reasignados manualmente.` (si aplica)
  - `• Si hay clientes inscritos, el borrado será bloqueado a menos que marques 'Forzar eliminación'.` (solo #1, #2)
- **Variante con color de advertencia:** cuando `clientes.isNotEmpty()` en `PlanDetailScreen.kt:178`, la segunda viñeta usa `WarningOrange (#FFAD00)` para enfatizar.
- **Separación:** `Spacer(12.dp)` antes y después del bloque.

**c) Control opcional (Checkbox o campos)**
- **Checkbox "Forzar":** `Row(verticalAlignment=CenterVertically)` con `Checkbox(checked, onCheckedChange)` + `Text("Forzar eliminación (ignorar clientes vinculados)", style=bodySmall)` con `Modifier.padding(start=8.dp)`. Colors: `CheckboxDefaults.colors(checkedColor=NeonPink)` en formularios, default M3 en diálogos simples.
- **Campos de formulario (solo #5, #7, #11):**
  - `#5 CambiarPlanDialog`: `ExposedDropdownMenuBox` con `OutlinedTextField` (readOnly) + `ExposedDropdownMenu` (opciones `Sin Plan Asignado` + lista `plan.nombre - $precio`) + `Checkbox forzar`.
  - `#7 EditarConfiguracionDialog`: 3x `MaryFitnessTextField` (`InputBackground #252525`, label `Nombre completo*` etc, `RoundedCornerShape(4dp)`, borde `SurfaceContainerHighest`) + 2x `Row Checkbox + Text`.
  - `#11 Editar Valor`: solo `OutlinedTextField(label="Nuevo valor (cm / kg)")`.

### 2.5 Botones (`confirmButton` / `dismissButton`)
- **Layout M3:** `Row` interno de `AlertDialog`, alineado `End` (derecha), con `horizontalArrangement = Arrangement.spacedBy(8.dp)` implícito. Orden visual: `[Cancelar]` a la izquierda, `[Confirmar]` a la derecha.
- **Componente:** `TextButton` (sin borde, sin fill). No `NeonPrimaryButton` ni `OutlinedButton` dentro del diálogo (excepto #3 que usa `Column` con dos `TextButton` en `confirmButton` para la elección binaria).
- **Confirmar destructivo:** `Text("Eliminar Plan", color=MaryFitnessColors.ErrorRed (#FF3131))` o `Text("Sí, eliminar todo", color=ErrorRed)`. En confirmaciones positivas: `Text("Sí, restaurar", color=SuccessCyan (#00F5FF))`.
- **Confirmar neutro:** `Text("Guardar")`, `Text("Confirmar Cambio")`, `Text("Desactivar", color=ErrorRed)` (#8 desactivar también es rojo aunque es reversible, porque oculta dato).
- **Cancelar/descartar:** siempre `Text("Cancelar", color=OnSurfaceVariant o Primary)` — sin color de error, alineado a la izquierda del par. Mismo `TextButton` sin tinte.
- **Touch target:** mínimo 48dp alto, padding horizontal 8dp.
- **Comportamiento:** `onClick` cierra diálogo (`planAEliminar = null` / `confirmarEliminar = false`) y luego dispara `viewModel.eliminar(...)` o `viewModel.desactivar(...)` en `Dispatchers.IO`. No hay loading interno en el diálogo.

### 2.6 Tipografía y color — tokens

| Rol | Token | Valor |
|-----|-------|-------|
| Fondo diálogo | `colorScheme.surface` | `#2A2A2A` (`SurfaceContainerHigh`) |
| Título | `OnSurface` | `#E5E2E1`, `titleLarge` (20sp, 600) |
| Cuerpo principal | `OnSurface` | `#E5E2E1`, `bodyMedium` (14sp) |
| Viñetas / helper | `OnSurfaceVariant` | `#DDBED2`, `bodySmall` (12sp) |
| Advertencia | `WarningOrange` | `#FFAD00` |
| Éxito / positivo | `SuccessCyan` | `#00F5FF` (usado en "Restaurar", "Reactivar Variable") |
| Destructivo | `ErrorRed` | `#FF3131` (confirmar eliminar/desactivar) |
| Acento checkbox | `NeonPink` | `#FF10F0` (solo en formularios edición) |
| Fondo inputs | `InputBackground` | `#252525` |

Fuente: `Type.kt` + `Color.kt`. Todo el diálogo hereda `MaryFitnessTheme` (dark).

---

## 3. Variantes visuales

### 3.1 Destructivo con consecuencias + forzar (nuevo patrón planes)
**Archivos:** `PlanesListScreen.kt:130-180`, `PlanDetailScreen.kt:108-156`  
**Uso:** eliminar plan (único con `Checkbox` forzar).

- Título pregunta + cuerpo con intro que menciona nombre del plan.
- Bloque `Consecuencias:` con 2–3 viñetas `•`.
- Si `clientes.isNotEmpty()`, viñeta 2 en `WarningOrange` + viñeta 3 explicando bloqueo.
- `Checkbox` forzar debajo de `Spacer(12.dp)`.
- Botones: `Cancelar` (izq) + `Eliminar Plan` (der, `ErrorRed`).

`PlanDetailScreen` además adapta texto si `clientes.isEmpty()` → `• No hay clientes vinculados, el plan se eliminará inmediatamente.`

### 3.2 Destructivo irreversible 2 niveles (cliente)
**Archivo:** `ClienteDetailScreen.kt:267-289`

- **Nivel 1:** Título `¿Cómo deseas eliminar al cliente?` + texto con 2 viñetas (`• Archivar (Recomendado): ... 6 meses.` / `• Eliminar definitivamente: ...`) + `confirmButton = Column { TextButton("Eliminar Definitivamente", ErrorRed); TextButton("Archivar Cliente") }` + `dismissButton "Cancelar"`. Dos acciones en la misma columna de confirm, no es el patrón estándar M3 (custom).
- **Nivel 2:** Título `¿Confirmas la eliminación permanente?` + texto `Se eliminarán permanentemente el historial de pagos, asistencias y medidas físicas... No se puede deshacer.` + `Confirm "Sí, eliminar todo" ErrorRed` / `Cancelar`.

Visualmente, nivel 1 es más alto (dos botones apilados verticalmente en `confirmButton`), nivel 2 es compacto estándar.

### 3.3 Desactivar (reversible, icono basura)
**Archivo:** `ConfiguracionMedidasScreen.kt:170-189` — disparo `IconButton(Delete, tint=ErrorRed)` en cada `GlassCard` de medida activa.

- Diálogo compacto: Título `Desactivar medida` (sin `¿`), cuerpo `¿Estás seguro de que deseas desactivar esta configuración de medida? No se mostrará en nuevas tomas de medidas.` (dos oraciones), botones `Desactivar` (`ErrorRed`) / `Cancelar`.
- No tiene lista de viñetas ni checkbox. Es el más simple. Color de confirm igualmente `ErrorRed` porque visualmente es acción negativa aunque sea reversible vía "Reactivar Variable" en sección amarilla `WarningOrange`.

### 3.4 Editar configuración (click en tarjeta)
**Archivo:** `ConfiguracionMedidasScreen.kt:160-244` — disparo `GlassCard(modifier=clickable { iniciarEdicion })` (toda la tarjeta, no solo botón).

- `AlertDialog` con `title "Editar configuración"` + `text = Column { 3x MaryFitnessTextField + 2x Row Checkbox }`.
- Inputs con `InputBackground #252525`, borde `SurfaceContainerHighest #353534`, `RoundedCornerShape(4dp)`, label con `*`.
- Botones: `Guardar` (neutral, sin ErrorRed) / `Cancelar`.
- Alto mayor que destructivos (ocupa ~60% de pantalla por 3 fields + 2 checkboxes).

### 3.5 Restaurar cliente (positivo)
**Archivo:** `ClientesOcultosScreen.kt:88-97`

- Dos diálogos hermanos, mismo contenedor pero distinto tono:
  - **Restaurar:** Título `¿Restaurar cliente?` + texto `El cliente volverá a estar activo en la lista principal y podrá registrar asistencias.` + `Confirm "Sí, restaurar"` (sin ErrorRed, default `Primary` o `SuccessCyan` según implementación) / `Cancelar`.
  - **Eliminar definitivo:** Título `¿Eliminar cliente definitivamente?` + texto `¡Atención! Esta acción no se puede deshacer. Se borrarán permanentemente sus pagos, historial de medidas y asistencias.` + `Confirm "Sí, eliminar todo" ErrorRed` / `Cancelar`.
- Visualmente idénticos en layout, solo cambia texto y color del confirm.

### 3.6 Editar / Eliminar medida puntual (historial)
**Archivo:** `HistorialMedidasScreen.kt:104-126`

- **Editar valor:** Título `Editar Valor de Medida` + `OutlinedTextField(label="Nuevo valor (cm / kg)")` + `Confirm "Guardar Cambio"` / `Cancelar`.
- **Eliminar registro:** Título `¿Eliminar este registro de medidas?` + texto `Esta toma de medidas se eliminará del historial del cliente. Esta acción no se puede deshacer.` + `Confirm "Eliminar Registro" ErrorRed` / `Cancelar`.

### 3.7 Seguridad PIN
**Archivo:** `AjustesSeguridadScreen.kt:71-79` — zona peligro `GlassCard(fullBorderColor=ErrorRed)` con `NeutralOutlineButton "Restablecer PIN..."`.

- Diálogo: Título `¿Restablecer el PIN de acceso?` + texto `Se eliminará la clave actual. La app te pedirá crear un nuevo PIN la próxima vez que la abras.` + `Confirm "Sí, restablecer" ErrorRed` / `Cancelar`.

### 3.8 Reasignar medición (2 pasos, nativo)
**Archivo:** `ReasignarMedicionDialog.kt:38-106`

- **Paso 1:** `DatePickerDialog` nativo M3 (calendario, header con año/mes, `isSelectableDate >= hoyInicioUtc`), botones `Siguiente` / `Cancelar`.
- **Paso 2:** `AlertDialog` título `Hora del recordatorio` + cuerpo `TimePicker(is24Hour=true, initial 09:00)` + `error Text` opcional (`ErrorRed bodySmall`) + `Confirm "Reasignar"` / `Cancelar`.
- Es el único flujo con picker nativo embebido, no lista de consecuencias.

---

## 4. Especificación para rediseño

Para modificar visualmente estos componentes, intervenir **solo** los siguientes puntos (mantener lógica `onDismissRequest` / `viewModel` intacta):

1. **Fondo y forma del contenedor:** `AlertDialog` usa `containerColor` y `shape`. Cambiar `RoundedCornerShape(28dp)` → ej: `16dp` o `8dp` para alinear con `GlassCard(8dp)` si se desea consistencia "gym-tech". Cambiar `containerColor` de `SurfaceContainerHigh` a `SurfaceDark (#1E1E1E)` con `BorderStroke(1.dp, OutlineVariant)`.
2. **Título:** `Text` dentro de `title`. Cambiar `style` (`titleLarge` → `titleMedium`, `headlineSmall`), `color` o añadir `Icon` leading (ej: `Icons.Filled.Warning` en `WarningOrange` para destructivos).
3. **Cuerpo — bloque consecuencias:** es `Column` de `Text`. Se puede reemplazar viñetas `Text("• ...")` por `Row(Icon(Check/Warning) + Text)` o `BulletList` custom. Mantener `style=bodySmall` para no romper jerarquía.
4. **Checkbox forzar:** `Checkbox` + `Text`. Cambiar `colors = CheckboxDefaults.colors(checkedColor=NeonPink → Primary)` o reemplazar por `Switch`.
5. **Botones:** `TextButton` → `OutlinedButton` / `NeonPrimaryButton` si se quiere más peso. El destructivo debe mantener `contentColor=ErrorRed`; el positivo `SuccessCyan`. Añadir `Modifier.weight(1f)` para botones full-width en mobile si se desea (como en `ClienteDetailScreen` nivel 1 que ya usa `Column`).
6. **Scrim:** no expuesto en `AlertDialog`; para cambiar opacidad, wrappear en `DialogProperties(scrimColor=...)` o custom `Dialog`.
7. **Inputs (solo #5, #7, #11):** son `MaryFitnessTextField` / `OutlinedTextField` con `InputBackground`. Cambiar `shape`, `colors`, `label` sin tocar validación.

**No modificar:** `onDismissRequest`, `confirmButton onClick` (debe seguir llamando `vm.eliminar(...)` / `vm.desactivar(...)` y cerrar con `planAEliminar = null`), ni el `viewModel`/`repository` subyacente.

**Archivos a editar para cambio global:** crear un composable wrapper `MaryFitnessConfirmDialog(@Composable content)` que centralice `AlertDialog` + tokens, y reemplazar cada `AlertDialog(...)` por él. Así el rediseño afecta a los 14 diálogos a la vez. Ubicación sugerida: `ui/common/MaryFitnessDialogs.kt`.

---

## 5. Referencias cruzadas

- Inventario de mensajes (textos): `documentacion/pantallas/mensajes-nuevas-pantallas.md` (§1.2–2.5, §3)
- Propuesta de mejora de textos: `documentacion/pantallas/mejora-mensajes-nuevas-pantallas.md` (§1.2–2.5)
- Colores: `app/src/main/java/com/maryfitness/app/ui/theme/Color.kt`
- Tema: `app/src/main/java/com/maryfitness/app/ui/theme/Theme.kt`
- Tarjeta base: `app/src/main/java/com/maryfitness/app/ui/common/GlassCard.kt`
- Botones: `app/src/main/java/com/maryfitness/app/ui/common/MaryFitnessButtons.kt`

---

*Documento generado para la tarea: "Modificar eliminación directa de planes y documentar diálogos de confirmación". Última edición de código: `PlanesListScreen.kt` y `PlanDetailScreen.kt` (diálogos con lista de consecuencias y confirmación explícita).*
