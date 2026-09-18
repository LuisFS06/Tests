# Prompt para Claude Code: análisis de estabilización de vinculación (nuevos vs. maduros)

> Usar en **modo plan** (Shift+Tab dos veces) antes de pegar el prompt.
> Reemplazar los valores entre `<...>` con los nombres reales.

---

Necesito que primero diseñes un PLAN (no escribas código todavía) para un análisis exploratorio de vinculación de clientes. Hazme las preguntas que necesites antes de cerrar el plan.

## Contexto

- Banco con un indicador de vinculación de clientes con 5 categorías ordinales (1 = menor, 5 = mayor), calculado a partir de la tenencia de varios productos.
- Fuente: SQL Server. Tabla `<esquema.tabla_vinculacion>` con ~7 millones de registros por mes y 24 meses de historia (~170M filas).
- Columnas clave: `<cliente_id>`, `<mes>` (periodo), `<categoria>`, `<fecha_alta_cliente>`. Hay más columnas de productos que por ahora no son necesarias.
- Stack: SQL Server para el procesamiento pesado; Python con Polars para el análisis (evitar pandas salvo para gráficos). Extracción vía connectorx o arrow-odbc, lo que sea más eficiente.

## Objetivo del negocio

Separar el comportamiento de clientes nuevos vs. antiguos para:

1. Medir cuánto tardan los nuevos en estabilizarse en una categoría y cómo se mueven antes de hacerlo.
2. Comparar sus distribuciones y dinámicas con las de los clientes maduros.
3. Evaluar si esto sirve para fijar metas diferenciadas: retención de vinculación para maduros y crecimiento o velocidad de vinculación para nuevos.

El análisis es **EXPLORATORIO, no confirmatorio**: la hipótesis de que los maduros son estables y los nuevos tienen más potencial puede ser incorrecta, y el diseño debe permitir descubrirlo.

## Componentes que debe cubrir el plan

### 1. Máquina de estados eficiente (en SQL Server)

- Comprimir el panel mensual en una tabla de **TRAMOS** (run-length / gaps-and-islands): una fila por periodo continuo en la misma categoría: `cliente_id, tramo, categoria, mes_inicio, mes_fin, n_meses, es_abierto`.
- Estados adicionales explícitos: `AUSENTE` (el cliente no aparece un mes) y `BAJA` (sale definitivamente; definir la regla, p. ej. N meses ausente). Esto es crítico para evitar sesgo de supervivencia.
- Carga inicial con `LAG` + `SUM` acumulada sobre los 24 meses.
- Carga incremental mensual: comparar el mes nuevo solo contra el tramo abierto de cada cliente (`MERGE` o `UPDATE` + `INSERT`) y extender o cerrar el tramo según corresponda.
- Proponer índices o columnstore adecuados, particionamiento por mes si aplica, y cómo validar que la tabla de tramos reconstruye exactamente el panel original.

### 2. Definición de cohortes

- Usar la fecha de alta real, no la primera aparición en la ventana.
- Tramos de antigüedad: 0–3, 4–6, 7–12, 13–24, >24 meses (parametrizable).
- Marcar la censura por la izquierda: los clientes con alta anterior a la ventana no tienen historia temprana observable y solo sirven como referencia de "maduros".
- Cohortes de alta por trimestre (y por canal si existe esa columna) para detectar efectos de cohorte.

### 3. Análisis exploratorios (Python + Polars)

a) **Matrices de transición** mensuales (5 categorías + BAJA) por tramo de antigüedad, con conteos y probabilidades. Comparar nuevos vs. maduros. Calcular la distribución estacionaria de la matriz de maduros.

b) **Curvas de distribución por mes de antigüedad** para las cohortes nuevas, con su distancia a la distribución de maduros (Jensen-Shannon y/o Hellinger) mes a mes. Identificar el punto donde la curva se aplana.

c) **Estabilización a nivel cliente**: primer mes a partir del cual el cliente permanece k meses consecutivos en la misma categoría (k parametrizable, default 6). Estimar el tiempo a estabilización con Kaplan-Meier (lifelines), tratando como censurados a quienes aún no se estabilizan, y desagregar por categoría final.

d) **Volatilidad**: número de cambios de categoría y proporción de subidas vs. bajadas por mes de antigüedad.

e) **Descomposición shift-share** de la variación mensual del indicador agregado en efecto mezcla (proporción nuevos/maduros) vs. efecto desempeño (cambio dentro de cada segmento).

### 4. Insumos para metas

- **Maduros**: tasa mensual de descenso de categoría, tasa de baja, % que se mantiene o sube.
- **Nuevos**: % que alcanza categoría ≥3 a los 6 y 12 meses, y tiempo mediano para lograrlo.
- Proponer cómo usar estas métricas como línea base para metas, dejando claro qué es observación y qué es supuesto.

### 5. Controles y riesgos

- Revisar si la definición del indicador cambió en el periodo.
- Controlar estacionalidad (cierres de año, campañas).
- Documentar supuestos y parámetros en un archivo de configuración.

## Entregables esperados

- Scripts SQL: creación de la tabla de tramos, carga inicial, carga incremental y validación.
- Módulos Python organizados (extracción, transformación, métricas, visualización), con parámetros centralizados.
- Notebook o reporte exploratorio con gráficos de cada análisis.
- Un README con la lógica de estados y las decisiones tomadas.

## Restricciones

- Eficiencia primero: el cálculo pesado se hace en SQL Server y a Python solo se traen agregados o la tabla de tramos. Nunca se trae el panel completo a memoria.
- Todo debe ser reproducible y parametrizable (ventana de fechas, k, tramos de antigüedad).

Entrega el plan por fases con dependencias, estimando el volumen esperado de cada tabla intermedia, e indica qué información me falta darte.
