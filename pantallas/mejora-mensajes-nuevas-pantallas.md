# Propuesta de Optimización de UX Writing y Mensajes de Interfaz
**Documento de Análisis y Mejora sobre `mensajes-nuevas-pantallas.md`**
**Proyecto:** Mary Fitness (Android App)  
**Fecha:** Septiembre 2026  
**Autor:** Especialista UX / Gemini Notebook  

---

## 1. Resumen Ejecutivo y Metodología UX

El presente documento analiza minuciosamente los ~85 mensajes expuestos en el archivo de inventario `mensajes-nuevas-pantallas.md`. Cada texto ha sido evaluado bajo principios fundamentales de UX Mobile y Psicología del Diseño:

1. **Ley de Hick (Reducción de Carga Cognitiva):** Eliminación de jerga técnica interna (p. ej., *"reporte 007 P2"*, *"soft delete"*, *"BCrypt"*, *"Helio G37"*), simplificando las opciones presentadas al usuario para agilizar la toma de decisiones.
2. **Ley de Jakob (Patrones Estándar Predictivos):** Uso de convenciones móviles familiares en Android (labels, modales, toasts, chips y diálogos) para alinearse con los modelos mentales preexistentes del usuario.
3. **Ubicación y Tono de Errores (Proximidad y Claridad):** Sustitución de variables de excepción crudas (`{it.message}`) por mensajes de error orientados a la solución, ubicados en el lugar exacto del conflicto (*inline*) o en modales cuando bloquean el flujo.
4. **Estados Vacíos Accionables (*Empty States*):** Transformación de pantallas en blanco o mensajes neutros en puentes de navegación con llamadas a la acción (*CTA*) claras para no dejar al usuario en un "callejón sin salida".
5. **Psicología de Carga y Espera:** Optimización de la retroalimentación en procesos asíncronos mediante textos dinámicos progresivos e indicadores continuos en lugar de *spinners* o textos estáticos aburridos.
6. **Prevención y Manejo de Errores Destructivos:** Transparencia total sobre acciones irreversibles (borrado en cascada) vs. acciones reversibles (ocultar/deshabilitar), utilizando verbos directos en lugar de etiquetas genéricas como "Aceptar" u "Ok".

---

## 2. Revisión y Mejora Detallada por Pantalla y Módulo

### 1. PANTALLAS COMPLETAMENTE NUEVAS

#### 1.1 Splash — Carga Inicial (`SplashScreen.kt`)

| Texto Actual | Contexto / Ubicación | Texto Propuesto / Mejorado | Justificación UX y Principio Aplicado |
| :--- | :--- | :--- | :--- |
| `MARY FITNESS` | Marca / logo textual | **Mary Fitness** | Formato de marca legible en caja mixta para evitar sensación de grito en pantalla inicial. |
| `Gestión Integral de Gimnasio` | Subtítulo descriptivo | **Gestión Integral de tu Gimnasio** | Tono más cercano y personal. |
| `Verificando estado de PIN y base local...` | Estado mientras `checking == true` | **Iniciando seguridad y datos locales...** | Elimina jerga técnica de programación (*"base local"*, *"PIN"*) por un concepto comprensible para el usuario. |
| `Iniciando...` | Estado cuando `checking == false` | **¡Todo listo! Entrando...** | Genera una transición positiva (psicología de estados de éxito/carga). |
| `Optimizado para Moto G22 · Helio G37 · Offline-First` | Footer técnico | **Modo Offline Activo · Funciona sin internet** | Traduce el dato de hardware/arquitectura técnica (*Helio G37*, *Offline-First*) a un beneficio real para la entrenadora. |
| `Mary Fitness v0.1.5 · Seguridad local BCrypt` | Pie de versión | **Mary Fitness v0.1.5 · Datos Protegidos** | Reemplaza la referencia al algoritmo *BCrypt* por una afirmación de seguridad comprensible. |
| *(CircularProgressIndicator sin texto)* | Indicador visual | *(Línea de progreso con porcentaje aproximado o micro-animación)* | Según la psicología de *spinners*, la retroalimentación visual continua reduce el tiempo de espera percibido. |

---

#### 1.2 Seguridad — PIN Administrador (`AjustesSeguridadScreen.kt`)

