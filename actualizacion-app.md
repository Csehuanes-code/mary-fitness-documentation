# Actualización de la app desde la aplicación

La app puede actualizarse sin Play Store (decisión del propietario, distribución
directa al Moto G22). Al abrir el panel principal se consulta silenciosamente si hay
versión nueva; también existe un botón manual en **Ajustes > Acerca de**. Si hay
actualización, la app descarga el APK desde Firebase Storage y lanza el instalador
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

## Publicación de una versión nueva (procedimiento)

1. Subir `mary-fitness-X.Y.Z.apk` a Firebase Storage (consola > Storage).
2. En los permisos del archivo usar acceso público por enlace (token) y copiar la
   URL de descarga (`...alt=media&token=...`) del APK.
3. Crear/actualizar `latest.json` en el bucket:

   ```json
   {
     "versionName": "0.2.0",
     "versionCode": 2,
     "urlApk": "<URL de descarga directa del APK>",
     "notas": "Resumen corto de cambios (opcional)"
   }
   ```

4. Copiar la URL de descarga de `latest.json` en `local.properties`:

   ```
   MARYFITNESS_UPDATE_INFO_URL=https://firebasestorage.googleapis.com/v0/b/<bucket>/o/latest.json?alt=media&token=<token>
   ```

5. **Incrementar `versionCode`** (y `versionName`) en `app/build.gradle.kts` antes de
   compilar el APK; de lo contrario la app no detectará la actualización.
6. Compilar `assembleRelease` o `assembleDebug` SIEMPRE con la misma llave de firma
   usada por la versión instalada en el dispositivo.

## Notas de seguridad

- El instalador verifica la firma: un APK ajeno nunca reemplazará a la app.
- Las URLs de Firebase Storage incluyen token: revocarlas invalida las descargas ya
  publicadas (regenerar token implica actualizar `urlApk` y `MARYFITNESS_UPDATE_INFO_URL`).
- El APK descargado vive solo en caché; puede borrarse tras instalar o limpiar datos.
