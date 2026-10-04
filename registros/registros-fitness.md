# Base de Datos MaryFitness - Formato Legible

## Configuración de Medidas
- **PESO**: Peso Corporal (kg) [Única, Obligatoria]
- **PECHO**: Pecho (cm) [Única, Obligatoria]
- **CINTURA**: Cintura (cm) [Única, Obligatoria]
- **CADERA**: Cadera (cm) [Única, Obligatoria]
- **BRAZO**: Brazo (cm) [Doble, Obligatoria]
- **PIERNA**: Pierna (cm) [Doble, Obligatoria]
- **PANTORRILLA**: Pantorrilla (cm) [Única, Obligatoria]
- **CINTURA_BAJA**: Cintura Baja (cm) [Única, Opcional]

## Catálogo de Planes
- **plan_mensual**: Mensualidad | Precio: 80000 | Días: 30 | Tipo: INDIVIDUAL | Max: NINGUNO | Seg: Sí
- **plan_diario**: Diario | Precio: 5000 | Días: 1 | Tipo: INDIVIDUAL | Max: NINGUNO | Seg: No
- **plan_retos**: Reto Individual | Precio: 70000 | Días: 42 | Tipo: INDIVIDUAL | Max: NINGUNO | Seg: Sí
- **plan_mensual_70**: Mensualidad 70 | Precio: 70000 | Días: 30 | Tipo: INDIVIDUAL | Max: NINGUNO | Seg: Sí
- **plan_reto_2x1**: Reto 2x1 | Precio: 35000 | Días: 21 | Tipo: GRUPAL | Max: 2 | Seg: Sí
- **plan_2x1_mensual**: 2x1 Mensual | Precio: 65000 | Días: 30 | Tipo: GRUPAL | Max: 2 | Seg: Sí
- **plan_estudiantil**: Plan Estudiantil | Precio: 50000 | Días: 30 | Tipo: INDIVIDUAL | Max: NINGUNO | Seg: Sí
- **plan_quincenal**: Plan Quincenal | Precio: 40000 | Días: 15 | Tipo: INDIVIDUAL | Max: NINGUNO | Seg: No
- **plan_uso_maquinas_mensual**: Uso Mensual de Maquinas | Precio: 40000 | Días: 30 | Tipo: INDIVIDUAL | Max: NINGUNO | Seg: Sí

---

## Clientes, Pagos y Medidas

### 1. Nereida Rocha (CEDULA_CIUDADANIA: 1082372923)
- **Teléfono**: 3103534455
- **Plan Actual**: plan_reto_2x1
- **Pago**: Total: 35000 | Pagado: 0 | Estado: PARCIAL | Fecha: 2026-08-11 | Vence: 2026-09-22 | Plazo: 2026-10-30
- **Medidas**:
  - **[BASE] 2026-04-23**:
    - PESO: 73.7
    - PECHO: 97.0
    - CINTURA: 84.0
    - CADERA: 112.0
    - BRAZO: Grande 32.0, Pequeña 31.0
    - PIERNA: Grande 68.0, Pequeña 54.0
    - PANTORRILLA: 39.0
  - **[SEGUIMIENTO] 2026-08-25**:
    - PESO: 74.9
    - PECHO: 99.0
    - CINTURA: 82.0
    - CADERA: 112.0
    - BRAZO: Grande 33.0, Pequeña 31.0
    - PIERNA: Grande 68.0, Pequeña 54.0
    - PANTORRILLA: 39.0
  - **[SEGUIMIENTO] 2026-09-02**:
    - PESO: 75.0
    - PECHO: 98.0
    - CINTURA: 82.0
    - CADERA: 111.0
    - BRAZO: Grande 32.0, Pequeña 30.0
    - PIERNA: Grande 65.0, Pequeña 54.0
    - PANTORRILLA: 39.0

