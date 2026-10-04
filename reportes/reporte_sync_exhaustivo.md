# REPORTE EXHAUSTIVO - Sistema de Sincronización de Datos
**Repositorio:** C:\Users\csehuanes\Downloads\mary-fitness (Android/Kotlin)  
**Fecha de análisis:** 2026-10-02  
**Modo:** Investigación exhaustiva (sin modificaciones)

## RESUMEN EJECUTIVO

El repositorio implementa un sistema de **backup/copia de seguridad LOCAL → NUBE** con las siguientes características clave:

- **Arquitectura:** Offline-first. Room es la **FUENTE ÚNICA DE VERDAD**. Las lecturas de negocio NUNCA consultan la nube.
- **Direccionalidad:** Upload-only (unidireccional). **NO existe restauración (download/restore) implementada**. La restauración asistida está planificada (Fase 3 en `documentacion/reportes/005-plan-mejoras-firebase.md`).
- **Proveedor de nube (backup estructurado):** **Firebase Firestore** para respaldo de entidades de negocio.
- **Proveedor de nube (archivos multimedia):** **Cloudinary** para subida de archivos (fotos, videos, GIFs) - implementación independiente del backup de Room.
- **Orquestación:** `ConnectivityManager.NetworkCallback` (tiempo real) + `WorkManager` periódico (fallback/reintentos) cada **3 horas** con `NetworkType.CONNECTED`.
- **Mecanismo de tracking:** Bandera `pendienteSync: Boolean` por fila en entidades Room. Solo filas con cambios pendientes se suben.

**ESTADO GENERAL:**
- **Implementado:** Backup upload-only a Firestore (6 colecciones), infraestructura de sync (manager, worker, scheduler, UI de estado), subida de archivos a Cloudinary, tests unitarios completos.
- **NO Implementado:** Download/restore desde Firestore (restauración asistida), Auth anónima Firebase (Fase 2 planeada), Firebase Storage (Fase 4 planeada), migración asistida de UID (Fase 2 planeada).
- **STUB/TODO/PLACEHOLDER:** **No existen** marcadores `TODO`, `FIXME`, `"not implemented"`, `throw UnsupportedOperationException`, métodos vacíos o stubs relacionados con sync/backup en el código fuente.

---

## 1. CREACIÓN DE COPIA DE SEGURIDAD (BACKUP)

### 1.1 Clases principales de backup

#### `app/src/main/java/com/maryfitness/app/sync/FirestoreSyncManager.kt` (Líneas 1-153)
**Clase:** `FirestoreSyncManager`  
**Package:** `com.maryfitness.app.sync`  
**Rol:** Gestor principal de sincronización upload-only a Firestore.

**Atributos relevantes:**
- `private val connectivityManager: ConnectivityManager` (líneas 50-52) - Lazy
- `private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)` (línea 54)
- `private var callbackRegistrado = false` (línea 55)
- `private val mutexSincronizacion = Mutex()` (línea 61) - Protección contra concurrencia
- `private const val TAG = "FirestoreSync"` (línea 20)

**Métodos relevantes (con líneas):**

| Método | Visibilidad/Tipo | Líneas | Descripción |
|---|---|---|---|
| `firestoreDisponible()` | `private fun` | 63-69 | Verifica si `FirebaseApp.getInstance()` existe. Retorna `true` si no lanza `IllegalStateException`, `false` en caso contrario. |
| `firestoreDisponibleActual()` | `fun` | 72 | Expone disponibilidad para UI/workers. |
| `contarPendientes()` | `suspend fun` | 75-81 | Suma pendientes de TODAS las colecciones: `clienteDao().obtenerPendientesSync().size + planDao()... + pagoDao()... + medidaDao().obtenerConfiguracionesPendientesSync().size + medidaDao().obtenerRegistrosPendientesSync().size + medidaDao().obtenerValoresPendientesSync().size` |
| `sincronizarAhora()` | `suspend fun` | 84-86 | Wrapper público para disparar sincronización manual (UI y workers). |
| `iniciarObservacionConectividad()` | `fun` | 88-99 | Registra `NetworkCallback` para `NET_CAPABILITY_INTERNET`. En `onAvailable(network: Network)` lanza `scope.launch { sincronizarPendientes() }`. Solo registra si `firestoreDisponible()` y `!callbackRegistrado`. |
| `subirTodo(uploader: suspend (coleccion: String, docId: String, data: Any) -> Unit)` | `internal suspend fun` | 106-131 | **Núcleo del backup.** Itera en **orden de dependencia**: clientes (107-110) → planes (111-114) → pagos (115-118) → configuraciones_medida (119-122) → medida_registros (123-126) → medida_valores (127-130). Por cada fila pendiente: llama `uploader(coleccion, entidad.id.toString(), entidad)`, luego marca `entidad.copy(pendienteSync = false)` y actualiza DAO. Limpia bandera **inmediatamente tras éxito** por fila (soporta fallo a mitad de lote). |
| `sincronizarPendientes()` | `private suspend fun` | 139-151 | Adquiere `mutexSincronizacion.tryLock()` (140). Si falla → retorna (descarta disparo redundante). Obtiene `FirebaseFirestore.getInstance()` dentro del try (142). Llama `subirTodo` con lambda que hace `firestore.collection(coleccion).document(docId).set(data).await()` (143-145). Si exitoso completo → `appSettingsManager.setUltimaSincronizacionExitosaMillis(System.currentTimeMillis())` (146). En excepción → log `Log.w(TAG, "...", e)` (148). En `finally` → `mutexSincronizacion.unlock()` (150). |

