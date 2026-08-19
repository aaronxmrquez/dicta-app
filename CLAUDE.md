# Dicta — contexto para Claude Code

App nativa de macOS de dictado por voz. Vive en la barra de menús: mantienes una
tecla, hablas, la sueltas, y el texto se escribe donde esté el cursor — en
cualquier app. Requiere Apple Silicon y macOS 14+. Autor: Aaron Márquez.

Este archivo es el briefing operativo del proyecto: decisiones tomadas, sus
razones y las trampas que ya nos costaron tiempo. El `README.md` es la
documentación para quien usa o instala la app.

## Comandos

```sh
./build.sh release            # compila → build/Dicta.app
./build.sh release install    # además instala en /Applications
./build.sh release dmg        # genera build/Dicta.dmg distribuible
```

Modos de desarrollo del binario:

```sh
Dicta --render-previews <dir>            # renderiza las pantallas a PNG
Dicta --transcribe-file <audio> [es|en|auto]  # prueba el motor sin micrófono
Dicta --splash                           # fuerza el splash de bienvenida
```

**Publicar una versión** (secuencia completa, no saltarse pasos):

1. Subir `CFBundleShortVersionString` y `CFBundleVersion` en `Support/Info.plist`
2. `./build.sh release install` → `./build.sh release dmg`
3. Commit + push
4. `gh release create vX.Y.Z build/Dicta.dmg --title "…" --notes "…"`

El DMG se llama siempre `Dicta.dmg` (sin versión) para que este link quede
estable: `releases/latest/download/Dicta.dmg`.

## Arquitectura

```
HotkeyMonitor (CGEventTap) → AppState (idle → grabando → transcribiendo → insertando)
                                  ↓
              AudioRecorder → TranscriptionEngine → HUD (parciales en vivo)
                                  ↓
                             TextInserter (portapapeles + ⌘V + restauración)
```

`AppState` es el único coordinador; las vistas solo observan. Dos puntos de
extensión:

- **`TranscriptionEngine`** (`Sources/Dicta/Transcription/`): agregar un motor
  nuevo es implementar el protocolo, nada más. Hoy: `AppleSpeechEngine` y
  `WhisperEngine`.
- **`Brand.swift` + `Theme.swift`** (`Sources/Dicta/UI/`): design system. Todas
  las pantallas se arman con `BrandScreen`, `BrandCard`, `ChipButton`,
  `BrandToggle`, `PrimaryButton`, `BrandFooter`.

## Decisiones y su porqué

**whisper.cpp vendored, no WhisperKit.** SwiftPM está roto en esta máquina y no
puede bajar dependencias. `build.sh` clona y compila whisper.cpp v1.7.4 en
`vendor/` (gitignored) la primera vez, estático + Metal con
`GGML_METAL_EMBED_LIBRARY` para no depender del compilador de Metal de Xcode.

**`swiftc` directo, no `swift build`.** El SwiftPM de los Command Line Tools
crashea (símbolo de llbuild faltante) y el compilador 6.1.2 no soporta el SDK
26.2 instalado → se pasa `-sdk MacOSX15.5.sdk` explícito. Si algún día se
reparan los CLT, se puede volver a `Package.swift`.

**Fuera del App Store.** El sandbox que exige el Store impide Accessibility y
CGEventTap; sin eso la app no puede detectar la tecla ni escribir en otras apps.
No es una limitación evitable: superwhisper y Wispr Flow se distribuyen igual.

**Inserción por portapapeles + ⌘V sintético**, no escritura carácter por
carácter ni AX API. Es lo único que funciona igual en apps nativas, Electron y
navegadores. Se guarda y restaura el portapapeles del usuario, y los eventos se
marcan con `HotkeyMonitor.syntheticEventMarker` (`0xD1C7A`) para que el tap no
los interprete como teclas del usuario.

**Motor híbrido en `WhisperEngine`.** Whisper no entrega resultados parciales en
streaming; el motor de Apple sí. Mientras hablas se muestran los parciales de
Apple y al soltar la tecla se inserta el texto final de Whisper (~1 s). Si
Whisper falla, cae al texto de Apple.