| Texto Actual | Contexto / Ubicación | Texto Propuesto / Mejorado | Justificación UX y Principio Aplicado |
| :--- | :--- | :--- | :--- |
| `Seguridad - PIN Administrador` | Título TopBar / Sección | **Seguridad y PIN de Acceso** | Título claro sin guiones innecesarios. |
| `El PIN protege el acceso a todos los datos. No hay recuperación remota (reporte 007 P2).` | Subtítulo explicativo | **Este PIN protege toda la información de la app. Guárdalo bien: por seguridad, no se puede recuperar por internet.** | **Ley de Hick:** Elimina el código interno *"reporte 007 P2"* y explica la consecuencia con lenguaje humano y preventivo. |
| `Cambiar PIN` | Título de GlassCard | **Cambiar PIN Actual** | Verbo explícito de acción. |
| `PIN actual (4-6 dígitos)` | Label de campo | **PIN actual** *(Placeholder: Ingresa de 4 a 6 dígitos)* | Mantiene el label limpio y traslada la restricción al *helper text* o *placeholder*. |
| `PIN nuevo` | Label de campo | **Nuevo PIN** | Sintaxis natural. |
| `Confirmar nuevo PIN` | Label de campo | **Confirmar nuevo PIN** | Claro y conciso. |
| `Cambiar PIN` | Texto de Botón Principal | **Guardar Nuevo PIN** | Verbo de confirmación claro (*Guardar*) para evitar duplicidad con el título de la tarjeta. |
| `Resetear PIN (emergencia)` | Título zona de peligro | **Restablecer PIN de Emergencia** | Término más estandarizado en Android (*Restablecer* vs. *Resetear*). |
| `Borra el PIN actual y obliga a configurar uno nuevo en el próximo inicio. Usa solo si olvidaste el PIN.` | Descripción zona de peligro | **Elimina el PIN actual para definir uno nuevo al reiniciar. Úsalo solo si no recuerdas tu PIN.** | Tono explicativo claro y directo. |
| `Resetear PIN` | Botón de zona de peligro | **Restablecer PIN...** | Los tres puntos (...) indican que la acción abrirá un diálogo de confirmación previo. |
| `El nuevo PIN no coincide con la confirmación.` | Mensaje inline de error | **Los PIN ingresados no coinciden. Verifícalos.** | Formulación amigable e instructiva. |
| `PIN actual incorrecto.` | Mensaje inline de error | **El PIN actual no es correcto. Inténtalo de nuevo.** | Guía al usuario sin frustrarlo. |
| `PIN cambiado con éxito.` | Toast / Feedback éxito | **¡PIN actualizado correctamente!** | Estado de éxito positivo y claro. |
| `PIN reseteado. Reinicia la app para crear uno nuevo.` | Feedback de reset | **PIN eliminado. Reinicia la aplicación para configurar uno nuevo.** | Instrucción clara paso a paso. |
| `{it.message}` | Error de excepción técnica | **No se pudo cambiar el PIN. Inténtalo nuevamente.** | Evita mostrar excepciones crudas de Kotlin/Java en la UI. |
| `¿Resetear PIN?` | Título diálogo confirmación | **¿Restablecer el PIN de acceso?** | Pregunta clara sobre la consecuencia. |
| `Se eliminará el PIN actual. Tendrás que crear uno nuevo al reiniciar la app.` | Cuerpo del diálogo | **Se eliminará la clave actual. La app te pedirá crear un nuevo PIN la próxima vez que la abras.** | Explica con claridad la consecuencia antes de ejecutar. |
| `Sí, resetear` | Botón de confirmación (ErrorRed) | **Sí, restablecer** | Coherencia con el término *Restablecer*. |
| `Cancelar` | Botón de descarte | **Cancelar** | Estándar predecible. |

---

#### 1.3 Clientes Ocultos / Deshabilitados (`ClientesOcultosScreen.kt`)

