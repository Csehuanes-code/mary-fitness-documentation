Para registrar un cliente desde cero cuando **no existe ninguna configuración previa** en el sistema, se debe respetar el orden estricto de dependencias del modelo de datos:

1. **Configuraciones de medidas (`configuraciones_medida`):** Las variables corporales que el gimnasio medirá (peso, cintura, brazo, etc.). Son indispensables antes de tomar cualquier medida.
2. **Planes (`planes`):** El catálogo de membresías o suscripciones disponibles (costo, duración, cupos). Sin esto, no se puede asociar un plan ni tarifar un pago.
3. **Cliente (`clientes`):** La ficha del nuevo miembro con sus datos personales, de contacto e identificación.
4. **Pagos (`pagos`):** La transacción financiera inicial asociada al cliente y su plan (monto pagado, saldo, vencimiento y plazo si es abono parcial).
5. **Registros y valores de medidas (`medida_registros` / `medida_valores`):** La toma de medidas antropométricas iniciales marcadas como medición base (`esBase: true`).

A continuación tienes el **molde JSON completo** estructurado según el modelo de datos de la aplicación.

---

### 1. Molde JSON Relacional Completo (Semilla / Base de datos en blanco)

Este formato representa las colecciones tal como se persisten y sincronizan (ordenadas por dependencias):

```json
{
  "$schema_description": "Plantilla para inicialización y registro integral desde cero",
  
  "configuraciones_medida": [
    {
      "id": 1,
      "nombreCompleto": "Peso Corporal",
      "nombreAbreviado": "PESO",
      "unidadMedida": "kg",
      "tieneDosSecciones": false,
      "obligatoria": true,
      "activa": true,
      "pendienteSync": true
    },
    {
      "id": 2,
      "nombreCompleto": "Cintura",
      "nombreAbreviado": "CINT",
      "unidadMedida": "cm",
      "tieneDosSecciones": false,
      "obligatoria": true,
      "activa": true,
      "pendienteSync": true
    },
    {
      "id": 3,
      "nombreCompleto": "Brazo",
      "nombreAbreviado": "BRZ",
      "unidadMedida": "cm",
      "tieneDosSecciones": true,
      "obligatoria": false,
      "activa": true,
      "pendienteSync": true
    },
    {
      "id": 4,
      "nombreCompleto": "Pierna / Muslo",
      "nombreAbreviado": "MUSL",
      "unidadMedida": "cm",
      "tieneDosSecciones": true,
      "obligatoria": false,
      "activa": true,
      "pendienteSync": true
    }
  ],

  "planes": [
    {
      "id": 1,
      "nombre": "Plan Mensual Estándar",
      "precio": 80000,
      "duracionDias": 30,
      "beneficios": "Acceso total al gimnasio, asesoría de rutina y seguimiento de medidas.",
      "tipo": "INDIVIDUAL",
      "maxIntegrantes": null,
      "incluyeSeguimientoMedidas": true,
      "habilitado": true,
      "pendienteSync": true
    },
    {
      "id": 2,
      "nombre": "Plan Dúo Pareja",
      "precio": 140000,
      "duracionDias": 30,
      "beneficios": "Acceso ilimitado para dos personas en la misma franja horaria.",
      "tipo": "GRUPAL",
      "maxIntegrantes": 2,
      "incluyeSeguimientoMedidas": true,
      "habilitado": true,
      "pendienteSync": true
    }
  ],

  "clientes": [
    {
      "id": 1,
      "primerNombre": "Carlos",
      "segundoNombre": "Alberto",
      "primerApellido": "Pérez",
      "segundoApellido": "Gómez",
      "tipoDocumento": "CEDULA_CIUDADANIA",
      "numeroDocumento": "1098765432",
      "telefono": "3001234567",
      "planId": 1,
      "fechaRegistro": 1725321600000,
      "fechaUltimaActividad": 1725321600000,
      "estadoCuenta": "ACTIVO",
      "habilitado": true,
      "oculto": false,
      "fechaOcultado": null,
      "fechaProximaRevisionInactividad": null,
      "pendienteSync": true
    }
  ],

  "pagos": [
    {
      "id": 1,
      "clienteId": 1,
      "planId": 1,
      "montoTotal": 80000,
      "montoPagado": 80000,
      "estado": "COMPLETO",
      "fechaPago": 1725321600000,
      "fechaVencimiento": 1727913600000,
      "plazoMaximoPago": null,
      "pendienteSync": true
    }
  ],

  "medida_registros": [
    {
      "id": 1,
      "clienteId": 1,
      "timestamp": 1725321600000,
      "esBase": true,
      "pendienteSync": true
    }
  ],

  "medida_valores": [
    {
      "id": 1,
      "registroId": 1,
      "configuracionMedidaId": 1,
      "seccion": "UNICA",
      "valor": 78.5,
      "pendienteSync": true
    },
    {
      "id": 2,
      "registroId": 1,
      "configuracionMedidaId": 2,
      "seccion": "UNICA",
      "valor": 84.0,
      "pendienteSync": true
    },
    {
      "id": 3,
      "registroId": 1,
      "configuracionMedidaId": 3,
      "seccion": "MAS_GRANDE",
      "valor": 34.5,
      "pendienteSync": true
    },
    {
      "id": 4,
      "registroId": 1,
      "configuracionMedidaId": 3,
      "seccion": "MAS_PEQUENA",
      "valor": 32.0,
      "pendienteSync": true
    },
    {
      "id": 5,
      "registroId": 1,
      "configuracionMedidaId": 4,
      "seccion": "MAS_GRANDE",
      "valor": 56.0,
      "pendienteSync": true
    },
    {
      "id": 6,
      "registroId": 1,
      "configuracionMedidaId": 4,
      "seccion": "MAS_PEQUENA",
      "valor": 53.5,
      "pendienteSync": true
    }
  ]
}
```

