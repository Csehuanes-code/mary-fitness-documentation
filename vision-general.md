## 3. VISIÓN GENERAL DEL PRODUCTO

* **Propósito:** Aplicación móvil nativa para la administración interna de un gimnasio pequeño, enfocada en el registro de clientes, control financiero de planes y traqueo de medidas corporales periódicas.
* **Enfoque de Arquitectura:** Arquitectura Pragmática MVVM (Model-View-ViewModel). Se omiten las capas complejas de Casos de Uso independientes para acelerar el desarrollo. La comunicación será directa: `Compose UI` $\leftrightarrow$ `ViewModel` $\leftrightarrow$ `Repository` $\leftrightarrow$ `Data Sources (Room/Firestore)`.
* **Modelo de Operación:** *Offline-First*. La aplicación debe ser 100% funcional en sótanos o zonas sin cobertura telefónica. La base de datos local siempre responde de inmediato; los cambios se propagan a la nube de manera transparente en background.
* **Formato de Salida:** Compilación en un paquete ejecutable autónomo de Android (**archivo .apk**), instalable mediante almacenamiento local o descarga directa, sin dependencia obligatoria de Google Play Store para su distribución interna.
