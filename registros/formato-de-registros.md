# Formato de Registros — Guía de Llenado Manual

Guía para registrar clientes, pagos, planes y medidas de forma manual mediante JSON.

---

## 1. Orden de dependencias

Antes de registrar cualquier cliente se deben definir los catálogos base:

1. **`configuraciones_medida`** — Variables corporales que el gimnasio medirá (peso, cintura, brazo, etc.). Se definen una vez y se reutilizan para todos los clientes.
2. **`planes`** — Catálogo de membresías disponibles (costo, duración, tipo). Los clientes referencian un `planId`.

Una vez existan这两种猫logos, se pueden registrar clientes con su historial asociado (pagos y medidas).

---

## 2. Catálogos base (llenar una sola vez)

### 2.1 Configuraciones de medida

Cada entrada define una variable corporal medible.

```json
{
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
    }
  ]
}
```

| Campo | Tipo | Obligatorio | Descripción |
|:---|:---|:---|:---|
| `id` | Long | Sí | Identificador único. Se usa como referencia en `medida_valores.configuracionMedidaId`. |
| `nombreCompleto` | String | Sí | Nombre descriptivo (ej: "Peso Corporal"). |
| `nombreAbreviado` | String | Sí | Abreviatura corta (ej: "PESO"). |
| `unidadMedida` | String | Sí | Unidad de medida (ej: "kg", "cm"). |
| `tieneDosSecciones` | Boolean | Sí | `false` = se registra un solo valor (`UNICA`). `true` = se registran dos valores (`MAS_GRANDE` y `MAS_PEQUENA`), ej: brazo izquierdo/derecho. |
| `obligatoria` | Boolean | Sí | Si `true`, el cliente debe tener al menos un valor para esta medida en su primer registro. |
| `activa` | Boolean | Sí | Si `false`, no se muestra en la app. |
| `pendienteSync` | Boolean | Sí | Siempre `true` para registros nuevos. |

### 2.2 Planes

Cada entrada define un tipo de membresía.

```json
{
  "planes": [
    {
      "id": 1,
      "nombre": "Mensualidad",
      "precio": 80000,
      "duracionDias": 30,
      "beneficios": "Acceso mensual al gimnasio",
      "tipo": "INDIVIDUAL",
      "maxIntegrantes": null,
      "incluyeSeguimientoMedidas": true,
      "habilitado": true,
      "pendienteSync": true
    }
  ]
}
```

| Campo | Tipo | Obligatorio | Descripción |
|:---|:---|:---|:---|
| `id` | Long | Sí | Identificador único. Se usa como referencia en `clientes.planId` y `pagos.planId`. |
| `nombre` | String | Sí | Nombre del plan (ej: "Mensualidad", "2x1"). |
| `precio` | Long | Sí | Precio en pesos colombianos (sin decimales). |
| `duracionDias` | Int | Sí | Duración en días (ej: 30, 1). |
| `beneficios` | String | No | Descripción de beneficios. Default: `""`. |
| `tipo` | Enum | Sí | `INDIVIDUAL` o `GRUPAL`. |
| `maxIntegrantes` | Int? | No | Requerido si `tipo = GRUPAL` (mínimo 2). Si `INDIVIDUAL`, debe ser `null`. |
| `incluyeSeguimientoMedidas` | Boolean | No | Si `true`, el plan incluye seguimiento de medidas. Default: `false`. |
| `habilitado` | Boolean | Sí | Si `false`, el plan no aparece disponible. |
| `pendienteSync` | Boolean | Sí | Siempre `true` para registros nuevos. |

---

## 3. Registro de clientes (formato centrado en el cliente)

Para cada cliente se especifica toda su información en una sola entrada anidada, incluyendo su historial de pagos y medidas.

### 3.1 Plantilla por cliente

```json
{
  "cliente": {
    "primerNombre": "NOMBRE",
    "segundoNombre": null,
    "primerApellido": "APELLIDO",
    "segundoApellido": null,
    "tipoDocumento": "CEDULA_CIUDADANIA",
    "numeroDocumento": "NUMERO",
    "telefono": "TELEFONO",
    "planId": 1,
    "fechaRegistro": 0,
    "fechaUltimaActividad": 0,
    "estadoCuenta": "ACTIVO",
    "habilitado": true,
    "oculto": false,
    "pendienteSync": true
  },
  "pagos": [
    {
      "planId": 1,
      "montoTotal": 80000,
      "montoPagado": 80000,
      "estado": "COMPLETO",
      "fechaPago": 0,
      "fechaVencimiento": 0,
      "plazoMaximoPago": null,
      "pendienteSync": true
    }
  ],
  "registros_medidas": [
    {
      "timestamp": 0,
      "esBase": true,
      "valores": [
        {
          "configuracionMedidaId": 1,
          "seccion": "UNICA",
          "valor": 0.0
        }
      ]
    }
  ]
}
```