**KDoc relevante (explicita diseño):**
- Líneas 22-44: Modelo **upload-only**. Room fuente única de verdad. Firestore NO es sync multi-dispositivo. Restauración planificada fase 3 (líneas 30-32).
- Líneas 34-39: Backup cubre **TODAS** entidades negocio. **Incluye clientes OCULTOS** (soft-delete). **NO respalda cache de ejercicios externos**. Purga física local NO se propaga (historial completo retenido en Firestore).
- Líneas 57-61, 134-138: Explican mutex + tryLock para evitar subidas paralelas.

**Estado:** IMPLEMENTADO (upload-only). Sin TODO/FIXME/stubs.

### 1.2 Entidades con tracking `pendienteSync`

| Entidad | DAO | Query pendientes | Líneas DAO |
|---|---|---|---|
| `ClienteEntity` | `app/src/main/java/com/maryfitness/app/data/local/dao/ClienteDao.kt` | `@Query("SELECT * FROM clientes WHERE pendienteSync = 1") suspend fun obtenerPendientesSync(): List<ClienteEntity>` | 26-29. Comentario soft-delete: 31-33. |
| `PlanEntity` | `app/src/main/java/com/maryfitness/app/data/local/dao/PlanDao.kt` | `@Query("SELECT * FROM planes WHERE pendienteSync = 1") suspend fun obtenerPendientesSync(): List<PlanEntity>` | 24-26 |
| `PagoEntity` | `app/src/main/java/com/maryfitness/app/data/local/dao/PagoDao.kt` | `@Query("SELECT * FROM pagos WHERE pendienteSync = 1") suspend fun obtenerPendientesSync(): List<PagoEntity>` | 24-26 |
| `ConfiguracionMedidaEntity` | `app/src/main/java/com/maryfitness/app/data/local/dao/MedidaDao.kt` | `@Query("SELECT * FROM configuraciones_medida WHERE pendienteSync = 1") suspend fun obtenerConfiguracionesPendientesSync(): List<ConfiguracionMedidaEntity>` | 36-38 |
| `MedidaRegistroEntity` | `app/src/main/java/com/maryfitness/app/data/local/dao/MedidaDao.kt` | `@Query("SELECT * FROM medida_registros WHERE pendienteSync = 1") suspend fun obtenerRegistrosPendientesSync(): List<MedidaRegistroEntity>` | 42-44 |
| `MedidaValorEntity` | `app/src/main/java/com/maryfitness/app/data/local/dao/MedidaDao.kt` | `@Query("SELECT * FROM medida_valores WHERE pendienteSync = 1") suspend fun obtenerValoresPendientesSync(): List<MedidaValorEntity>` | 48-50 |

**Mapeo colecciones Firestore (orden de dependencia):**
1. `clientes` (docId = `cliente.id.toString()`) — FirestoreSyncManager:107-110
2. `planes` (docId = `plan.id.toString()`) — 111-114
3. `pagos` (docId = `pago.id.toString()`) — 115-118
4. `configuraciones_medida` (docId = `config.id.toString()`) — 119-122
5. `medida_registros` (docId = `registro.id.toString()`) — 123-126
6. `medida_valores` (docId = `valor.id.toString()`) — 127-130

### 1.3 Otros mecanismos "export/backup" (no nube)
- **`NotificacionLogRepository.exportar()`** (`data/repository/NotificacionLogRepository.kt:82-156`): Exporta log de notificaciones a archivos CSV/JSON en `filesDir/exportes/` para compartir con soporte. **No es backup a nube** - exportación local.
- **`AppDatabase.exportSchema = false`** (`data/local/AppDatabase.kt:37`).

