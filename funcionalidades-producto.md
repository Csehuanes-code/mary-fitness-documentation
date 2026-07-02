## FUNCIONALIDADES Y USOS DE LA APLICACIÓN

La aplicación se compone de cuatro módulos críticos administrados bajo un único flujo de trabajo:

1. **Gestión de Clientes (CRUD Básico):**
* Registro de nuevos miembros con datos personales básicos (Nombre, Identificación, Teléfono, Estado de cuenta).
* Visualización de listas de clientes optimizadas mediante `LazyColumn` para evitar sobrecarga de memoria en el Moto G22.


2. **Control de Planes y Finanzas:**
* Creación de planes con atributos de Nombre, Costo y Duración en días.
* Asociación de un cliente a un plan específico (un cliente también puede configurarse "Sin Plan").
* Registro de transacciones de pago: Monto pagado, Fecha del pago y cálculo automático de la Fecha de Vencimiento.


3. **Trazabilidad de Medidas Corporales:**
* **Medición Inicial (Base):** Registro inicial de parámetros antropométricos (Peso, Estatura, Hombros, Bíceps Izquierdo/Derecho, Pierna Izquierda/Derecha, etc.).
* **Mediciones Posteriores:** El sistema debe obligar a que las re-mediciones utilicen exactamente el mismo set de variables tomado en la medición base para garantizar la coherencia histórica de los datos y gráficos de progreso.


4. **Sistema de Notificaciones Locales Autónomas:**
* **Alerta de Pago:** Envío de una notificación push local en el dispositivo cuando falten $X$ días para el vencimiento del plan de un cliente.
* **Alerta de Re-medición:** Envío de un recordatorio periódico configurable para volver a tomar las medidas antropométricas del cliente tomando como referencia la fecha del último registro.