| Texto Actual | Contexto / Ubicación | Texto Propuesto / Mejorado | Justificación UX y Principio Aplicado |
| :--- | :--- | :--- | :--- |
| `Clientes ocultos / deshabilitados` | Título de pantalla | **Gestión de Clientes Inactivos** | Término unificado que engloba ambos estados sin jerga técnica de arquitectura. |
| `Vista administrativa (reporte 007 P2): restaura o purga definitiva.` | Subtítulo explicativo | **Administra clientes archivados o bloqueados. Puedes restaurarlos o eliminarlos de forma definitiva.** | Elimina la referencia interna *"reporte 007 P2"* y la palabra *"purga"* por términos de uso cotidiano (*archivados*, *eliminación definitiva*). |
| `No hay clientes ocultos ni deshabilitados.` | Empty state | **No tienes clientes archivados ni deshabilitados** *(Subtexto: Los clientes que ocultes o bloquees aparecerán en esta sección)* | **Empty State Accionable:** Explica por qué está vacío para reducir la confusión inicial. |
| `Ocultos (${N}) - purga automática en 6 meses` | Header sección ocultos | **Archivados (${N}) · Eliminación automática en 6 meses** | Reemplaza *"Ocultos"* por *"Archivados"* (patrón estándar de la Ley de Jakob) y *"purga"* por *"Eliminación"*. |
| `Deshabilitados (${N})` | Header sección deshabilitados | **Deshabilitados (${N}) · Sin acceso a clases** | Añade contexto útil sobre qué implica estar deshabilitado. |
| `{nombreCompleto}` / `{numeroDocumento}` | Datos en tarjeta | `{nombreCompleto}` · Doc: `{numeroDocumento}` | Formato de datos ordenado. |
| `Oculto` | Chip de estado (ErrorRed) | **Archivado** | Etiqueta más precisa para un *soft-delete*. |
| `Deshabilitado` | Chip de estado (WarningOrange) | **Bloqueado** / **Inactivo** | Comunicación de estado más intuitiva. |
| `Restaurar` | Botón para archivados | **Desarchivar / Restaurar** | Verbo directo de acción. |
| `Eliminar definitivo` | Botón de peligro | **Eliminar definitivamente** | Adverbio que enfatiza el carácter permanente. |
| `Rehabilitar` | Botón para deshabilitados | **Reactivar cuenta** | Acción clara para restablecer servicios al cliente. |
| `Restaurar cliente` | Título diálogo restaurar | **¿Restaurar cliente?** | Pregunta directa en título de modal. |
| `Se restaurará y habilitará al cliente.` | Cuerpo diálogo restaurar | **El cliente volverá a estar activo en la lista principal y podrá registrar asistencias.** | **Progressive Disclosure:** Detalla exactamente el impacto positivo de la acción. |
| `Restaurar` | Botón confirmar restaurar | **Sí, restaurar** | Confirmación afirmativa explícita. |
| `Cancelar` | Botón descartar | **Cancelar** | Estándar. |
| `Eliminar definitivo` | Título diálogo eliminación | **¿Eliminar cliente definitivamente?** | Advertencia clara en el título. |
| `Borrado irreversible (pagos/medidas en cascada).` | Cuerpo diálogo eliminación | **¡Atención! Esta acción no se puede deshacer. Se borrarán permanentemente sus pagos, historial de medidas y asistencias.** | **Manejo de Destrucción de Datos:** Detalla el borrado en cascada para evitar pérdidas fatales involuntarias. |
| `Eliminar` | Botón confirmar (ErrorRed) | **Sí, eliminar todo** | Verbo de alto impacto visual para confirmar una acción destructiva. |

---

#### 1.4 Detalle de Plan (`PlanDetailScreen.kt`)