---

### 2. Molde Unificado / Transaccional (Para un endpoint o registro conjunto)

Si necesitas un único payload para registrar todo en una sola operación atómica:

```json
{
  "cliente": {
    "primerNombre": "Carlos",
    "segundoNombre": "Alberto",
    "primerApellido": "Pérez",
    "segundoApellido": "Gómez",
    "tipoDocumento": "CEDULA_CIUDADANIA",
    "numeroDocumento": "1098765432",
    "telefono": "3001234567"
  },
  "configuracion_inicial_plan": {
    "crearNuevoPlan": true,
    "plan": {
      "nombre": "Plan Mensual Estándar",
      "precio": 80000,
      "duracionDias": 30,
      "beneficios": "Acceso completo a sala y seguimiento de medidas",
      "tipo": "INDIVIDUAL",
      "maxIntegrantes": null,
      "incluyeSeguimientoMedidas": true
    }
  },
  "primer_pago": {
    "montoPagado": 50000,
    "estado": "PARCIAL",
    "fechaPagoEpochMillis": 1725321600000,
    "plazoMaximoPagoEpochMillis": 1725926400000
  },
  "configuraciones_medida_sistema": [
    {
      "claveReferencia": "PESO",
      "nombreCompleto": "Peso Corporal",
      "nombreAbreviado": "PESO",
      "unidadMedida": "kg",
      "tieneDosSecciones": false,
      "obligatoria": true
    },
    {
      "claveReferencia": "BRZ",
      "nombreCompleto": "Brazo",
      "nombreAbreviado": "BRZ",
      "unidadMedida": "cm",
      "tieneDosSecciones": true,
      "obligatoria": false
    }
  ],
  "toma_medidas_base": {
    "esBase": true,
    "timestampEpochMillis": 1725321600000,
    "valores": [
      {
        "claveReferencia": "PESO",
        "seccion": "UNICA",
        "valor": 78.5
      },
      {
        "claveReferencia": "BRZ",
        "seccion": "MAS_GRANDE",
        "valor": 34.5
      },
      {
        "claveReferencia": "BRZ",
        "seccion": "MAS_PEQUENA",
        "valor": 32.0
      }
    ]
  }
}
```

---

### 3. Diccionario de Enums y Reglas de Validación

Para asegurar la integridad de datos en el sistema:

| Enumeración / Campo | Valores permitidos | Regla de negocio asociada |
| :--- | :--- | :--- |
| `tipoDocumento` | `CEDULA_CIUDADANIA`, `TARJETA_IDENTIDAD`, `CEDULA_EXTRANJERIA`, `PASAPORTE`, `NIT` | `numeroDocumento` debe ser único en la base de datos. |
| `tipo` (Plan) | `INDIVIDUAL`, `GRUPAL` | Si es `GRUPAL`, `maxIntegrantes` debe ser $\ge 2$. Si es `INDIVIDUAL`, debe ser `null`. |
| `estado` (Pago) | `COMPLETO`, `PARCIAL`, `CANCELADO` | Si es `PARCIAL`, es **obligatorio** especificar `plazoMaximoPago`. |
| `estadoCuenta` (Cliente) | `SIN_PLAN`, `ACTIVO`, `DEUDA`, `VENCIDO` | Se calcula automáticamente en base a pagos y vigencias. |
| `seccion` (Medida) | `UNICA`, `MAS_GRANDE`, `MAS_PEQUENA` | Si la medida tiene `tieneDosSecciones = true`, se deben registrar ambas partes (`MAS_GRANDE` y `MAS_PEQUENA`). |
| `esBase` (Medida) | `true`, `false` | El primer registro de un cliente **siempre** debe ser base (`esBase = true`). Las mediciones posteriores deberán usar exactamente el mismo conjunto de variables. |
| Timestamps | Números enteros en milisegundos (`Long` epoch) | Evita fechas futuras con margen de reloj superior a 60 segundos. |
