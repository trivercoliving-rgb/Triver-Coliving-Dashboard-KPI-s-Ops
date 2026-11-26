# BLUEPRINT — Pestaña "Evolución & YTD" · Dashboard TRIVER KPIs

**Versión:** 1.1 · **Fecha:** 26/11/2026 · **Responsable:** Diego Medina (Head of Operations)
**Repo:** `trivercoliving-rgb/Triver-Coliving-Dashboard-KPI-s-Ops` · **Base:** `main` posterior al commit `3bf9be6` ("Modo visor cifrado v1")
**Tiempo objetivo:** ≤ 1.5 h de desarrollo

---

## 1. Objetivo

Agregar una pestaña nueva, **"Evolución & YTD"**, que muestre en tablas y gráficas la evolución mes a mes y el acumulado del año de todos los KPIs del dashboard. Debe verse igual en el **modo editor** y en el **modo visor**.

## 2. Qué NO es

- No reemplaza "Tendencia — Últimos 6 Meses" ni "YTD Score Analysis". Esas secciones se quedan como están.
- No captura datos nuevos. **Todo se calcula con datos que ya existen** en `appData.meses`.
- No modifica procesadores, reglas de conteo de QA ni fórmulas existentes.

## 3. Ventana de tiempo

- **Columnas de meses:** de Enero al mes seleccionado en `#monthSelector`, del **mismo año**.
- **YTD:** acumulado de esos mismos meses. Si Diego selecciona Agosto 2026, el YTD es Ene–Ago 2026.
- Debajo de cada valor YTD se muestra cuántos meses tienen dato: `n = 7 meses`.

## 4. Reglas de cálculo (validadas con Diego)

| Tipo de KPI | Regla YTD |
|---|---|
| KPIs de reseñas que **ya tienen YTD ponderado** (CSAT, Staff, Limpieza: `csatAcum`, `ytdSC`; Score Acumulado por canal) | **Reusar exactamente** la función existente. El número debe ser idéntico al que ya muestra el Dashboard |
| KPIs de reseñas por mes sin YTD previo (score mensual por canal) | Ponderado por número de reseñas del mes, con la misma lógica de `wavg()` |
| KPIs operativos (Hosts, Housekeeping %, Mantenimiento) | **Promedio simple** de los meses que tienen dato. Un mes vacío (`null`) **no** cuenta como cero |
| Conteos (limpiezas, checkout, cortesías, auditorías, hallazgos) | **Suma** |

Valores mensuales: siempre salen de `getKPIs(m)` o de los campos guardados. **No recalcular** nada que ya calcula el código.

## 5. Contenido de la pestaña

### 5.1 Tarjetas resumen YTD (fila superior)
Usar el estilo `mkCard()` existente: **CSAT YTD · Total limpiezas YTD · Total hallazgos YTD · % Tickets en tiempo YTD**.

### 5.2 Gráfica de evolución
- Selector de familia: **CSAT general · Canales · Hosts · Housekeeping · Mantenimiento**.
- Gráfica de líneas en `<canvas>` puro, siguiendo el estilo de `renderChart()`. Sin librerías.
- Línea punteada horizontal con el **promedio YTD** de cada serie.
- Escala Y automática por familia, porque minutos, %, scores /5 y horas no comparten eje. Si una familia mezcla unidades, mostrar una serie a la vez con chips para elegirla.
- Paleta de marca: teal `#168C95`, orange `#F9A058`, coral `#E26147`, navy `#38495D`.

### 5.3 Tabla maestra de KPIs
Filas = KPIs agrupados por familia. Columnas = Ene … mes seleccionado + **YTD** (resaltada).

| Familia | KPIs (fuente) |
|---|---|
| CSAT general | CSAT, Staff, Limpieza (`getKPIs`) |
| Canales | Booking, Airbnb, Expedia, Google, HostelWorld: score mensual /5 + n.º de reseñas |
| Hosts | Response Time (`hosts.response_time`), % Auditorías QA (`hosts.qa_completion`), auditorías por host (`aud_sofia`, `aud_andrea`, `aud_daniela`), total (`aud_total`) |
| Housekeeping | Limpiezas, Checkout, Cortesías (`hk.*`), % Hallazgos (`hk.hallazgos`) |
| Mantenimiento | Tiempo prom. ticket (`mant.tiempo`), % Tickets en tiempo (`mant.cumplimiento`) |

- Cada celda lleva un punto de semáforo con **las mismas metas y umbrales que ya usa el dashboard**: Response ≤ 60 min, Ticket ≤ 24 h, Cumplimiento ≥ 90 %, CSAT ≥ 4.5, etc. Reusar las funciones existentes; no inventar metas nuevas.
- Celda sin dato: `—`.
- Tabla con `overflow-x:auto` y primera columna fija (`position:sticky`) para celular.

### 5.4 Hallazgos HK por categoría (suma)
- Fuente: `m.qa.by_category`, formato `[["Categoría — respuesta", conteo], ...]`.
- Agrupar por **categoría**, que es el texto antes de `" — "`.
- Filas = categorías, ordenadas por total YTD de mayor a menor. Columnas = meses + **Total YTD**.
- Cada categoría se puede expandir (▸) para ver el detalle de respuestas sumadas.
- Fila final de **Total**.

