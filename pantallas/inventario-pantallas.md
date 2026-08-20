# Inventario de Pantallas de la Aplicación

Este documento describe, a nivel funcional, las pantallas que compone la aplicación. Para cada una se indica qué información se muestra, qué acciones puede realizar el administrador desde ella y con qué otras pantallas se conecta. No se describen componentes visuales, botones, colores ni detalles de implementación: el objetivo es responder **qué hace la pantalla**, no **cómo se ve**.

Referencia de dominio: `vision-general.md`, `funcionalidades-producto.md`, `especificaciones-negocio.md`, `especificaciones-generales.md`, `decisiones-tecnicas.md`, `api-externa/`.

---

## 1. Pantallas de arranque y acceso

### 1.1 Carga inicial (Splash)
- **Qué muestra:** una pantalla transitoria mientras la aplicación verifica su estado interno (si existe un PIN configurado, si hay una sesión de administrador activa, si la base de datos local está lista).
- **Qué se puede hacer:** nada interactivo; es un paso de tránsito automático.
- **Conexión con otras pantallas:** redirige automáticamente a *Configuración inicial de acceso* (si es el primer uso, sin PIN definido) o a *Inicio de sesión* (si ya existe un PIN configurado).

### 1.2 Configuración inicial de acceso (primer uso)
- **Qué muestra:** una solicitud para que el administrador defina su PIN de acceso por primera vez.
- **Qué se puede hacer:** definir y confirmar un PIN numérico que quedará asociado a la única cuenta de administrador de la aplicación.
- **Conexión con otras pantallas:** al completarse, lleva directamente al *Panel principal*, sin pasos intermedios adicionales.

### 1.3 Inicio de sesión (validación de PIN)
- **Qué muestra:** una solicitud del PIN de acceso para el administrador ya configurado.
- **Qué se puede hacer:** ingresar el PIN para acceder al panel administrativo; en caso de error, se informa que el PIN no es correcto y se permite reintentar. Existe una opción para iniciar el restablecimiento del PIN si fue olvidado.
- **Conexión con otras pantallas:** si el acceso es correcto, se pasa al *Panel principal*. Si se solicita recuperación, se dirige a *Restablecimiento de PIN*.

### 1.4 Restablecimiento de PIN
- **Qué muestra:** una advertencia de que el restablecimiento borra el PIN actual y obliga a definir uno nuevo, ya que no existe recuperación remota (no hay multiusuario ni backend de autenticación).
- **Qué se puede hacer:** confirmar explícitamente el restablecimiento y, tras confirmar, definir un nuevo PIN.
- **Conexión con otras pantallas:** vuelve a *Configuración inicial de acceso* para definir el nuevo PIN, y de ahí al *Panel principal*.

---

## 2. Panel principal

### 2.1 Panel principal / Dashboard administrativo
- **Qué muestra:** un resumen general del estado del gimnasio: cantidad de clientes activos, en deuda y vencidos; próximos vencimientos de planes; próximas mediciones pendientes; accesos a los módulos principales (clientes, planes, medidas, biblioteca de ejercicios, configuración).
- **Qué se puede hacer:** navegar a cualquiera de los módulos principales; revisar de un vistazo alertas pendientes (deudas, vencimientos próximos, recordatorios de medición) sin tener que entrar a cada cliente.
- **Conexión con otras pantallas:** es el punto central desde el cual se accede a *Lista de clientes*, *Lista de planes*, *Gestión de usuarios inactivos*, *Biblioteca de ejercicios* y *Configuración general*. Es también el destino tras un inicio de sesión exitoso.

---

## 3. Módulo de clientes

### 3.1 Lista de clientes
- **Qué muestra:** el listado completo de clientes registrados (excluyendo los marcados como ocultos/eliminados), con su estado de cuenta visible por cada uno (activo, en deuda, vencido) y el plan al que están asociados (o "sin plan").
- **Qué se puede hacer:** buscar o filtrar clientes por nombre, número de documento, estado de cuenta o plan asociado; ordenar el listado; acceder al detalle de un cliente tocándolo; iniciar el registro de un nuevo cliente.
- **Conexión con otras pantallas:** lleva a *Detalle de cliente* y a *Registro de nuevo cliente*. Es alcanzable desde el *Panel principal*.

### 3.2 Registro de nuevo cliente
- **Qué muestra:** un formulario para capturar los datos personales básicos de un nuevo miembro: nombre completo (al menos primer nombre y primer apellido), tipo de documento (con cédula de ciudadanía como opción por defecto, entre otras opciones predefinidas) y número de documento, teléfono.
- **Qué se puede hacer:** registrar al cliente sin asignarle plan (queda "sin plan") o asociarlo de una vez a un plan existente durante el mismo registro.
- **Conexión con otras pantallas:** al guardar, dirige al *Detalle de cliente* recién creado. Puede requerir seleccionar un plan desde *Lista de planes* si se decide asignar uno durante el registro.