### 2. Ana Karina Marin Boscan (CEDULA_CIUDADANIA: 12851674)
- **Teléfono**: 3217408170
- **Plan Actual**: plan_mensual_70
- **Pago**: Total: 70000 | Pagado: 70000 | Estado: COMPLETO | Fecha: 2026-09-29 | Vence: 2026-10-29 | Plazo: NINGUNO
- **Medidas**:
  - **[BASE] 2026-04-27**:
    - PESO: 66.7
    - PECHO: 96.0
    - CINTURA: 81.0
    - CADERA: 105.0
    - BRAZO: Grande 32.0, Pequeña 30.0
    - PIERNA: Grande 61.0, Pequeña 49.0
    - PANTORRILLA: 37.0
  - **[SEGUIMIENTO] 2026-09-25**:
    - PESO: 67.3
    - PECHO: 94.0
    - CINTURA: 80.0
    - CADERA: 106.0
    - BRAZO: Grande 32.0, Pequeña 30.0
    - PIERNA: Grande 61.0, Pequeña 51.0
    - PANTORRILLA: 38.0

### 3. Esther Masias Machuca (CEDULA_CIUDADANIA: 1010124952)
- **Teléfono**: 3107569871
- **Plan Actual**: plan_mensual
- **Pago**: Total: 80000 | Pagado: 0 | Estado: PARCIAL | Fecha: 2026-09-07 | Vence: 2026-10-07 | Plazo: 2026-09-30
- **Medidas**:
  - **[BASE] 2026-05-04**:
    - PESO: 86.0
    - PECHO: 111.0
    - CINTURA: 89.0
    - CADERA: 122.0
    - BRAZO: Grande 36.0, Pequeña 35.0
    - PIERNA: Grande 67.0, Pequeña 56.0
    - PANTORRILLA: 42.0
  - **[SEGUIMIENTO] 2026-06-05**:
    - PESO: 82.5
    - PECHO: 106.0
    - CINTURA: 86.0
    - CADERA: 115.0
    - BRAZO: Grande 35.0, Pequeña 34.0
    - PIERNA: Grande 68.0, Pequeña 56.0
    - PANTORRILLA: 42.0
  - **[SEGUIMIENTO] 2026-07-10**:
    - PESO: 81.5
    - PECHO: 104.0
    - CINTURA: 83.0
    - CADERA: 114.0
    - BRAZO: Grande 35.0, Pequeña 34.0
    - PIERNA: Grande 66.0, Pequeña 55.0
    - PANTORRILLA: 43.0
  - **[SEGUIMIENTO] 2026-08-12**:
    - PESO: 78.9
    - PECHO: 101.0
    - CINTURA: 80.0
    - CADERA: 112.0
    - BRAZO: Grande 34.0, Pequeña 33.0
    - PIERNA: Grande 66.0, Pequeña 54.0
    - PANTORRILLA: 42.0

### 4. Katy Angulo Jimenez (CEDULA_CIUDADANIA: 1027283005)
- **Teléfono**: 3212915524
- **Plan Actual**: plan_reto_2x1
- **Pago**: Total: 35000 | Pagado: 35000 | Estado: COMPLETO | Fecha: 2026-09-03 | Vence: 2026-10-21 | Plazo: NINGUNO
- **Medidas**:
  - **[BASE] 2026-08-10**:
    - PESO: 61.0
    - PECHO: 89.0
    - CINTURA: 74.0
    - CADERA: 98.0
    - BRAZO: Grande 27.0, Pequeña 26.0
    - PIERNA: Grande 56.0, Pequeña 48.0
    - PANTORRILLA: 35.0
  - **[SEGUIMIENTO] 2026-09-03**:
    - PESO: 57.8
    - PECHO: 90.0
    - CINTURA: 73.0
    - CADERA: 98.0
    - BRAZO: Grande 27.0, Pequeña 26.0
    - PIERNA: Grande 55.0, Pequeña 48.0
    - PANTORRILLA: 34.0