### 5.5 Hallazgos HK por integrante (suma + % de hallazgos)
- Fuente: `m.qa.by_cleaner`, formato `[[nombre, total, detalles], ...]`. También `m.qa.audits_by_cleaner`.
- Columnas: Integrante · meses (hallazgos por mes) · **Total hallazgos YTD** · **Auditorías YTD** · **% Hallazgos YTD**.
- **% Hallazgos YTD = Total hallazgos YTD ÷ Auditorías YTD × 100**, con 1 decimal. Es la misma definición que el `% Hallazgos QA HK` general, que es `total_details / total_audits × 100`. Puede superar 100 % porque una auditoría puede tener varios hallazgos.
- Semáforo con la meta existente del dashboard: **< 20 % verde · 20–25 % amarillo · > 25 % rojo**.
- **Orden: de mejor a peor desempeño**, es decir, % Hallazgos YTD **ascendente** (menor % = mejor).
  - Empate: primero quien tenga más auditorías.
  - Integrantes con 0 auditorías YTD: `%` = `—` y van al final.
  - "Desconocido" va siempre al final.
- Debajo de la tabla, una nota: *"Ordenado de mejor a menor desempeño (% de hallazgos por auditoría, YTD)."*
- **Normalización de nombres** para sumar entre meses: minúsculas, sin acentos, espacios colapsados y recortados, y sin prefijos numéricos (`/^\d+[-.\s]+/`). Se muestra la grafía más frecuente. La misma normalización aplica a `audits_by_cleaner` para que hallazgos y auditorías crucen bien.
- Se puede expandir (▸) para ver el detalle por categoría de esa persona.

### 5.6 Exportar
Botón ⬇ PNG en cada sección, reutilizando `exportPNG(id)` y html2canvas, que ya existen.

### 5.7 Meses sin detalle de QA
Si un mes no tiene objeto `qa` (por ejemplo, se cargó antes de que existiera el detalle), la columna muestra `—` y aparece una nota al pie: *"Sin detalle de QA en: Ene, Feb…"*.

## 6. Integración

- Nuevo botón `.tab-btn` **"Evolución & YTD"**, ubicado antes de "Captura de datos". **No** lleva `data-editor-only`: debe verse en el visor.
- Nuevo panel `#tab-evolucion` y función `renderEvolucion()`.
- Llamar `renderEvolucion()` al activar la pestaña en `switchTab()` y al cambiar `#monthSelector` si la pestaña está activa.
- Solo lectura: no llama a `saveData()`.

## 7. Plan de desarrollo

| # | Fase | Criterio de terminado |
|---|---|---|
| 0 | Verificación | `git pull`; confirmar que el último commit de `index.html` incluye "Modo visor cifrado v1" |
| 1 | Capa de datos | Funciones puras `ytdRange()`, `ytdSimple()`, `sumBy()`, `normName()` con pruebas en consola |
| 2 | Tabla maestra + tarjetas | Valores mensuales iguales a los del Dashboard en cada mes |
| 3 | Hallazgos por categoría e integrante | Las sumas cuadran con lo mostrado en la pestaña Housekeeping de cada mes |
| 4 | Gráfica + PNG | Selector de familia funcional y línea YTD visible |
| 5 | Pruebas y regresión | Checklist de la sección 8 en verde |

## 8. Checklist de pruebas

- [ ] El CSAT, Staff y Limpieza YTD de la pestaña nueva **coinciden exactamente** con los del Dashboard.
- [ ] El promedio simple ignora meses vacíos: con meses de 50, — y 70, el YTD es 60, no 40.
- [ ] El total de limpiezas YTD es igual a la suma manual de `hk.limpiezas`.
- [ ] Por cada mes, el total de hallazgos por categoría es igual a `qa.total_details` de ese mes.
- [ ] "María López" y "maria lopez " se suman como una sola persona, tanto en hallazgos como en auditorías.
- [ ] % Hallazgos YTD por integrante = hallazgos ÷ auditorías × 100. Ejemplo: 6 hallazgos en 40 auditorías = 15.0 %, en verde.
- [ ] La tabla de integrantes queda ordenada de menor a mayor %; los de 0 auditorías y "Desconocido" van al final.
- [ ] Al cambiar el mes en el selector, se recalcula la ventana Ene → mes.
- [ ] Los semáforos usan las mismas metas que el Dashboard.
- [ ] Celular (375 px): las tablas hacen scroll interno y la página no tiene scroll horizontal.
- [ ] ⬇ PNG funciona en las 4 secciones.
- [ ] **Visor:** la pestaña aparece y funciona con datos descifrados (`?modo=visor` + `data.enc.json` de prueba con datos ficticios).
- [ ] **Regresión:** editor, PIN, Publicar snapshot y el resto de pestañas funcionan igual.

## 9. Fuera de alcance

Comparativo contra año anterior, metas editables, filtros por canal en tablas, % de hallazgos por integrante mes a mes (solo YTD) y exportar a Excel.