### 3.3 Edición de datos de cliente
- **Qué muestra:** los mismos datos personales del cliente ya registrados, disponibles para modificación.
- **Qué se puede hacer:** actualizar nombre, tipo y número de documento, teléfono, y reasignar o quitar el plan asociado.
- **Conexión con otras pantallas:** se accede desde el *Detalle de cliente* y regresa a él tras guardar los cambios.

### 3.4 Detalle de cliente
- **Qué muestra:** toda la información relevante de un cliente en un mismo lugar: datos generales, plan asignado y su vigencia, estado de cuenta actual (activo, en deuda o vencido), historial resumido de pagos, historial de mediciones corporales, y —si aplica— un recordatorio visible de deuda pendiente o de plazo máximo de pago vencido.
- **Qué se puede hacer:** editar los datos del cliente; registrar un nuevo pago o abono; tomar una nueva medición; revisar el historial completo de pagos; revisar el historial y comparación de medidas; cambiar el plan asignado; y, si el cliente lleva más de cinco meses sin registros y no tiene deudas pendientes, decidir si se elimina (oculta), se deshabilita o se ignora por este ciclo. Si el cliente tiene una deuda pendiente, cualquier intento de modificación muestra primero el recordatorio de deuda antes de continuar.
- **Conexión con otras pantallas:** es el nodo central del módulo de clientes; conecta con *Edición de datos de cliente*, *Registro de pago/abono*, *Historial de pagos*, *Registro de nueva medición*, *Historial y comparación de medidas*, y *Selección/edición de plan*. También es el destino directo al tocar "Revisar" o "Reasignar" desde una notificación de recordatorio de medición, y el destino de la acción "Reasignar" mostrando además la selección de nueva fecha/hora directamente en esta misma pantalla.

### 3.5 Gestión de usuarios inactivos
- **Qué muestra:** el listado de clientes que llevan más de cinco meses sin registros o modificaciones asociadas (mediciones, pagos, actualizaciones), excluyendo a quienes tengan deudas pendientes (estos no pueden aparecer como accionables aquí).
- **Qué se puede hacer:** para cada cliente inactivo listado, decidir entre eliminarlo (se oculta de listados y búsquedas, sin borrar sus datos), deshabilitarlo (se mantiene visible su historial pero no puede vincularse a nuevos planes o servicios), o ignorarlo por un mes adicional antes de volver a evaluarlo.
- **Conexión con otras pantallas:** accesible desde el *Panel principal* o desde *Lista de clientes*; cada cliente listado puede abrirse en su *Detalle de cliente* antes de tomar la decisión.

---

## 4. Módulo de planes

### 4.1 Lista de planes
- **Qué muestra:** todos los planes existentes con su nombre, precio, duración (o vigencia), tipo (individual o grupal) y estado (habilitado o deshabilitado).
- **Qué se puede hacer:** buscar y filtrar planes; acceder al detalle de un plan; iniciar la creación de uno nuevo; deshabilitar o eliminar un plan existente.
- **Conexión con otras pantallas:** lleva a *Detalle de plan* y a *Creación de plan*. Accesible desde el *Panel principal*.

### 4.2 Creación / edición de plan
- **Qué muestra:** un formulario con los atributos del plan: nombre, precio, duración en días, beneficios/servicios incluidos, si es individual o grupal (y en ese caso, la cantidad máxima de integrantes), y si incluye o no seguimiento periódico de medidas corporales.
- **Qué se puede hacer:** definir o modificar cada uno de estos atributos; habilitar o deshabilitar el plan.
- **Conexión con otras pantallas:** se accede desde *Lista de planes* y, al guardar, regresa al *Detalle de plan* correspondiente.

### 4.3 Detalle de plan
- **Qué muestra:** toda la información del plan (los mismos atributos de creación) junto con la lista de clientes actualmente asociados a él y, si es grupal, el desglose de integrantes y el estado de pago individual de cada uno dentro del período de vigencia compartido.
- **Qué se puede hacer:** editar el plan, deshabilitarlo o eliminarlo (si no tiene clientes activos vinculados), y navegar a cualquiera de los clientes asociados.
- **Conexión con otras pantallas:** conecta con *Creación/edición de plan* y con el *Detalle de cliente* de cada integrante asociado.

---

## 5. Módulo de pagos