| Texto Actual | Contexto / Ubicación | Texto Propuesto / Mejorado | Justificación UX y Principio Aplicado |
| :--- | :--- | :--- | :--- |
| `Cargando plan...` | Estado inicial de carga | **Cargando información del plan...** | Texto contextual claro mientras carga la tarjeta. |
| `{nombre}` | Título de plan | `{nombre}` | Correcto. |
| `$${precio} / {duracionDias}d` | Precio y duración | **$${precio} USD / {duracionDias} días** | Agrega unidades explícitas para evitar ambigüedad. |
| `Sin beneficios descritos` | Fallback descripción | *Sin descripción de beneficios agregada.* | Tono informativo neutro e itálico. |
| `ACTIVO` / `DESHABILITADO` | Chips de estado | **DISPONIBLE** / **INACTIVO** | Estado enfocado en la oferta comercial del plan. |
| `Tipo: {tipo} (máx {N})` | Info de cupo | **Modalidad: {tipo} · Cupo máximo: {N} personas** | Claridad en la redacción de parámetros del plan. |
| `Seguimiento medidas: Sí` / `No` | Control de medidas | **Incluye control de medidas: Sí** / **No** | Redacción completa y legible. |
| `Habilitado` | Label de Switch | **Plan disponible para venta** | Claridad sobre la función del interruptor. |
| `Editar` / `Eliminar` | Botones de acción | **Editar Plan** / **Eliminar** | Verbos claros. |
| `Clientes asociados ({N})` | Header lista asociados | **Clientes inscritos en este plan ({N})** | Vocabulario más natural (*inscritos* vs. *asociados*). |
| `Ningún cliente vinculado a este plan.` | Empty state | **No hay clientes inscritos en este plan actualmente.** | Estado vacío claro. |
| `{nombreCompleto} / {numeroDocumento}` | Lista de inscritos | `{nombreCompleto}` · ID: `{numeroDocumento}` | Separación visual limpia. |
| `{mensaje}` | Feedback del ViewModel | *(Mensajes contextuales de éxito u error)* | Asegurar estilos `SuccessCyan` y `ErrorRed`. |
| `¿Eliminar plan?` | Título diálogo eliminación | **¿Eliminar este plan?** | Pregunta directa. |
| `Si hay clientes vinculados, el borrado será bloqueado. Marca 'Forzar' para eliminar de todas formas (admin override).` | Cuerpo del diálogo | **Este plan tiene clientes inscritos. Para eliminarlo, debes reasignar a los clientes o marcar 'Forzar eliminación' para omitir el bloqueo.** | Explica la regla de negocio sin usar la jerga técnica *"admin override"*. |
| `Forzar eliminación` | Checkbox de excepción | **Forzar eliminación (ignorar clientes vinculados)** | Aclara exactamente qué hace la casilla. |
| `Eliminar` / `Cancelar` | Botones de diálogo | **Eliminar Plan** / **Cancelar** | Etiquetas de acción específicas. |

---

### 2. MODIFICACIONES EN PANTALLAS EXISTENTES

#### 2.1 Detalle de Cliente — Gobernanza de Cuenta (`ClienteDetailScreen.kt`)

| Texto Actual | Contexto / Ubicación | Texto Propuesto / Mejorado | Justificación UX y Principio Aplicado |
| :--- | :--- | :--- | :--- |
| `Gobernanza de Cuenta` | Header de tarjeta administrativa | **Administración de Cuenta** | Término más estándar en aplicaciones comerciales que *"Gobernanza"*. |
| `Reporte 007` | Badge descriptivo | **Ajustes de Administrador** | Sustituye el número de reporte interno por una etiqueta funcional. |
| `Estado del Perfil` | Label del toggle | **Estado del cliente** | Directo y conciso. |
| `Cliente Activa (con acceso)` | Subtítulo estado activo | **Cliente Activo · Acceso permitido** | Unificación de género neutro o dinámico y lenguaje claro. |
| `Cliente Inactiva (bloqueada)` | Subtítulo estado bloqueado | **Cliente Bloqueado · Acceso denegado** | Explicación clara de la consecuencia de estar inactivo. |
| `Cambiar / Reasignar Plan (Admin override)` | Botón reasignar | **Cambiar o Reasignar Plan** | **Ley de Hick:** Remueve la acotación técnica *(Admin override)* de la cara del usuario. |
| `Restaurar cliente` | Botón restauración | **Reactivar Cuenta** | Mantiene consistencia con la acción del sistema. |
| `Eliminar Cliente del Sistema` | Botón zona destructiva | **Eliminar Cliente...** | Mantiene la advertencia pero reduce el dramatismo en la etiqueta del botón. |
| `{accionResultado}` | Feedback inferior | *(Mensaje dinámico de resultado)* | Correctamente configurado con duraciones y colores legibles. |
| `Cliente habilitado. Acceso restablecido.` | Mensaje de éxito | **Cliente activado. Acceso restablecido con éxito.** | Mensaje positivo y completo. |
| `Cliente deshabilitado.` | Mensaje de éxito | **Cliente deshabilitado correctamente.** | Confirmación de la acción. |
| `Cliente eliminado del sistema.` | Mensaje de éxito | **Cliente eliminado del sistema.** | Confirmación directa. |
| `Cliente ocultado (soft delete). Se purgará en 6 meses.` | Mensaje de éxito | **Cliente archivado. Permanecerá en la papelera durante 6 meses antes de eliminarse.** | Traduce el término *"soft delete"* y *"purgará"* a conceptos familiares (*archivado/papelera*). |
| `Cliente restaurado y habilitado.` | Mensaje de éxito | **Cliente restaurado y activado correctamente.** | Retroalimentación clara. |
| `Plan asignado con éxito.` | Mensaje de éxito | **Plan asignado correctamente.** | Breve y directo. |
| `Plan reasignado en modo administrador (override).` | Mensaje de éxito | **Plan reasignado mediante autorización especial.** | Sustituye el término en inglés *override*. |

