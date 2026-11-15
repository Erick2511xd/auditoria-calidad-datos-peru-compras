# Auditoría Automatizada de Calidad de Datos y Arquitectura Medallion – PERÚ COMPRAS

Construí en **Databricks (PySpark y Delta Lake)** un pipeline que audita la calidad de **17,623 órdenes de compra reales** de los Catálogos Electrónicos de PERÚ COMPRAS (periodos 2026-08 a 2026-10), publicadas en la Plataforma Nacional de Datos Abiertos del Perú.

El código completo, con sus resultados, está en el notebook [`Auditoria_Calidad_Datos_Medallion.ipynb`](Auditoria_Calidad_Datos_Medallion.ipynb).

## Resultado en una línea
Audité las 17,623 órdenes con 4 reglas de calidad y **no encontré incumplimientos**. Para comprobar que mi motor de reglas funciona, inyecté 759 anomalías controladas y **detectó el 100%**.

## Arquitectura del flujo (Medallion + Quarantine)
1. **Bronze (`bronze_peru_compras`):** cargué los archivos crudos desde Unity Catalog Volumes sin inferir el esquema, para no perder registros corruptos. Estandaricé los encabezados (sin tildes) y agregué metadatos de trazabilidad (`fecha_auditoria` y `archivo_origen`).
2. **Motor de reglas de calidad (basado en DAMA-DMBOK2):** definí 4 reglas. Cada registro guarda una lista (`ARRAY`) con todas las que incumple:
   * **Unicidad:** órdenes electrónicas repetidas o nulas.
   * **Validez de RUC:** 11 dígitos que empiezan con 10 o 20 (el RUC de la entidad solo con 20). Valido el formato, no el dígito verificador.
   * **Consistencia financiera:** `SUB_TOTAL + IGV = TOTAL` con tolerancia de S/ 0.10, y monto mayor a cero.
   * **Integridad temporal:** `FECHA_ULTIMO_ESTADO` no puede ser anterior a `FECHA_FORMALIZACION`.
3. **Silver y Quarantine:** los registros que cumplen todas las reglas pasan a Silver, con los tipos de datos estandarizados. Los que incumplen alguna van a Quarantine con sus motivos de rechazo, y el pipeline no se detiene.
4. **Gold:** generé un resumen de la corrida real, un scorecard de anomalías por regla y un scorecard de la prueba negativa.
5. **Gobierno en Unity Catalog:** documenté las tablas y columnas con `COMMENT ON TABLE` y `ALTER COLUMN`, y marqué las tablas de prueba como tales.

## Resultados

### Corrida real (datos sin modificar)
| Capa | Registros |
|---|---|
| Auditados (Bronze) | 17,623 |
| Silver (aprobados) | 17,623 |
| Quarantine (observados) | 0 |

La fuente pública cumple las 4 reglas que definí.

### Prueba negativa (anomalías inyectadas)
Para validar el motor, modifiqué registros reales de forma determinística (ordenados por `ORDEN_ELECTRONICA`) en tres campos: RUC del proveedor (cada 50 filas), TOTAL (cada 65) y fecha de último estado (cada 80).

| Dato | Valor |
|---|---|
|