### 5.1 Historial de pagos de cliente
- **Qué muestra:** el listado cronológico de todos los pagos y abonos registrados para un cliente: monto pagado, fecha de pago, estado (completo o parcial) y fecha de vencimiento resultante de cada registro.
- **Qué se puede hacer:** revisar el detalle de cada pago individual; iniciar el registro de un nuevo pago o abono.
- **Conexión con otras pantallas:** accesible desde el *Detalle de cliente*; lleva a *Registro de pago/abono*.

### 5.2 Registro de pago / abono
- **Qué muestra:** los datos del plan vigente del cliente (monto total esperado según el plan) para contextualizar el pago a registrar.
- **Qué se puede hacer:** registrar un pago completo (que activa o renueva el plan desde la fecha indicada, calculando automáticamente la fecha de vencimiento) o un abono parcial (que deja al cliente en estado de deuda y solicita definir un plazo máximo de pago). En planes grupales con pago dividido, el registro se hace de forma individual por integrante, sobre el mismo período de vigencia grupal.
- **Conexión con otras pantallas:** se accede desde el *Detalle de cliente* o desde el *Historial de pagos*; al confirmar, regresa a cualquiera de esas dos pantallas con el estado de cuenta del cliente ya actualizado.

---

## 6. Módulo de medidas corporales

### 6.1 Configuración de variables de medida
- **Qué muestra:** el catálogo de medidas antropométricas disponibles para ser tomadas (por ejemplo peso, pecho, cintura, cadera, brazo, pantorrilla, entre otras), indicando para cada una su nombre completo, nombre abreviado, unidad de medida, y si aplica en dos secciones (lado más grande y lado más pequeño, como en brazo o pierna).
- **Qué se puede hacer:** definir qué medidas existen y cuáles son obligatorias como mínimo para toda toma de mediciones; añadir nuevas variables de medida o modificar las existentes.
- **Conexión con otras pantallas:** accesible desde *Configuración general*; determina qué campos aparecerán en *Medición inicial* y *Registro de nueva medición* para todos los clientes.

### 6.2 Medición inicial (base) de cliente
- **Qué muestra:** el formulario con el conjunto de variables de medida configuradas, vacío, para la primera toma de datos antropométricos de un cliente que aún no tiene mediciones registradas.
- **Qué se puede hacer:** registrar los valores iniciales de cada variable; al guardar, este conjunto de variables queda fijado como referencia obligatoria para todas las mediciones futuras de ese cliente.
- **Conexión con otras pantallas:** se accede desde el *Detalle de cliente* cuando no existen mediciones previas; al guardar, lleva al *Historial y comparación de medidas* del cliente.

### 6.3 Registro de nueva medición (seguimiento)
- **Qué muestra:** el mismo conjunto de variables usado en la medición base del cliente (no se permite un set distinto), junto con un recordatorio de cuándo fue la medición anterior.
- **Qué se puede hacer:** registrar los nuevos valores, quedando la fecha y hora exactas del registro guardadas automáticamente junto con la medición.
- **Conexión con otras pantallas:** accesible desde el *Detalle de cliente* y desde la acción "Revisar" o "Reasignar" de una notificación de recordatorio de medición; al guardar, lleva al *Historial y comparación de medidas*.

### 6.4 Historial y comparación de medidas
- **Qué muestra:** la evolución de cada variable de medida a lo largo del tiempo para un cliente, permitiendo ver lado a lado la medición actual, la medición inmediatamente anterior y la medición base original, para evidenciar el progreso neto.
- **Qué se puede hacer:** revisar cualquier medición histórica puntual; comparar visualmente el avance entre distintos períodos.
- **Conexión con otras pantallas:** accesible desde el *Detalle de cliente*; permite volver a iniciar un *Registro de nueva medición* desde aquí mismo.

---

## 7. Notificaciones y recordatorios

### 7.1 Notificación de vencimiento de pago
- **Qué muestra (fuera de la app, como notificación del sistema):** un aviso de que el plan de un cliente está por vencer, enviado dos días antes del vencimiento.
- **Qué se puede hacer:** tocar la notificación para ir directamente al *Detalle de cliente* correspondiente.
- **Conexión con otras pantallas:** lleva a *Detalle de cliente*.

### 7.2 Notificación de recordatorio de medición
- **Qué muestra (fuera de la app):** un aviso de que corresponde volver a tomar las medidas de un cliente, enviado un día antes en tres momentos del día, con tres respuestas rápidas posibles: revisar, reasignar u marcar como visto.
- **Qué se puede hacer:** elegir "Revisar" para ver el detalle completo del cliente (datos generales, plan, medidas y registros anteriores); elegir "Reasignar" para ver lo mismo que "Revisar" pero además definir una nueva fecha y/u hora para el recordatorio (puede indicarse solo fecha, solo hora, o ambas, regenerando el ciclo de avisos de un día antes y tres el día indicado); elegir la opción de "visto" para descartar el aviso sin tomar ninguna acción inmediata.
- **Conexión con otras pantallas:** "Revisar" y "Reasignar" llevan al *Detalle de cliente*, este último con la selección de nueva fecha/hora visible de inmediato en esa misma pantalla.