### 3.2 Campos del cliente

| Campo | Tipo | Obligatorio | Descripción |
|:---|:---|:---|:---|
| `primerNombre` | String | Sí | Primer nombre. |
| `segundoNombre` | String? | No | Segundo nombre. `null` si no aplica. |
| `primerApellido` | String | Sí | Primer apellido. |
| `segundoApellido` | String? | No | Segundo apellido. `null` si no aplica. |
| `tipoDocumento` | Enum | Sí | Ver sección 5. |
| `numeroDocumento` | String | Sí | Número de documento. **Debe ser único** en la base de datos. |
| `telefono` | String? | No | Teléfono de contacto. `null` si no se tiene. |
| `planId` | Long? | Sí | ID del plan asociado. `null` si el cliente no tiene plan (`estadoCuenta = SIN_PLAN`). |
| `fechaRegistro` | Long | Sí | Fecha de registro en **epoch millis**. |
| `fechaUltimaActividad` | Long | Sí | Fecha de última actividad en **epoch millis**. Generalmente igual a `fechaRegistro` para registros iniciales. |
| `estadoCuenta` | Enum | Sí | Ver sección 5. Para registros iniciales usar `ACTIVO` o `SIN_PLAN`. |
| `habilitado` | Boolean | Sí | `true` si el cliente está activo. |
| `oculto` | Boolean | Sí | `false` para clientes visibles. |
| `pendienteSync` | Boolean | Sí | Siempre `true` para registros nuevos. |

### 3.3 Campos de pagos

Cada entrada en `pagos` representa una transacción financiera del cliente.

| Campo | Tipo | Obligatorio | Descripción |
|:---|:---|:---|:---|
| `planId` | Long | Sí | ID del plan que se está pagando. |
| `montoTotal` | Long | Sí | Monto total del plan en pesos. |
| `montoPagado` | Long | Sí | Monto que el cliente ha pagado. Si es igual a `montoTotal`, el estado es `COMPLETO`. |
| `estado` | Enum | Sí | `COMPLETO` (pagó todo), `PARCIAL` (abono parcial), `CANCELADO`. |
| `fechaPago` | Long | Sí | Fecha del pago en **epoch millis**. |
| `fechaVencimiento` | Long | Sí | Fecha de vencimiento del plan en **epoch millis**. |
| `plazoMaximoPago` | Long? | Condicionado | Solo requerido si `estado = PARCIAL`. Fecha límite para completar el pago. `null` en otros casos. |
| `pendienteSync` | Boolean | Sí | Siempre `true` para registros nuevos. |

### 3.4 Campos de registros de medidas

Cada entrada en `registros_medidas` representa una toma de medidas antropométricas en un momento dado.

| Campo | Tipo | Obligatorio | Descripción |
|:---|:---|:---|:---|
| `timestamp` | Long | Sí | Fecha y hora de la toma de medidas en **epoch millis**. |
| `esBase` | Boolean | Sí | `true` si es la primera toma de medidas del cliente (medición base). `false` para tomas de seguimiento. |
| `valores` | Array | Sí | Lista de valores medidos. Ver abajo. |

### 3.5 Campos de valores de medidas

Cada entrada en `valores` representa una medición individual dentro de un registro.

| Campo | Tipo | Obligatorio | Descripción |
|:---|:---|:---|:---|
| `configuracionMedidaId` | Long | Sí | ID de la configuración de medida que se está registrando. |
| `seccion` | Enum | Sí | `UNICA` (medida única), `MAS_GRANDE` (lado mayor), `MAS_PEQUENA` (lado menor). Si `configuracionMedida.tieneDosSecciones = false`, usar siempre `UNICA`. |
| `valor` | Double | Sí | Valor numérico de la medición. |

---

## 4. Ejemplos de uso

### 4.1 Cliente nuevo (sin historial previo)

Cliente que se acaba de inscribir con una toma de medidas base y un pago completo.

