# Consideraciones de la aplicación
En este documento se describen algunas especificaciones importantes que se deberán tener en cuenta para la aplicación.
Las indicaciones se harán de manera general o especifica, no tendrán una estructura muy compleja, es generada de manera manual por el usuario. La información en este documento deberá ser refactorizada por un modelo de IA, de tal manera que estructure la información y la divida en las secciones pertinentes.

## Registros por Administrador
### Gestionar planes
El administrador puede gestionar todo lo relacionado con los planes y sus indicaciones, pudiendo crear, modificar, deshabilitar y eliminar planes.
Un plan tiene un nombre, precio, tiempo de vigencia, beneficios, 
Un plan puede ser individual o grupal, si es grupal, se deberá indicar la cantidad máxima de integrantes. En el caso de ser grupal, el registro del pago puede hacerse de manera parcial, dividido entre los integrantes o una sola persona.
Dentro de los servicios y comodidades que ofrece un plan puede o no estar el registro de medidas de manera periodica. Para esto, deberá ser posible 

### Notificaciones y recordatorios
Un usuario puede ser admitido si no ha cancelado la totalidad del plan (Pago parcial) si el administrador lo permite, en tal caso, el pago del usuario se registra como incompleto, cada que se intente modificar o actualizar información de dicho usuario, ya sean medidas o se interactue de cualquier manera con el usuario, se deberá mostrar un recordatorio del pago incompleto, de igual manera, se debe seleccionar un plazo máximo de pago.
Si un usuario no ha pagado y el administrador sigue realizando actualizaciones sobre sus datos (Registros, medidas, etc.) se le deberá registrar una deuda al usuario y un constante recordatorio al administrador de dicha deuda, cada que intente hacer modificación o actualización sobre dicho usuario, se deberá mostrar el mensaje de deuda.
Los recordatorios para las medidas de un usuario dependen enteramente de su plan, si el plan es mensual e incluye la toma de medidas y seguimiento, entonces se deberá mostrar notificaciones y/o recordatorios dentro y fuera de la aplicación sobre la actualización de medidas al usuario, es pertinente aclarar que para que esto suceda, el usuario debe tener un plan activo. Para la toma de medidas se realizará una notificación 1 día antes, y en 3 secciones del día, 8 AM, 2 PM y 5:30 PM. La notificación deberá tener opciones de respuesta de acceso rapido, una para revisar, otra para reasignar y okay.
**Revisar:** Envía a la sección de detalle del usuario, mostrando datos generales, especificaciones de medida, plan y registros anteriores.
**Reasignar:** Muestra lo mismo que la seccion _Revisar_ con la diferencia que esta solicita una fecha y hora de reasignación, puede indicarse solo la fecha, solo la hora o ambas. Si se indica solo fecha, se deberá realizar recordatorios para esa fecha siguiendo el estandar (Un recordatorio un dia antes y 3 el dia pertinente)
**Okay:** Indica que el administrador vió la notificación actual y no hará nada al respecto en ese preciso momento.

En caso de existir usuarios inactivos, con más de 5 meses sin tener modificaciones o registros vinculados, se deberán considerar como inactivos, para estos usuarios, se deberá tomar una desición crucial, eliminar, deshabilitar o ignorar.
**Eliminar:** Se elimina el acceso a información o detalles relacionados con ese cliente, manteniendo datos en el sistema pero no serán mostrados en el sistema ni consultados para disminuir el uso de recursos.
**Deshabilitar:** El administrador tendrá acceso a la información del usuario, sin embargo, dicho usuario está totalmente deshabilitado para nuevas asignaciones a planes y servicios prestados.
**Ignorar:** Indica que no se tomarán acciones sobre el usuario en cuestion, se mantendrá tal y como está. El sistema le deberá añadir un mes adicional para considerarlo nuevamente como inactivo, el mes adicional aplica únicamente para ese usuario en especifico, no afecta la integridad de los demás usuarios ni su lógica de funcionamiento.

Un usuario no podrá ser considerado inactivo, borrable o deshabilitable si tiene deudas pendientes.


### Registrar Usuarios
El administrador deberá estar en capacidad de añadir usuarios no registrados indicando campos como nombre completo (al menos primer nombre y primer apellido), tipo de documento (Cedula por defecto, este campo es un desplegable de opciones predeterminadas) y Numero de documento.
Un usuario podrá estár vinculado a un unico plan o ninguno, dependiendo el plan al que esté vinculado tendrá o no un seguimiento de medidas

### Metricas y registros
Un usuario puede tener varias medidas a traves del tiempo, es decir, se podrán registrar varias o ninguna medida a un mismo usuario siempre y cuando sea un usuario registrado.
Las medidas a tomar son definidas por el administrador, como mínima cantidad de medidas a tomar y medidas existentes: Peso (kg), pecho (cm), cintura (cm), cadera (cm), Brazo (cm), Pantorrilla (cm), tambien hay medidas que se toman en dos secciones parte más grande y parte más pequeña, tal es el caso de Brazo y pierna, ambas en cm. Una vez guardado el registro, se debe añadir la hora y fecha en la que se realizó el registro de las nuevas medidas.
El administrador deberá estar en capacidad de ingresar metricas a un usuario registrado, indicando si se tomará medida de una zona con configuración más grande y más pequeña y la unidad de medida.
Una medida tiene un nombre completo, un nombre abreviado y unidad de medida.

### Pagos de usuarios
- [ ] Registrar Pago
- [ ] Registrar Abono

### 


**FUNCIONALIDADES DE ADMINISTRADOR**
> CRUD indica 4 funcionalidades basicas y una adicional, Crear, Ver, Modificar, Eliminar y Deshabilitar.

`Gestionar Planes`
- CRUD planes
- Ver lista de Planes

`Gestionar Usuarios`
- CRUD Usuario
- Ver lista de Usuarios

`Configuración general`
- Configurar Notificaciones
- Configurar Aspectos visuales
- Configurar Acciones por Defecto