### 7.3 Centro de alertas pendientes
- **Qué muestra:** un listado consolidado de todas las alertas activas de la aplicación: clientes con pago próximo a vencer, clientes en deuda, clientes con medición pendiente o atrasada, y clientes candidatos a revisión por inactividad.
- **Qué se puede hacer:** revisar cada alerta y navegar directamente al cliente involucrado para resolverla.
- **Conexión con otras pantallas:** accesible desde el *Panel principal*; lleva al *Detalle de cliente* de cada alerta.

---

## 8. Biblioteca de ejercicios (consulta)

### 8.1 Listado de ejercicios
- **Qué muestra:** un catálogo consultable de ejercicios (proveniente de una fuente externa), con su nombre, grupo muscular objetivo y una vista previa asociada, disponible únicamente cuando hay conexión a internet. Sin conexión, se informa que la biblioteca no está disponible en ese momento en lugar de mostrar contenido incompleto o roto.
- **Qué se puede hacer:** buscar ejercicios por nombre (con sugerencias mientras se escribe), filtrar por grupo muscular, equipamiento, dificultad o tipo de ejercicio; abrir el detalle de un ejercicio.
- **Conexión con otras pantallas:** accesible desde el *Panel principal*; lleva a *Detalle de ejercicio*. Es una sección informativa/de consulta para el administrador, no se asigna todavía a planes ni a clientes.

### 8.2 Detalle de ejercicio
- **Qué muestra:** la información ampliada de un ejercicio: instrucciones paso a paso, músculos involucrados (incluyendo una visualización de activación muscular) y video demostrativo, cuando hay conexión disponible.
- **Qué se puede hacer:** revisar las instrucciones completas y la visualización muscular del ejercicio seleccionado.
- **Conexión con otras pantallas:** se accede únicamente desde el *Listado de ejercicios*.

---

## 9. Configuración general

### 9.1 Configuración general (menú)
- **Qué muestra:** las categorías disponibles de configuración de la aplicación.
- **Qué se puede hacer:** navegar a cada subsección de configuración.
- **Conexión con otras pantallas:** accesible desde el *Panel principal*; da acceso a *Configuración de notificaciones*, *Configuración de aspectos visuales*, *Configuración de acciones por defecto*, *Configuración de variables de medida*, *Estado de sincronización* y *Acerca de la aplicación*.

### 9.2 Configuración de notificaciones
- **Qué muestra:** los recordatorios activos configurables: aviso de vencimiento de pago, recordatorio periódico de re-medición y sus horarios asociados.
- **Qué se puede hacer:** activar o desactivar cada tipo de recordatorio; ajustar la periodicidad del recordatorio de re-medición.
- **Conexión con otras pantallas:** accesible desde *Configuración general*.

### 9.3 Configuración de aspectos visuales
- **Qué muestra:** las preferencias de presentación de la aplicación disponibles para el administrador.
- **Qué se puede hacer:** ajustar dichas preferencias de presentación a su gusto.
- **Conexión con otras pantallas:** accesible desde *Configuración general*.

### 9.4 Configuración de acciones por defecto
- **Qué muestra:** los comportamientos predeterminados del sistema que pueden ajustarse, como el plazo máximo de pago sugerido para abonos parciales, o la decisión por defecto sugerida ante usuarios inactivos.
- **Qué se puede hacer:** modificar estos valores predeterminados para agilizar los flujos frecuentes de registro de pagos y revisión de inactividad.
- **Conexión con otras pantallas:** accesible desde *Configuración general*.

### 9.5 Estado de sincronización
- **Qué muestra:** el estado de la sincronización en segundo plano de los datos locales hacia la nube (última sincronización exitosa, pendientes por subir, estado de conectividad actual). Los datos locales en el dispositivo son siempre la fuente de verdad inmediata; esta pantalla es meramente informativa sobre el respaldo remoto.
- **Qué se puede hacer:** revisar si existen cambios pendientes de sincronizar y forzar un intento de sincronización manual si hay conexión disponible.
- **Conexión con otras pantallas:** accesible desde *Configuración general*.

### 9.6 Acerca de la aplicación
- **Qué muestra:** información general de la aplicación (versión, propósito, dispositivo objetivo).
- **Qué se puede hacer:** consultar esta información de referencia.
- **Conexión con otras pantallas:** accesible desde *Configuración general*.
