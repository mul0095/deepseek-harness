# Local Whisper Dictation in the Electron Desktop Application

## Goal

The packaged Windows Electron application provides microphone dictation through a local Whisper process, while preserving the existing conversation UI and keeping captured audio out of OpenAI and browser speech-recognition services.

## Scope

The first vertical slice targets the existing Electron desktop composition and its shared conversation composer. It adds a local transcription route, a managed `whisper.cpp` process, desktop-only client selection, Windows icon packaging, bounded audio intake, cancellation, and focused tests. The browser Web Speech implementation remains available for non-desktop web sessions until a separate deployment provides the local provider.

## Architecture

The browser renderer captures one short recording with `MediaRecorder` and posts the audio to an authenticated exact Connection Fetch route. The Desktop Host owns the route and passes a temporary WAV representation to a managed `whisper.cpp` CLI process. The response contains only the transcript; the temporary audio is removed during success, failure, cancellation, and disposal.

The desktop page receives an explicit transport capability marker so the shared `InputBar` selects local dictation only when the Desktop Host has the provider. The renderer does not receive Node or Electron APIs directly. Electron continues to carry requests over `dsh-app://` and the existing framed byte-pipe transport.

## Local ASR provider

The provider uses a configured executable and multilingual GGML model. Development resolves explicit environment overrides; packaged Windows builds resolve the executable and model from application resources. The provider validates request size, duration metadata, executable/model availability, process exit status, output format, and cancellation. It never sends network requests and does not retain audio after a request settles.

The initial implementation uses the `whisper.cpp` CLI contract with a multilingual `base` model. Model and executable versions are pinned by the desktop preparation step and their checksums are recorded before packaging. The provider keeps the executable path separate from the model path so future CPU/GPU binaries can be selected without changing the client protocol.

## Client behavior

The microphone control has explicit recording and transcription states. Starting requires an editable composer and a supported `MediaRecorder` MIME type. Stopping sends the completed Blob, inserts the returned text at the editor caret, and clears transient state. Empty audio, unsupported capture, denied permission, HTTP failures, cancellation, and provider failures become locale-owned toast messages. A desktop session does not fall back to browser speech recognition after local ASR is selected.

## Security and resource limits

The route is available only through the authenticated Connection transport. It accepts only `POST` audio requests, applies a byte limit before transcription, does not write predictable shared files, and uses a private temporary directory with exclusive filenames. Child processes receive a scrubbed environment, inherit the request cancellation signal, and are awaited during teardown. The route returns `Cache-Control: no-store` and never logs audio bytes or transcript content.

## Packaging

The supplied `deepseek_256.ico` is copied into the desktop application assets and used by electron-builder for the Windows target. The packaged resource inventory includes the selected Whisper executable, model, and metadata. Unsigned Windows directory packaging is used for local verification; signed release packaging retains the existing release requirements.

## Verification

Focused tests cover request validation, temporary-file cleanup, process success/failure/cancellation, route registration and disposal, client recording transitions, desktop capability selection, icon configuration, and resource inventory. The implementation also runs the existing client tests, desktop tests, typecheck, focused build, `git diff --check`, and the repository’s relevant documentation and i18n gates. A real packaged Windows smoke test must still verify microphone capture and Ukrainian transcription on the user’s machine.
