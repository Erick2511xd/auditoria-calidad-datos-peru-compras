# Auditoría Automatizada de Calidad de Datos y Arquitectura Medallion – PERÚ COMPRAS

Construí un pipeline de auditoría de calidad de datos en **Databricks (PySpark y Delta Lake)** sobre **17,623 órdenes de compra reales** de los Catálogos Electrónicos de PERÚ COMPRAS (periodos 2026-08 a 2026-10), publicadas en la Plataforma Nacional de Datos Abiertos del Perú.

## Resultado en una línea
Auditó 17,623 órdenes reales con 4 reglas de calidad y **no encontró incumplimientos**. Para comprobar que el motor funciona, inyecté 759 anomalías controladas y **detectó el 100%**.

## Arquitectura del flujo (Medallion + Quarantine)
1. **Bronze (`bronze_peru_compras`):** ingesta cruda desde Unity Catalog Volumes sin inferir esquema (para no perder registros corruptos), estandarización de encabezados (sin tildes) y metadatos de trazabilidad (`fecha_auditoria` y `archivo_origen`).
2. **Motor de reglas de calidad (basado en DAMA-DMBOK2):** cada registro guarda una lista (`ARRAY`) con todas las reglas que incumple:
   * **Unicidad:** órdenes electrónicas repetidas o nulas.
   * **Validez de RUC:** 11 dígitos que empiezan con 10 o 20 (formato, no dígito verificador).
   * **Consistencia financiera:** `SUB_TOTAL + IGV = TOTAL` con tolerancia de S/ 0.10, y monto mayor a cero.
   * **Integridad temporal:** `FECHA_ULTIMO_ESTADO` no puede ser anterior a `FECHA_FORMALIZACION`.
3. **Silver y Quarantine:** los registros que cumplen todas las reglas pasan a Silver con tipos de datos estandarizados. Los que incumplen alguna van a Quarantine con sus motivos de rechazo, sin detener el pipeline.
4. **Gold:** resumen de la corrida real, scorecard de anomalías por tipo de regla y scorecard de la prueba negativa.
5. **Gobierno en Unity Catalog:** comentarios a nivel de tabla y columna (`COMMENT ON TABLE` y `ALTER COLUMN`). Las tablas de prueba están marcadas como tales.

## Resultados

### Corrida real (datos sin modificar)
| Capa | Registros |
|---|---|
| Auditados (Bronze) | 17,623 |
| Silver (aprobados) | 17,623 |
| Quarantine (observados) | 0 |

La fuente pública cumple las 4 reglas definidas.

### Prueba negativa (anomalías inyectadas)
Para validar el motor, modifiqué de forma determinística registros reales (ordenados por `ORDEN_ELECTRONICA`) en tres campos: RUC del proveedor (cada 50 filas), TOTAL (cada 65) y fecha de último estado (cada 80).

| Dato | Valor |
|---|---|
| Anomalías esperadas (por conteo) | 759 |
| Anomalías detectadas por el motor | 759 |
| Detección | 100% |

Los resultados de la prueba están en tablas separadas (`*_test`) para no mezclarlos con los datos reales. El detalle por regla está en `gold_scorecard_prueba_negativa`.

## Limitaciones
* Las reglas validan formato y coherencia interna; no verifican el dígito verificador del RUC ni contrastan los datos con otra fuente.
* Que la fuente salga limpia significa que cumple estas 4 reglas, no que esté libre de cualquier error.
* La carga es completa (`overwrite`), no incremental.

## Tecnologías
* **Plataforma y procesamiento:** Databricks (Free Edition), PySpark, Delta Lake, Unity Catalog.
* **Enfoque:** arquitectura Medallion, Data Quality Gate, Negative Testing, DAMA-DMBOK2.
* **Fuente de datos:** Plataforma Nacional de Datos Abiertos del Perú – Central de Compras Públicas (PERÚ COMPRAS).