**Certificado local estable, no firma ad-hoc.** Con ad-hoc la identidad de
código cambia en cada build y macOS invalida el permiso de Accesibilidad (el
switch se ve encendido pero no aplica). `Support/make_signing_cert.sh` crea
"Dicta Local Signing" una vez; `build.sh` lo usa si existe y cae a ad-hoc si no
(y en ese caso hace `tccutil reset` para que el estado sea honesto).

**Fuentes embebidas** vía `ATSApplicationFontsPath` en el Info.plist. Space Mono
e Inter viajan en `Support/Fonts/` (OFL) para verse igual en Macs ajenas;
`Theme.mono`/`Theme.sans` caen a las del sistema si faltaran.

**HUD como `NSPanel` no activante.** Nunca debe robar el foco del campo donde el
usuario está escribiendo.

**Limpieza del texto con IA: descartada.** Decisión explícita de Aaron —
Whisper ya transcribe limpio y no valía el costo ni sacar el audio de la Mac. El
punto de enganche natural sería entre `finish()` e `insert()` en `AppState`.

## Convenciones

- **Idioma**: comentarios y mensajes de commit en español; **la UI de la app va
  en inglés**. Los commits terminan con `Co-Authored-By: Claude …`.
- **Tipografía**: Space Mono para titulares, labels y chips; Inter para texto
  corrido, descripciones y botones principales.
- **Nunca hardcodear** un color, una fuente o un componente: todo sale de
  `Theme.swift` (`background`, `accent` verde `#38D610`, `dictaGray` `#5D5B5B`
  para el "DICTA." de los títulos, `footerGray` `#545454` para footer y versión)
  y de `Brand.swift`.
- **Medidas de Figma**: el canvas del diseño es de 884 px y la ventana real de
  560 → multiplicar cada valor por **0.6335**.
- **Ventanas**: todas 560×792, header y footer fijos, scroll solo donde el
  contenido no cabe (`scrollBounceBehavior(.basedOnSize)`). Solo se abre una
  ventana a la vez (`closeOtherWindows` en `DictaApp.swift`).
- **Verificar antes de instalar**: `--render-previews` y mirar los PNG. Para
  cambios de geometría de ventana, medir además la ventana viva (CGWindowList),
  no confiar solo en el render.

## Trampas conocidas

- **`NSHostingView` redimensiona la ventana** al asignarse como `contentView` y
  deshace cualquier `setFrame` previo → `sizingOptions = []` y llamar
  `BrandWindow.applyChrome(to:)` **después** de asignar el contenido. Está así en
  las 4 ventanas y en `PreviewRenderer`.
- **El titlebar inserta ~28 pt de safe area** que empujan todo el contenido →
  `.ignoresSafeArea()` en todas las pantallas (el diseño es full-bleed e incluye
  la zona del semáforo).
- **Los renders de verificación deben usar ventanas con titlebar real**; con
  ventanas borderless los bugs de layout quedan invisibles.
- **`ImageRenderer` no dibuja el contenido de un `ScrollView`** → esas pantallas
  se renderizan con `renderInWindow` (`cacheDisplay`).
- **Figma por MCP**: el archivo *Playground* de Aaron no da acceso (hay que
  pedirle exports PNG a `~/Downloads`); el de *Diners Design System — Benchmark*
  sí funciona (`fileKey RzlNTuBfcjk1OKHLugZMkO`, frames `dicta - *`).
- El modelo Whisper vive en `~/Library/Application Support/Dicta/models/`
  (`ggml-large-v3-turbo-q5_0.bin`, 574 MB) y **no** está en el repo; lo descarga
  `ModelManager` desde Settings.

## Pendientes

- **Sin notarizar**: quien la instala pasa una vez por "Abrir de todos modos".
  Se resuelve con el Apple Developer Program ($99/año) + `notarytool`.
- **Solo arm64**: no hay build universal para Intel.
- Ideas futuras: vocabulario personalizado, auto-updates con Sparkle, hotkey
  configurable por app.
