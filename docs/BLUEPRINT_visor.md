# BLUEPRINT — Modo Visor Cifrado · Dashboard TRIVER KPIs

**Versión:** 1.0 · **Fecha:** 26/11/2026 · **Responsable:** Diego Medina (Head of Operations)
**Repo:** `trivercoliving-rgb/Triver-Coliving-Dashboard-KPI-s-Ops` · **Hosting:** GitHub Pages
**Archivo base:** `index.html` versión 54 (4,124 líneas · git blob `d50e829c8e49ff23cec22b07c47ae037d456abd6`)
**Tiempo objetivo:** ≤ 1.5 h de desarrollo

---

## 1. Problema

El dashboard guarda todos sus datos en el `localStorage` del navegador de Diego (clave `triver_kpi_v3`). Publicarlo en GitHub Pages no comparte los datos: cualquier otra persona ve el `SEED` vacío. Además, el PIN actual (`DEFAULT_PIN = '2025'`) está escrito en el código y es visible para cualquiera en un repo público.

## 2. Objetivo

Que integrantes autorizados de TRIVER y la dirección puedan **consultar** el dashboard completo desde un link, protegido con una **contraseña compartida**, mientras Diego sigue siendo el único que captura datos.

## 3. Decisiones de negocio (validadas con Diego)

| Tema | Decisión |
|---|---|
| Quién captura datos | Solo Diego, en su Mac, como hoy |
| Quién consulta | Equipo y dirección autorizados de TRIVER |
| Control de acceso | Una contraseña compartida. Diego la define, la comunica y la rota cuando alguien sale |
| Qué ve el visor | Las tres pestañas: **Dashboard, Hosts y Housekeeping**, incluido el desglose individual |
| Qué NO ve el visor | Pestaña "Captura de datos", "+ Nuevo mes", Exportar, Importar |
| Frescura de datos | Snapshot manual: el visor muestra el último corte publicado por Diego |

## 4. Arquitectura

**Un solo `index.html` con dos modos.** No se crea un archivo separado para evitar divergencia de versiones.

```
MODO_VISOR = (location.protocol !== 'file:') || URLSearchParams(location.search).get('modo') === 'visor'
```

| | Modo editor (`file://` en la Mac de Diego) | Modo visor (`https://` GitHub Pages) |
|---|---|---|
| Fuente de datos | `localStorage` (sin cambios) | `data.enc.json` descifrado **solo en memoria** |
| Pantalla de bloqueo | PIN actual (sin cambios) | Contraseña compartida |
| Guardado | `saveData()` normal | `saveData()` = no-op |
| Botón nuevo | **"🔐 Publicar snapshot"** | — |
| Pestaña Captura | Visible | Oculta |
| Exportar PNG | Sí | Sí |

Archivos en la raíz del repo:

```
index.html        ← editor + visor
data.enc.json     ← snapshot cifrado (lo genera Diego)
docs/BLUEPRINT_visor.md
docs/RUTINA_publicacion.md
```

## 5. Especificación de cifrado

Solo Web Crypto API nativa (`crypto.subtle`), sin librerías.

- **Derivación de clave:** PBKDF2 · SHA-256 · **600,000 iteraciones** · salt aleatorio de 16 bytes.
- **Cifrado:** AES-GCM 256 bits · IV aleatorio de 12 bytes por publicación.
- **Texto plano cifrado:** `JSON.stringify({ publishedAt: <ISO string>, data: <objeto de triver_kpi_v3> })`. La fecha va **dentro** del cifrado.
- **Formato de `data.enc.json`:**

```json
{
  "v": 1,
  "alg": "AES-GCM-256",
  "kdf": "PBKDF2-SHA256",
  "iter": 600000,
  "salt": "<base64>",
  "iv": "<base64>",
  "ct": "<base64>"
}
```

- Base64 con un helper propio que procese en bloques (`String.fromCharCode` por chunks), porque el JSON puede pesar varios cientos de KB.
- Si `crypto.subtle` no existe, mostrar el error: *"Este navegador no soporta el cifrado necesario"*.

## 6. Cambios en el código (mapa sobre la versión 54)

