# GenMedia Creative Studio — CLAUDE.md

This file provides AI assistants with a comprehensive guide to the codebase structure, conventions, and development workflows.

## Project Overview

A production-grade **Vertex AI Generative Media showcase application** built with Python, [Mesop](https://mesop-dev.github.io/mesop/) (Google's Python UI framework), and FastAPI. It exposes Vertex AI APIs — Veo (video), Imagen (image), Lyria (music), Chirp 3 HD (speech), Gemini TTS, and Gemini image generation — through a unified web UI, with media stored in Google Cloud Storage and metadata in Firestore.

- **Python ≥ 3.13**, **Mesop 1.1.0**, **FastAPI 0.121.3**
- Deployed to **Cloud Run** behind **IAP** (Identity-Aware Proxy)
- Package manager: **UV** (not pip)

---

## Repository Structure

```
vertex-ai-creative-studio/
├── main.py                    # FastAPI app entry point; imports all pages
├── app_factory.py             # create_app() factory; on_load handler
├── Procfile                   # gunicorn + uvicorn worker startup
├── Dockerfile                 # Python 3.13-slim container
├── pyproject.toml             # Project metadata and dependencies
├── requirements.txt           # Pinned deps (auto-generated via UV)
├── dotenv.template            # Required env vars template
├── cloudbuild.yaml            # Cloud Build CI/CD pipeline
├── devserver.sh               # Local dev server startup script
├── build.sh                   # Build & deploy to Cloud Build
│
├── pages/                     # One file per UI page (42+ pages)
├── components/                # Reusable Mesop UI components (25+ dirs)
├── models/                    # AI model integration & business logic
├── common/                    # Shared services and utilities
├── config/                    # Configuration, nav, prompt templates
├── state/                     # Global Mesop state (AppState)
├── routers/                   # FastAPI route handlers (veo_router.py)
├── services/                  # Service layer (veo_service.py)
├── workflows/                 # Compound workflow pages
├── test/                      # pytest integration & feature tests
├── experiments/               # Standalone experimental apps (MCP, etc.)
├── plans/                     # Feature planning documents
├── docs/                      # Documentation assets
├── prompts/                   # Prompt template files
├── infra/                     # Terraform IaC (main.tf, variables.tf)
└── assets/                    # Static assets (ffmpeg binaries)
```

---

## Key Files

| File | Purpose |
|---|---|
| `main.py` | Imports all pages (registers them), mounts Mesop on `/app`, FastAPI routes for `/api/*` and media proxy |
| `app_factory.py` | `create_app()` creates FastAPI instance; `create_on_load_handler()` sets AppState from IAP headers |
| `state/state.py` | Global `AppState` stateclass: `user_email`, `session_id`, `theme_mode`, `sidenav_open`, `current_page` |
| `config/default.py` | All env-var-backed configuration (PROJECT_ID, model IDs, bucket names). **Source of truth for all config values** |
| `config/navigation.json` | Navigation structure defining sidebar menu entries |
| `common/metadata.py` | `MediaItem` dataclass + Firestore read/write helpers |
| `common/analytics.py` | `@track_click`, `log_ui_click`, `track_model_call` instrumentation |
| `common/utils.py` | `gcs_uri_to_https_url()`, `https_url_to_gcs_uri()` — always use these for GCS URI conversion |
| `common/storage.py` | Cloud Storage operations |

---

## Architecture: Mesop + FastAPI Integration

```
Browser
  │
  ▼
FastAPI (main.py)
  ├── /              → RedirectResponse to /app/home
  ├── /api/*         → FastAPI route handlers (REST API)
  ├── /media/*       → GCS signed-URL proxy
  └── /app/*         → WSGIMiddleware → Mesop WSGI app
```

**Critical:** Mesop is mounted at `/app` (not `/`). FastAPI handles root-level paths first. Never mount Mesop at `/` or FastAPI routes at sub-paths under `/app`.

### Request Flow for a Generative Feature

1. User interacts with **`pages/`** → Mesop event handler fires
2. Handler calls function in **`models/`** to invoke Vertex AI API
3. Result saved to Firestore via **`common/metadata.py`**
4. Handler updates **`state/`** and `yield`s → UI re-renders

---

## Development Setup

```bash
# Install dependencies (use UV, not pip)
uv sync

# Copy and configure environment
cp dotenv.template .env
# Edit .env: set PROJECT_ID at minimum

# Run local dev server
./devserver.sh
# OR
gunicorn --bind :8080 --workers 1 --threads 8 --timeout 0 \
  -k uvicorn.workers.UvicornWorker main:app
```

### Environment Variables (`.env`)

```bash
PROJECT_ID=             # REQUIRED: Google Cloud project ID
LOCATION=us-central1    # Optional, default: us-central1
MODEL_ID=               # Optional, default: gemini-2.0-flash
GENMEDIA_BUCKET=        # Optional, default: {PROJECT_ID}-assets
VEO_PROJECT_ID=         # Optional, defaults to PROJECT_ID
VEO_MODEL_ID=           # Optional, default: veo-2.0-generate-001
LYRIA_MODEL_VERSION=    # Optional, default: lyria-base-001
```

### Running Tests

```bash
uv run pytest test/
```

---

## Adding a New Page (Standard Pattern)

### 1. Create `pages/my_feature.py`

```python
import mesop as me
from state.state import AppState
from components.header import header
from components.page_scaffold import page_frame, page_scaffold

@me.stateclass
class PageState:
    my_value: str = ""

def my_feature_content():
    state = me.state(PageState)
    with page_frame():
        header("My Feature", "icon_name")
        me.text(state.my_value)

@me.page(
    path="/my_feature",
    title="My Feature - GenMedia Creative Studio",
)
def page():
    with page_scaffold(page_name="my_feature"):
        my_feature_content()
```

### 2. Register in `main.py`

```python
from pages import my_feature as my_feature_page
```

### 3. Add to `config/navigation.json`

```json
{
  "id": 70,
  "display": "My Feature",
  "icon": "icon_name",
  "route": "/my_feature",
  "group": "workflows"
}
```

---

## Critical Mesop Patterns

### State Management

- **Page-local state must live in the same file as the page.** Only `AppState` goes in `state/state.py`.
- **Get global state inside event handlers** by calling `app_state = me.state(AppState)`. Never pass state as parameters to components.
- **Mutable defaults** in stateclass: use `field(default_factory=list)` not `[]`.

```python
def on_click(e: me.ClickEvent):
    app_state = me.state(AppState)   # correct way to access global state
    state = me.state(PageState)       # correct way to access page state
```

### Event Handlers

- The function assigned to `on_click`, `on_value_change`, etc. **must be the generator that `yield`s**. Never wrap a generator in a `lambda`.
- Always `yield` after updating state to trigger UI refresh.
- `me.yield_value(...)` is removed — use plain `yield`.

### Custom Components

- **ALWAYS read the source file before using a custom component.** Their APIs are project-specific and may differ from standard Mesop patterns. A `TypeError: unexpected keyword argument` means you need to read the component source.
- `me.EventHandler` does not exist — use `typing.Callable` for callback type hints.
- The `key` parameter is for **native Mesop components only**. Custom `@me.component` functions need a dedicated `component_id: str` parameter if keying is needed.
- `me.Style` has **no `combine` method** — manually construct a new `me.Style` when merging.

### Theme and Styling

- **Never hardcode colors.** Use `me.theme_var("surface")` etc.
- Use `overflow_y="auto"` on scrollable containers.
- `me.link` does not support `target` parameter.
- For icon buttons: use `me.content_button` containing `me.icon` (standard `me.button` has no icon support).

### `@me.content_component`

```python
# CORRECT: always render me.slot(), control visibility via style
@me.content_component
def my_dialog(is_open: bool):
    with me.box(style=me.Style(display="block" if is_open else "none")):
        me.slot()
```

### Lambda Late Binding in Loops

```python
# WRONG: captures variable by reference
on_click=lambda e: handle(item)

# CORRECT: capture value at definition time
on_click=lambda e, v=item: handle(v)
# OR use functools.partial
on_click=partial(handle, arg=item)
```

### Dialogs and Responsiveness

Close dialog *before* doing work for perceived responsiveness:

```python
def on_select(e):
    state = me.state(PageState)
    state.show_dialog = False
    yield  # close dialog immediately
    # then do slow work
    do_slow_work()
    yield
```

---

## Analytics Instrumentation

Every new feature should be instrumented. The `page_scaffold` handles page view tracking automatically.

```python
from common.analytics import track_click, log_ui_click, track_model_call

# Button clicks
@track_click(element_id="imagen_generate_button")
def on_generate(e: me.ClickEvent):
    yield

# Sliders, selects, text inputs
def on_slider_change(e: me.SliderValueChangeEvent):
    app_state = me.state(AppState)
    log_ui_click(
        element_id="veo_duration_slider",
        page_name=app_state.current_page,
        session_id=app_state.session_id,
        extras={"value": e.value},
    )
    yield

# Model calls
with track_model_call("veo-2.0-generate-001", prompt_length=len(prompt)):
    result = model.generate(...)
```

**Element ID convention:** `{page_name}_{element_type}_{name}` — e.g., `imagen_generate_button`, `veo_aspect_ratio_select`.

---

## GCS URI Handling

Always use the centralized utilities:

```python
from common.utils import gcs_uri_to_https_url, https_url_to_gcs_uri

# For display in me.image / me.video
url = gcs_uri_to_https_url("gs://bucket/path/file.jpg")

# For model API calls (require gs:// URI)
uri = https_url_to_gcs_uri("https://storage.googleapis.com/bucket/path/file.jpg")
```

GCS URI format: `gs://<bucket-name>/<object-name>` — never include `gs://` twice.

Use `uuid.uuid4()` for unique filenames when saving to GCS:
```python
filename = f"output_{uuid.uuid4()}.mp4"
```

---

## Data Lifecycle Checklist (New Feature Fields)

When adding a new data field, verify all steps:

- [ ] **State:** Added to appropriate `@me.stateclass`
- [ ] **UI Input:** Input component added in `pages/`
- [ ] **Request Schema:** `models/requests.py` updated
- [ ] **Model Logic:** `models/` generation function uses the field
- [ ] **Persistence Write:** `MediaItem` in `common/metadata.py` updated
- [ ] **Persistence Read:** `_create_media_item_from_dict` in `common/metadata.py` updated ← **most often missed**
- [ ] **UI Display:** Field shown in library detail view
- [ ] **Edge Cases:** Clear/Reset buttons handle the new field

---

## Circular Import Prevention

- `main.py` is the root of the dependency tree. **Nothing should import from `main.py`.**
- `AppState` lives in `state/state.py` — import from there everywhere.
- Avoid module-level I/O in config files. Wrap in functions and call lazily:

```python
# BAD (module-level side effect)
CONFIG_DATA = load_from_file()

# GOOD (lazy)
def get_config_data():
    return load_from_file()
```

---

## Refactoring Protocols

### Before Changing a Function Signature

1. Search codebase for all callers of the function
2. List every file that calls it
3. Update every call site

### When a `replace` Fails

If a file edit fails with "0 occurrences found", **read the file first** to get the current content — do not rely on memory.

### Renaming Data Fields

1. `grep` for all usages across the entire codebase
2. Update **both** write paths and read paths
3. Run tests after

---

## Configuration Conventions

- All config values (model IDs, bucket names, endpoints) come from `config/default.py`. **Never hardcode magic strings** in pages or models.
- Navigation entries go in `config/navigation.json`.
- Prompt templates go in `config/image_prompt_templates.json` or `prompts/`.

---

## Deployment

```bash
# Build and deploy via Cloud Build
./build.sh

# Procfile (Gunicorn + Uvicorn worker for ASGI)
web: gunicorn --bind :$PORT --workers 1 --threads 8 --timeout 0 \
  -k uvicorn.workers.UvicornWorker main:app
```

**Cloud Run timeout:** Set `--timeout 3600` on `gcloud run deploy` for long-running video generations (Cloud Run default is 300s regardless of Gunicorn's `--timeout 0`).

**CI/CD:** `cloudbuild.yaml` → build Docker image → push to Artifact Registry → deploy to Cloud Run.

---

## Content Security Policy

Applied globally via FastAPI middleware in `main.py` (not per-page Mesop decorator), as page-level headers don't propagate through `WSGIMiddleware`.

---

## Common Pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| `TypeError: unexpected keyword argument` | Wrong custom component API | Read component source file |
| `AttributeError: me.EventHandler` | Invalid type hint | Use `typing.Callable` |
| `NameError: AppState not defined` | Circular import or import-time side effect | Check import order; defer I/O |
| Event handler uses last loop value | Python late binding | Use default arg or `functools.partial` |
| UI doesn't update after state change | `yield` missing | Add `yield` after state mutation |
| `TypeError` passing state to component | Mesop state is not serializable as param | Call `me.state(MyState)` inside handler |
| Blank UI with 404s for `prod_bundle.js` | Missing static file mount | Check `StaticFiles` mount in `main.py` |
| 503 after 5 minutes for video | Cloud Run timeout | Set `--timeout 3600` on deploy |