**Conclusión Sección 1:** Implementación completa para upload-only. Cobertura total entidades negocio. Diseño robusto ante fallos (limpieza por fila post-éxito, mutex, degradación elegante si Firebase no configurado).

---

## 2. SUBIR A LA NUBE (UPLOAD)

### 2.1 Firebase Firestore (upload estructurado)
**Triggers:**
1. **Tiempo real:** `ConnectivityManager.NetworkCallback.onAvailable()` → `sincronizarPendientes()` (FirestoreSyncManager:94-99). Registrado por `iniciarObservacionConectividad()` llamado en `MaryFitnessApplication.onCreate()` (línea 18).
2. **Periódico/reintentos:** `SyncWorker` cada 3h con `NetworkType.CONNECTED` → `sincronizarAhora()` (WorkScheduler:93-106, SyncWorker:18-27).
3. **Manual (UI):** `SincronizacionViewModel.sincronizarAhora()` (AjustesViewModels.kt:94-101) → `syncManager.sincronizarAhora()`. Botón "Sincronizar ahora" en `AjustesSincronizacionScreen.kt:63-69`.

**Flujo upload Firestore:** Ver Sección 8 (flujo end-to-end).

### 2.2 Cloudinary (upload de archivos multimedia)
Implementación **independiente** de Firestore (no backup BD Room).

| Componente | Ruta | Líneas | Descripción |
|---|---|---|---|
| `CloudinaryManager` | `data/cloudinary/CloudinaryManager.kt` | 1-35 | `init(context, cloudName)`: inicializa MediaManager una vez. Si `cloudName.isBlank()` → log warning y NO inicializa (degradación). `isInitialized()`, `reset()`. Llamado en `MaryFitnessApplication.onCreate()` línea 16: `CloudinaryManager.init(this, BuildConfig.CLOUDINARY_CLOUD_NAME)`. |
| `CloudinaryRepository` | `data/cloudinary/CloudinaryRepository.kt` | 1-61 | `suspend fun uploadMedia(fileUri: Uri, uploadPreset: String = "mobile_app_preset"): String` envuelve callbacks Cloudinary en `suspendCoroutine`. Retorna `secure_url`. Usa `resource_type="auto"`, `unsigned(uploadPreset)`. Cancelación: `invokeOnCancellation { MediaManager.get().cancelRequest(requestId) }`. Errores lanzan `Exception("Cloudinary Error: ${error.description}")`. |
| `CloudinaryViewModel` | `data/cloudinary/CloudinaryViewModel.kt` | 1-42 | Gestiona estado con `UploadUiState` (Idle/Loading/Success/Error). ViewModelScope para corutinas. |
| `UploadScreen` | `data/cloudinary/UploadScreen.kt` | 1-61 | Pantalla Compose ejemplo para subir archivo seleccionado. Muestra estados UI. |

**Búsqueda términos upload/subir:** Coincide con Cloudinary y `subirTodo` en FirestoreSyncManager. **NO hay referencias** a Drive, Firebase Storage, S3 (solo Cloudinary + Firestore). Firebase Storage aparece **planeado** Fase 4 (plan 005).

**Conclusión Sección 2:** Upload implementado (Firestore estructurado + Cloudinary multimedia). Múltiples triggers para robustez.

---

## 3. DESCARGAR DESDE LA NUBE (DOWNLOAD/RESTORE)

### 3.1 Estado actual: NO IMPLEMENTADO
**Evidencias:**

1. **FirestoreSyncManager.kt:30-32 (KDoc):** *"La restauracion asistida desde la nube (instalacion limpia con Room vacio) esta planificada en documentacion/reportes/005-plan-mejoras-firebase.md fase 3; mientras esa fase no exista, la subida es unicamente local -> nube."*
2. **Búsqueda exhaustiva términos restore/restaurar/download/descargar/fetch/pull:**
- `ClienteRepository.restaurarCliente(clienteId: Long)` (`data/repository/ClienteRepository.kt:292`) — **NO es restauración desde nube.** Es **restauración de soft-delete local** (desocultar cliente): marca `oculto = false`, actualiza timestamps, programa alarmas, marca `pendienteSync = true`. Operación inversa local.
- `NotificationGrouper.kt:179` — Comentario conceptual ("tras restaurar un backup..."), no código.
- `ActualizacionManager.descargarApkRemoto()` (`data/actualizacion/ActualizacionManager.kt:46,70,74,98`) — Descarga APKs para actualizaciones de app, **no datos de negocio/backup**.