| Línea aprox. | Elemento actual | Cambio |
|---|---|---|
| ~1124 | `const STORAGE_KEY` | Agregar arriba la constante `MODO_VISOR` y la variable `VISOR_DATA = null` |
| ~1155 | `loadData()` | Si `MODO_VISOR`, devolver `VISOR_DATA` (nunca leer `localStorage`) |
| ~1158 | `saveData()` | Si `MODO_VISOR`, no hacer nada (`return`) |
| ~1309 | `render()` llama a `loadData()` | Sin cambio; funciona gracias al punto anterior |
| ~556–561 | Botones del encabezado | En visor: ocultar "+ Nuevo mes", Exportar, Importar. En editor: agregar "🔐 Publicar snapshot" junto a Exportar |
| ~571–574 | `.tab-btn` | En visor: ocultar el botón de "Captura de datos" |
| ~538–543 | Lock screen | En visor: etiqueta "Ingresa la contraseña del equipo", quitar `maxlength="6"`, quitar hint "PIN de 4–6 dígitos" |
| ~4085 | `checkPIN()` | En visor: llamar a una nueva función `desbloquearVisor(pwd)` |
| ~4102 | `lockDashboard()` | En visor: además poner `VISOR_DATA = null` y `appData = null` (borrar datos de memoria) |
| ~4031 | `exportBackup()` | Sin cambio. Agregar al lado una nueva función `publicarSnapshot()` |

### 6.1 `publicarSnapshot()` (solo editor)

1. Abre un modal con estilo de marca, con dos campos: contraseña y confirmación.
2. Valida que coincidan y que tengan **mínimo 12 caracteres**.
3. Lee `localStorage[STORAGE_KEY]`, cifra según la sección 5 y descarga **`data.enc.json`** (nombre fijo, sin timestamp).
4. Toast: *"✓ Snapshot cifrado listo. Súbelo al repo como data.enc.json"*.
5. La contraseña **nunca** se guarda en ningún lado.

### 6.2 `desbloquearVisor(pwd)` (solo visor)

1. `fetch('data.enc.json', { cache: 'no-store' })`.
   - Si falla con 404: *"Aún no hay datos publicados."*
   - Si falla por `file://` (prueba local): mostrar un `<input type="file">` para cargar el `.enc.json` manualmente.
2. Descifrar con la contraseña.
   - Si falla AES-GCM: *"Contraseña incorrecta."* No dar más detalle.
3. Si funciona: `VISOR_DATA = payload.data`, ocultar lock screen, ejecutar `render()` y mostrar el sello.

### 6.3 Sello de vigencia

En el encabezado del visor, con estilo de pastilla teal `#168C95`: **"Datos al: 26 nov 2026, 14:30"**, a partir de `publishedAt`.

## 7. Reglas que NO se tocan

- Lógica de procesadores (Booking, Airbnb, Expedia/Apify, Google, HostelWorld, QA Auditorías, Housekeeping XLSX).
- Reglas de conteo de QA y cálculo de Triver CSAT / Score Acumulado.
- Paleta de marca, logos base64, `html2canvas` 1.4.1 de cdnjs.
- Workflow de GitHub Actions existente.
- Comportamiento completo del modo editor.

## 8. Plan de desarrollo

| # | Fase | Criterio de terminado |
|---|---|---|
| 0 | Verificación | `git hash-object index.html` = `d50e829c…`. Si no coincide, **detenerse y avisar** |
| 1 | Detección de modo | `MODO_VISOR` funciona con protocolo y con `?modo=visor` |
| 2 | Cifrado | Ida y vuelta: cifrar → descifrar devuelve un JSON idéntico |
| 3 | Visor | Lock screen con contraseña, render completo de las 3 pestañas, Captura oculta, sello visible |
| 4 | Pruebas | Checklist de la sección 9 en verde |

## 9. Checklist de pruebas

- [ ] Editor (`file://`): todo funciona igual que la versión 54, incluido el PIN.
- [ ] Editor: "Publicar snapshot" rechaza contraseñas que no coinciden o con menos de 12 caracteres.
- [ ] Visor (`python3 -m http.server` + `?modo=visor`): contraseña correcta → dashboard completo.
- [ ] Visor: contraseña incorrecta → mensaje genérico, sin datos visibles.
- [ ] Visor: sin `data.enc.json` → "Aún no hay datos publicados".
- [ ] Visor: no aparecen Captura, "+ Nuevo mes", Exportar ni Importar.
- [ ] Visor: después de usarlo, `localStorage` **no** contiene `triver_kpi_v3`.
- [ ] Visor: 🔒 Bloquear y el auto-lock de 30 min borran los datos de memoria y vuelven a pedir contraseña.
- [ ] Visor: ⬇ PNG sigue funcionando.
- [ ] Visor: se ve bien en celular (dirección lo abrirá desde el teléfono).
- [ ] `data.enc.json` abierto en texto no revela nombres, fechas ni scores.

## 10. Fuera de alcance (MVP)

Accesos individuales por correo, base de datos en tiempo real, edición desde el visor, histórico de snapshots y analítica de quién entra.