##### Diálogos Modales de Gobernanza

| Texto Actual | Contexto / Ubicación | Texto Propuesto / Mejorado | Justificación UX y Principio Aplicado |
| :--- | :--- | :--- | :--- |
| `Cambiar / Reasignar plan` | Título CambiarPlanDialog | **Reasignar Plan de Cliente** | Título claro del diálogo. |
| `Selecciona el plan. Con 'Forzar' ignoras bloqueo por deuda/vigencia y cupo grupal.` | Cuerpo CambiarPlanDialog | **Selecciona el nuevo plan. Si marcas 'Forzar', la reasignación se completará omitiendo restricciones de deuda o cupo.** | Explica las consecuencias del botón forzar en lenguaje sencillo. |
| `Plan` / `Sin plan` | Dropdown de selección | **Seleccionar Plan** / **Sin Plan Asignado** | Etiquetas claras de formulario. |
| `Forzar asignación (admin override)` | Checkbox de excepción | **Forzar asignación (omitir deudas y cupos)** | Explicación clara de la función de la casilla. |
| `Asignar` / `Cancelar` | Botones de diálogo | **Confirmar Cambio** / **Cancelar** | Verbo de confirmación explícito. |
| `¿Eliminar cliente?` | Título Diálogo Eliminar (Nivel 1) | **¿Cómo deseas eliminar al cliente?** | Enfoca el diálogo en la elección entre las dos alternativas. |
| `Opción 1: ocultar (soft delete, se conserva en backup y purga en 6 meses). Opción 2: eliminar definitivo e irreversible (borra pagos/medidas en cascada).` | Cuerpo Diálogo Nivel 1 | **• Archivar (Recomendado):** Mantiene copias de seguridad y permite restaurarlo durante 6 meses.<br>**• Eliminar definitivamente:** Borra para siempre todos sus pagos, medidas y registros. | **Progressive Disclosure:** Presenta ambas alternativas organizadas con viñetas para facilitar la lectura rápida. |
| `Eliminar definitivo` | Botón Opción 1 | **Eliminar Definitivamente** | Acción destacada en color rojo. |
| `Ocultar (soft delete)` | Botón Opción 2 | **Archivar Cliente** | Término amigable y seguro en color secundario. |
| `Confirmar eliminación definitiva` | Título Diálogo Nivel 2 | **¿Confirmas la eliminación permanente?** | Diálogo de seguridad de segundo nivel para evitar destrucción accidental. |
| `Esta acción borra todos los datos del cliente de forma irreversible.` | Cuerpo Diálogo Nivel 2 | **Se eliminarán permanentemente el historial de pagos, asistencias y medidas físicas de este cliente. Esta acción no se puede deshacer.** | Advierte explícitamente sobre el impacto real. |
| `Sí, eliminar` / `Cancelar` | Botones Nivel 2 | **Sí, eliminar todo** / **Cancelar** | Botón primario en rojo (*ErrorRed*). |

---

#### 2.2 Lista de Planes — Acciones Inline y Toggle (`PlanesListScreen.kt`)

| Texto Actual | Contexto / Ubicación | Texto Propuesto / Mejorado | Justificación UX y Principio Aplicado |
| :--- | :--- | :--- | :--- |
| `Habilitado` | Label Switch en tarjeta | **Disponible** | Indica si el plan está a la venta. |
| `Detalle` / `Editar` / `Eliminar` | Acciones de la tarjeta | **Ver detalle** / **Editar** / **Eliminar** | Verbos de acción completos y legibles. |
| `Plan habilitado.` / `Plan deshabilitado.` | ViewModel feedback | **Plan disponible para inscripción.** / **Plan pausado.** | Retroalimentación clara sobre el estado del producto. |
| `Plan eliminado.` | ViewModel feedback | **Plan eliminado con éxito.** | Confirmación positiva. |
| `{it.message}` | Excepción técnica de error | **No se puede eliminar: el plan tiene clientes activos vinculados.** | Sustituye errores crudos por mensajes con la causa y solución. |