**Conclusión:** **NO existe implementación de download/restore desde Firestore**. Únicamente upload. Restauración desde nube **planificada** (Fase 3), **no implementada** (sin métodos, sin UI para restore desde nube, sin lógica lectura Firestore→Room).

### 3.2 WorkManager/JobScheduler/periodic (orquestación)
| Componente | Ruta | Líneas | Propósito |
|---|---|---|---|
| `SyncWorker` | `workers/SyncWorker.kt` | 1-28 | Worker WorkManager. `doWork()`: verifica `manager.firestoreDisponibleActual()` → si false retorna `Result.success()` (silencioso). Si true → `manager.sincronizarAhora()`. Retorna `Result.success()`. Comentario: sincronizarAhora degrada fallos a log; siguiente ciclo reintenta pendientes. |
| `WorkScheduler.programarSincronizacionPeriodica()` | `workers/WorkScheduler.kt` | 93-106 | Programa `PeriodicWorkRequestBuilder<SyncWorker>(3, TimeUnit.HOURS)` con `Constraints(NetworkType.CONNECTED)`. Usa `ExistingPeriodicWorkPolicy.KEEP`. Nombre: `NOMBRE_SYNC_PERIODICO = "sync_backup_maryfitness"` (línea 20). |
| `WorkScheduler.programarChequeoDiario()` | `workers/WorkScheduler.kt` | 22-34 | Periódico 8h para alarmas (no backup). `PERIODICIDAD_RECONCILIACION_HORAS = 8L` (83). |
| `DailyCheckWorker` | `workers/DailyCheckWorker.kt` | 1-187 | Reconciliación alarmas/notificaciones (orígenes PERIODICO/MANUAL/RESTAURACION). No relacionado a sync. |

**Periodicidad sync backup:** 3 horas, requiere red conectada. Además NetworkCallback tiempo real.

---

## 4. ORQUESTACIÓN / ESTADO

### 4.1 Máquinas de estado / Enums
**Búsqueda:** No existen enums `EstadoSincronizacion`, `SyncState`, `SyncStatus` en código.

**Estados vía UI State:**
- `SincronizacionUiState` (`ui/ajustes/AjustesViewModels.kt:65-70`):
```kotlin
data class SincronizacionUiState(
    val disponible: Boolean = false,
    val pendientes: Int = 0,
    val ultimaSincronizacionMillis: Long? = null,
    val sincronizando: Boolean = false
)
```
- `UploadUiState` (`data/cloudinary/CloudinaryViewModel.kt:37-42`): `Idle`, `Loading`, `Success(val url)`, `Error(val message)` (sealed interface).

**Conclusión:** Sin FSM compleja. Estado simple basado en flags/flows.

### 4.2 Componentes (Repositorios, ViewModels, UI)
| Componente | Ruta | Líneas | Rol |
|---|---|---|---|
| `FirestoreSyncManager` | `sync/FirestoreSyncManager.kt` | 1-153 | Core sync (upload). Singleton vía ServiceLocator. |
| `SyncWorker` | `workers/SyncWorker.kt` | 1-28 | WorkManager periódico. |
| `WorkScheduler` | `workers/WorkScheduler.kt` | 1-107 | Programación workers (incluye sync periódico). |
| `SincronizacionViewModel` | `ui/ajustes/AjustesViewModels.kt:72-102` | ViewModel pantalla sincronización. Lee estado via `syncManager` + `settings`. Dispara `sincronizarAhora()`. En init hace `refrescar()`. |
| `AjustesSincronizacionScreen` | `ui/ajustes/AjustesSincronizacionScreen.kt` | 1-71 | UI Compose: chip "Disponible"/"No configurada" (líneas 38-41), pendientes (43), última sync (45-49), texto explicativo (51-55), botón "Sincronizar ahora" (63-69), `CircularProgressIndicator` mientras `sincronizando` (60-62). |

### 4.3 Notificaciones relacionadas a sync
Sin notificaciones de usuario para sync. Solo logs con `TAG="FirestoreSync"` (FirestoreSyncManager.kt:20). Fallos logueados con `Log.w` (línea 148). Éxito silencioso (actualiza timestamp).

### 4.4 Scheduling (inicio aplicación)
`MaryFitnessApplication.onCreate()` (líneas 13-31):
```kotlin
CloudinaryManager.init(this, BuildConfig.CLOUDINARY_CLOUD_NAME)  // línea 16
ServiceLocator.notificationScheduler(this)                        // 17
ServiceLocator.syncManager(this).iniciarObservacionConectividad()  // 18 (NetworkCallback)
WorkScheduler.programarChequeoDiario(this)                         // 19
WorkScheduler.programarSincronizacionPeriodica(this)              // 21 (SyncWorker 3h)
WorkScheduler.ejecutarChequeoInmediato(this)                      // 25
```
`ServiceLocator.syncManager(context)` (ServiceLocator.kt:116-123) retorna singleton `FirestoreSyncManager`.

