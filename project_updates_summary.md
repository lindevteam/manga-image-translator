# Project Analysis & Recent Updates Summary
*Date: 2025-12-11*

## 1. Project Structure Analysis

The project is a modular Manga/Image Translation system with a core processing engine and multiple interfaces.

### Core Component: `manga_translator/`
This is the heart of the project (`python package`).
-   **Core Logic**: `manga_translator.py` (Class `MangaTranslator`) orchestrates the pipeline: Extraction -> OCR -> Translation -> Inpainting/Rendering.
-   **Submodules**: `detection/`, `ocr/`, `translators/`, `rendering/`, `inpainting/`.
-   **Entry Point**: Can be run via CLI (`python -m manga_translator`).

### Interfaces (Consumers)
1.  **Web API Server (`server/`)**: FastAPI backend. Imports `manga_translator` to serve REST endpoints (e.g., `/translate/image`). Manages queues (`myqueue.py`).
2.  **Desktop App (`MangaStudioMain.py`)**: Standalone PySide6/Qt application. Uses `manga_translator` as a local library.
3.  **Web Frontend (`front/`)**: Vite + React/Vue modern frontend, communicates with the `server` API.

---

## 2. Recent Code Updates (`manga_translator/`)
*Timeline: September - October 2025*

The development focus has been on modularity and cleanup.

-   **Refactoring Renderer (Oct 3 & Sep 16, 2025)**:
    -   The `rendering` module was significantly refactored to be installable as a pip module.
    -   Merged `renderer-module` PR.
    -   Updated `text_render.py` and `text_render_pillow_eng.py`.
-   **Utils Cleanup (Sep 16, 2025)**:
    -   Consolidated utility functions.
    -   Added `sort.py` and updated `generic2.py`, `textblock.py`.
    -   Standardized imports to be relative.
-   **Revert Operation (Aug 13, 2025)**:
    -   Rolled back a major feature attempt ("add panel detector") due to likely stability issues.

---

## 3. Dependency Changes (`requirements.txt`)
*Timeline: June - August 2025*

A major architectural shift occurred in July, moving away from PaddlePaddle.

-   **Current Status (Aug 31, 2025)**:
    -   Unlocked `ctranslate2` version (removed `<=3.24.0` constraint).
-   **New Language Support (Jul 30, 2025)**:
    -   Added `python-bidi` for Arabic/RTL language support.
-   **Major Engine Swap (Jul 15, 2025)**:
    -   **REMOVED**: `paddleocr`, `paddlepaddle`, `paddlepaddle-gpu`.
    -   **ADDED**: `rusty-manga-image-translator` (Rust + ONNX implementation).
    -   *Impact*: Significant reduction in dependency bloat and likely performance improvements by checking `extra-index-url` for prebuilt Rust wheels.

---

## 4. Infrastructure Changes (`Dockerfile`)
*Timeline: February - March 2025*

Updates focused on GPU environment stability and entry point standardization.

-   **Entrypoint Standardization (Mar 6, 2025)**:
    -   Changed to `ENTRYPOINT ["python", "-m", "manga_translator"]`.
-   **GPU & Runtime Fixes (Feb 5, 2025)**:
    -   Manually installed `libcudnn8` (CUDA 11.8 support).
    -   Added `libvulkan1` (required for Waifu2x).
    -   Cleaned up build deps (`g++`, `wget`) to keep image small.
-   **Permission Fixes (Feb 2, 2025)**:
    -   Explicitly fixed `/tmp` directory permissions (`chmod 1777`) to prevent runtime errors with libraries like GIMP or multiprocessing.

---

## 5. Architecture & Runtime Insights (Added Dec 12, 2025)

### Running on Mac M1 (Apple Silicon)
- **Recommendation**: Run locally via `server/main.py` instead of Docker.
- **Reason**: Docker images are often amd64/CUDA optimized. Local execution allows `torch` to use `mps` (Metal Performance Shaders) for native GPU acceleration.
- **Command**: `python main.py --use-gpu --verbose` (Start inside `server/` dir after installing requirements).

### Parameter Flow: `--use-gpu`
The flag propagates through the system architecture as follows:
1.  **User Input**: CLI argument passed to `server/main.py`.
2.  **Web Server (Parent Process)**: Parses `--use-gpu`, then spawns a subprocess for the translator client via `server/main.py -> prepare` function.
3.  **Translator (Child Process)**: Receives `--use-gpu` in the spawn command line arguments in `manga_translator/__main__.py`.
4.  **Core Logic**: `MangaTranslator` class initializes. It checks `torch.backends.mps.is_available()` (on Mac). If true and `use_gpu` is set, it sets internal `device='mps'`.

### Web Server Architecture (Task Queue Pattern)
The server uses a **Task Queue + Worker Process** model to handle high-load translation tasks.
-   **Request Ingestion**: `server/request_extraction.py` receives image (Multipart/Base64), converts to PIL Image, and wraps it in a `QueueElement`.
-   **Queuing**: `server/myqueue.py` adds the element to `task_queue`.
-   **Scheduling**:
    -   `wait_in_queue` function monitors queue position.
    -   It waits for an available `ExecutorInstance` (subprocess).
    -   Supports streaming updates (SSE) to notify client of position (State 3).
-   **Execution**:
    -   When an instance is free, `server/instance.py` sends an HTTP request to the subprocess (e.g., `http://127.0.0.1:5004/execute/translate`).
    -   The subprocess performs the heavy lifting (Detection -> OCR -> Translate -> Render).
-   **Result**: The processed image is returned to the main process and then to the user.

### Batch Processing Support
The architecture fully supports simultaneous multi-image processing.
-   **Server Layer**: Endpoints `/translate/batch/json` and `/translate/batch/images` accept lists of images. They wrap requests in `BatchQueueElement`.
-   **Transmission**: The list of images is sent to the translator subprocess in a single HTTP request (`sent_batch`).
-   **Core Layer (`MangaTranslator.translate_batch`)**:
    -   **Efficiency**: Instead of looping single translations, it batches operations where possible.
    -   **Pipeline**:
        1.  **Pre-processing**: Runs detection and OCR for all images.
        2.  **Translation**: Batches text contexts together for efficient model inference (especially useful for LLMs).
        3.  **Post-processing**: Performs inpainting and rendering for each image.
