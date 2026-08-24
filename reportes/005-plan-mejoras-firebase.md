# Plan 005: Mejoras Firebase (backup seguro, restauración, Storage, Crashlytics)

**Fecha:** 2026-08
**Origen:** decisión del propietario de ampliar el rol de Firebase.
**Documentos que este plan actualiza:** respuesta revisada a la pregunta #8 de `Pendientes/003-respuesta-decisiones-pendientes.md` (antes "Firestore es solo backup") y nuevo ADR #7 en `decisiones-tecnicas.md`.
**Principios que NO cambian:** Room es la única fuente de verdad; la app es 100% offline-first; el acceso a la app es el PIN local (ADR #2); un solo dispositivo; sin FCM (notificaciones por AlarmManager/WorkManager).

---

## Resumen de fases

| Fase | Alcance | Dependencias | Tamaño relativo |
|------|---------|--------------|-----------------|
| 1 | Activar el backup existente en Firestore | Cuenta Firebase + `google-services.json` | Pequeña |
| 2 | Auth anónima + reglas de seguridad | Fase 1 | Pequeña |
| 3 | Restauración asistida desde la nube | Fases 1 y 2 | Mediana |
| 4 | Firebase Storage: fotos/comprobantes WebP | Fase 2 (reglas por UID) | Grande |
| 5 | Crashlytics | Fase 1 (independiente funcionalmente) | Pequeña |

El orden recomendado es 1 → 2 → 3 → 4, con 5 en cualquier momento posterior a la 1.

---

## Fase 1 — Activar el backup existente en Firestore

**Objetivo:** poner en producción el sistema ya implementado (`FirestoreSyncManager`, `SyncWorker`, banderas `pendienteSync`), que hoy está dormido por falta de configuración.

### Prerrequisitos

1. Crear proyecto en la consola de Firebase (plan Spark gratuito es suficiente) y registrar una app Android con package exacto `com.maryfitness.app`.
2. Descargar `google-services.json` y colocarlo en `app/` (ignorado por git; ver `.gitignore`).
3. En la consola, crear la base Firestore en **modo producción** (reglas se cierran en Fase 2; entre Fase 1 y 2 mantener reglas de solo lectura o denegar escritura externa lo antes posible — ventana de riesgo mínima, aceptada por ser proyecto privado).

### Cambios en el código

* Raíz `build.gradle.kts`: descomentar `id("com.google.gms.google-services")`.
* `app/build.gradle.kts`: descomentar el mismo plugin. No hay cambios de dependencias (firebase-bom 33.1.2 + firestore-ktx ya están).
* README.md: quitar la nota "(Opcional)" del paso 4 de compilación.

### Procedimiento de verificación

1. `./gradlew assembleDebug` compila sin errores con el plugin activo.
2. Instalar en el Moto G22, crear datos de prueba, verificar en Ajustes > Sincronización: chip "Disponible", conteo "Pendientes por subir" baja tras "Sincronizar ahora".
3. Verificar en consola de Firestore las colecciones: `clientes`, `planes`, `pagos`, `configuraciones_medida`, `medida_registros`, `medida_valores`.
4. Activar modo avión, crear un registro, reconectar: la subida automática debe ocurrir (callback de conectividad o `SyncWorker` cada 3 h).

### Criterios de aceptación

* [ ] Con red, ninguna fila queda con `pendienteSync = true` más allá de un ciclo de worker.
* [ ] Sin `google-services.json` la app sigue funcionando al 100% offline (regresión: compilar variante sin el archivo).
* [ ] `ultimaSincronizacionExitosaMillis` se actualiza solo tras subidas completas.

### Riesgos

* Costo/cuota Spark: volumen ínfimo (un gimnasio, un dispositivo); sin riesgo práctico.
* Datos sensibles (nombres, cédulas) en Firestore: mitigado por Fase 2 inmediatamente después.

---

## Fase 2 — Auth anónima + reglas de seguridad

**Objetivo:** impedir que terceros lean o escriban el backup. Hoy, sin autenticación, las reglas tendrían que estar abiertas.

### Diseño

* Se agrega `firebase-auth` (mismo BOM). Al iniciar la app (en `MaryFitnessApplication`, antes del primer sync) se ejecuta `signInAnonymously()`.
* El UID anónimo se propaga como campo `propietarioUid` en cada documento subido (requiere tocar los data classes respaldados o envolverlos en un DTO de subida; se prefiere el wrapper para no ensuciar las entidades Room).
* Reglas de Firestore:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /{coleccion}/{doc} {
      allow create, update: if request.auth != null
                            && request.resource.data.propietarioUid == request.auth.uid;
      allow read, delete: if request.auth != null && resource.data.propietarioUid == request.auth.uid;
    }
  }
}
```

* El login visible de la app NO cambia: sigue siendo el PIN local (ADR #2 intacta). Anonymous Auth es credencial técnica invisible para el usuario; esto se deja explícito en el ADR #7 para evitar contradicciones.

### Problema conocido: pérdida del UID al reinstalar

Al reinstalar/borrar datos, Android genera un UID anónimo nuevo y el backup viejo queda inaccesible para la app. **Mitigación elegida (sin servidor):** migración asistida desde la consola de Firebase — el propietario copia los documentos del UID antiguo al nuevo usando la consola (procedimiento documentado paso a paso dentro de esta fase). Alternativas descartadas: tokens personalizados (requieren backend), cuenta email/contraseña embebida (mismo problema de persistencia local y superficie extra de ataque).

### Cambios en el código

* `app/build.gradle.kts`: `implementation("com.google.firebase:firebase-auth-ktx")`.
* Nuevo `CloudAuthManager` (patrón `AdminAuthManager`): garantiza sesión anónima activa y expone el UID actual.
* `FirestoreSyncManager.subirTodo()`: firma del uploader recibe `(coleccion, docId, data)` → se añade el campo `propietarioUid` vía wrapper.
* `FirestoreSyncManagerTest`: ajustar mocks del uploader.

### Criterios de aceptación

* [ ] Petición sin sesión anónima es rechazada por las reglas (verificado desde consola con request simulado).
* [ ] La app sincroniza igual que en Fase 1 (regresión verde).
* [ ] Documento en consola contiene `propietarioUid` correcto.

---

## Fase 3 — Restauración asistida desde la nube

**Objetivo:** recuperar los datos tras pérdida/cambio del dispositivo. Es la mejora que convierte el backup en útil de verdad. Es recuperación puntual, no sincronización continua (ADR #7).

### Diseño

* Disparador: pantalla Ajustes > Sincronización muestra botón "Restaurar backup" **solo si**: sesión anónima activa Y todas las tablas de negocio de Room están vacías Y el PIN fue configurado en esta instalación (instalación limpia). Si hay datos locales, el botón se deshabilita con explicación (evita mezclar estados).
* Descarga en orden de dependencia inversa a la subida: `clientes` → `planes` → `pagos` → `configuraciones_medida` → `medida_registros` → `medida_valores`. Cada colección se inserta en un batch atómico; si una falla, se cancela todo y Room queda vacío (reintentable).
* Filas restauradas entran con `pendienteSync = false` (no re-subir lo que ya está en la nube).
* Consulta filtrada por `propietarioUid == uidActual`. Tras reinstalación el UID es nuevo: primero aplicar migración asistida de Fase 2 (la UI explica este paso con texto claro para el propietario).
* Los IDs numéricos locales se preservan tal cual venían del documento (el `docId` ya es el id local original): las relaciones cliente↔pago↔medida quedan íntegras.
* Textos a actualizar junto con esta fase: tarjeta de Estado de la nube en `AjustesSincronizacionScreen.kt` (quitar "subida unidireccional" si aún estuviera) y sección Limitaciones del README.

### Cambios en el código

* `FirestoreSyncManager`: agregar `restaurarSiLocalVacio(): Result<Int>` con guardas descritas; reutiliza mutex.
* DAOs: métodos `insertarTodo(lista)` transaccionales (ya existen inserts individuales; agrupar en `@Transaction` por repositorio).
* ViewModel + UI en `AjustesSincronizacionScreen` con estado de progreso y resultado.
* Tests unitarios: caso feliz, caso tablas-no-vacías (rechaza), fallo a mitad (rollback lógico).

### Criterios de aceptación

* [ ] Instalación limpia + backup en la nube → restauración completa verificable (conteo local vs nube coincide).
* [ ] Con datos locales presentes, restauración rechazada con mensaje explicativo.
* [ ] Fallo de red a mitad no deja datos parciales en Room.

### Riesgos

* Cambio de esquema futuro entre backup y restauración: mitigar versionando documentos (campo `versionEsquema` agregado en cada subida desde esta fase; la restauración valida compatibilidad).

---

## Fase 4 — Firebase Storage: fotos y comprobantes (WebP)

**Objetivo:** cerrar el pendiente histórico del README (fotos de perfil y comprobantes de pago) hospedando binarios en Storage, coherente con el modelo offline-first.

### Diseño

* Captura con `TakePicture` (cámara del Moto G22) o selección de galería; conversión a WebP (pendiente declarado en README:34) con límite razonable (~512 px lado mayor, calidad ~80) antes de guardar.
* Copia local en `filesDir/media/` (Room guarda solo la ruta relativa + hash; nunca URLs remotas como fuente de verdad, coherente con ADR #3 sobre rotación de medios externos).
* Subida a Storage bajo `backups/{uid}/clientes/{clienteId}/foto.webp` y `backups/{uid}/pagos/{pagoId}/comprobante.webp`, disparada por el mismo mecanismo de conectividad (se extiende `FirestoreSyncManager` o se crea `StorageSyncManager` hermano).
* Reglas de Storage espejo de Firestore: lectura/escritura solo con `auth.uid == uid` del path.
* Visualización con Coil desde URI local; remota solo como fallback.

### Cambios en el código

* Dependencias: `firebase-storage-ktx`; `androidx.exifinterface` para orientación; nada nuevo para WebP (`Bitmap.compress(WEBP)` disponible en minSdk 31; evaluar `WEBP_LOSSY` API 30+).
* Entidades: `ClienteEntity.fotoRuta?`, `PagoEntity.comprobanteRuta?` → migración Room v4 con columnas nullable y `pendienteSync` para media (tabla nueva `media_pendiente` recomendada para no ensuciar entidades).
* UI: avatar/foto en detalle de cliente, botón adjuntar comprobante en registro de pago, visor a pantalla completa.
* Nueva migración `MIGRACION_3_4` en `AppDatabase`.

### Criterios de aceptación

* [ ] Foto capturada se ve correcta (orientación EXIF) offline y online.
* [ ] Comprobante queda asociado al pago y sobrevive reinstalación vía restauración (Fase 3 extiende descarga a media pendiente).
* [ ] Binarios NUNCA dentro de Firestore (solo rutas/hash en documentos).

### Riesgos

* Cuota de Storage Spark (5 GB): con compresión WebP agresiva, años de uso caben holgado; monitoreo mensual documentado.
* Privacidad de imágenes: reglas por UID obligatorias antes de subir la primera foto (depende de Fase 2).

---

## Fase 5 — Crashlytics

**Objetivo:** detectar errores en producción (recordatorio: la app aún no se probó end-to-end en dispositivo real, README:30).

### Cambios

* Dependencias `firebase-crashlytics-ktx` + plugin gradle `com.google.firebase.crashlytics` (raíz y app).
* Inicialización perezosa tras consentimiento: setting "Compartir reportes de error" en Ajustes > Acerca de (default OFF por privacidad; DataStore en `AppSettingsManager`). Sin consentimiento, `FirebaseCrashlytics.getInstance().setCrashlyticsCollectionEnabled(false)` en el arranque.
* Logs personalizados en puntos críticos ya identificados (fallos de sync, migraciones Room, errores de AscendAPI) con `recordException` para no-fatales.

### Criterios de aceptación

* [ ] Crash forzado en build debug aparece en consola (< 10 min).
* [ ] Con ajuste OFF no se envía ningún evento (verificar con `adb logcat`).
* [ ] Release con minify sube mapping automáticamente (plugin lo hace por defecto).

---

## Impacto documental cubierto por este plan

| Documento | Cambio |
|-----------|--------|
| `Pendientes/003-respuesta-decisiones-pendientes.md` #8 | Respuesta revisada: backup + restauración asistida + seguridad |
| `decisiones-tecnicas.md` | Nuevo ADR #7 que gobierna la evolución y aclara convivencia con ADR #2 |
| `Pendientes/listado-pendientes.md` | Restauración pasa de "pendiente de decisión" a decidida; quedan abiertos retención e integridad |
| README.md | Limitación de sync referenciada al plan; pasos de compilación actualizados en Fase 1 |
| `FirestoreSyncManager.kt` / test | KDoc actualizado a la decisión evolucionada |
| `AjustesSincronizacionScreen.kt` | Texto de usuario sin la afirmación de subida unidireccional permanente |

Los reportes históricos (`001`, `002` GAP 7, `003-decisiones-pendientes`) no se modifican: son fotografías de su momento; el ADR #7 prima sobre ellos según la regla de precedencia de `decisiones-tecnicas.md`.
