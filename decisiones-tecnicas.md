# Decisiones Técnicas (ADR)

Este documento resuelve las ambigüedades y contradicciones identificadas en `reportes/001-especificaciones-pendientes.md`. Estas decisiones son vinculantes para la implementación y priman sobre lecturas contradictorias de otros documentos.

## 1. Lógica de pagos: se permiten abonos parciales

No se prohíben los abonos. Lo que se prohíbe es el estado *indeterminado*: todo pago queda registrado con un estado explícito.

* Un `Pago` tiene: `montoTotal` (heredado del plan), `montoPagado`, `estado` (`COMPLETO` | `PARCIAL`), `fechaPago`, `fechaVencimiento`, `plazoMaximoPago` (solo si `PARCIAL`).
* Un `Cliente` tiene un `estadoCuenta` derivado, no almacenado como fuente de verdad ambigua: `ACTIVO`, `DEUDA` (pago parcial dentro del plazo), `VENCIDO` (plan expiró o se superó el plazo de pago).
* Cada vez que el administrador interactúa con un cliente en `DEUDA`, la UI muestra un banner de recordatorio de deuda (no bloqueante).
* Para planes grupales con pago dividido, cada integrante tiene su propio registro de `Pago` referenciando el mismo `Plan` y el mismo periodo de vigencia grupal.

## 2. Autenticación: PIN local, sin Firebase Auth

* PIN numérico (4-6 dígitos), hash con `BCrypt` (librería `at.favre.lib:bcrypt`), almacenado en `DataStore<Preferences>`.
* Si no existe PIN configurado, la app fuerza una pantalla de configuración inicial antes del dashboard.
* No hay multiusuario ni recuperación remota de PIN (coherente con "único rol Administrador" y offline-first). La recuperación es un reset manual local (borra el PIN y fuerza reconfiguración), protegido detrás de una confirmación explícita.

## 3. AscendAPI: incluida como sección básica de consulta

* Se agrega una pantalla "Biblioteca de ejercicios" que consume ExerciseDB vía Retrofit, solo cuando hay conectividad.
* No se persisten URLs de medios (GIFs/imágenes/video) más allá de la sesión en memoria/caché HTTP corta, por la rotación semanal documentada en `api-externa/`.
* Sí se pueden cachear en Room, de forma opcional, los campos estables: `exerciseId`, `name`, `bodyParts`, `targetMuscles` — nunca URLs.
* Sin conexión, la pantalla muestra un aviso "Biblioteca no disponible sin conexión" en vez de fallar o mostrar imágenes rotas. Esta sección es informativa/de consulta para el administrador (no se asigna a planes en esta fase).
* La API key de RapidAPI se inyecta vía `local.properties` / `BuildConfig`, nunca hardcodeada en el repo.

## 4. Gestión de usuarios inactivos (>5 meses sin registros)

* **Eliminar** = soft delete: campo `oculto: Boolean = true` en `Cliente`. Los datos permanecen en Room/Firestore pero se excluyen de todas las consultas de listados y búsquedas (`WHERE oculto = 0`).
* **Deshabilitar**: campo `habilitado: Boolean = false`. El admin sigue viendo al cliente y su historial, pero no puede asociarlo a nuevos planes/servicios.
* **Ignorar**: no cambia flags, pero registra `fechaProximaRevisionInactividad = ahora + 1 mes` únicamente para ese cliente; no afecta el cálculo global de inactividad de otros clientes.
* Chequeo automático obligatorio: ninguna de las tres acciones se habilita en la UI si el cliente tiene `estadoCuenta == DEUDA` con saldo pendiente > 0. La app bloquea el botón y explica por qué.

## 5. Tipos de documento (dropdown)

Lista por defecto (Colombia, dado el uso de "Cédula"): `Cédula de Ciudadanía` (default), `Tarjeta de Identidad`, `Cédula de Extranjería`, `Pasaporte`, `NIT`. Configurable a futuro, pero fija para el MVP.

## 6. Notificación "Reasignar" fuera de la app

Al tocar "Reasignar" desde la notificación del sistema, se abre la app directamente en la pantalla de detalle del cliente con un diálogo de selección de fecha/hora ya visible (no una pantalla intermedia adicional), reutilizando el mismo componente de detalle usado desde "Revisar".

## 7. Evolución del rol de Firebase (revisa la respuesta original a la pregunta 003 #8)

La respuesta original a `Pendientes/003-respuesta-decisiones-pendientes.md` #8 fue "Firestore es solo backup", con subida unidireccional y descarga descartada. El propietario aprobó (2026-08) ampliar ese alcance manteniendo sus principios base:

* **Room sigue siendo la única fuente de verdad** y la app sigue siendo 100% offline-first: ninguna lectura de negocio consulta la nube.
* **Sigue prohibida la sincronización multi-dispositivo**: un solo dispositivo (Moto G22) opera la app.
* **Restauración asistida:** se permite descargar el backup una única vez, solo cuando Room está vacío (instalación limpia tras pérdida/cambio de dispositivo). Es recuperación ante desastres, no sincronización.
* **Auth anónima:** se usa exclusivamente como credencial técnica para restringir Firestore/Storage por UID en las reglas de seguridad. **No modifica el ADR #2**: el acceso a la app sigue siendo el PIN local; no hay login de usuario ni multiusuario. Tras una reinstalación el UID cambia y la recuperación del backup viejo se hace con migración asistida desde la consola de Firebase (procedimiento en el plan).
* **Firebase Storage:** además de hospedar `latest.json`/APKs, alojará fotos de perfil y comprobantes de pago comprimidas a WebP (pendiente histórico del README).
* **Crashlytics:** telemetría de errores, activable/desactivable desde Ajustes por privacidad.

Plan detallado por fases con criterios de aceptación: `reportes/005-plan-mejoras-firebase.md`.
