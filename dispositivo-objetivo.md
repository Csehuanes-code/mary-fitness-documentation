## ESPECIFICACIONES DEL DISPOSITIVO OBJETIVO

El agente de IA debe optimizar todo el código generado considerando estrictamente las limitaciones de hardware del dispositivo objetivo para evitar problemas de rendimiento (ANR - Android Not Responding) o degradación de la interfaz de usuario.

* **Dispositivo Objetivo:** Motorola Moto G22.
* **Sistema Operativo:** Android 12 / 13 (API de destino recomendada: API 33+).
* **Procesador:** MediaTek Helio G37 (Octa-core, gama de entrada).
* **Memoria RAM:** 4 GB.
* **Almacenamiento Interno:** eMMC 5.1 (Velocidad de lectura/escritura limitada).
* **Restricciones para el Agente de IA:**
1. **Hilos de Ejecución:** Queda estrictamente prohibido realizar operaciones de lectura/escritura en la base de datos o procesamiento de datos pesados en el hilo principal de la interfaz de usuario (`Main Thread`). Se deben delegar obligatoriamente a contextos asíncronos adecuados.
2. **Jerarquía de Vistas:** Diseñar pantallas planas. Evitar la anidación profunda de componentes visuales para no saturar la GPU del procesador Helio G37.
3. **Procesamiento de Archivos:** Cualquier imagen (como fotos de perfil o comprobantes) debe ser comprimida localmente al formato **WebP** y reducir su resolución antes de almacenarse en el almacenamiento interno o enviarse a la nube.