### 5. Ana Del Carmen Jimenez (CEDULA_CIUDADANIA: 36733696)
- **Teléfono**: 3205096592
- **Plan Actual**: plan_reto_2x1
- **Pago**: Total: 35000 | Pagado: 35000 | Estado: COMPLETO | Fecha: 2026-09-03 | Vence: 2026-10-21 | Plazo: NINGUNO
- **Medidas**:
  - **[BASE] 2026-04-15**:
    - PESO: 87.3
    - PECHO: 111.0
    - CINTURA: 95.0
    - CADERA: 120.0
    - BRAZO: Grande 39.0, Pequeña 37.0
    - PIERNA: Grande 67.0, Pequeña 55.0
    - PANTORRILLA: 45.0
  - **[SEGUIMIENTO] 2026-05-15**:
    - PESO: 86.1
    - PECHO: 108.0
    - CINTURA: 90.0
    - CADERA: 119.0
    - BRAZO: Grande 38.5, Pequeña 36.5
    - PIERNA: Grande 69.0, Pequeña 57.0
    - PANTORRILLA: 45.0
  - **[SEGUIMIENTO] 2026-06-15**:
    - PESO: 82.0
    - PECHO: 106.0
    - CINTURA: 87.0
    - CADERA: 114.0
    - BRAZO: Grande 38.0, Pequeña 36.0
    - PIERNA: Grande 70.0, Pequeña 59.0
    - PANTORRILLA: 43.0
  - **[SEGUIMIENTO] 2026-09-03**:
    - PESO: 81.7
    - PECHO: 106.0
    - CINTURA: 87.0
    - CADERA: 113.0
    - BRAZO: Grande 36.0, Pequeña 35.0
    - PIERNA: Grande 69.0, Pequeña 59.0
    - PANTORRILLA: 41.0
  
### 6. Orlis Casseres (CEDULA_CIUDADANIA: 39462922)
- **Teléfono**: 3135362460
- **Plan Actual**: plan_reto_2x1
- **Pago**: Total: 35000 | Pagado: 0 | Estado: PARCIAL | Fecha: 2026-09-01 | Vence: 2026-09-22 | Plazo: 2026-09-30
- **Medidas**:
  - **[BASE] 2026-05-25**:
    - PESO: 83.5
    - PECHO: 106.0
    - CINTURA: 91.0
    - CADERA: 118.0
    - BRAZO: Grande 35.0, Pequeña 32.0
    - PIERNA: Grande 66.0, Pequeña 55.0
    - PANTORRILLA: 41.0

### 7. Yorgelis Morales (CEDULA_CIUDADANIA: 1082371998)
- **Teléfono**: 3127471618
- **Plan Actual**: plan_reto_2x1
- **Pago**: Total: 35000 | Pagado: 0 | Estado: PARCIAL | Fecha: 2026-09-01 | Vence: 2026-09-22 | Plazo: 2026-09-30
- **Medidas**:
  - **[BASE] 2026-05-25**:
  - PESO: 70.3
  - PECHO: 106.0
  - CINTURA: 90.0
  - CADERA: 99.0
  - BRAZO: Grande 34.0, Pequeña 30.0
  - PIERNA: Grande 56.0, Pequeña 46.0
  - PANTORRILLA: 36.0
- **[SEGUIMIENTO] 2026-06-03**:
  - PESO: 70.3
  - PECHO: 104.0
  - CINTURA: 90.0
  - CADERA: 100.0
  - BRAZO: Grande 32.0, Pequeña 29.0
  - PIERNA: Grande 56.0, Pequeña 50.0
  - PANTORRILLA: 35.0
- **[SEGUIMIENTO] 2026-08-02**:
  - PESO: 68.7
  - PECHO: 102.0
  - CINTURA: 87.0
  - CADERA: 101.0
  - BRAZO: Grande 34.0, Pequeña 29.0
  - PIERNA: Grande 57.0, Pequeña 51.0
  - PANTORRILLA: 36.0
- **[SEGUIMIENTO] 2026-09-01**:
  - PESO: 67.5
  - PECHO: 98.0
  - CINTURA: 87.0
  - CADERA: 99.0
  - BRAZO: Grande 33.0, Pequeña 29.0
  - PIERNA: Grande 56.0, Pequeña 46.0
  - PANTORRILLA: 35.0
- **[SEGUIMIENTO] 2026-09-20**:
  - PESO: 66.5
  - PECHO: 98.0
  - CINTURA: 87.0
  - CADERA: 97.0
  - BRAZO: Grande 32.0, Pequeña 29.0
  - PIERNA: Grande 56.0, Pequeña 47.0
  - PANTORRILLA: 35.0

### 8. Zurley Larios (CEDULA_CIUDADANIA: 26905411)
- **Teléfono**: 3145900766
- **Plan Actual**: plan_retos
- **Pago**: Total: 70000 | Pagado: 70000 | Estado: COMPLETO | Fecha: 2026-09-16 | Vence: 2026-10-28 | Plazo: NINGUNO
- **Medidas**:
  - **[BASE] 2026-06-09**:
    - PESO: 82.0
    - PECHO: 98.0
    - CINTURA: 83.0
    - CADERA: 121.0
    - BRAZO: Grande 35.0, Pequeña 36.0
    - PIERNA: Grande 72.0, Pequeña 62.0
    - PANTORRILLA: 41.0

