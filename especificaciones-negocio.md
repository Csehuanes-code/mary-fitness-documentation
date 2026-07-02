## ESPECIFICACIONES DEL NEGOCIO (LÓGICA)

El agente de IA debe implementar las siguientes reglas de negocio de manera estricta en las validaciones de software:

* **Control de Roles de Acceso:** Existe únicamente el rol de **Administrador**. No se debe implementar lógica de login multiusuario, permisos dinámicos ni perfiles de cliente interactivos. La aplicación se abre directamente en el panel de control administrativo. Los clientes son únicamente registros de datos persistidos.
* **Consistencia Financiera:** No se permiten estados de pago parciales indeterminados; un pago registra la activación o renovación del plan a partir de una fecha específica. Si el plan vence, el estado del cliente cambia automáticamente a "Inactivo" o "Vencido" visualmente.
* **Regla de Trazabilidad de Medidas:** La app debe guardar la estampa de tiempo (`timestamp`) exacta de cada toma de medidas. La interfaz debe permitir la comparación visual directa entre la medición actual, la medición inmediatamente anterior y la medición base original para mostrar el progreso neto.
* **Estrategia de Sincronización de Datos Transaccionales:** Los datos de texto (registros de clientes, planes, históricos de pagos y medidas) tienen prioridad absoluta de almacenamiento inmediato en Room. El procesamiento de subida a Firestore ocurrirá en segundo plano tan pronto el sistema operativo reporte conectividad activa a través de los callbacks de conectividad nativos de Android.
