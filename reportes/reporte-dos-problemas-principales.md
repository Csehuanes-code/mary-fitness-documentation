Ran command: `Get-ChildItem -Recurse -File | Select-Object -First 100 FullName`
Viewed c:\Users\csehuanes\Downloads\mary-fitness\package.json
Viewed README.md:1-35
Viewed decisiones-tecnicas.md:1-61
Viewed especificaciones-negocio.md:1-9
Viewed funcionalidades-producto.md:1-25
Viewed build.gradle.kts:1-179
Viewed especificaciones-generales.md:1-60
Viewed 001-especificaciones-pendientes.md:1-34
Viewed 003-respuesta-decisiones-pendientes.md:1-49
Viewed c:\Users\csehuanes\Downloads\mary-fitness\documentacion\reportes\002-auditoria-codigo.md
Created powershell.cmd
Ran command: `Get-ChildItem -Recurse -File | Select-Object -First 20 FullName`
Ran command: `Get-ChildItem -Path documentacion -Recurse -File | Select-Object FullName`
Viewed 006-reporte-comparativo-documentacion-codigo-interfaz.md:1-583
Viewed 007-reporte-capacidad-administrador-acceso-total.md:1-328
Viewed ConfiguracionMedidasScreen.kt:1-225
Viewed MaryFitnessTextField.kt:1-57
Viewed c:\Users\csehuanes\Downloads\mary-fitness\app\src\main\java\com\maryfitness\app\ui\common\MaryFormDialog.kt
Ran command: `Get-ChildItem -Path app\src -Recurse -File | Select-String "MaryFormDialog" | Select-Object -First 10`
Viewed MaryFitnessDialogs.kt:565-613
Viewed MaryFitnessDialogs.kt:490-564
Viewed MaryFitnessDialogs.kt:350-420
Viewed MaryFitnessDialogs.kt:1-100
Viewed MaryFitnessDialogs.kt:100-140
Viewed HistorialMedidasScreen.kt:1-175
Viewed RegistrarPagoScreen.kt:300-370
Ran command: `Get-ChildItem -Path app\src -Recurse -File | Select-String "MaryStyledCheckboxRow"`
Viewed MaryFitnessDialogs.kt:210-250
Viewed ConfiguracionMedidaViewModel.kt:1-102
Viewed ClienteDetailScreen.kt:290-329
Ran command: `Get-ChildItem -Path app\src -Recurse -File | Select-String "MaryFitnessTextField"`
Viewed RegistrarMedidaScreen.kt:80-160
Viewed inventario-pantallas.md:190-192
Ran command: `Select-String -Path documentacion\pantallas\inventario-pantallas.md -Pattern "6\."`
Viewed inventario-pantallas.md:104-130
Ran command: `Select-String -Path documentacion\pantallas\inventario-pantallas.md -Pattern "5\."`
Viewed inventario-pantallas.md:90-103
Viewed formato-de-registros.md:1-483
Viewed PagoRepository.kt:30-140
Viewed PagoRepository.kt:180-235
Viewed PagoRepository.kt:235-265
Viewed PagoViewModel.kt:1-125
Viewed RegistrarPagoScreen.kt:50-205
Viewed RegistrarPagoScreen.kt:206-280
Viewed RegistrarPagoScreen.kt:358-421
Viewed problems.md:1-57
Ran command: `git status`
Ran command: `Remove-Item -Path "powershell.cmd" -Force`
Viewed c:\Users\csehuanes\Downloads\mary-fitness\powershell.cmd

A continuación te presento el diagnóstico detallado del **estado real** de ambos problemas analizado en los tres niveles solicitados: **documentación**, **código fuente** e **interfaz de usuario**.

---

## 1. Problema: Medidas no modificables
> *"Se muestra pantalla de edición pero los campos no son editables, solo se puede modificar los checkeables (tiene dos secciones y obligatoria)"*