### 9. Karen Casseres (CEDULA_CIUDADANIA: 1082373158)
- **Teléfono**: 3117951905
- **Plan Actual**: plan_reto_2x1
- **Pago**: Total: 35000 | Pagado: 0 | Estado: PARCIAL | Fecha: 2026-09-16 | Vence: 2026-10-07 | Plazo: 2026-09-29
- **Medidas**:
  - **[BASE] 2026-06-01**:
    - PESO: 80.0
    - PECHO: 100.0
    - CINTURA: 86.0
    - CADERA: 113.0
    - BRAZO: Grande 37.0, Pequeña 36.0
    - PIERNA: Grande 69.0, Pequeña 62.0
    - PANTORRILLA: 42.0

### 10. Viviana Oliveros (CEDULA_CIUDADANIA: 0000000010)
- **Teléfono**: 3218311610
- **Plan Actual**: plan_reto_2x1
- **Pago**: Total: 35000 | Pagado: 35000 | Estado: COMPLETO | Fecha: 2026-08-11 | Vence: 2026-09-22 | Plazo: NINGUNO
- **Medidas**:
- **[BASE] 2026-06-10**:
  - PESO: 73.7
  - PECHO: 98.0
  - CINTURA: 83.0
  - CADERA: 109.0
  - BRAZO: Grande 35.0, Pequeña 33.0
  - PIERNA: Grande 64.0, Pequeña 54.0
  - PANTORRILLA: 38.0
- **[SEGUIMIENTO] 2026-08-11**:
  - PESO: 72.8
  - PECHO: 98.0
  - CINTURA: 82.0
  - CADERA: 108.0
  - BRAZO: Grande 35.0, Pequeña 33.0
  - PIERNA: Grande 68.0, Pequeña 57.0
  - PANTORRILLA: 38.0
- **[SEGUIMIENTO] 2026-09-20**:
  - PESO: 68.2
  - PECHO: 95.0
  - CINTURA: 78.0
  - CADERA: 103.0
  - BRAZO: Grande 35.0, Pequeña 33.0
  - PIERNA: Grande 66.0, Pequeña 57.0
  - PANTORRILLA: 37.0

### 11. Aura Royero (CEDULA_CIUDADANIA: 114161736)
- **Teléfono**: 3008468830
- **Plan Actual**: plan_mensual
- **Pago**: Total: 80000 | Pagado: 0 | Estado: PARCIAL | Fecha: 2026-09-16 | Vence: 2026-10-16 | Plazo: 2026-09-30
- **Medidas**:
  - **[BASE] 2026-07-31**:
    - PESO: 57.5
    - PECHO: 97.0
    - CINTURA: 81.0
    - CADERA: 98.0
    - BRAZO: Grande 31.0, Pequeña 28.0
    - PIERNA: Grande 56.0, Pequeña 49.0
    - PANTORRILLA: 32.0

### 12. Paola Arrieta (CEDULA_CIUDADANIA: 1004320420)
- **Teléfono**: 3205685034
- **Plan Actual**: plan_reto_2x1
- **Pago**: Total: 35000 | Pagado: 35000 | Estado: COMPLETO | Fecha: 2026-09-01 | Vence: 2026-09-22 | Plazo: NINGUNO
- **Medidas**:
  - **[BASE] 2026-06-16**:
    - PESO: 67.2
    - PECHO: 94.0
    - CINTURA: 80.0
    - CADERA: 102.0
    - BRAZO: Grande 31.0, Pequeña 29.0
    - PIERNA: Grande 56.0, Pequeña 46.0
    - PANTORRILLA: 36.0
  - **[SEGUIMIENTO] 2026-07-22**:
    - PESO: 67.5
    - PECHO: 94.0
    - CINTURA: 80.0
    - CADERA: 103.0
    - BRAZO: Grande 31.0, Pequeña 29.0
    - PIERNA: Grande 58.0, Pequeña 48.0
    - PANTORRILLA: 36.0
  - **[SEGUIMIENTO] 2026-08-01**:
    - PESO: 66.2
    - PECHO: 92.0
    - CINTURA: 80.0
    - CADERA: 101.0
    - BRAZO: Grande 31.0, Pequeña 28.0
    - PIERNA: Grande 59.0, Pequeña 49.0
    - PANTORRILLA: 35.6
  - **[SEGUIMIENTO] 2026-08-22**:
    - PESO: 66.3
    - PECHO: 93.0
    - CINTURA: 79.0
    - CADERA: 101.0
    - BRAZO: Grande 31.0, Pequeña 28.0
    - PIERNA: Grande 59.0, Pequeña 49.0
    - PANTORRILLA: 36.0