### 4.5 Concurrencia
- `private val mutexSincronizacion = Mutex()` (FirestoreSyncManager.kt:61)
- `sincronizarPendientes()` usa `mutexSincronizacion.tryLock()` (línea 140): si lock falla → retorna inmediatamente (descarta disparo redundante). `unlock()` en `finally` (150).

---

## 5. CONFIGURACIÓN (CLAVES/CONSTANTES)

### 5.1 Archivos de propiedades
| Archivo | Presente | Notas |
|---|---|---|
| `local.properties` | Sí (gitignored) | Contiene configuración local. **Nombres de claves detectados (solo nombres, SIN valores):** `ANDROID_KEYSTORE_PASSWORD`, `ANDROID_KEY_ALIAS`, `ANDROID_KEY_PASSWORD`, `ASCENDAPI_KEY`, `ASCENDAPI_HOST`, `CLOUDINARY_CLOUD_NAME`. Valores presentes (no vacíos). |
| `local.properties.example` | Sí | Ejemplo de estructura con placeholders. |
| `production.properties` | Sí | Config producción. **Nombres de claves:** `ASCENDAPI_KEY`, `ASCENDAPI_HOST`, `CLOUDINARY_CLOUD_NAME`. |
| `gradle.properties` | Sí | Props Gradle. Sin claves cloud/sync evidentes. |
| `app/google-services.json` | Sí | Configuración Firebase (proyecto Android). Contiene `project_id`, `api_key`, `google_app_id`, `mobilesdk_app_id`, etc. **Presente en repo.** (Verificar política .gitignore - típicamente google-services.json no se commitea en algunos proyectos; presente aquí). |
| `.gitignore` | Sí | Reglas de ignore del repo. |

**Nota de seguridad:** Conforme solicitud, **SOLO se reportan nombres de claves y si el valor está presente/vacío/masked. NO se imprimen valores de secretos/API keys.**

### 5.2 Firebase / Google Services
- **`google-services.json`**: Presente en `app/`.
- **Plugin `com.google.gms.google-services`**: **NO habilitado** en `app/build.gradle.kts`. En `build.gradle.kts` raíz: línea comentada `// id("com.google.gms.google-services") version "4.4.2" apply false`. En `app/build.gradle.kts` no aparece aplicación del plugin. Esto significa Firebase no se inicializa vía plugin en estado actual → `FirebaseApp.getInstance()` lanzará `IllegalStateException` → `firestoreDisponible()` retorna `false` (comportamiento degradado correcto: app 100% offline).
- **Dependencias Firebase (presentes en `app/build.gradle.kts:150-152`):** `implementation(platform(libs.firebase.bom))`, `implementation(libs.firebase.analytics)`, `implementation(libs.firebase.firestore)`.
- **Dependencias Firebase FALTANTES (planificadas):** `firebase-auth-ktx` (Fase 2), `firebase-storage-ktx` (Fase 4). No presentes actualmente.
- **Cloudinary:** `BuildConfig.CLOUDINARY_CLOUD_NAME` usado en `MaryFitnessApplication.kt:16`. Generado desde propiedades (local/production).

**Conclusión configuración:** Infraestructura Firebase presente (google-services.json + dependencias Firestore) pero **plugin google-services NO aplicado** → sync Firestore inactivo por config (comportamiento esperado offline-first). Listo para activar según Fase 1 plan 005.

### 5.3 Constantes relevantes
- `FirestoreSyncManager.TAG = "FirestoreSync"` (20)
- `WorkScheduler.NOMBRE_SYNC_PERIODICO = "sync_backup_maryfitness"` (20)
- `WorkScheduler.NOMBRE_TRABAJO_DIARIO = "chequeo_diario_maryfitness"` (17)
- `WorkScheduler.PERIODICIDAD_RECONCILIACION_HORAS = 8L` (83)
- Periodicidad SyncWorker: `3, TimeUnit.HOURS` (WorkScheduler.kt:94)

---

## 6. TESTS RELACIONADOS CON SYNC/BACKUP

### `app/src/test/java/com/maryfitness/app/sync/FirestoreSyncManagerTest.kt` (1-173)
**Suite completa de unit tests** para `FirestoreSyncManager`. Usa **MockK** (mocks DAOs/DB). No depende de Firebase real (inyecta uploader de prueba vía lambda). 5 tests.