```json
{
  "cliente": {
    "primerNombre": "Juan",
    "segundoNombre": null,
    "primerApellido": "Pérez",
    "segundoApellido": null,
    "tipoDocumento": "CEDULA_CIUDADANIA",
    "numeroDocumento": "1234567890",
    "telefono": "3001234567",
    "planId": 1,
    "fechaRegistro": 1757500800000,
    "fechaUltimaActividad": 1757500800000,
    "estadoCuenta": "ACTIVO",
    "habilitado": true,
    "oculto": false,
    "pendienteSync": true
  },
  "pagos": [
    {
      "planId": 1,
      "montoTotal": 80000,
      "montoPagado": 80000,
      "estado": "COMPLETO",
      "fechaPago": 1757500800000,
      "fechaVencimiento": 1760092800000,
      "plazoMaximoPago": null,
      "pendienteSync": true
    }
  ],
  "registros_medidas": [
    {
      "timestamp": 1757500800000,
      "esBase": true,
      "valores": [
        { "configuracionMedidaId": 1, "seccion": "UNICA", "valor": 75.0 },
        { "configuracionMedidaId": 3, "seccion": "UNICA", "valor": 82.0 },
        { "configuracionMedidaId": 5, "seccion": "MAS_GRANDE", "valor": 33.0 },
        { "configuracionMedidaId": 5, "seccion": "MAS_PEQUENA", "valor": 31.0 }
      ]
    }
  ]
}
```

### 4.2 Cliente antiguo con historial completo

Cliente que ya tiene meses en el gimnasio, con múltiples pagos y tomas de medidas.

```json
{
  "cliente": {
    "primerNombre": "María",
    "segundoNombre": "Ana",
    "primerApellido": "García",
    "segundoApellido": "López",
    "tipoDocumento": "CEDULA_CIUDADANIA",
    "numeroDocumento": "9876543210",
    "telefono": "3109876543",
    "planId": 2,
    "fechaRegistro": 1743993600000,
    "fechaUltimaActividad": 1757500800000,
    "estadoCuenta": "ACTIVO",
    "habilitado": true,
    "oculto": false,
    "pendienteSync": true
  },
  "pagos": [
    {
      "planId": 2,
      "montoTotal": 35000,
      "montoPagado": 35000,
      "estado": "COMPLETO",
      "fechaPago": 1743993600000,
      "fechaVencimiento": 1746585600000,
      "plazoMaximoPago": null,
      "pendienteSync": true
    },
    {
      "planId": 2,
      "montoTotal": 35000,
      "montoPagado": 0,
      "estado": "PARCIAL",
      "fechaPago": 1746585600000,
      "fechaVencimiento": 1749177600000,
      "plazoMaximoPago": 1747190400000,
      "pendienteSync": true
    },
    {
      "planId": 2,
      "montoTotal": 35000,
      "montoPagado": 35000,
      "estado": "COMPLETO",
      "fechaPago": 1749177600000,
      "fechaVencimiento": 1751769600000,
      "plazoMaximoPago": null,
      "pendienteSync": true
    }
  ],
  "registros_medidas": [
    {
      "timestamp": 1743993600000,
      "esBase": true,
      "valores": [
        { "configuracionMedidaId": 1, "seccion": "UNICA", "valor": 80.0 },
        { "configuracionMedidaId": 3, "seccion": "UNICA", "valor": 88.0 },
        { "configuracionMedidaId": 4, "seccion": "UNICA", "valor": 95.0 },
        { "configuracionMedidaId": 5, "seccion": "MAS_GRANDE", "valor": 30.0 },
        { "configuracionMedidaId": 5, "seccion": "MAS_PEQUENA", "valor": 28.0 }
      ]
    },
    {
      "timestamp": 1746585600000,
      "esBase": false,
      "valores": [
        { "configuracionMedidaId": 1, "seccion": "UNICA", "valor": 78.5 },
        { "configuracionMedidaId": 3, "seccion": "UNICA", "valor": 86.0 },
        { "configuracionMedidaId": 4, "seccion": "UNICA", "valor": 93.0 },
        { "configuracionMedidaId": 5, "seccion": "MAS_GRANDE", "valor": 31.0 },
        { "configuracionMedidaId": 5, "seccion": "MAS_PEQUENA", "valor": 29.0 }
      ]
    },
    {
      "timestamp": 1749177600000,
      "esBase": false,
      "valores": [
        { "configuracionMedidaId": 1, "seccion": "UNICA", "valor": 77.0 },
        { "configuracionMedidaId": 3, "seccion": "UNICA", "valor": 84.0 },
        { "configuracionMedidaId": 4, "seccion": "UNICA", "valor": 91.0 },
        { "configuracionMedidaId": 5, "seccion": "MAS_GRANDE", "valor": 32.0 },
        { "configuracionMedidaId": 5, "seccion": "MAS_PEQUENA", "valor": 30.0 }
      ]
    }
  ]
}
```

### 4.3 Cliente sin plan

Cliente que solo tiene ficha registrada pero no ha adquirido un plan.

