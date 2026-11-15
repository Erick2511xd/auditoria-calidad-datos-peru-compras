# Auditoría Automatizada de Calidad de Datos y Arquitectura Medallion – PERÚ COMPRAS

Pipeline de aseguramiento de calidad y gobierno de datos implementado en **Databricks (PySpark y Delta Lake)** sobre **17,623 órdenes de compra reales** de los Catálogos Electrónicos de PERÚ COMPRAS (periodos 2026-08 a 2026-10).

## Arquitectura del Flujo (Medallion + Quarantine)
1. **Capa Bronze (`bronze_peru_compras`):** Ingesta cruda desde Unity Catalog Volumes sin inferencia de esquema, estandarización de encabezados (eliminación de tildes) y registro de metadatos técnicos de trazabilidad (`fecha_auditoria` y `archivo_origen`).
2. **Motor de Reglas de Calidad (DAMA-DMBOK2 & Negative Testing):** Validación de unicidad, formato oficial de RUC (SUNAT), consistencia matemática (`SUB_TOTAL + IGV = TOTAL`) e integridad temporal, incluyendo pruebas negativas controladas sobre el ~4% de los registros para validar el aislamiento de defectos.
3. **Bifurcación Automatizada (`Silver` vs. `Quarantine`):**
   * **Capa Silver (`silver_peru_compras`):** **16,863 registros (95.69%)** aprobados al 100% con tipos de datos estandarizados.
   * **Capa Quarantine (`quarantine_peru_compras`):** **760 registros (4.31%)** aislados sin detener el pipeline, etiquetados con una lista (`ARRAY`) de todos los motivos de rechazo acumulados.
4. **Capa Gold (`gold_scorecard_calidad_peru_compras`):** Desglose ejecutivo (`explode`) por tipo de anomalía y cuantificación financiera del monto observado en soles.
5. **Gobierno en Unity Catalog:** Documentación de metadatos y reglas de negocio a nivel de tabla y columna mediante SQL (`COMMENT ON TABLE` / `ALTER COLUMN`).

## Principales Hallazgos (Capa Gold)
* **RUC de Proveedor Inválido:** 353 registros observados (S/ 3,435,585.20).
* **Inconsistencia Financiera:** 272 registros observados (-S/ 32,640.00).
* **Fecha Inconsistente:** 221 registros observados (S/ 2,181,284.69).

## Tecnologías Utilizadas
* **Plataforma y Procesamiento:** Databricks, PySpark, Delta Lake, Unity Catalog.
* **Enfoque:** Arquitectura Medallion, Data Quality Gate, Negative Testing, DAMA-DMBOK2.
* **Fuente de Datos:** Plataforma Nacional de Datos Abiertos del Perú – Central de Compras Públicas (PERÚ COMPRAS).