| Test | Líneas | Descripción/Verificación |
|---|---|---|
| ``sube las seis colecciones en orden de dependencia`` | 101-115 | Verifica orden exacto: `["clientes","planes","pagos","configuraciones_medida","medida_registros","medida_valores"]`. Verifica docIds `"1"`, `"3"`, etc. |
| ``incluye clientes ocultos para que su soft delete llegue al backup`` | 118-129 | Confirma cliente con `oculto = true` se incluye/sube. Verifica `clienteDao.actualizar(clienteSubido.copy(pendienteSync = false))` llamado. |
| ``limpia la bandera de cada fila tras subirla`` | 132-143 | Verifica que **TODAS** las 6 entidades actualizan `pendienteSync=false` tras subida exitosa (clientes, planes, pagos, configs, registros, valores). |
| ``un fallo a mitad del lote conserva la bandera de lo no subido`` | 146-164 | **Caso crítico.** Simula excepción al subir `"pagos"`. Verifica: clientes/planes YA subidos quedan limpios (`pendienteSync=false`); pagos y medidas posteriores **NO** actualizados (conservan pendienteSync). Resultado `isFailure`. Demuestra comportamiento correcto para reintentar lo faltante. |
| ``cuenta los pendientes de todas las colecciones`` | 167-172 | Verifica `manager.contarPendientes()` retorna `6` con un pendiente por colección. |

**Cobertura:** Excelente. Prueba lógica core, orden de dependencia, inclusión de ocultos, idempotencia/parcialidad ante fallos.

**Otros tests sync/backup:** Búsqueda `sync|backup` en tests → **solo** este archivo. No hay tests para restore (no implementado), ni para Cloudinary, ni para Workers.

**Conclusión tests:** Sólidos, bien diseñados, validan comportamiento correcto de upload-only.

---

## 7. DOCUMENTACIÓN RELACIONADA CON SINCRONIZACIÓN/BACKUP

| Archivo | Ruta | Relación con sync/backup |
|---|---|---|
| **ADR #7 + Decisiones Técnicas** | `documentacion/decisiones-tecnicas.md` (líneas ~45-70) | Expone evolución rol Firebase. Define: Room única fuente verdad, prohibida sync multi-dispositivo, **restauración asistida SOLO si Room vacío** (instalación limpia), Auth anónima técnica (Fase 2), Firebase Storage (Fase 4), Crashlytics (Fase 5). Referencia plan 005. |
| **Plan 005 - Mejoras Firebase** | `documentacion/reportes/005-plan-mejoras-firebase.md` | **Plan maestro por fases (Fase 1-5).** Detalla: activar backup existente (Fase 1), Auth anónima + reglas (Fase 2), **Restauración asistida desde nube (Fase 3)**, Firebase Storage (Fase 4), Crashlytics (Fase 5). Incluye criterios aceptación, riesgos, cambios código. |
| **ADR Notificaciones** | `documentacion/ADR-notificaciones.md` | No relacionado directamente. |
| **Especificaciones Negocio** | `documentacion/especificaciones-negocio.md` | Sin menciones explícitas backup/sync (según revisión). |
| **Reglas Codificación** | `documentacion/reglas-codificacion.md` | Sin menciones explícitas sync/backup. |
| **Reportes históricos (001-004)** | `documentacion/reportes/` | Fotografía de momentos históricos. ADR #7 prima sobre contradicciones (según reglas precedencia). |

**Conclusión documentación:** Muy completa. Expone claramente estado actual (upload-only) y roadmap (restore + auth + storage).

---

## 8. FLUJO END-TO-END ACTUAL (IMPLEMENTADO)

### FASE 1: Detección de cambios locales
1. Usuario crea/modifica/elimina (soft-delete) entidades negocio (Cliente, Plan, Pago, Configuraciones/Registros/Valores Medida).
2. Operaciones marcan entidades con `pendienteSync = true` (patrón Room + flags).