### 13. Iris Diaz (CEDULA_CIUDADANIA: 49762044)
- **Teléfono**: 3113597580
- **Plan Actual**: plan_mensual
- **Pago**: Total: 80000 | Pagado: 80000 | Estado: COMPLETO | Fecha: 2026-08-28 | Vence: 2026-09-28 | Plazo: NINGUNO
- **Medidas**:
  - **[BASE] 2026-07-28**:
    - PESO: 59.8
    - PECHO: 93.0
    - CINTURA: 79.0
    - CADERA: 98.0
    - BRAZO: Grande 31.0, Pequeña 28.0
    - PIERNA: Grande 54.0, Pequeña 44.0
    - PANTORRILLA: 32.0
  - **[SEGUIMIENTO] 2026-08-28**:
    - PESO: 59.0
    - PECHO: 92.0
    - CINTURA: 78.0
    - CADERA: 97.0
    - BRAZO: Grande 30.0, Pequeña 29.0
    - PIERNA: Grande 56.0, Pequeña 50.0
    - PANTORRILLA: 32.0

### 14. Dalma Kamila Davila Machado (CEDULA_CIUDADANIA: 1082374728)
- **Teléfono**: 3182223885
- **Plan Actual**: plan_reto_2x1
- **Pago**: Total: 35000 | Pagado: 0 | Estado: PARCIAL | Fecha: 2026-09-16 | Vence: 2026-10-07 | Plazo: 2026-09-29
- **Medidas**:
  - **[BASE] 2026-06-29**:
    - PESO: 95.2
    - PECHO: 118.0
    - CINTURA: 102.0
    - CADERA: 124.0
    - BRAZO: Grande 38.0, Pequeña 33.0
    - PIERNA: Grande 66.0, Pequeña 52.0
    - PANTORRILLA: 33.0

### 15. Norma Díaz (CEDULA_CIUDADANIA: 26801269)
- **Teléfono**: NINGUNO
- **Plan Actual**: plan_reto_2x1
- **Pago**: Total: 35000 | Pagado: 35000 | Estado: COMPLETO | Fecha: 2026-09-01 | Vence: 2026-09-22 | Plazo: NINGUNO
- **Medidas**:
  - **[BASE] 2026-08-01**:
    - PESO: 75.3
    - PECHO: 108.0
    - CINTURA: 97.0
    - CINTURA_BAJA: 107.0
    - CADERA: 103.0
    - BRAZO: Grande 34.0, Pequeña 32.0
    - PIERNA: Grande 50.0, Pequeña 44.0
    - PANTORRILLA: 34.0
  - **[SEGUIMIENTO] 2026-08-22**:
    - PESO: 75.3
    - PECHO: 106.0
    - CINTURA: 94.0
    - CINTURA_BAJA: 94.0
    - CADERA: 103.0
    - BRAZO: Grande 34.0, Pequeña 32.0
    - PIERNA: Grande 50.0, Pequeña 44.0
    - PANTORRILLA: 34.0

