# Trace Recording for Selenium — Design Doc

## Overview

Add a `--pipe` mode to Selenium Manager that acts as an HTTP proxy between
the binding and the WebDriver, recording every command + screenshot into a
trace.zip compatible with Playwright / Vibium record player.

## Architecture

```
Binding (sin cambios en la API pública)
  │
  │ HTTP REST JSON (mismo formato que siempre)
  ▼
selenium-manager --pipe --record trace.zip --browser chrome
  │
  ├─► HTTP proxy → chromedriver/geckodriver/msedgedriver
  │
  └─► BiDi WebSocket → takeScreenshot (después de cada acción)
  │
  └─► trace.zip (al cerrar la sesión)
```

## Pipe mode

Nuevo flag `--pipe` en Selenium Manager. Cuando está presente:

1. SM descubre browser y driver (como hoy)
2. Arranca el WebDriver (chromedriver, etc.)
3. Conecta vía BiDi WebSocket al WebDriver
4. Arranca un servidor HTTP interno en puerto libre
5. Imprime JSON a stdout con `{"proxy_url": "http://localhost:PORT"}`
6. El binding lee la URL y apunta sus peticiones HTTP ahí
7. SM reenvía cada petición REST al WebDriver y devuelve la respuesta
8. Tras comandos de acción (click, navigate, etc.), envía BiDi takeScreenshot
9. Al recibir DELETE /session/{id}, empaqueta trace.zip y termina

## Recording

- **Qué capturar**: solo comandos de acción (click, navigate, findElement con
  interacción, type, keys, back, forward, etc.). Consultas puras (title,
  cookies, url) sin screenshot.
- **BiDi para screenshots**: `browserContext.takeScreenshot` vía el WebSocket
  BiDi. Si falla o no hay soporte, se continúa sin screenshot.
- **Video**: Firefox 154+ vía screencast BiDi. Chrome no lo soporta aún.
  `--record --video` falla con error explícito si no hay soporte.
- **Buffer**: screenshots en memoria hasta el final. Para sesiones largas,
  opción `--record-chunk-size` (post-MVP).

## Trace.zip format

Compatible con Playwright trace viewer / player.vibium.dev:

```
trace.zip
├── trace.json       # metadatos (comandos, timestamps, grupos)
├── screenshots/     # capturas PNG/JPEG
│   ├── s0.png
│   └── ...
└── trace.webm       # video (solo Firefox, opcional)
```

`trace.json` schema:

```json
{
  "type": "trace",
  "version": 2,
  "pages": [{
    "pageId": "...",
    "title": "...",
    "url": "...",
    "commands": [{
      "type": "click",
      "action": "...",
      "timestamp": 1234567890,
      "duration": 123,
      "screenshotIndex": 0,
      "url": "...",
      "response": {...}
    }]
  }],
  "screenshots": ["screenshots/s0.png", ...]
}
```

## Binding integration

Cada binding añade un setter en las Options:

```java
ChromeOptions options = new ChromeOptions();
options.setRecord("trace.zip");
WebDriver driver = new ChromeDriver(options);
// → binding invoca SM con --pipe --record trace.zip --browser chrome
// → lee proxy_url de stdout
// → apunta el driver HTTP a proxy_url
driver.quit();
// → SM empaqueta trace.zip
```

Cambio por binding: ~100 líneas. El 95% de la lógica está en Rust.

## Edge cases

| Caso | Comportamiento |
|------|---------------|
| BiDi no disponible | Recording sin screenshots (solo comandos) |
| Screenshot falla | Se registra el error, no se detiene la sesión |
| Video no disponible | `--record` sin problema; `--record --video` falla |
| Sesiones concurrentes | Cada una su propio proceso SM, sin estado compartido |
| trace.zip existe | Se sobrescribe |
| WebDriver timeout | Proxy devuelve 502 |
| Conexión BiDi caída | Recording continúa sin screenshots |

## Testing

1. **Unit tests (Rust)**: Recorder module, Archiver, protocol parsing
2. **Integration (Rust)**: SM --pipe --record con chromedriver real, verificar
   trace.zip generado
3. **Integration (bindings)**: Java test con ChromeOptions.setRecord,
   verificar que el ZIP se genera al hacer quit()
4. **Compatibilidad**: Verificar que el trace.zip generado se abre en
   player.vibium.dev sin errores

## Roadmap

1. **MVP**: --pipe mode solo para Chrome, screenshots PNG, sin video
2. **Firefox**: Añadir soporte BiDi + video screencast
3. **Edge**: Añadir soporte
4. **Mejoras**: --record-chunk-size, grupos, chunks, HTML snapshots
5. **HTTP proxy standalone**: Que SM pueda actuar como proxy sin recording
   (útil para debug)