# LocalFlow for macOS: Product Requirements & Technical Design Document

## 1. High-Level Architecture (macOS Restructure)

To replicate Wispr Flow’s lightning-fast experience natively on a Mac, the architecture must pivot to take full advantage of macOS-specific APIs and Apple Silicon (M-Series chips).

Apple's **Unified Memory Architecture (UMA)** and **Metal Performance Shaders (MPS)** allow us to run heavy STT and LLM models locally with near-zero latency, often outperforming network-bound cloud calls.

*   **macOS / Client Layer (Menubar App):** Instead of a heavy UI, the app will live in the macOS Menu Bar (using Python's `rumps` or a lightweight Swift UI). It requests **macOS Accessibility Permissions** and **Microphone Permissions** on first launch.
*   **Audio Capture & Hardware VAD:** Captures CoreAudio streams. Voice Activity Detection (VAD) monitors the mic and chunks audio efficiently to prevent memory bloat.
*   **Mac-Optimized STT Engine (Apple MLX):** Instead of standard `faster-whisper`, we utilize **`mlx-whisper`**. MLX is Apple’s native array framework built for Apple Silicon, allowing Whisper to run purely on the GPU with zero-copy memory overhead.
*   **Local LLM Layer (Ollama via Metal):** The raw transcript is passed to a locally running instance of Ollama. Ollama natively utilizes macOS Metal, allowing ultra-fast inference on small models like `llama3.2:3b` or `phi3`.
*   **macOS Context & Injection Layer:**
    *   *Context Grabber:* Uses macOS `AppKit` (`NSWorkspace`) to instantly read the name of the foreground application (e.g., "Slack", "Xcode", "Safari").
    *   *Injection:* Copies the polished text to the macOS clipboard (`NSPasteboard`) and uses `CGEvent` (via `pynput` or `Quartz`) to synthesize a native `Cmd + V` keystroke, instantly pasting the text into the active field.

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
| **Global Hotkey** | A system-wide customizable shortcut (default: `Cmd + Option + Space`) mapped via macOS Accessibility APIs. | P0 |
| **Apple Silicon STT** | Transcribe speech offline using `mlx-whisper` for maximum GPU utilization on Mac. | P0 |
| **Local LLM Editing** | Pass raw text to an Ollama-hosted model to remove filler words, fix grammar, and apply context formatting. | P0 |
| **Native Text Injection** | Synthesize `Cmd + V` to paste text instantly into the user's active window. | P0 |
| **macOS Context Awareness** | Detect active App (e.g., Xcode vs. Messages) via `NSWorkspace` and adjust LLM prompt tone automatically. | P1 |
| **Menubar UI** | A lightweight native-feeling status bar icon to show recording state, model loading status, and settings. | P1 |
| **Voice Commands** | "New paragraph", "Delete last sentence", or "Format as python code". | P2 |

### Non-Functional Requirements
*   **Privacy:** 100% offline. Zero network calls outside of `localhost` (Ollama).
*   **Latency:** Audio-stop to text-insertion must be **< 1.5 seconds** (achievable via Apple Silicon Unified Memory).
*   **Hardware Compatibility:** Optimized for Apple Silicon (M1/M2/M3/M4) with at least 8GB Unified Memory (16GB recommended). Intel Macs supported gracefully via CPU fallbacks.
*   **Permissions:** Must gracefully handle macOS sandboxing (Microphone access, Accessibility access).

---

## 3. Technical Design Document (TDD) for macOS

### Architecture Stack
*   **Orchestrator:** Python 3.10+ (Simplest for tying ML and OS APIs together).
*   **UI / Menubar:** `rumps` (Ridiculously Uncomplicated macOS Python Statusbar apps) for the system tray icon.
*   **Audio Engine:** `sounddevice` or `pyaudio` + `silero-vad` (fast voice activity detection).
*   **STT Engine:** `mlx-whisper` (Apple's MLX framework) with `distil-whisper-large-v3` or `base.en`.
*   **LLM Engine:** `Ollama` running as a background service. Models: `llama3.2:3b` or `phi3:mini`.
*   **macOS Automation:**
    *   Context: `pyobjc` framework (specifically `AppKit.NSWorkspace.sharedWorkspace().frontmostApplication().localizedName()`).
    *   Hotkey & Injection: `pynput` for listening to global hotkeys and simulating the `Cmd + V` keystroke. `pyperclip` for clipboard manipulation.

### Sequential Data Flow (Mac Context)
1.  **Idle State:** App lives in the macOS Menubar. Models reside in Unified Memory (if cached).
2.  **Trigger:** User presses and holds `Option + Space`.
3.  **Context Grab:** Python script instantly queries `AppKit` to determine the active app (e.g., "Notes").
4.  **Capture:** Audio buffers into a temporary `.wav` in memory/`/tmp` while the hotkey is held.
5.  **Release:** User releases the hotkey (or VAD detects silence).
6.  **STT Phase:** Audio array is passed to `mlx-whisper`. Transcript generated using Mac GPU.
7.  **LLM Phase:** Transcript + App Context ("Notes") passed to `localhost:11434` (Ollama). System prompt instructs the model to format text appropriately for a note-taking app.
8.  **Output & Injection Phase:**
    *   Text is placed on the macOS `NSPasteboard`.
    *   `pynput` synthesizes `Cmd (Command) + V`.
    *   Text appears in the user's active window seamlessly.

---

## 4. Implementation Roadmap (macOS Action Plan)

### Phase 1: Environment & Core STT
1. Setup Python `venv` on macOS.
2. Install Mac-specific ML packages: `pip install mlx-whisper sounddevice numpy`.
3. Write the audio recording loop and test `mlx-whisper` transcription latency on Apple Silicon.

### Phase 2: Local LLM Integration
1. Install Ollama for Mac (native app via `ollama.com`).
2. Pull a fast model: `ollama run phi3` or `ollama run llama3.2`.
3. Integrate the `requests` library in Python to send the transcription output to the local Ollama API.
4. Refine the system prompt to prevent conversational padding (e.g., "Here is your text:").

### Phase 3: macOS Deep Integration (The Glue)
1. Install `pyobjc-core` and `pyobjc-framework-Cocoa`.
2. Write the active window detection logic using `NSWorkspace`.
3. Set up `pynput` to bind the global push-to-talk hotkey. *Note: Running this will prompt macOS to ask for Accessibility permissions in System Settings.*
4. Implement clipboard pasting logic.

### Phase 4: UI & Packaging
1. Wrap the script using `rumps` to create a clickable Menubar icon (Start/Stop, Settings, Quit).
2. Use `pyinstaller` to package the Python script into a standalone `.app` bundle for macOS so it can be added to "Login Items".