### 16. Yina Alvarado (CEDULA_CIUDADANIA: 0000000016)
- **Teléfono**: 3193464038
- **Plan Actual**: plan_mensual_70
- **Pago**: Total: 70000 | Pagado: 70000 | Estado: COMPLETO | Fecha: 2026-09-10 | Vence: 2026-10-10 | Plazo: NINGUNO
- **Medidas**:
  - **[BASE] 2026-07-10**:
    - PESO: 77.6
    - PECHO: 103.0
    - CINTURA: 94.0
    - CADERA: 116.0
    - BRAZO: Grande 34.0, Pequeña 30.0
    - PIERNA: Grande 70.0, Pequeña 56.0
    - PANTORRILLA: 37.0
  - **[SEGUIMIENTO] 2026-07-25**:
    - PESO: 77.4
    - PECHO: 103.0
    - CINTURA: 86.0
    - CADERA: 115.0
    - BRAZO: Grande 33.0, Pequeña 31.0
    - PIERNA: Grande 70.0, Pequeña 57.0
    - PANTORRILLA: 37.0
  - **[SEGUIMIENTO] 2026-08-10**:
    - PESO: 76.5
    - PECHO: 103.0
    - CINTURA: 86.0
    - CADERA: 114.0
    - BRAZO: Grande 33.0, Pequeña 31.0
    - PIERNA: Grande 71.0, Pequeña 56.0
    - PANTORRILLA: 37.0

### 17. Zulay Morales (CEDULA_CIUDADANIA: 0000000017)
- **Teléfono**: 3126433187
- **Plan Actual**: plan_mensual
- **Pago**: Total: 80000 | Pagado: 80000 | Estado: COMPLETO | Fecha: 2026-08-27 | Vence: 2026-09-27 | Plazo: NINGUNO
- **Medidas**:
  - **[BASE] 2026-04-07**:
    - PESO: 69.4
    - PECHO: 93.0
    - CINTURA: 79.0
    - CADERA: 111.0
    - BRAZO: Grande 31.0, Pequeña 30.0
    - PIERNA: Grande 65.0, Pequeña 52.0
    - PANTORRILLA: 37.0

### 18. Loly Luz Monterrosa (CEDULA_CIUDADANIA: 0000000018)
- **Teléfono**: NINGUNO
- **Plan Actual**: plan_uso_maquinas_mensual
- **Pago**: Total: 40000 | Pagado: 40000 | Estado: COMPLETO | Fecha: 2026-10-30 | Vence: 2026-11-30 | Plazo: NINGUNO
- **Medidas**:
  - **[BASE] 2026-09-10**:
    - PESO: 62.3
    - PECHO: 99.0
    - CINTURA: 81.0
    - CADERA: 99.0
    - BRAZO: Grande 31.0, Pequeña 26.0
    - PIERNA: Grande 56.0, Pequeña 46.0
    - PANTORRILLA: 35.0

### 19. Angie Niebles (CEDULA_CIUDADANIA: 1003004668)
- **Teléfono**: 3116599940
- **Plan Actual**: plan_mensual_70
- **Pago**: Total: 70000 | Pagado: 70000 | Estado: COMPLETO | Fecha: 2026-08-25 | Vence: 2026-09-25 | Plazo: NINGUNO
- **Medidas**:
  - **[BASE] 2026-04-08**:
    - PESO: 59.2
    - PECHO: 93.0
    - CINTURA: 80.0
    - CADERA: 98.0
    - BRAZO: Grande 30.0, Pequeña 27.0
    - PIERNA: Grande 56.0, Pequeña 48.0
    - PANTORRILLA: 35.0
  - **[SEGUIMIENTO] 2026-04-25**:
    - PESO: 59.0
    - PECHO: 91.0
    - CINTURA: 78.0
    - CADERA: 97.0
    - BRAZO: Grande 31.0, Pequeña 28.0
    - PIERNA: Grande 57.0, Pequeña 48.0
    - PANTORRILLA: 35.0
  - **[SEGUIMIENTO] 2026-09-10**:
    - PESO: 59.0
    - PECHO: 91.0
    - CINTURA: 78.0
    - CADERA: 97.0
    - BRAZO: Grande 31.0, Pequeña 27.0
    - PIERNA: Grande 58.0, Pequeña 50.0
    - PANTORRILLA: 35.0 

### 20. Marisol Mejia Oliveros (CEDULA_CIUDADANIA: 1051655301)
- **Teléfono**: 3106565530
- **Plan Actual**: plan_mensual
- **Pago**: Total: 80000 | Pagado: 80000 | Estado: COMPLETO | Fecha: 2026-08-25 | Vence: 2026-09-25 | Plazo: NINGUNO
- **Medidas**:
  - **[BASE] 2026-04-13**:
    - PESO: 68.4
    - PECHO: 93.0
    - CINTURA: 80.0
    - CADERA: 103.0
    - BRAZO: Grande 29.0, Pequeña 28.0
    - PIERNA: Grande 59.0, Pequeña 51.0
    - PANTORRILLA: 34.0

