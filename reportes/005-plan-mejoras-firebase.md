# Plan 005: Mejoras Firebase (backup seguro, restauración, Storage, Crashlytics)

**Fecha:** 2026-08
**Origen:** decisión del propietario de ampliar el rol de Firebase.
**Documentos que este plan actualiza:** respuesta revisada a la pregunta #8 de `Pendientes/003-respuesta-decisiones-pendientes.md` (antes "Firestore es solo backup") y nuevo ADR #7 en `decisiones-tecnicas.md`.
**Principios que NO cambian:** Room es la única fuente de verdad; la app es 100% offline-first; el acceso a la app es el PIN local (ADR #2); un solo dispositivo; sin FCM (notificaciones por AlarmManager/WorkManager).

---

## Resumen de fases

| Fase | Alcance | Dependencias | Tamaño relativo | Estado |
|------|---------|--------------|-----------------|--------|
| 1 | Activar el backup existente en Firestore | Cuenta Firebase + `google-services.json` | Pequeña | Código listo; falta el proyecto en la consola |
| 2 | Auth anónima + reglas de seguridad | Fase 1 | Pequeña | **Implementada en código** (2026-10-02); falta proyecto en la consola |
| 3 | Restauración asistida desde la nube | Fases 1 y 2 | Mediana | No iniciada |
| 4 | Firebase Storage: fotos/comprobantes WebP | Fase 2 (reglas por UID) | Grande | No iniciada |
| 5 | Crashlytics | Fase 1 (independiente funcionalmente) | Pequeña | No iniciada |

El orden recomendado es 1 → 2 → 3 → 4, con 5 en cualquier momento posterior a la 1.

## Incidente: el proyecto de Firebase fue eliminado (2026-10)

El proyecto `mary-fitness` (project number `847997858505`) fue borrado. Consecuencias y estado
actual:

* El `app/google-services.json` del repositorio de trabajo apunta a ese proyecto muerto: la API key ya no
  sirve, cada escritura a Firestore falla y el respaldo quedo sin guardar nada. **Los datos que se
  submieron alli se perdieron de forma irrecuperable.**
* El fallo era invisible para el propietario: `FirestoreSyncManager` capturaba la excepcion y solo
  escribia un `Log.w`, mientras la tarjeta de Ajustes > Sincronizacion seguia mostrando el chip
  "Disponible" porque `FirebaseApp.getInstance()` si inicializa con un JSON obsoleto. Esto violaba la
  regla #4 de `reglas-codificacion.md` (nada de fallos silenciosos) y ya esta corregido: el gestor
  expone `ultimoError` y la pantalla lo muestra al propietario.
* Decision tomada (2026-10-02): **recuperar el backup se considera mas valioso que exponerlo**, asi
  que antes de crear el proyecto nuevo se implemento la fase 2 completa para no repetir el error con
  reglas abiertas.
* **`google-services.json` esta en `.gitignore` y no se versiona**: el archivo del working tree es
  local. Quien clone el repositorio arranca sin configuracion de nube y la app funciona 100% offline.
* Riesgo de repeticion: en el plan Spark, Firebase puede eliminar proyectos por inactividad. El
  `SyncWorker` (cada 3 h) solo genera escrituras cuando hay cambios locales por subir, de modo que un
  gimnasio sin movimientos durante semanas podria perder el proyecto otra vez. Crashlytics (fase 5)
  tambien generaria actividad.

---

## Fase 1 — Activar el backup existente en Firestore

**Objetivo:** poner en producción el sistema ya implementado (`FirestoreSyncManager`, `SyncWorker`, banderas `pendienteSync`), que hoy está dormido por falta de configuración.

### Prerrequisitos

1. Crear proyecto en la consola de Firebase (plan Spark gratuito es suficiente) y registrar una app Android con package exacto `com.maryfitness.app`.
2. Descargar `google-services.json` y colocarlo en `app/` (ignorado por git; ver `.gitignore`).
3. En la consola, crear la base Firestore en **modo producción** (reglas se cierran en Fase 2; entre Fase 1 y 2 mantener reglas de solo lectura o denegar escritura externa lo antes posible — ventana de riesgo mínima, aceptada por ser proyecto privado).

### Cambios en el código

* Raíz `build.gradle.kts` y `app/build.gradle.kts`: el plugin `com.google.gms.google-services` **ya esta aplicado** (se activo en el commit `8bfce38`, 2026-09-28). No queda nada por descomentar.
* No hay cambios de dependencias en esta fase: firebase-bom 33.1.2 + firestore ya estan.
* README.md: el paso 4 de compilacion ya no es "(Opcional)" ni menciona descomentar el plugin.

### Procedimiento de verificación

1. `./gradlew assembleDebug` compila sin errores con el plugin activo.
2. Instalar en el Moto G22, crear datos de prueba, verificar en Ajustes > Sincronizacion: chip "Disponible", conteo "Pendientes por subir" baja tras "Sincronizar ahora" **y la tarjeta no muestra ninguna linea de error**.
3. Verificar en consola de Firestore las colecciones: `clientes`, `planes`, `pagos`, `configuraciones_medida`, `medida_registros`, `medida_valores`. Cada documento debe verse como `{ propietarioUid, versionEsquema, datos: {...} }`, no como la entidad plana.
4. Activar modo avion, crear un registro, reconectar: la subida automatica debe ocurrir (callback de conectividad o `SyncWorker` cada 3 h).

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

El archivo autoritativo de esta fase es `firestore.rules` en la raíz del repo (versionado junto al
código que depende de él). El esqueleto de reglas es:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    function esPropio() {
      return request.auth != null
        && resource.data.propietarioUid == request.auth.uid;
    }
    function escribeComoPropietario() {
      return request.auth != null
        && request.resource.data.propietarioUid == request.auth.uid;
    }
    match /clientes/{clienteId} {
      allow read, delete: if esPropio();
      allow create, update: if escribeComoPropietario();
    }
    // ... una por cada coleccion de negocio ...
    match /{coleccion}/{doc} { allow read, write: if false; }
  }
}
```

En la consola, además de publicarlas en Firestore Database > Reglas, hay que **habilitar el método de
acceso anónimo** (Authentication > Sign-in method > Anonymous > Habilitar). Si no se habilita,
`signInAnonymously()` falla y no se sube nada.

* El login visible de la app NO cambia: sigue siendo el PIN local (ADR #2 intacta). Anonymous Auth es credencial técnica invisible para el usuario; esto se deja explícito en el ADR #7 para evitar contradicciones.

### Problema conocido: pérdida del UID al reinstalar

Al reinstalar/borrar datos, Android genera un UID anónimo nuevo y el backup viejo queda inaccesible para la app. **Mitigación elegida (sin servidor):** migración asistida desde la consola de Firebase — el propietario copia los documentos del UID antiguo al nuevo usando la consola (procedimiento documentado paso a paso dentro de esta fase). Alternativas descartadas: tokens personalizados (requieren backend), cuenta email/contraseña embebida (mismo problema de persistencia local y superficie extra de ataque).

### Cambios en el código

* `app/build.gradle.kts`: `implementation(libs.firebase.auth)` (alias en `libs.versions.toml`, mismo BOM; no se usa el artifact `-ktx` porque Firebase Fusion lo integro en el principal y `await()` ya viene de `kotlinx-coroutines-play-services`, que estaba en el proyecto).
* Nuevo `CloudAuthManager` (patrón `AdminAuthManager`): garantiza sesión anónima activa y expone el UID actual.
* `MaryFitnessApplication`: pide la sesión anónima al arrancar, antes de que dispare el callback de conectividad, para que la primera subida de la sesión no se pierda.
* `FirestoreSyncManager`: recibe el `CloudAuthManager` por constructor y exige sesión válida antes de subir; si no la consigue, no escribe nada y deja el motivo en `ultimoError`.
* `FirestoreSyncManager.subirTodo()`: **no cambia de firma**. El sobre `DocumentoRespaldo` (nuevo, en el mismo paquete) se aplica en el lambda del uploader, en el borde con Firestore. Así las entidades de Room quedan intactas y los cinco tests existentes siguen valiendo tal cual.
* `firestore.rules` (nuevo, en la raíz del repo): reglas con funciones `esPropio()` / `esEscribeComoPropietario()`, una regla por colección de negocio y un `match /{coleccion}/{doc} { allow read, write: if false; }` final que cierra cualquier colección inesperada. Se pegan en la consola (Firestore Database > Reglas > Publicar).
* `AjustesSincronizacionScreen` / `SincronizacionViewModel`: `SincronizacionUiState` gana `error`, alimentado por un colector de `syncManager.ultimoError`, y la tarjeta muestra la línea `⚠️ ...`. `refrescar()` paso a usar `copy` con `MutableStateFlow.update` en lugar de reconstruir el estado, porque al reconstruirlo borraba `error` y `sincronizando`.
* Tests nuevos: `DocumentoRespaldoTest` (uid en la raíz del documento, entidad intacta, versión de esquema por defecto). `FirestoreSyncManagerTest` solo cambio en la construcción del `FirestoreSyncManager` (nuevo parámetro).

### Criterios de aceptación

* [x] Petición sin sesión anónima es rechazada por las reglas (estructura de `firestore.rules` verificada; la comprobación en vivo requiere el proyecto recreado).
* [x] La app sincroniza igual que en Fase 1: los 5 tests de `FirestoreSyncManagerTest` siguen en verde sin cambiar su lógica, y `DocumentoRespaldoTest` cubre el sobre.
* [ ] Documento en consola contiene `propietarioUid` correcto (requiere el proyecto recreado).
* [x] El fallo de sincronización es visible para el propietario: `ultimoError` + línea `⚠️` en Ajustes > Sincronización, en vez de solo un `Log.w`.

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
