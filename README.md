# 🚀 Refactorización DAX: De Fórmulas Ineficientes a Código Profesional

## 📌 Contexto del Proyecto
En el rol de **Analista de Datos SSR** en *RetailPro*, se heredó un modelo de datos en Power BI basado en el dataset **Sample Superstore**. Si bien las medidas DAX originales devolvían resultados correctos, presentaban serios problemas de rendimiento, redundancia, falta de variables (`VAR`/`RETURN`) y baja legibilidad/mantenibilidad.

El objetivo de esta práctica es refactorizar la lógica DAX aplicando buenas prácticas profesionales, documentar los hallazgos, verificar la coincidencia técnica de resultados mediante una matriz de control y analizar el impacto técnico en el motor de consultas (Formula Engine vs. Storage Engine).

---

## 🛠️ Dataset y Modelo de Datos
* **Dataset:** Sample Superstore (Ventas, Ganancias, Fechas de Pedido, Clientes, Productos, Regiones).
* **Tabla de Fechas (`dim_fechas`):** Relacionada de `1` a `N` (`dim_fechas[Date]` $
ightarrow$ `Ventas[Order Date]`).
* **Tabla de Medidas:** `_Medidas` (organización centralizada de indicadores).

---

## 🔍 Paso 1: Análisis de Diagnóstico (Medidas Originales)

### 1. `Crecimiento Anual`
```dax
Crecimiento Anual = DIVIDE(
    SUM(Ventas[Sales]) - CALCULATE(SUM(Ventas[Sales]), PREVIOUSYEAR(dim_fechas[Date])),
    CALCULATE(SUM(Ventas[Sales]), PREVIOUSYEAR(dim_fechas[Date]))
)
```
* **Diagnóstico:**
  * **Cálculos redundantes:** El cálculo del período anterior (`CALCULATE(..., PREVIOUSYEAR(...))`) se ejecuta **dos veces** por cada celda de evaluación (numerador y denominador).
  * **Mantenibilidad crítica:** Si la regla de negocio para comparar años cambia (ej. usar `SAMEPERIODLASTYEAR` o manejar filtros de contexto adicionales), hay que modificar el código en dos lugares distintos, aumentando la probabilidad de error.

### 2. `Margen %`
```dax
Margen % = DIVIDE(
    SUM(Ventas[Profit]),
    SUM(Ventas[Sales])
) * 100
```
* **Diagnóstico:**
  * **Falta de modularización:** Aunque es una fórmula corta, recalcula agregaciones directamente sin almacenamiento intermedio de contexto.
  * **Formato vs. Multiplicador:** Realizar la multiplicación `* 100` de forma manual dentro del DAX en lugar de delegar el formato al lienzo visual de Power BI dificulta la reusabilidad en tarjetas u otros cálculos derivados.

### 3. `Clasificacion Rendimiento`
```dax
Clasificacion Rendimiento = IF(
    DIVIDE(SUM(Ventas[Profit]), SUM(Ventas[Sales])) * 100 >= 20,
    "Alto",
    IF(
        DIVIDE(SUM(Ventas[Profit]), SUM(Ventas[Sales])) * 100 >= 10,
        "Medio",
        "Bajo"
    )
)
```
* **Diagnóstico:**
  * **Evaluación repetida en `IF` anidados:** La división completa `DIVIDE(SUM(Ventas[Profit]), SUM(Ventas[Sales])) * 100` se ejecuta en la primera condición y, si no se cumple, vuelve a calcularse exactamente igual para evaluar el segundo tramo.
  * **Estructura rígida:** Los `IF` anidados dificultan la lectura rápida y complican la adición de nuevos rangos de clasificación.

---

## ⚙️ Paso 2: Código Refactorizado (`_v2`)

Se reescribieron las tres medidas aplicando patrones profesionales:
1. Almacenamiento previo en variables descriptivas con `VAR`.
2. Reutilización de variables en los pasos subsiguientes.
3. Uso de la función `SWITCH(TRUE(), ...)` para estructuras condicionales complejas.
4. Salida clara con `RETURN`.

