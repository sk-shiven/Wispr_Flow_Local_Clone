# LocalFlow for macOS: Product Requirements & Technical Design Document

## 1. High-Level Architecture (macOS Restructure)

To replicate Wispr Flow’s lightning-fast experience natively on a Mac, the architecture must pivot to take full advantage of macOS-specific APIs and Apple Silicon (M-Series chips).

Apple's **Unified Memory Architecture (UMA)** and **Metal Performance Shaders (MPS)** allow us to run heavy STT and LLM models locally with near-zero latency, often outperforming network-bound cloud calls.

*   **macOS / Client Layer (Menubar App):** Instead of a heavy UI, the app will live in the macOS Menu Bar (using Python's `rumps` or a lightweight Swift UI). It requests **macOS Accessibility Permissions** and **Microphone Permissions** on first launch.
*   **Audio Capture & Hardware VAD:** Captures CoreAudio streams via `sounddevice` into a ring buffer. Voice Activity Detection (VAD) monitors the mic and chunks audio efficiently to prevent memory bloat.
*   **Mac-Optimized STT Engine (Apple MLX):** We utilize **`whisper-mlx`** or **`parakeet-mlx`** (NVIDIA Parakeet) for maximum GPU utilization on Apple Silicon, running STT with zero-copy memory overhead. Crucially, the STT engine processes **streaming chunks** to provide sub-second perceived latency via partial transcripts.
*   **Local LLM Layer (Ollama via Metal):** The raw transcript is passed to a locally running instance of Ollama. Ollama natively utilizes macOS Metal, allowing ultra-fast inference. It uses **streaming** (`stream: true`) to paste text incrementally.
*   **macOS Context & Injection Layer:**
    *   *Deep Context Grabber:* Uses macOS `AppKit` (`NSWorkspace`) and AppleScript/Accessibility APIs to gather: the frontmost application name, active URL (for browsers), the last 200 characters from the clipboard, and any selected text.
    *   *Robust Injection:* Copies the polished text to `NSPasteboard`. It first tries injecting via `Quartz.CGEventCreateKeyboardEvent` (`Cmd+V`). If the app rejects synthetic events (common in Electron or SwiftUI apps), it falls back to an AppleScript injection (`keystroke "v" using command down`).

### Threading & Process Model
To guarantee smooth performance and low latency, the system utilizes asynchronous concurrency:
```
Main Thread:  rumps UI, PyObjC (Quartz) global hotkey listener
Audio Thread: sounddevice callback → ring buffer
STT Worker:   consumes chunks from ring buffer, emits partials via streaming
LLM Worker:   asyncio + aiohttp consumes final/partial STT, streams output to injector
Injector:     places on clipboard + fires robust CGEvent/AppleScript fallback
```

---

## 2. Product Requirements Document (PRD)

**Product Name:** LocalFlow (macOS Edition)
**Objective:** An open-source, privacy-first, fully offline macOS menu-bar application that provides intelligent, context-aware dictation and text-formatting using local AI models.

### Target Audience
*   Mac-based developers, writers, and professionals handling sensitive data (Proprietary code, HIPAA/GDPR data).
*   Power users utilizing Apple Silicon hardware who want zero-latency, offline dictation.

### Core Features & Prioritization

| Feature | Description | Priority |
| :--- | :--- | :--- |
| **Robust Global Hotkey** | Push-to-talk via `Right Command` or `Fn` (avoids Spotlight `Cmd+Space` conflicts). Mapped via `NSEvent.addGlobalMonitorForEvents` instead of `pynput`. | P0 |
| **Streaming STT** | Transcribe speech offline using `whisper-mlx` or `parakeet-mlx` with streaming chunks for sub-second partials. | P0 |
| **Streaming LLM Edit** | Pass raw text to Ollama. Stream output to inject text sentence-by-sentence. | P0 |
| **Robust Text Injection** | Synthesize `Cmd+V` via `Quartz.CGEvent`. Fallback to AppleScript for stubborn apps. | P0 |
| **Deep Context Awareness** | Detect active App, active URL, clipboard history, and selected text to adjust LLM prompt tone. | P1 |
| **"Skip LLM" Mode** | Modifier hotkey (e.g., `Option+Fn`) to output raw transcript bypassing the LLM. | P1 |
| **Menubar UI** | A lightweight native-feeling status bar icon. Includes error states (Ollama missing, Mic denied). | P1 |

### Non-Functional Requirements
*   **Privacy:** 100% offline. Zero network calls outside of `localhost` (Ollama).
*   **Latency Budget:** Time from audio-stop to first token insertion must be **< 1.5 seconds**. Mitigations include concurrent STT/LLM processing and skipping LLM for short, clean transcripts.
*   **Memory Budget & Model Tiers:**
    *   **8 GB Unified Memory:** `base.en` STT + `qwen2.5:1.5b` LLM (Fallback profile)
    *   **16 GB Unified Memory:** `small.en` STT + `qwen2.5:3b` LLM (Recommended profile)
    *   **32 GB+ Unified Memory:** `large-v3` STT + `llama3.1:8b` LLM (High-quality profile)
*   **Error States:** App must gracefully handle: missing Ollama daemon, unloaded models, empty STT transcripts, and denied macOS permissions (linking to System Settings).

---

## 3. Technical Design Document (TDD) for macOS

### Architecture Stack
*   **Orchestrator:** Python 3.10+ with `asyncio`.
*   **UI / Menubar:** `rumps`.
*   **Audio Engine:** `sounddevice` + ring buffer + `silero-vad`.
*   **STT Engine:** `whisper-mlx` or `parakeet-mlx` (for streaming capabilities and better performance than `mlx-whisper`).
*   **LLM Engine:** `Ollama` running as a background service.
*   **macOS Automation:**
    *   Context: `pyobjc` (`AppKit.NSWorkspace`), AppleScript.
    *   Hotkey: `PyObjC` (`NSEvent.addGlobalMonitorForEvents`).
    *   Injection: `Quartz.CGEventCreateKeyboardEvent`, AppleScript fallback, `NSPasteboard`.

### Recommended Ollama Models (Ranked)

1.  **`qwen2.5:3b`** (~2.0 GB): Best instruction-following at this size. Excellent filler removal. ~250ms latency to first token (M2 Pro).
2.  **`qwen2.5:1.5b`** (~1.0 GB): Faster fallback for 8GB Macs (~150ms latency).
3.  **`llama3.2:3b`** (~2.0 GB): Solid baseline, very stable instruction adherence (~280ms latency).
4.  **`qwen2.5-coder:3b`**: For code contexts (Xcode, VS Code).

### Default Prompt Template
```text
SYSTEM: You are a dictation cleanup engine. Rewrite the transcript
to remove filler words ("um", "uh", "like"), fix grammar, and apply
formatting appropriate for {app_context}. Output ONLY the cleaned
text with no preamble, no quotes, no explanation.

USER: {raw_transcript}
```

### Sequential Data Flow (Mac Context)
1.  **Idle State:** App lives in Menubar. Listeners active via `NSEvent`.
2.  **Trigger:** User presses and holds `Right Command`.
3.  **Context Grab:** Python queries `AppKit` and AppleScript for deep context (app, URL, clipboard).
4.  **Capture & Stream:** Audio buffers into a ring buffer. The STT worker consumes chunks and generates partial transcripts.
5.  **Release:** User releases hotkey.
6.  **LLM Phase:** Final transcript + Deep Context passed to `localhost:11434` via `aiohttp` with `stream: true`.
7.  **Output & Injection Phase:**
    *   As tokens stream back, the Injector places complete words/sentences on `NSPasteboard`.
    *   Fires robust `Quartz.CGEvent` (or AppleScript fallback) to paste into the active window instantly.

---

## 4. Implementation Roadmap (macOS Action Plan)

### Phase 1: Environment & Core STT
1. Setup Python `venv` on macOS.
2. Install Mac-specific ML packages: `pip install whisper-mlx sounddevice numpy`.
3. Write the asynchronous audio recording ring buffer and implement streaming partial transcription.

### Phase 2: Local LLM Integration
1. Install Ollama for Mac (native app via `ollama.com`).
2. Pull recommended models: `ollama pull qwen2.5:3b` and `qwen2.5:1.5b`.
3. Integrate `aiohttp` in Python to send transcripts and receive streaming responses.
4. Implement the "Skip LLM" hotkey logic and latency mitigations (skipping LLM for clean/short inputs).

### Phase 3: macOS Deep Integration (The Glue)
1. Install `pyobjc-core`, `pyobjc-framework-Cocoa`, and `pyobjc-framework-Quartz`.
2. Write the deep context logic (AppKit + AppleScript).
3. Set up the `NSEvent` global listener for robust hotkeys (avoiding `pynput` conflicts).
4. Implement the dual-path clipboard pasting logic (`Quartz.CGEvent` -> AppleScript fallback).

### Phase 4: UI, Error States & Packaging
1. Wrap the script using `rumps`. Add error state indicators (Ollama status, missing permissions).
2. Implement custom vocabulary (proper nouns, code identifiers).
3. Build a simple test suite using `pytest` and golden transcript fixtures.
4. Use `pyinstaller` to package into `.app`. Implement auto-updates via GitHub releases or `sparkle`. Code sign and notarize the bundle.