### 21. Vanesa Perez Amaris (CEDULA_CIUDADANIA: 10823701081)
- **Teléfono**: NINGUNO
- **Plan Actual**: plan_2x1_mensual
- **Pago**: Total: 65000 | Pagado: 60000 | Estado: PARCIAL | Fecha: 2026-09-16 | Vence: 2026-10-16 | Plazo: 2026-10-10
- **Medidas**:
  - **[BASE] 2026-09-16**:
    - PESO: 59.1
    - PECHO: 86.0
    - CINTURA: 70.0
    - CADERA: 102.0
    - BRAZO: Grande 31.0, Pequeña 29.0
    - PIERNA: Grande 58.0, Pequeña 45.0
    - PANTORRILLA: 35.0

### 22. Elianis S/A (CEDULA_CIUDADANIA: 1082362968)
- **Teléfono**: 3107223351
- **Plan Actual**: plan_2x1_mensual
- **Pago**: Total: 65000 | Pagado: 0 | Estado: PARCIAL | Fecha: 2026-09-16 | Vence: 2026-10-16 | Plazo: 2026-10-10
- **Medidas**:
  - **[BASE] 2026-09-16**:
    - PESO: 61.6
    - PECHO: 87.0
    - CINTURA: 71.0
    - CADERA: 106.0
    - BRAZO: Grande 28.0, Pequeña 27.0
    - PIERNA: Grande 60.0, Pequeña 47.0
    - PANTORRILLA: 34.0

### 23. Katya S/A (CEDULA_CIUDADANIA: 1082375864)
- **Teléfono**: 3154271279
- **Plan Actual**: plan_2x1_mensual
- **Pago**: Total: 65000 | Pagado: 65000 | Estado: COMPLETO | Fecha: 2026-09-11 | Vence: 2026-10-11 | Plazo: NINGUNO
- **Medidas**:
  - **[BASE] 2026-08-11**:
    - PESO: 71.8
    - PECHO: 98.0
    - CINTURA: 81.0
    - CADERA: 105.0
    - BRAZO: Grande 31.0, Pequeña 28.0
    - PIERNA: Grande 65.0, Pequeña 56.0
    - PANTORRILLA: 41.0
  - **[SEGUIMIENTO] 2026-09-11**:
    - PESO: 68.9
    - PECHO: 92.0
    - CINTURA: 78.0
    - CADERA: 105.0
    - BRAZO: Grande 30.5, Pequeña 30.0
    - PIERNA: Grande 65.0, Pequeña 57.0
    - PANTORRILLA: 41.0
  
### 24. Isamar Barrios (CEDULA_CIUDADANIA: 1065819279)
- **Teléfono**: 3225962589
- **Plan Actual**: plan_2x1_mensual
- **Pago**: Total: 65000 | Pagado: 0 | Estado: PARCIAL | Fecha: 2026-09-11 | Vence: 2026-10-11 | Plazo: 2026-09-30
- **Medidas**:
  - **[BASE] 2026-08-11**:
    - PESO: 97.9
    - PECHO: 120.0
    - CINTURA: 107.0
    - CINTURA_BAJA: 124.0
    - CADERA: 121.0
    - BRAZO: Grande 41.0, Pequeña 39.0
    - PIERNA: Grande 71.0, Pequeña 61.0
    - PANTORRILLA: 40.0
  - **[SEGUIMIENTO] 2026-09-10**:
    - PESO: 93.9
    - PECHO: 114.0
    - CINTURA: 101.0
    - CINTURA_BAJA: 120.0
    - CADERA: 118.0
    - BRAZO: Grande 41.0, Pequeña 38.0
    - PIERNA: Grande 71.0, Pequeña 62.0
    - PANTORRILLA: 40.0

### 25. Karen Herrera (CEDULA_CIUDADANIA: 0000000025)
- **Teléfono**: 3102723389
- **Plan Actual**: plan_mensual
- **Pago**: Total: 80000 | Pagado: 80000 | Estado: COMPLETO | Fecha: 2026-09-08 | Vence: 2026-10-08 | Plazo: NINGUNO
- **Medidas**:
  - **[BASE] 2026-09-08**:
    - PESO: 72.5
    - PECHO: 106.0
    - CINTURA: 90.0
    - CADERA: 102.0
    - BRAZO: Grande 36.0, Pequeña 34.0
    - PIERNA: Grande 63.0, Pequeña 50.0
    - PANTORRILLA: 36.0
    
