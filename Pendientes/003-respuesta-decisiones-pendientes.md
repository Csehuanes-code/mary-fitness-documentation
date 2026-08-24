# Decisiones tomadas y preguntas resueltas

## 1. Acumulación de Pagos Parciales
### Respuesta
**A. Acumular pagos**, el sistema deberá acumular varios pagos, llevando el registro de los pagos y mostrando la realidad financiera.\
Dado que los registros de pagos son manuales, no se manejaran reembolsos ni pagos duplicados. El administrador tiene acceso total a todos los pagos y lo relacionado con ellos, sin distinción podrá, eliminar, registrar, modificar un pago.

## 2. Periodicidad de Re-mediciones
**B. Usar remedición de periodicidad en días**, donde el administrador configura cada cuántos días se piden medidas sin importar el plan. El valor por defecto son 30 días, como rango permitido es mínimo 15 días y máximo 90 días.

## 3. Comportamiento del Botón "Reasignar"
1. Es un DatePicker + TimePicker nativo
2. Solo aplica a la siguiente medición
3. No se puede reasignar una fecha pasada, pero si se puede añadir un registro de medidas con fecha pasada.
4. Solo aplaza el recordatorio
5. Igual deberá solicitar las medidas esto debido a que el recordatorio fue solicitado mientras el plan estaba vigente.

## 4. Planes Grupales: Validación de Cupo
**A Validar en pago**, se debe validar, mostrar visualmente lo que pasa e impedir la acción.

## 5. Overlapping de planes en un Cliente
1. Si, primero debe cancelar el plan actual para actualizar al nuevo plan deseado
2. No hay reembolso ni ajuste
3. Si
4. Depende de varios factores, estado de pago, ultima fecha de pago, fecha actual y tipo de plan.\
   Plan trimestral vencido, al validar la fecha actual menos la cantidad de dias de duracion del plan se muestra una fecha reciente (1 a 7 dias) indica que es su plan actual.

## 6. Permisos de Notificación en Android 13+
**A Solicitar en onboarding**, se debe pedir durante la configuración inicial del PIN, si no se consede y luego es requerida una acción con este permiso, se deberá solicitar el permiso. Si no se concede, deberá mostrarse al usuario que el permiso afecta directamente el funcionamiento.

## 7. Persistencia de Datos tras Soft Delete
**B. Purgar despues de 6 meses**

## 8. Sincronización Firestore: Estrategia de Merge
**Respuesta original:** Firestore es solo backup.
**Respuesta revisada (2026-08):** Firestore continúa siendo un respaldo secundario — Room es la única fuente de verdad y NO habrá sincronización multi-dispositivo — pero el rol de la nube se amplía según lo detallado en el ADR #7 de `decisiones-tecnicas.md` y el plan `reportes/005-plan-mejoras-firebase.md`:

1. **Backup activo y seguro:** subida de todas las colecciones de negocio con Auth anónima de Firebase para poder cerrar las reglas de seguridad (el login de la app sigue siendo el PIN local; ver ADR #2).
2. **Restauración asistida:** se permite DESCARGAR el backup una sola vez, en instalación limpia con base local vacía, para recuperar datos tras pérdida o cambio del dispositivo. No hay merge ni descarga continua.
3. **Firebase Storage:** hospedaje del `latest.json`/APK de actualización (ya existente) y, más adelante, fotos de perfil y comprobantes de pago.
4. **Crashlytics:** telemetría de errores en producción.

Lo que sigue prohibido: sincronización bidireccional continua, edición de datos desde la nube y operación multi-dispositivo simultánea.

## 9. Manejo de Planes Vencidos sin Acción
**D Banner persistente** Mostrar contador de clientes vencidos en Dashboard sin fecha de caducidad

## 10. Requisitos de Pruebas Automatizadas
**Todas las opciones** se debe probar todo antes de release.