### FASE 2: Subida automática al detectar conectividad (tiempo real)
1. App arranca → `MaryFitnessApplication.onCreate()` llama `syncManager.iniciarObservacionConectividad()`.
2. `FirestoreSyncManager` registra `NetworkCallback` para `NET_CAPABILITY_INTERNET`.
3. Red disponible (`onAvailable`) → callback lanza `scope.launch { sincronizarPendientes() }`.
4. `sincronizarPendientes()` hace `tryLock()` (mutex). Si falla → retorna (descarta redundante).
5. Obtiene `FirebaseFirestore.getInstance()`. Si Firebase no configurado → excepción capturada → `Log.w(TAG, ...)` → unlock → termina (app 100% offline-first). **No bloquea app**.
6. Llama `subirTodo()` con uploader que hace `firestore.collection(coleccion).document(docId).set(data).await()`.
7. **Orden subida:** clientes → planes → pagos → configuraciones_medida → medida_registros → medida_valores.
8. **Por fila:** sube con docId = id.toString(). Tras éxito → marca `pendienteSync = false` (update DAO). Si falla a mitad → excepción propagada; filas ya subidas limpias, restantes pendientes (correcto para retry).
9. Éxito completo → `appSettingsManager.setUltimaSincronizacionExitosaMillis(System.currentTimeMillis())`.
10. `finally` → unlock mutex.

### FASE 3: Subida periódica/reintentos
1. `WorkScheduler.programarSincronizacionPeriodica()` programa `SyncWorker` cada **3h** con `NetworkType.CONNECTED`, `ExistingPeriodicWorkPolicy.KEEP`.
2. `SyncWorker.doWork()`: si `!firestoreDisponibleActual()` → `Result.success()` (silencioso). Si disponible → `manager.sincronizarAhora()` → `Result.success()`.
3. Corre en background (WorkManager). Garantiza reintentos.

### FASE 4: Subida manual (UI)
1. Usuario abre Ajustes > Sincronización.
2. `SincronizacionViewModel.refrescar()` (init): consulta `disponible`, `pendientes = contarPendientes()`, `ultimaSincronizacionMillis`.
3. UI muestra estado. Pulsar "Sincronizar ahora": setea `sincronizando=true` → `syncManager.sincronizarAhora()` → `refrescar()` → `sincronizando=false`.

### FASE 5: Subida archivos multimedia (Cloudinary) - independiente
1. Usuario selecciona archivo (Uri) vía UI (UploadScreen o flujo con CloudinaryViewModel).
2. `CloudinaryViewModel.uploadFile(uri)`: Loading → `repository.uploadMedia(uri)`.
3. `CloudinaryRepository` sube vía MediaManager (`unsigned(uploadPreset="mobile_app_preset")`, `resource_type="auto"`) → retorna `secure_url`.
4. ViewModel actualiza Success(url) o Error(message).

**NOTA:** **No existe Fase 6 (download/restore)** en implementación actual. Referencias a "restaurar" en código son restore local (soft-delete) o planificadas (docs).

---

## 9. PARTES IMPLEMENTADAS vs STUB/TODO/PLACEHOLDER

### IMPLEMENTADAS (completas y funcionales)
- ✅ `FirestoreSyncManager` completo (upload-only): disponibilidad, conteo pendientes, observación conectividad, `subirTodo` con orden correcto, `sincronizarPendientes` con mutex + tryLock.
- ✅ DAOs con `obtenerPendientesSync()` para 6 colecciones.
- ✅ `SyncWorker` + `WorkScheduler.programarSincronizacionPeriodica()` (3h, constraint CONNECTED).
- ✅ Integración `MaryFitnessApplication` (inicio automático NetworkCallback + workers).
- ✅ `SincronizacionViewModel` + `AjustesSincronizacionScreen` (UI estado + sync manual).
- ✅ Tests exhaustivos FirestoreSyncManager (5 tests, casos edge: fallo mitad lote, clientes ocultos).
- ✅ Infraestructura Cloudinary completa (Manager, Repository, ViewModel, UploadScreen).
- ✅ `ServiceLocator.syncManager()` singleton.
- ✅ `AppSettingsManager.ultimaSincronizacionExitosaMillis` (persistencia timestamp).

### PARCIALMENTE IMPLEMENTADAS / CONFIGURACIÓN PENDIENTE
- ⚠️ **Plugin google-services**: Comentado/no aplicado (build.gradle.kts). Dependencias Firestore presentes. Comportamiento actual correcto (offline-first). Pendiente activar Fase 1 plan 005.
- ⚠️ **Firebase Auth/Storage:** Dependencias NO presentes (`firebase-auth-ktx`, `firebase-storage-ktx`). Planificadas Fase 2 y 4.

### NO IMPLEMENTADAS (planificadas, sin código)
- ❌ **Restauración asistida desde nube (download/restore):** Ningún método lee Firestore→Room. Solo comentarios/docs (Fase 3 plan 005). UI sin botón funcional de restore desde nube.
- ❌ **Auth anónima + propietarioUid:** Sin `CloudAuthManager`, sin añadir `propietarioUid` a documentos, sin filtrado por UID (Fase 2).
- ❌ **Firebase Storage para binarios:** Solo Cloudinary existe (Fase 4).
- ❌ **Versionado esquema (`versionEsquema`) en documentos:** Mencionado Fase 3, no implementado.
- ❌ **Wrappers con propietarioUid en subida:** FirestoreSyncManager sube entidades tal cual (requiere cambio Fase 2).

