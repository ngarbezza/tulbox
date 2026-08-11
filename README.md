# tulbox

Caja de herramientas web: utilidades pequeñas, autocontenidas y sin dependencias de build.

Cada herramienta vive en su propia carpeta y corre directamente desde el navegador (sin `npm install`, sin bundler, sin servidor).

[`index.html`](index.html) en la raíz es una landing page con una card por herramienta.

## Herramientas

### [`beracode/`](beracode/)

Visor de reportes de vulnerabilidades generados por [Veracode SCA](https://docs.veracode.com/r/veracode-sca).

**Cómo usarlo:**

1. Correr un escaneo con Veracode CLI:
   ```sh
   veracode scan --source <carpeta> --type directory --format json > result.json
   ```
2. Abrir `beracode/index.html` en el navegador.
3. Arrastrar el `result.json` al área de drop (o hacer clic para buscar el archivo).

**Funcionalidades:**
- Tabla de vulnerabilidades filtrable por severidad, paquete y texto libre
- Detalle de cada finding: CVSS, EPSS, CWE, versión con fix
- Soporte para múltiples escaneos simultáneos (persiste en `localStorage`)
- Tema claro/oscuro

### [`moving-motivators/`](moving-motivators/)

Actividad de [Moving Motivators](https://management30.com/practice/moving-motivators/) de Management 3.0: ordenar 10 motivadores según preferencia personal.

**Cómo usarlo:**

1. Abrir `moving-motivators/index.html` en el navegador.
2. Arrastrar las tarjetas para ordenarlas según tu preferencia.
3. Posicionarte sobre una tarjeta muestra su descripción.

**Funcionalidades:**
- Reordenamiento por drag-and-drop
- Traducciones a 7 idiomas (en, es, de, it, pt, sv, bn)

## Estructura

```
tulbox/
├── beracode/
│   ├── index.html       # app completa (React via CDN + Babel standalone)
│   └── tweaks-panel.jsx # componente compartido de controles de UI
└── moving-motivators/
    └── index.html       # app completa (Vue via CDN, sin build step)
```

## Agregar una herramienta nueva

1. Crear una carpeta con el nombre de la herramienta.
2. Poner un `index.html` autocontenido dentro.
3. Documentarla en este README.
4. Agregar su card en [`index.html`](index.html).