```json
{
  "cliente": {
    "primerNombre": "Pedro",
    "segundoNombre": null,
    "primerApellido": "Martínez",
    "segundoApellido": null,
    "tipoDocumento": "CEDULA_CIUDADANIA",
    "numeroDocumento": "5555555555",
    "telefono": null,
    "planId": null,
    "fechaRegistro": 1757500800000,
    "fechaUltimaActividad": 1757500800000,
    "estadoCuenta": "SIN_PLAN",
    "habilitado": true,
    "oculto": false,
    "pendienteSync": true
  },
  "pagos": [],
  "registros_medidas": []
}
```

---

## 5. Diccionario de Enums

| Enum | Valores permitidos | Uso |
|:---|:---|:---|
| `tipoDocumento` | `CEDULA_CIUDADANIA`, `TARJETA_IDENTIDAD`, `CEDULA_EXTRANJERIA`, `PASAPORTE`, `NIT` | Tipo de documento de identidad del cliente. |
| `tipo` (Plan) | `INDIVIDUAL`, `GRUPAL` | Si es `GRUPAL`, `maxIntegrantes` debe ser ≥ 2. Si es `INDIVIDUAL`, debe ser `null`. |
| `estado` (Pago) | `COMPLETO`, `PARCIAL`, `CANCELADO` | Si es `PARCIAL`, es obligatorio especificar `plazoMaximoPago`. |
| `estadoCuenta` (Cliente) | `SIN_PLAN`, `ACTIVO`, `DEUDA`, `VENCIDO` | Para registros iniciales usar `ACTIVO` o `SIN_PLAN`. Los demás valores se calculan automáticamente. |
| `seccion` (Medida) | `UNICA`, `MAS_GRANDE`, `MAS_PEQUENA` | Si `configuracionMedida.tieneDosSecciones = false`, usar `UNICA`. Si `true`, registrar ambas secciones. |
| `esBase` (Medida) | `true`, `false` | El primer registro de un cliente **siempre** debe ser base (`true`). Las tomas posteriores usan `false`. |

---

## 6. Conversión de fechas

Las fechas se almacenan como **epoch millis** (milisegundos desde el 1 de enero de 1970).

Para convertir una fecha legible a epoch millis:

```
Fecha: 2025-04-05 (5 de abril de 2025)
Epoch millis: 1743820800000
```

Herramientas útiles:
- [epochconverter.com](https://www.epochconverter.com/)
- En Kotlin: `LocalDate.of(2025, 4, 5).atStartOfDay(ZoneId.of("America/Bogota")).toInstant().toEpochMilli()`

---

## 7. Formato relacional de salida

El formato centrado en el cliente (sección 3) se convierte al siguiente formato relacional para persistir en la base de datos:

```json
{
  "configuraciones_medida": [ ... ],
  "planes": [ ... ],
  "clientes": [
    {
      "id": 1,
      "primerNombre": "...",
      "segundoNombre": null,
      "primerApellido": "...",
      "segundoApellido": null,
      "tipoDocumento": "CEDULA_CIUDADANIA",
      "numeroDocumento": "...",
      "telefono": "...",
      "planId": 1,
      "fechaRegistro": 0,
      "fechaUltimaActividad": 0,
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
      "fechaPago": 0,
      "fechaVencimiento": 0,
      "plazoMaximoPago": null,
      "pendienteSync": true
    }
  ],
  "medida_registros": [
    {
      "id": 1,
      "clienteId": 1,
      "timestamp": 0,
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
      "valor": 75.0,
      "pendienteSync": true
    }
  ]
}
```

**Reglas de asignación de IDs:**
- Los `id` de cada colección se asignan de forma secuencial (1, 2, 3, ...).
- Los `clienteId`, `planId` y `registroId` referencian los `id` de sus respectivas colecciones.
- Los `configuracionMedidaId` referencian los `id` de `configuraciones_medida`.

---

## 8. Reglas de validación

| Regla | Descripción |
|:---|:---|
| `numeroDocumento` único | No puede existir dos clientes con el mismo número de documento. |
| `esBase = true` obligatorio | El primer registro de medidas de un cliente **siempre** debe ser base. |
| Medidas con dos secciones | Si `configuracionMedida.tieneDosSecciones = true`, el registro debe incluir ambos valores (`MAS_GRANDE` y `MAS_PEQUENA`). |
| `plazoMaximoPago` condicionado | Solo se permite si `estado = PARCIAL`. En otros casos debe ser `null`. |
| `maxIntegrantes` condicionado | Solo se requiere si `tipo = GRUPAL` (mínimo 2). Si `INDIVIDUAL`, debe ser `null`. |
| `fechaVencimiento > fechaPago` | La fecha de vencimiento debe ser posterior a la fecha de pago. |
| Timestamps válidos | Las fechas no deben ser futuras con más de 60 segundos de margen. |
