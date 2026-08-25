# Actualización de la app desde la aplicación

La app puede actualizarse sin Play Store (decisión del propietario, distribución
directa al Moto G22). Al abrir el panel principal se consulta silenciosamente si hay
versión nueva; también existe un botón manual en **Ajustes > Acerca de**. Si hay
actualización, la app descarga el APK desde Cloudinary y lanza el instalador
del sistema, donde el usuario confirma la instalación.

## Flujo técnico

1. `ActualizacionManager.verificar()` descarga `latest.json` y compara su
   `versionCode` con `BuildConfig.VERSION_CODE` (solo avanza si el remoto es mayor).
2. `descargar()` guarda el APK en `cache/actualizaciones/`.
3. La UI lo expone vía `FileProvider` e inicia `ACTION_VIEW`
   (`application/vnd.android.package-archive`). Android exige que el APK nuevo esté
   firmado con la misma llave que el instalado; la primera vez pide habilitar
   "Instalar apps desconocidas" para Mary Fitness.

Sin `MARYFITNESS_UPDATE_INFO_URL` configurada, la función queda deshabilitada y la app
sigue funcionando 100% offline.

## Configuración por ambiente

La app usa archivos `local.properties.<ambiente>` para configurar cada entorno.
Estos archivos **no se versionan** (ver `.gitignore`).

| Archivo                    | Uso                                    |
|----------------------------|----------------------------------------|
| `local.properties.local`   | Desarrollo local (tu máquina)          |
| `local.properties.development` | Entorno de desarrollo compartido    |
| `local.properties.test`    | Entorno de QA / pruebas                |
| `local.properties.production` | Builds de release / producción      |

Al compilar, **copia el archivo correspondiente a `local.properties`** (sin sufijo).
Ejemplo en máquina local:

```bash
cp local.properties.local local.properties
```

En CI/CD, el pipeline copia el archivo del ambiente correspondiente antes de compilar.

Variables requeridas en `local.properties`:

```
# Clave RapidAPI (ejercicios)
ASCEND_API_KEY=...

# Cloudinary (subida de archivos y hosting latest.json)
CLOUDINARY_CLOUD_NAME=tu_cloud_name
CLOUDINARY_UPLOAD_PRESET=mobile_app_preset

# URL del latest.json en Cloudinary (raw upload)
# Formato: https://res.cloudinary.com/<cloud_name>/raw/upload/latest.json
MARYFITNESS_UPDATE_INFO_URL=https://res.cloudinary.com/tu_cloud_name/raw/upload/latest.json
```

## Publicación de una versión nueva (procedimiento)

### 1. Preparar Cloudinary

- Crear un **Upload Preset no firmado** (unsigned) en Cloudinary Console:
  - Settings > Upload > Upload presets > Add upload preset
  - Signing Mode: **Unsigned**
  - Nombre sugerido: `mobile_app_preset` (o con sufijo por ambiente: `_dev`, `_test`, `_prod`)
  - Folder opcional: `mary-fitness/apk/`

- Subir el `latest.json` como **raw** (no image/video):
  ```bash
  curl -X POST "https://api.cloudinary.com/v1_1/<cloud_name>/raw/upload" \
    -F "file=@latest.json" \
    -F "upload_preset=mobile_app_preset" \
    -F "folder=mary-fitness/updates"
  ```
  La respuesta incluye `secure_url`: `https://res.cloudinary.com/<cloud_name>/raw/upload/v123456/mary-fitness/updates/latest.json`

### 2. Subir el APK a Cloudinary

```bash
curl -X POST "https://api.cloudinary.com/v1_1/<cloud_name>/raw/upload" \
  -F "file=@mary-fitness-0.2.0.apk" \
  -F "upload_preset=mobile_app_preset" \
  -F "folder=mary-fitness/apk"
```

Copiar la `secure_url` del APK (ej: `https://res.cloudinary.com/<cloud_name>/raw/upload/v123456/mary-fitness/apk/mary-fitness-0.2.0.apk`).

### 3. Actualizar `latest.json`

Crear/actualizar `latest.json` con la URL del APK:

```json
{
  "versionName": "0.2.0",
  "versionCode": 2,
  "urlApk": "https://res.cloudinary.com/<cloud_name>/raw/upload/v123456/mary-fitness/apk/mary-fitness-0.2.0.apk",
  "notas": "Resumen corto de cambios (opcional)"
}
```

Subir este `latest.json` a Cloudinary (ver paso 1) y copiar su `secure_url`.

### 4. Configurar `MARYFITNESS_UPDATE_INFO_URL`

Pegar la URL del `latest.json` en el `local.properties` del ambiente correspondiente:

```
MARYFITNESS_UPDATE_INFO_URL=https://res.cloudinary.com/<cloud_name>/raw/upload/v123456/mary-fitness/updates/latest.json
```

### 5. Compilar y firmar

- **Incrementar `versionCode`** (y `versionName`) en `app/build.gradle.kts` antes de compilar.
- Compilar `assembleRelease` **SIEMPRE con la misma llave de firma** usada por la versión instalada en el dispositivo.

```bash
./gradlew assembleRelease
```

El APK generado en `app/build/outputs/apk/release/` se instala en el dispositivo y la app detectará la actualización al siguiente arranque.

## Notas de seguridad

- El instalador verifica la firma: un APK ajeno nunca reemplazará a la app.
- Las URLs de Cloudinary son públicas pero no adivinables (incluyen version hash `v123456`).
- Para revocar una versión: eliminar el archivo en Cloudinary Console o regenerar el upload preset.
- El APK descargado vive solo en caché; puede borrarse tras instalar o limpiar datos.

## Flujo en CI/CD (resumen)

```yaml
# Ejemplo conceptual para GitHub Actions / GitLab CI
jobs:
  build:
    steps:
      - name: Setup environment
        run: cp local.properties.${{ env.ENVIRONMENT }} local.properties
      - name: Build release
        run: ./gradlew assembleRelease
      - name: Upload APK to Cloudinary
        run: |
          curl -X POST "https://api.cloudinary.com/v1_1/${CLOUD_NAME}/raw/upload" \
            -F "file=@app/build/outputs/apk/release/app-release.apk" \
            -F "upload_preset=${UPLOAD_PRESET}" \
            -F "folder=mary-fitness/apk"
      - name: Update latest.json
        run: |
          # Generar latest.json con la nueva URL del APK
          # Subir a Cloudinary
          # Actualizar variable MARYFITNESS_UPDATE_INFO_URL en secrets del siguiente build
```
