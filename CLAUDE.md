# tulbox — contexto para Claude

## Qué es este repo

Colección de herramientas web estáticas. Cada herramienta es una carpeta con un `index.html` autocontenido que corre directo en el navegador, sin build step, sin bundler, sin servidor de desarrollo.

## Convenciones del proyecto

- **Sin npm, sin bundler.** Las dependencias se cargan desde CDN con integridad SRI.
- **React via CDN + Babel standalone.** Los archivos `.jsx` se transpilan en el navegador; no hay paso de compilación.
- **Sin comentarios de documentación genéricos.** Solo comentarios que expliquen el *por qué* de algo no obvio.
- **Sin frameworks CSS externos.** Estilos inline o `<style>` en el HTML.

## Herramientas actuales

### `beracode/`

Visor de reportes de vulnerabilidades de Veracode SCA.

- **`index.html`**: app React completa (componentes, lógica, estilos) en un único archivo.
- **`tweaks-panel.jsx`**: componente compartido de controles de UI (panel flotante con sliders, toggles, radios, color pickers). Se expone en `window` para que `index.html` lo consuma.

Formato de entrada: JSON producido por `veracode scan --source <carpeta> --type directory`.

El campo clave es `data.vulnerabilities.matches[]`, con subcampos `vulnerability`, `artifact`, `matchDetails`, `relatedVulnerabilities`.

Los escaneos se persisten en `localStorage` bajo la clave `beracode_scans`.

## Cómo probar cambios

Abrir el `index.html` directamente en el navegador (no hace falta servidor). Si hay cambios en `tweaks-panel.jsx`, recargar la página.

## Guía de estilo

- Variables CSS centralizadas en `:root` en `index.html`.
- Paleta semántica: `--critical`, `--high`, `--medium`, `--low` (y sus variantes `-bg`).
- Tipografía monoespaciada (`--font`) para código/datos; sans-serif (`--font-ui`) para UI.
