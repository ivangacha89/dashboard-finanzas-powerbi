# 📏 Documentación de Medidas DAX

---

## 🗂️ Índice

1. [Medidas Base](#1-medidas-base)
2. [Medidas Financieras](#2-medidas-financieras)
3. [Variaciones y Presupuesto](#3-variaciones-y-presupuesto)
4. [Inteligencia de Tiempo](#4-inteligencia-de-tiempo)
5. [Visualización y Formato](#5-visualización-y-formato)
6. [Otras Medidas](#6-otras-medidas)

---

## 1. Medidas Base

Estas medidas son la base del modelo. Todas las demás medidas financieras se construyen sobre ellas, por lo que cualquier ajuste en este nivel impacta el resto del reporte.

---

### `Monto Total`

```dax
Monto Total = SUM(Finanzas[Monto])
```

**¿Para qué se usa?**  
Suma todos los montos de la tabla de hechos `Finanzas`, sin filtrar por escenario. Es la medida raíz del modelo: agrega tanto los valores reales como los presupuestados. No se usa directamente en visualizaciones, sino como insumo para `Monto Real` y `Monto Presupuesto`.

---

### `Monto Real`

```dax
Monto Real = 
CALCULATE(
    [Monto Total],
    Escenario[Escenario]="Real"
)
```

**¿Para qué se usa?**  
Filtra `Monto Total` para retornar únicamente los valores del escenario **Real**, es decir, lo que efectivamente se ejecutó. Es la medida central del dashboard: alimenta las tarjetas KPI, los gráficos de evolución mensual, el estado de resultados y el waterfall financiero.

---

### `Monto Presupuesto`

```dax
Monto Presupuesto = 
CALCULATE(
    [Monto Total],
    Escenario[Escenario]="Presupuesto"
)
```

**¿Para qué se usa?**  
Filtra `Monto Total` para retornar únicamente los valores del escenario **Presupuesto**, que representa la meta aprobada al inicio del año. Se usa en las comparaciones Real vs Presupuesto, en el cálculo de variaciones y en el texto ejecutivo dinámico.

---

## 2. Medidas Financieras

Estas medidas descomponen los montos reales en las categorías del estado de resultados, utilizando la tabla `Cuentas` (columna `Tipo`) como clasificador.

---

### `Ingresos`

```dax
Ingresos = 
CALCULATE(
    [Monto Real],
    Cuentas[Tipo]="Ingreso"
)
```

**¿Para qué se usa?**  
Suma los montos reales de todas las cuentas clasificadas como **Ingreso** en `Cuentas` (ventas de productos y servicios profesionales). Es el punto de partida del estado de resultados y el denominador del cálculo de margen. Se muestra en tarjetas KPI y en el waterfall financiero.

---

### `Costos`

```dax
Costos = 
ABS(
    CALCULATE(
        [Monto Real],
        Cuentas[Tipo]="Costo"
    )
)
```

**¿Para qué se usa?**  
Suma los montos reales de cuentas clasificadas como **Costo** (costo de mercadería, servicios, depreciación, intereses e impuesto a la renta) y los convierte a valor absoluto. El `ABS()` es necesario porque estos montos están registrados en negativo en la fuente de datos. Se usa en el estado de resultados y en el cálculo de utilidad.

---

### `Gastos`

```dax
Gastos = 
ABS(
    CALCULATE(
        [Monto Real],
        Cuentas[Tipo]="Gasto Operacional"
    )    
)
```

**¿Para qué se usa?**  
Suma los montos reales de cuentas clasificadas como **Gasto Operacional** (sueldos, marketing digital, arriendo, software y licencias) y los expresa en positivo mediante `ABS()`. Se usa en el estado de resultados, en el gráfico de gastos por centro de costo y en el cálculo de utilidad.

---

### `Utilidad`

```dax
Utilidad = [Ingresos] - [Costos] - [Gastos]
```

**¿Para qué se usa?**  
Calcula la **utilidad neta** del período como la diferencia entre ingresos y la suma de costos y gastos operacionales. Es el resultado financiero principal del reporte y se presenta en tarjetas KPI, en el estado de resultados y en el gráfico de evolución de utilidad.

---

### `Margen %`

```dax
Margen % = 
DIVIDE(
    [Utilidad],
    [Ingresos],
    0
)
```

**¿Para qué se usa?**  
Calcula qué porcentaje de los ingresos se convierte en utilidad neta. El tercer argumento de `DIVIDE()` retorna `0` en caso de que los ingresos sean cero, evitando errores de división. Se muestra como indicador de eficiencia en el estado de resultados y en tarjetas KPI.

---

## 3. Variaciones y Presupuesto

Estas medidas permiten comparar la ejecución real frente al presupuesto, identificando brechas favorables o desfavorables.

---

### `Variacion vs Presupuesto`

```dax
Variacion vs Presupuesto = [Monto Real] - [Monto Presupuesto]
```

**¿Para qué se usa?**  
Calcula la diferencia absoluta entre el monto real y el presupuesto. Un valor positivo puede indicar superávit en ingresos o sobrecosto según el contexto. Es la base para `Variacion %`, `Indicador Variación` y el texto ejecutivo dinámico.

---

### `Variacion %`

```dax
Variacion % = 
DIVIDE(
    [Variacion vs Presupuesto],
    ABS([Monto Presupuesto]),
    0
)
```

**¿Para qué se usa?**  
Expresa la variación frente al presupuesto en términos porcentuales. El `ABS()` en el denominador normaliza el cálculo cuando el presupuesto es negativo (como ocurre con costos y gastos). Se usa en tablas comparativas y tarjetas de variación.

---

### `Cumplimiento Presupuesto %`

```dax
Cumplimiento Presupuesto % = 
DIVIDE(
    [Monto Real],
    [Monto Presupuesto],
    0
)
```

**¿Para qué se usa?**  
Indica qué proporción del presupuesto fue ejecutada. Un valor de `1.0` (100%) significa que se ejecutó exactamente lo presupuestado. Se usa en el texto ejecutivo dinámico para comunicar el nivel de cumplimiento de manera comprensible para la gerencia.

---

### `Indicador Variación`

```dax
Indicador Variación = 
IF(
    [Variacion vs Presupuesto] >= 0,
    "🔴 Desfavorable",
    "✅ Favorable"
)
```

**¿Para qué se usa?**  
Convierte la variación numérica en una etiqueta visual con emoji para facilitar la lectura ejecutiva. La lógica es financiera: una variación positiva en el monto total indica que los egresos superaron lo esperado (desfavorable), mientras que una variación negativa puede indicar ahorro (favorable). Se usa en tarjetas KPI y en el resumen ejecutivo.

> ⚠️ **Nota:** Esta lógica aplica correctamente para el monto agregado del modelo. Si se usa en contextos donde se filtra solo por ingresos, el signo se invertiría (más ingreso = favorable).

---

## 4. Inteligencia de Tiempo

Medidas que usan funciones de inteligencia de tiempo de DAX para análisis de tendencias y acumulados.

---

### `Real Mes Anterior`

```dax
Real Mes Anterior = 
CALCULATE(
    [Monto Real],
    PREVIOUSMONTH(Calendario[Date])
)
```

**¿Para qué se usa?**  
Calcula el monto real del mes inmediatamente anterior al período seleccionado, usando la tabla `Calendario` como referencia. Requiere que el modelo tenga una tabla de calendario con relación activa a la tabla de hechos. Se usa como base para calcular la variación mensual.

---

### `Variacion Mes Anterior`

```dax
Variacion Mes Anterior = [Monto Real] - [Real Mes Anterior]
```

**¿Para qué se usa?**  
Calcula la diferencia entre el monto real del período actual y el del mes anterior, permitiendo identificar si la ejecución mejoró o empeoró respecto al período previo. Base para `Variacion % Mes Anterior`.

---

### `Variacion % Mes Anterior`

```dax
Variacion % Mes Anterior = 
DIVIDE(
    [Variacion Mes Anterior],
    [Real Mes Anterior],
    0
)
```

**¿Para qué se usa?**  
Expresa en porcentaje el cambio del monto real respecto al mes anterior. Se usa en tarjetas KPI para mostrar tendencia de crecimiento o decrecimiento mensual de manera porcentual.

---

### `Monto Real YTD`

```dax
Monto Real YTD = 
TOTALYTD(
    [Monto Real],
    Calendario[Date]
)
```

**¿Para qué se usa?**  
Acumula el monto real desde el inicio del año hasta la fecha seleccionada (*Year-To-Date*). Permite ver la ejecución acumulada en lo que va del año, útil para gráficos de evolución mensual donde se quiere mostrar el avance acumulado en lugar del valor puntual del mes.

---

## 5. Visualización y Formato

Medidas orientadas a controlar cómo se presenta la información en las visualizaciones del dashboard.

---

### `Valor Estado Resultados`

```dax
Valor Estado Resultados = 
VAR _concepto = SELECTEDVALUE(TablaEstadoResultados[Concepto])
RETURN
SWITCH(
    _concepto,
    "Ingresos",         [Ingresos],
    "(-) Costos",       [Costos],
    "(-) Gastos Oper.", [Gastos],
    "Utilidad neta",    [Utilidad],
    "Margen %",         [Margen %],
    BLANK()
)
```

**¿Para qué se usa?**  
Es la medida dinámica que alimenta la visual del **Estado de Resultados**. Dado que ese componente usa una tabla de parámetros (`TablaEstadoResultados`) con los conceptos del P&L, esta medida detecta qué fila está siendo evaluada y devuelve el valor de la medida DAX correspondiente. Sin esta medida, el estado de resultados no podría funcionar como una tabla dinámica.

---

### `Valor Formateado Estado Resultados`

```dax
Valor Formateado Estado Resultados = 
VAR _concepto = SELECTEDVALUE(TablaEstadoResultados[Concepto])
VAR _val = [Valor Estado Resultados]
RETURN
IF(
    _concepto = "Margen %",
    FORMAT(_val, "0.00%"),
    FORMAT(_val, "$ #,##0.00")
)
```

**¿Para qué se usa?**  
Aplica el formato correcto según la fila del estado de resultados: porcentaje para `Margen %` y formato monetario para el resto. Se usa cuando se muestra el estado de resultados como una tarjeta o tabla de una sola columna, donde el formato debe ajustarse por fila en lugar de ser uniforme.

---

### `Color Valor ER`

```dax
Color Valor ER = 
SWITCH(
    SELECTEDVALUE(TablaEstadoResultados[Concepto]),
    "Ingresos",         "#86EFAC",
    "(-) Costos",       "#FCA5A5",
    "(-) Gastos Oper.", "#FCA5A5",
    "Utilidad neta",    "#7EB8F7",
    "Margen %",         "#FCD34D",
    "#94A3B8"
)
```

**¿Para qué se usa?**  
Asigna un color HEX a cada fila del estado de resultados para usarlo como color condicional en la visual. Permite que cada concepto tenga su propio color sin necesidad de reglas de formato condicional manuales en Power BI:

| Concepto | Color | Significado |
|---|---|---|
| Ingresos | `#86EFAC` Verde claro | Positivo / generador de valor |
| Costos y Gastos | `#FCA5A5` Rojo claro | Egresos / reductores de utilidad |
| Utilidad neta | `#7EB8F7` Azul claro | Resultado final |
| Margen % | `#FCD34D` Amarillo | Indicador de eficiencia |
| Default | `#94A3B8` Gris | Filas sin clasificar |

---

### `Texto Ejecutivo Dinamico`

```dax
Texto Ejecutivo Dinamico = 
VAR REAL = FORMAT([Monto Real], "#,##0.00")
VAR PPTO = FORMAT([Monto Presupuesto], "#,##0.00")
VAR VARIACION = [Variacion vs Presupuesto]
VAR CUMPLIMIENTO = FORMAT(ABS([Cumplimiento Presupuesto %] - 1), "0.0%")
VAR SIGNO = IF(VARIACION >= 0, "superó", "quedó por debajo de")
RETURN
"El monto real fue $" & REAL &
" frente a un presupuesto de $" & PPTO &
". La ejecución " & SIGNO & " el presupuesto en " & CUMPLIMIENTO & "."
```

**¿Para qué se usa?**  
Genera automáticamente un párrafo narrativo que resume la ejecución financiera del período seleccionado. Se actualiza dinámicamente con cada interacción de filtro (año, mes, centro de costo). Es la medida que alimenta el componente de **Texto ejecutivo dinámico** del dashboard, permitiendo que cualquier usuario entienda el resultado sin necesidad de interpretar los gráficos.

**Ejemplo de salida:**  
> *"El monto real fue $1,234,567.00 frente a un presupuesto de $1,200,000.00. La ejecución superó el presupuesto en 2.9%."*

---

### `Titulo_Dinamico_Grafico_Real_vs_Presupuesto`

```dax
Titulo_Dinamico_Grafico_Real_vs_Presupuesto = 
"Real vs Presupuesto - " &
SELECTEDVALUE(Calendario[Año], "Todos los años")
```

**¿Para qué se usa?**  
Genera el título dinámico del gráfico de comparación Real vs Presupuesto. Cuando el usuario selecciona un año en el filtro, el título se actualiza automáticamente (ej. *"Real vs Presupuesto - 2024"*). Si no hay año seleccionado, muestra *"Real vs Presupuesto - Todos los años"*. Mejora la experiencia ejecutiva del dashboard.

---

### `Titulo_Dinamico_Grafico_Utilidad`

```dax
Titulo_Dinamico_Grafico_Utilidad = 
"Utilidad - " &
SELECTEDVALUE(Calendario[Año], "Todos los años")
```

**¿Para qué se usa?**  
Funciona igual que la medida anterior pero para el gráfico de evolución de utilidad. Permite que ambos gráficos muestren siempre el año en contexto, evitando que el usuario tenga que recordar qué filtro está activo.

---

## 6. Otras Medidas

---

### `Ranking CentroCosto`

```dax
Ranking CentroCosto = 
IF(
    ISINSCOPE(CentroCosto[CentroCosto]),
    RANKX(
        ALL(CentroCosto[CentroCosto]),
        [Monto Real],
        ,
        DESC,
        Dense
    ),
    BLANK()
)
```

**¿Para qué se usa?**  
Asigna un ranking numérico a cada centro de costo según su monto real de mayor a menor. El uso de `ISINSCOPE()` es clave: hace que el ranking solo aparezca cuando la visual está desglosada por centro de costo, devolviendo `BLANK()` en totales o subtotales para evitar números sin sentido. Se usa en la visual de **gastos por centro de costo** para identificar cuáles centros concentran mayor ejecución.

---

## 📌 Dependencias del Modelo

Para que estas medidas funcionen correctamente, el modelo debe tener las siguientes tablas y relaciones configuradas:

| Tabla | Tipo | Descripción |
|---|---|---|
| `Finanzas` | Fact | Tabla de hechos con los montos transaccionales |
| `Dim_Cuentas` / `Cuentas` | Dimensión | Clasificación de cuentas por tipo (Ingreso, Costo, Gasto Operacional) |
| `Dim_Escenario` / `Escenario` | Dimensión | Clasificación Real vs Presupuesto |
| `Dim_CentroCosto` / `CentroCosto` | Dimensión | Centros de costo con área y responsable |
| `Calendario` | Dimensión de tiempo | Tabla de fechas con columnas Año, Mes, Date |
| `TablaEstadoResultados` | Parámetro | Tabla auxiliar con los conceptos del P&L |

---

*Documentación generada para el repositorio del proyecto — Power BI Avanzado Finanzas*