---

#### 2.3 Formulario de Plan (`PlanFormScreen.kt`)

| Texto Actual | Contexto / Ubicación | Texto Propuesto / Mejorado | Justificación UX y Principio Aplicado |
| :--- | :--- | :--- | :--- |
| `Plan habilitado` | Label de Switch | **Habilitar plan para venta inmediata** | Explica claramente la consecuencia de activar el Switch al crear o editar el plan. |

---

#### 2.4 Historial y Comparación de Medidas (`HistorialMedidasScreen.kt`)

| Texto Actual | Contexto / Ubicación | Texto Propuesto / Mejorado | Justificación UX y Principio Aplicado |
| :--- | :--- | :--- | :--- |
| `Medida / Base / Anterior / Actual + ✎` | Encabezado de tabla | **Variable | Inicial | Anterior | Actual | Editar** | Sustituye el símbolo numérico/icono por la palabra explícita para mejorar la accesibilidad (lectores de pantalla). |
| `✎` | Botón por fila | **Editar** *(o icono con `contentDescription = "Corregir medida"`) | Garantiza accesibilidad UX Mobile. |
| `Registros ({N}) - eliminar si hay error de digitación` | Header lista de registros | **Historial de Registros ({N})** *(Subtexto: Elimina registros si hubo un error de ingreso)* | Separa el título principal de la instrucción secundaria. |
| `Base - dd/MM/yyyy` | Label de registro inicial | **Medición Inicial · dd/MM/yyyy** | Término más claro que *"Base"*. |
| `Registro dd/MM/yyyy HH:mm` | Label de registro normal | **Medición del dd/MM/yyyy a las HH:mm** | Formato de fecha y hora natural. |
| `id:{id}` | Subtexto de registro | *(Omitir de la UI pública del usuario)* | El ID numérico de base de datos no aporta valor a la entrenadora y genera ruido visual. |
| `🗑` | Botón de eliminación | *(Icono de papelera con `contentDescription = "Eliminar registro"`) | Mejora de accesibilidad para herramientas de lectura por voz. |
| `Registro eliminado.` | Toast / Feedback | **Registro de medida eliminado.** | Confirmación clara. |
| `Valor corregido.` | Toast / Feedback | **Medida actualizada correctamente.** | Feedback positivo. |
| `Corregir valor` | Título diálogo edición | **Editar Valor de Medida** | Verbo estándar. |
| `Nuevo valor` | Label de campo | **Nuevo valor (cm / kg)** | Muestra las unidades de medida esperadas en la etiqueta. |
| `Guardar` / `Cancelar` | Botones de diálogo | **Guardar Cambio** / **Cancelar** | Etiquetas explícitas. |
| `¿Eliminar registro?` | Título diálogo eliminar | **¿Eliminar este registro de medidas?** | Titular claro y específico. |
| `Se eliminará el registro y sus valores asociados de forma irreversible.` | Cuerpo del diálogo | **Esta toma de medidas se eliminará del historial del cliente. Esta acción no se puede deshacer.** | Tono de advertencia claro. |
| `Eliminar` | Botón de confirmación | **Eliminar Registro** | Verbo explícito en color ErrorRed. |

---

#### 2.5 Configuración de Variables de Medida (`ConfiguracionMedidasScreen.kt`)

| Texto Actual | Contexto / Ubicación | Texto Propuesto / Mejorado | Justificación UX y Principio Aplicado |
| :--- | :--- | :--- | :--- |
| `Medidas desactivadas (puedes reactivar)` | Título de sección inactiva | **Variables Desactivadas** *(Subtexto: Puedes reactivarlas en cualquier momento)* | Separa la categoría del consejo de uso. |
| `{nombreCompleto} ({nombreAbreviado})` | Card de inactiva | `{nombreCompleto}` **({nombreAbreviado})** | Tipografía limpia. |
| `Desactivada` | Subtexto de estado | **Inactiva** | Etiqueta coherente con el resto del sistema. |
| `Reactivar` | Botón de acción | **Reactivar Variable** | Verbo específico. |

---

#### 2.6 Biblioteca de Ejercicios — Filtros (`EjerciciosScreen.kt`)