### STUBS/TODO/FIXME/PLACEHOLDERS
**Búsqueda exhaustiva:**
```bash
grep -rn "TODO\|FIXME\|not implemented\|UnsupportedOperationException\|stub\|placeholder" 
  app/src/main/java/com/maryfitness/app/sync 
  app/src/main/java/com/maryfitness/app/workers 
  app/src/main/java/com/maryfitness/app/data/cloudinary
```
**Resultado:** 0 coincidencias.

**Búsqueda global app/src (sync/backup):** 0 marcadores relacionados con sincronización/backup.

**Conclusión:** **No existen** stubs, TODO, FIXME, "not implemented", `throw UnsupportedOperationException`, ni métodos vacíos en código de sync/backup/cloudinary. Funcionalidad futura **debidamente documentada** (ADR #7 + plan 005), no dejada como placeholder.

---

## 10. RESUMEN POR CATEGORÍA SOLICITADA

| Categoría | Estado | Hallazgos clave |
|---|---|---|
| **1. Backup (creación)** | ✅ IMPLEMENTADO | `subirTodo()` sube 6 colecciones en orden, incluye clientes ocultos, limpia banderas post-éxito, maneja fallo a mitad correctamente. Cobertura completa entidades negocio. Cache ejercicios NO respaldado. |
| **2. Upload (subir a nube)** | ✅ IMPLEMENTADO | Firebase Firestore (estructurado) + Cloudinary (multimedia). Triggers: NetworkCallback (realtime) + SyncWorker 3h (periódico) + manual UI. Orden dependencia correcto. |
| **3. Download/Restore** | ❌ NO IMPLEMENTADO | Solo planificado Fase 3 (plan 005). Código solo restore local (soft-delete). Sin lectura Firestore→Room. |
| **4. Orquestación/Estado** | ✅ IMPLEMENTADO | Mutex + tryLock, NetworkCallback + WorkManager, `SincronizacionUiState` (disponible/pendientes/ultimaSync/sincronizando). Sin FSM compleja. |
| **5. Configuración** | ⚠️ PARCIAL | `google-services.json` presente, dependencias Firestore presentes, **plugin google-services NO aplicado** (pendiente activar Fase 1). `CLOUDINARY_CLOUD_NAME` vía BuildConfig. Firebase Auth/Storage no presentes (Fase 2/4). |
| **6. Tests sync/backup** | ✅ IMPLEMENTADOS | 5 tests unitarios completos en `FirestoreSyncManagerTest.kt`. Cubren orden, ocultos, limpieza banderas, fallo parcial, conteo pendientes. |
| **7. Documentación** | ✅ COMPLETA | ADR #7 + 005-plan-mejoras-firebase.md muy detallados (estado actual + roadmap). |

---

## 11. CONCLUSIÓN

El sistema de **sincronización de backup** está **sólidamente implementado** para el caso de uso **upload-only local→nube**, con degradación elegante (funciona 100% offline si Firebase no configurado). La **restauración desde nube NO está implementada** - existe únicamente como diseño planificado. **No hay código incompleto (stubs/TODO)** relacionado; toda la funcionalidad futura está documentada en el plan de mejoras por fases.

La implementación demuestra buenas prácticas: protección contra concurrencia (mutex), manejo correcto de fallos parciales (retry de lo pendiente), inclusión explícita de soft-deletes, separación clara entre backup estructurado (Firestore) y upload de archivos (Cloudinary), y tests unitarios exhaustivos.

**Archivos más relevantes:**
1. `app/src/main/java/com/maryfitness/app/sync/FirestoreSyncManager.kt` - Núcleo
2. `app/src/test/java/com/maryfitness/app/sync/FirestoreSyncManagerTest.kt` - Especificación comportamiento
3. `app/src/main/java/com/maryfitness/app/workers/SyncWorker.kt` + `WorkScheduler.kt` - Orquestación
4. `app/src/main/java/com/maryfitness/app/ui/ajustes/AjustesViewModels.kt` + `AjustesSincronizacionScreen.kt` - UI/Estado
5. `documentacion/decisiones-tecnicas.md` (ADR #7) + `documentacion/reportes/005-plan-mejoras-firebase.md` - Arquitectura y roadmap

---

*Reporte generado sin modificar ningún archivo del repositorio. Solo investigación y análisis estructurado.*