```dax
-- =============================================
-- Medida 1: Crecimiento Anual Refactorizado
-- =============================================
Crecimiento Anual_v2 = 
VAR VentasActuales = SUM(Ventas[Sales])
VAR VentasAnoAnterior = CALCULATE(SUM(Ventas[Sales]), PREVIOUSYEAR(dim_fechas[Date]))
VAR Resultado = DIVIDE(VentasActuales - VentasAnoAnterior, VentasAnoAnterior)
RETURN
    Resultado

-- =============================================
-- Medida 2: Margen % Refactorizado
-- =============================================
Margen %_v2 = 
VAR VentasTotales = SUM(Ventas[Sales])
VAR GananciaTotal = SUM(Ventas[Profit])
VAR MargenPorcentaje = DIVIDE(GananciaTotal, VentasTotales) * 100
RETURN
    MargenPorcentaje

-- =============================================
-- Medida 3: Clasificación de Rendimiento Refactorizada
-- =============================================
Clasificacion Rendimiento_v2 = 
VAR VentasTotales = SUM(Ventas[Sales])
VAR GananciaTotal = SUM(Ventas[Profit])
VAR MargenPorcentaje = DIVIDE(GananciaTotal, VentasTotales) * 100
VAR Clasificacion = 
    SWITCH(
        TRUE(),
        MargenPorcentaje >= 20, "Alto",
        MargenPorcentaje >= 10, "Medio",
        "Bajo"
    )
RETURN
    Clasificacion
```

---

## 📊 Paso 3: Matriz de Verificación y Resolución de Errores Comunes

Para garantizar la integridad de los datos, se compararon las 6 medidas (3 originales vs. 3 refactorizadas) en una matriz consolidada por año.

### 🛠️ Resolución de Errores Frecuentes en Power BI Desktop:
1. **Problema de Sumarización en el Campo Año:**
   * *Sintoma:* Al arrastrar `dim_fechas[Año]`, Power BI suma los números de año (ej. `2021 + 2022 = 4043`).
   * *Solución:* En el panel de campos/filas, hacer clic derecho en `Año` y seleccionar **"No resumir"** (*Don't summarize*). Para fijarlo de forma definitiva en el modelo, seleccionar la columna en la vista de modelo y configurar `Default Summarization = Don't summarize`.
2. **Medidas Repetidas por Fila (Problema de Relaciones):**
   * *Sintoma:* El mismo valor se repite en todos los años.
   * *Solución:* Verificar que la relación en la vista de modelo sea de `1` (en `dim_fechas[Date]`) a `N` (en `Ventas[Order Date]`), con dirección de filtro único hacia la tabla de hechos.

### Table de Verificación de Resultados:

| Año | Crecimiento Anual | Crecimiento Anual_v2 | Margen % | Margen %_v2 | Clasificacion Rendimiento | Clasificacion Rendimiento_v2 |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **2014** | — | — | 10.23% | 10.23% | Medio | Medio |
| **2015** | -2.83% | -2.83% | 13.10% | 13.10% | Medio | Medio |
| **2016** | 29.47% | 29.47% | 13.43% | 13.43% | Medio | Medio |
| **2017** | 20.36% | 20.36% | 12.74% | 12.74% | Medio | Medio |

*Confirmación:* Los resultados de la versión `_v2` son **100% idénticos** a las versiones originales en todos los contextos de filtro.

---

## ⏱️ Paso 4: Análisis de Rendimiento con DAX Studio (Server Timing)

Al evaluar las consultas mediante la herramienta **Server Timing** de DAX Studio, se analiza el comportamiento interno del motor de Power BI:

### 1. Componentes del Engine:
* **Formula Engine (FE):** Procesa la lógica de negocio, estructuras de control (`IF`, `SWITCH`), uniones dinámicas y formato. Trabaja en un **único hilo** (monohilo).
* **Storage Engine (SE / VertiPaq):** Escanea, filtra y agrupa volúmenes masivos de datos comprimidos en memoria RAM. Trabaja en **múltiples hilos** (multihilo) y es extremadamente rápido.

### 2. Impacto de la Refactorización:
* **Reducción de Callback DataID / Evaluaciones en FE:** En la medida original de Clasificación, el cálculo se enviaba en múltiples pasos al Formula Engine para resolver cada rama condicional del `IF`. Con `VAR` y `SWITCH(TRUE())`, el valor se materializa una sola vez en la memoria local de la consulta.
* **Reducción de SE Queries:** Se eliminan llamadas duplicadas al motor de almacenamiento para recuperar la misma suma de ventas/ganancias en el mismo contexto.

### 3. ¿Por qué es crucial medir antes de optimizar?
* **Evita la optimización a ciegas:** Permite diagnosticar si el cuello de botella proviene de un modelo mal diseñado (Storage Engine) o de fórmulas ineficientes (Formula Engine).
* **Priorización de impacto:** Asegura que el tiempo del analista se invierta en las medidas con alto costo computacional en tableros con millones de filas.

---