| Texto Actual | Contexto / Ubicación | Texto Propuesto / Mejorado | Justificación UX y Principio Aplicado |
| :--- | :--- | :--- | :--- |
| `Todos` | FilterChip para limpiar | **Todos los músculos** | Especifica la categoría del filtro. |
| `{bodyPart}` | Chips dinámicos | `{bodyPart}` *(ej. Pecho, Espalda)* | Formato dinámico en mayúscula inicial. |
| `Sin resultados para el filtro.` | Empty state de búsqueda | **No encontramos ejercicios para este grupo muscular.**<br>*(CTA: Botón "Ver todos los ejercicios")* | **Empty State Accionable:** Ofrece una vía rápida para limpiar los filtros sin obligar al usuario a adivinar. |

---

#### 2.7 Menú de Ajustes y Navegación (`AjustesScreen.kt` & NavGraph)

| Texto Actual | Contexto / Ubicación | Texto Propuesto / Mejorado | Justificación UX y Principio Aplicado |
| :--- | :--- | :--- | :--- |
| `Seguridad (PIN)` | Item de menú Ajustes | **PIN de Seguridad** | Nombre de sección limpio. |
| `Clientes ocultos / deshabilitados` | Item de menú Ajustes | **Clientes Archivados y Bloqueados** | Coherencia con la nueva nomenclatura propuesta para la pantalla. |
| `Detalle de Plan` | TopBar Title | **Detalle del Plan** | Título de navegación natural. |
| `Clientes Ocultos` | TopBar Title | **Clientes Archivados** | Título coherente con el flujo de navegación. |
| `Seguridad` | TopBar Title | **Seguridad** | Conciso y claro. |

---

### 3. MENSAJES DE ERROR DE REPOSITORIO Y MÓDULO BACKEND

Sustitución de excepciones técnicas arrojadas por `IllegalStateException` en la capa de datos por mensajes de error con lenguaje de usuario:

| Mensaje Técnico Actual (Repository) | Situación de Disparo | Mensaje Propuesto para la UI | Justificación UX |
| :--- | :--- | :--- | :--- |
| `Cliente no encontrado.` | El ID de cliente no existe en DB local. | **No se encontró el perfil del cliente seleccionado.** | Explica el conflicto en términos del dominio de la aplicación. |
| `Configuración no encontrada.` | La variable de medida no existe. | **La variable de medida seleccionada no está disponible.** | Lenguaje claro sin tecnicismos de base de datos. |
| `Plan no encontrado.` | ID de plan inexistente. | **El plan seleccionado ya no está disponible.** | Indica el estado de la entidad. |
| `No se puede eliminar: {N} cliente(s) aún vinculados a este plan. Reasigna o cancela primero.` | Intentar eliminar un plan con clientes inscritos. | **No es posible eliminar el plan: tiene {N} cliente(s) inscrito(s). Reasigna a los clientes antes de borrarlo.** | **Mensaje con Solución:** Explica la causa exacta y indica la acción preventiva requerida. |
| `No se puede eliminar: {N} cliente(s) aún vinculados.` | Validación previa local en ViewModel. | **No se puede borrar: hay {N} cliente(s) vinculados a este plan.** | Redacción coherente con la pantalla de detalle. |

---

## 3. Matriz Guía de Estilo y Nomenclatura UX (Mary Fitness)

Para asegurar la consistencia del sistema de diseño en futuras iteraciones, se establecen las siguientes equivalencias de lenguaje:

| Término Técnico / Anterior | Término Optimizado UX | Justificación de Marca / Usuario |
| :--- | :--- | :--- |
| *Soft Delete / Oculto* | **Archivado / Papelera** | Concepto universal acuñado por la Ley de Jakob (familiaridad con Gmail, WhatsApp, etc.). |
| *Purga automática* | **Eliminación permanente** | Explicación clara sin connotaciones extrañas o agresivas. |
| *Deshabilitado* | **Bloqueado / Inactivo** | Describe con precisión la suspensión temporal de un usuario. |
| *Admin Override / Forzar* | **Autorización Especial / Omitir Validación** | Oculta la jerga de desarrollo y mantiene un tono profesional de gestión. |
| *Base local / BCrypt* | **Seguridad Local / Datos Protegidos** | Brinda tranquilidad al usuario sin marearlo con detalles de infraestructura. |
| *Resetear* | **Restablecer** | Corrección ortográfica y alineación con el estándar oficial de Android. |
