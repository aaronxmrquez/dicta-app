
# Dicta

Voice dictation for macOS. Lives in the menu bar: hold a key, talk,
let go — and the text is typed wherever your cursor is, in any app
(Slack, email, browser, editors).

[DOWNLOAD HERE](https://github.com/aaronxmrquez/dicta-app/releases/latest/download/Dicta.dmg)

Requirements: Mac with Apple Silicon (M1+) and macOS 14+. The app is
not notarized, so the first launch asks you to click "Open anyway"
in System Settings → Privacy & Security.


## Use

- **Hold to talk:** (right ⌘ by default; switchable to right ⌥ or fn): hold, speak, release.
  Esc cancels. Regular shortcuts (⌘C…) keep working while you dictate.
- **Toggle mode:** ⌥ Space starts and stops (selectable in Settings).
- **Language:** Auto (automatic es/en detection with Whisper), Spanish, or English.
- **History:** the last 100 dictations, click to copy. Always local.
- The app UI is in English. Branding: charcoal + Space Mono + Inter with a green accent,
  plus a welcome splash on first install.

## Demo


https://github.com/user-attachments/assets/7e0714d4-d418-4a83-90a2-d2c238a2155b



## Engines
- **Whisper (default):** whisper.cpp + Metal, quantized large-v3-turbo model (574 MB, downloaded
  once from Settings). Highest accuracy in Spanish and English, 100% local. Hybrid:
  live partials from Apple's engine, final text from Whisper (~1s).
- **Apple:** SFSpeechRecognizer. No downloads, lighter.



## Build
```bash
./build.sh release            # compiles and assembles build/Dicta.app
./build.sh release install    # also installs to /Applications
./build.sh release dmg        # generates a distributable build/Dicta.dmg
```
Notes for this machine: compiles with `swiftc` directly against the macOS 15.5 SDK
(SwiftPM from the CLT is broken — see the comment in `build.sh`). whisper.cpp is cloned
and built into `vendor/` on the first run. Signing uses the local "Dicta Local Signing"
certificate if present (`Support/make_signing_cert.sh`); without it, it falls back to ad-hoc.

Brand assets live in `Support/`: logo, app icon and menu bar icon (`make_icon.swift` regenerates them), 
splash pattern (SVG from Figma), and embedded Space Mono / Inter fonts (OFL).



## Architecture

`HotkeyMonitor` (CGEventTap) → `AppState` (idle → recording → transcribing → inserting) → `AudioRecorder` 
(AVAudioEngine) feeds the active engine → floating HUD (non-activating `NSPanel`) with live partials → `TextInserter` (clipboard + synthetic ⌘V + restore).

Engines implement the `TranscriptionEngine` protocol (Sources/Dicta/Transcription/): 
adding a new one doesn't touch the rest.

Development modes: `--render-previews <dir>` (renders the views to PNG), `--transcribe-file <audio> [es|en|auto]` 
(tests the engine without a microphone), `--splash` (shows the welcome splash).
