# Beracode: soporte para reportes de Static Analysis (SAST)

## Contexto

Beracode hoy solo entiende reportes de Veracode SCA (`data.vulnerabilities.matches[]`). Se agrega soporte para reportes de Veracode Static Analysis (`findings[]` con `scan_id` en la raíz), que tienen un modelo de datos y una vista de detalle completamente distintos (flaws en código propio, no dependencias vulnerables).

## Detección de formato

Al parsear un JSON en el `DropZone`:
- `data.vulnerabilities.matches` presente → **SCA**.
- `scan_id` (string) + `modules` (array) en la raíz → **SAST**. Nota: `findings` puede estar ausente cuando el scan no tiene hallazgos (visto en un caso real), así que la detección no puede depender de esa clave.
- Ninguno de los dos → error "No se pudo reconocer el formato" (mismo UX de error ya existente).

## Modelo de datos

Cada scan persistido en `localStorage` (`beracode_scans`) gana `kind: 'sca' | 'sast'`.

Finding SAST normalizado:

```js
{
  id, severity,            // 'Critical'|'High'|'Medium'|'Low', mapeado desde severidad numérica 0-5
  issueType, cwe, exploitLevel,
  description,             // display_text crudo (HTML), se sanitiza al renderizar
  file, line, functionName, qualifiedFunctionName, functionPrototype,
  stackFrames: [{ file, line, functionName }],
  detailsLink,
}
```

Mapeo de severidad: `5→Critical, 4→High, 3→Medium, 2 y 1→Low, 0→Low`. Se reutiliza la paleta de colores existente (`--critical/--high/--medium/--low`), sin agregar un 5to nivel.

Para SAST: `name` = `modules[0]`, `scannedAt` = momento de carga (el JSON no trae timestamp).

## UI

**Home / `ScanCard`**: lista unificada (sin tabs ni secciones separadas). Badge de texto "SCA"/"SAST" junto al nombre. El pie de la card muestra "N packages affected" (SCA) o "N issue types" (SAST) según corresponda.

**`SastReportView`** (nuevo componente, paralelo a `ReportView`, no se modifica el existente):
- Sidebar: buscador + filtro por Severity + filtro por **Issue Type** (reemplaza a Package).
- Summary strip: igual a SCA (5 tarjetas clicleables).
- Tabla: `Severity | Issue | CWE | Location (archivo:línea + función) | Exploit`. Sort solo por Severity.
- Drawer de detalle (`SastDetailDrawer`): descripción (display_text sanitizado, whitelist de tags `span`/`a`/`br`, un solo bloque), ubicación (archivo/línea/función calificada/prototipo), data flow (stack frames aplanados, sin `VarNames`) si existen, referencia a `flaw_details_link`.

**`App`**: dispatch a `ReportView` o `SastReportView` según `scan.kind`.

## Seeds de demo

Se agregan 1-2 scans `[DEMO]` ficticios de tipo SAST (nombres y hallazgos inventados, cubriendo variedad de severidades y tipos de issue, con y sin CWE) junto a los 3 seeds SCA existentes.

## Fuera de alcance

- No se toca `tweaks-panel.jsx`.
- No se separan visualmente descripción/remediación/referencias dentro del `display_text` (se muestra tal cual, sanitizado).
- No se agrega un 5to nivel de severidad "Info".