### 26. Juana Palomino (CEDULA_CIUDADANIA: 33221141)
- **Teléfono**: 3126124652
- **Plan Actual**: plan_reto_2x1
- **Pago**: Total: 35000 | Pagado: 35000 | Estado: COMPLETO | Fecha: 2026-09-02 | Vence: 2026-09-23 | Plazo: NINGUNO
- **Medidas**:
  - **[BASE] 2026-09-02**:
    - PESO: 70.1
    - PECHO: 101.0
    - CINTURA: 86.0
    - CINTURA_BAJA: 101.0
    - CADERA: 99.0
    - BRAZO: Grande 32.0, Pequeña 30.0
    - PIERNA: Grande 56.0, Pequeña 49.0
    - PANTORRILLA: 34.0
    
### 27. Tatiana Otalora (CEDULA_CIUDADANIA: 0000000027)
- **Teléfono**: 3146111239
- **Plan Actual**: plan_reto_2x1
- **Pago**: Total: 35000 | Pagado: 35000 | Estado: COMPLETO | Fecha: 2026-08-30 | Vence: 2026-09-21 | Plazo: NINGUNO
- **Medidas**:
  - **[BASE] 2026-08-30**:
    - PESO: 98.4
    - PECHO: 103.0
    - CINTURA: 99.0
    - CADERA: 129.0
    - BRAZO: Grande 37.0, Pequeña 34.0
    - PIERNA: Grande 72.0, Pequeña 60.0
    - PANTORRILLA: 38.0
  - **[SEGUIMIENTO] 2026-09-20**:
    - PESO: 95.7
    - PECHO: 111.0
    - CINTURA: 97.0
    - CADERA: 127.0
    - BRAZO: Grande 37.0, Pequeña 33.0
    - PIERNA: Grande 73.0, Pequeña 60.0
    - PANTORRILLA: 38.0
    
### 28. Maryuris S/A (CEDULA_CIUDADANIA: 1004320637)
- **Teléfono**: 3106842522
- **Plan Actual**: plan_reto_2x1
- **Pago**: Total: 35000 | Pagado: 35000 | Estado: COMPLETO | Fecha: 2026-08-30 | Vence: 2026-09-21 | Plazo: NINGUNO
- **Medidas**:
  - **[BASE] 2026-08-30**:
    - PESO: 76.4
    - PECHO: 98.0
    - CINTURA: 85.0
    - CADERA: 107.0
    - BRAZO: Grande 34.0, Pequeña 31.0
    - PIERNA: Grande 62.0, Pequeña 54.0
    - PANTORRILLA: 34.0

### 29. Taikys S/A (CEDULA_CIUDADANIA: 1018494810)
- **Teléfono**: NINGUNO
- **Plan Actual**: plan_2x1_mensual
- **Pago**: Total: 65000 | Pagado: 65000 | Estado: COMPLETO | Fecha: 2026-08-24 | Vence: 2026-10-24 | Plazo: NINGUNO
- **Medidas**:
  - **[BASE] 2026-08-24**:
    - PESO: 94.2
    - PECHO: 109.0
    - CINTURA: 97.0
    - CADERA: 121.0
    - BRAZO: Grande 38.0, Pequeña 35.0
    - PIERNA: Grande 74.0, Pequeña 58.0
    - PANTORRILLA: 43.0

### 30. Dayana Angulo (CEDULA_CIUDADANIA: 118858278)
- **Teléfono**: 3206242401
- **Plan Actual**: plan_reto_2x1
- **Pago**: Total: 35000 | Pagado: 35000 | Estado: COMPLETO | Fecha: 2026-08-31 | Vence: 2026-09-21 | Plazo: NINGUNO
- **Medidas**:
  - **[BASE] 2026-08-31**:
    - PESO: 69.5
    - PECHO: 93.0
    - CINTURA: 80.0
    - CADERA: 103.0
    - BRAZO: Grande 34.0, Pequeña 32.0
    - PIERNA: Grande 63.0, Pequeña 54.0
    - PANTORRILLA: 37.0