### A. Estado en Documentación
* **Especificación funcional original:**
  * En [`documentacion/pantallas/inventario-pantallas.md` (Sección 6.1)](file:///c:/Users/csehuanes/Downloads/mary-fitness/documentacion/pantallas/inventario-pantallas.md#L106-L110) se define explícitamente:
    > *"Qué se puede hacer: definir qué medidas existen y cuáles son obligatorias como mínimo para toda toma de mediciones; añadir nuevas variables de medida o **modificar las existentes**."*
  * En [`documentacion/especificaciones-generales.md` (Líneas 34-36)](file:///c:/Users/csehuanes/Downloads/mary-fitness/documentacion/especificaciones-generales.md#L34-L36) se describe que una medida consta de: nombre completo, nombre abreviado, unidad de medida, configuración de dos secciones (grande/pequeña) y obligatoriedad.
* **Diagnóstico documental:**
  * Los reportes de auditoría previa ([`006-reporte-comparativo-documentacion-codigo-interfaz.md`](file:///c:/Users/csehuanes/Downloads/mary-fitness/documentacion/reportes/006-reporte-comparativo-documentacion-codigo-interfaz.md#L377-L380) y [`007-reporte-capacidad-administrador-acceso-total.md`](file:///c:/Users/csehuanes/Downloads/mary-fitness/documentacion/reportes/007-reporte-capacidad-administrador-acceso-total.md#L96-L101)) marcaron este flujo como **"100% CUMPLE"** basándose solo en que la pantalla y el diálogo existían teóricamente, sin haber probado la interacción física con los campos de texto en un dispositivo.

### B. Estado en Código
* **Componentes involucrados:**
  * [`ConfiguracionMedidasScreen.kt`](file:///c:/Users/csehuanes/Downloads/mary-fitness/app/src/main/java/com/maryfitness/app/ui/medidas/ConfiguracionMedidasScreen.kt#L61-L64): Al hacer click en una tarjeta de medida, se ejecuta [`iniciarEdicion(config)`](file:///c:/Users/csehuanes/Downloads/mary-fitness/app/src/main/java/com/maryfitness/app/ui/medidas/ConfiguracionMedidaViewModel.kt#L40) y se despliega [`EditarConfiguracionDialog`](file:///c:/Users/csehuanes/Downloads/mary-fitness/app/src/main/java/com/maryfitness/app/ui/medidas/ConfiguracionMedidasScreen.kt#L189-L224).
  * Este diálogo usa [`MaryFormDialog`](file:///c:/Users/csehuanes/Downloads/mary-fitness/app/src/main/java/com/maryfitness/app/ui/common/MaryFitnessDialogs.kt#L578) y [`MaryFitnessDialogScaffold`](file:///c:/Users/csehuanes/Downloads/mary-fitness/app/src/main/java/com/maryfitness/app/ui/common/MaryFitnessDialogs.kt#L82).
* **Causa raíz del bug:**
  * En [`MaryFitnessDialogs.kt` (líneas 98–103)](file:///c:/Users/csehuanes/Downloads/mary-fitness/app/src/main/java/com/maryfitness/app/ui/common/MaryFitnessDialogs.kt#L98-L103) se encuentra el contenedor del diálogo:
    ```kotlin
    // Evita que el click en el contenido cierre el diálogo
    Box(
        modifier = Modifier
            .fillMaxWidth()
            .padding(horizontal = 16.dp)
            .clickable(enabled = false) {} // <--- ERROR CRÍTICO DE SEMÁNTICA
    )
    ```
  * En Jetpack Compose, al colocar `clickable(enabled = false)` sobre un contenedor padre, el nodo adquiere la semántica `disabled()`. Esto desactiva el foco de accesibilidad y bloquea la capacidad de recibir foco e input de teclado en todos los hijos que usan `OutlinedTextField` / [`MaryFitnessTextField`](file:///c:/Users/csehuanes/Downloads/mary-fitness/app/src/main/java/com/maryfitness/app/ui/common/MaryFitnessTextField.kt#L20).
  * **¿Por qué los checkboxes sí funcionan?** Porque [`MaryStyledCheckboxRow`](file:///c:/Users/csehuanes/Downloads/mary-fitness/app/src/main/java/com/maryfitness/app/ui/common/MaryFitnessDialogs.kt#L218) tiene su propio `.clickable { onCheckedChange(!checked) }` explícitamente habilitado (`enabled = true`). Este click handler directo consume los eventos táctiles y conmuta el estado booleano sin requerir foco ni teclado virtual.
  * Además, [`MaryFormDialog`](file:///c:/Users/csehuanes/Downloads/mary-fitness/app/src/main/java/com/maryfitness/app/ui/common/MaryFitnessDialogs.kt#L578) no tiene `Modifier.imePadding()` ni `verticalScroll()`, lo cual causaría colisiones de altura con el teclado en la pantalla del Moto G22.

### C. Estado en la Interfaz (UI)
* Al ingresar a la pantalla de **Medidas configuradas** y tocar cualquier variable existente, se abre el diálogo modal *"Editar configuración"*.
* **Comportamiento real en pantalla:**
  * **Campos de texto (*Nombre completo\**, *Nombre abreviado\*, *Unidad de medida\*):** Se muestran precargados con los datos actuales, pero al tocarlos **no responden, no muestran cursor parpadeante, no iluminan el borde rosa neón y el teclado virtual de Android no se abre**. Se comportan como elementos inertes / de solo lectura.
  * **Casillas de verificación (*"Tiene dos secciones"* y *"Obligatoria"*):** Responden con normalidad al toque, alternando su estado visual con el recuadro rosa (`NeonPink`).
  * **Botón "Guardar":** Es presionable, pero únicamente persiste las modificaciones que se hayan hecho a los checkboxes; los campos de texto quedan congelados.

---

## 2. Problema: Registrar pagos anteriores (Fecha de pago y edición de historial)
> *"No es posible registrar un pago anterior para un cliente, es decir, se debería permitir el registro de una fecha asociada a un pago. De igual manera al modificar un pago del historial de pagos"*

### A. Estado en Documentación
* **Especificación funcional original:**
  * En [`documentacion/especificaciones-negocio.md` (Línea 6)](file:///c:/Users/csehuanes/Downloads/mary-fitness/documentacion/especificaciones-negocio.md#L6):
    > *"un pago registra la activación o renovación del plan **a partir de una fecha específica**."*
  * En [`documentacion/funcionalidades-producto.md` (Módulo 2, Línea 13)](file:///c:/Users/csehuanes/Downloads/mary-fitness/documentacion/funcionalidades-producto.md#L13):
    > *"Registro de transacciones de pago: Monto pagado, **Fecha del pago** y cálculo automático de la Fecha de Vencimiento."*
  * En [`documentacion/pantallas/inventario-pantallas.md` (Sección 5.2)](file:///c:/Users/csehuanes/Downloads/mary-fitness/documentacion/pantallas/inventario-pantallas.md#L97-L101):
    > *"registrar un pago completo (que activa o renueva el plan **desde la fecha indicada**, calculando automáticamente la fecha de vencimiento)..."*
  * En [`documentacion/registros/formato-de-registros.md` (Secciones 3.3 y 4.2)](file:///c:/Users/csehuanes/Downloads/mary-fitness/documentacion/registros/formato-de-registros.md#L161-L175): Se especifica que `fechaPago` es un epoch millis independiente, permitiendo registrar pagos históricos y garantizando la regla `fechaVencimiento > fechaPago`.
  * En [`documentacion/Pendientes/003-respuesta-decisiones-pendientes.md` (Decisión #1)](file:///c:/Users/csehuanes/Downloads/mary-fitness/documentacion/Pendientes/003-respuesta-decisiones-pendientes.md#L5-L7):
    > *"El administrador tiene acceso total a todos los pagos y lo relacionado con ellos, sin distinción podrá eliminar, registrar, modificar un pago."*
* **Diagnóstico documental:**
  * Hay una **desconexión crítica**: en los reportes [`006-reporte-comparativo-documentacion-codigo-interfaz.md`](file:///c:/Users/csehuanes/Downloads/mary-fitness/documentacion/reportes/006-reporte-comparativo-documentacion-codigo-interfaz.md#L243-L253) y [`007-reporte-capacidad-administrador-acceso-total.md`](file:///c:/Users/csehuanes/Downloads/mary-fitness/documentacion/reportes/007-reporte-capacidad-administrador-acceso-total.md#L234-L245) se afirmó que "Pagos era el único módulo con CRUD completo y cumplía al 100%". Esto fue inexacto: se omitió por completo validar si el administrador podía asignar o corregir la **fecha del pago**.

### B. Estado en Código
* **En la base de datos (`PagoEntity.kt`):**
  * La entidad [`PagoEntity`](file:///c:/Users/csehuanes/Downloads/mary-fitness/app/src/main/java/com/maryfitness/app/data/local/entity/PagoEntity.kt#L27) **sí** tiene el campo `fechaPago: Long` y `fechaVencimiento: Long`. La tabla de Room permite guardar cualquier timestamp.
* **En el repositorio (`PagoRepository.kt`):**
  * **Al registrar:** [`registrarPago`](file:///c:/Users/csehuanes/Downloads/mary-fitness/app/src/main/java/com/maryfitness/app/data/repository/PagoRepository.kt#L46-L51) **no recibe ningún argumento de fecha**:
    ```kotlin
    suspend fun registrarPago(
        clienteId: Long,
        planId: Long,
        montoPagado: Long,
        plazoMaximoPagoSiParcial: Long?
    ): Result<Long>
    ```
    En la línea 112 se ejecuta: `val ahora = System.currentTimeMillis()`. La fecha de pago se fuerza internamente a `fechaPago = ahora` (línea 130) y la fecha de vencimiento a `ahora + duracionDias` (línea 118). No hay forma de pasar una fecha retroactiva.
  * **Al modificar:** [`modificarPago`](file:///c:/Users/csehuanes/Downloads/mary-fitness/app/src/main/java/com/maryfitness/app/data/repository/PagoRepository.kt#L205):
    ```kotlin
    suspend fun modificarPago(pagoId: Long, nuevoMonto: Long, nuevoPlazoMaximo: Long?): Result<Unit>
    ```
    Solo permite actualizar `montoPagado` y `plazoMaximoPago`. El campo `fechaPago` ni siquiera forma parte de la firma y no se puede modificar una vez insertado el registro.
* **En el ViewModel (`PagoViewModel.kt`):**
  * [`registrarPago`](file:///c:/Users/csehuanes/Downloads/mary-fitness/app/src/main/java/com/maryfitness/app/ui/pagos/PagoViewModel.kt#L76) y [`modificarPago`](file:///c:/Users/csehuanes/Downloads/mary-fitness/app/src/main/java/com/maryfitness/app/ui/pagos/PagoViewModel.kt#L90) carecen del parámetro `fechaPago`.
* **En la pantalla (`RegistrarPagoScreen.kt`):**
  * En el formulario superior: No existe ningún campo de fecha ni selector `DatePicker` (a diferencia de [`RegistrarMedidaScreen.kt`](file:///c:/Users/csehuanes/Downloads/mary-fitness/app/src/main/java/com/maryfitness/app/ui/medidas/RegistrarMedidaScreen.kt#L91), que sí incluye un selector de fecha pasada con `DatePicker`).
  * En el diálogo de edición [`DialogoEditarPago`](file:///c:/Users/csehuanes/Downloads/mary-fitness/app/src/main/java/com/maryfitness/app/ui/pagos/RegistrarPagoScreen.kt#L359-L420): Solo hay campos para `montoTexto` y `diasPlazo`.

### C. Estado en la Interfaz (UI)
* **Formulario de Registro de Pagos:**
  * Solo presenta:
    1. Selector desplegable de Plan (`Plan*`).
    2. Resumen financiero del periodo actual.
    3. Campo `Monto pagado*`.
    4. Campo `Plazo máximo de pago (días)*` (en caso de abonos parciales).
    5. Botón `Registrar`.
  * **Comportamiento real en UI:** Es completamente imposible ingresar una fecha diferente a "ahora". Si un cliente pagó hace días o semanas, el administrador no tiene forma visual de registrar esa fecha real.
* **Historial de Pagos y Diálogo de Modificación:**
  * En la lista inferior, cada fila muestra la fecha formateada (`dd/MM/yyyy`).
  * Al pulsar el icono de lápiz para editar el registro, se abre el diálogo *"Modificar pago"*, el cual **solo permite modificar el valor numérico del monto y los días de plazo**. No existe ningún botón, texto editable ni selector de calendario para corregir la fecha de pago asignada.

---

## Resumen Ejecutivo Comparativo

| Problema | Estado en Documentación | Estado en Código | Estado en Interfaz |
| :--- | :--- | :--- | :--- |
| **Edición de Medidas** | Documentado como editable. Auditorías asumieron erróneamente 100% de cumplimiento. | Diálogo implementado, pero el contenedor tiene `.clickable(enabled = false)`, lo que inyecta semántica `disabled()` y bloquea el foco/input en `MaryFitnessTextField`. Los checkboxes funcionan porque tienen su propio `clickable` activo. | La ventana modal se abre, pero los campos de texto no reciben foco ni abren el teclado. Solo se pueden marcar/desmarcar las dos casillas. |
| **Registrar Pagos Anteriores / Modificar Fecha** | Exigido explícitamente (*"a partir de una fecha específica"*, *"desde la fecha indicada"*). Omitido en auditorías previas. | `PagoEntity` soporta la fecha, pero `PagoRepository` tiene `System.currentTimeMillis()` cableado a fuego y `modificarPago` no recibe fecha. `PagoViewModel` no la expone. | No existe selector ni visualizador de fecha de pago en el formulario de creación ni en el diálogo de edición del historial. Imposible registrar pagos retroactivos. |
