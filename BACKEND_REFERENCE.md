# Backend Technical Reference

Last updated: 2026-03-03

## 1) Backend scope

The backend is a FastAPI service that:

- Serves frontend HTML/CSS/JS files
- Handles authentication (JWT + bcrypt)
- Stores chat data in SQLite and user profiles/settings in JSON
- Processes uploaded files into prompt-ready text
- Runs local `autobot-instruct` inference with streaming output
- Supports stop/continue generation workflow
- Exposes model status and health endpoints

## 2) Backend file map

- `backend/main.py`: FastAPI app, routing, middleware, chat streaming pipeline
- `backend/config.py`: settings/defaults, paths, allowed upload extensions, default prompt
- `backend/auth.py`: users JSON management, password hashing, JWT token lifecycle
- `backend/database.py`: SQLite schema and CRUD for users/conversations/messages
- `backend/file_processor.py`: file parsing for text/docs/pdf/sheets/images
- `backend/model_manager.py`: model load/unload, prompt construction, generation, tool loop
- `backend/requirements.txt`: runtime dependencies

Related runtime modules:

- `tools/tool_registry.py`: tool runtime registry (currently `web_search`)
- `tools/tool_detector.py`: parse model tool-call outputs

## 3) Runtime architecture

## 3.1 App startup and shutdown

- FastAPI app uses `lifespan` in `backend/main.py`.
- On startup:
  - verifies model path
  - optionally preloads model when `PRELOAD_MODEL=true`
- On shutdown:
  - calls `model_manager.unload_model()`

## 3.2 Middleware and app config

- CORS enabled with:
  - `allow_origins=settings.CORS_ORIGINS` (default `["*"]`)
  - `allow_methods=["*"]`
  - `allow_headers=["*"]`
- App title/version comes from config.

## 3.3 Authentication model

- Security scheme: `HTTPBearer(auto_error=False)`
- Access token:
  - JWT signed with `SECRET_KEY` and `HS256`
  - expiration default: `ACCESS_TOKEN_EXPIRE_MINUTES = 7 days`
- Protected endpoints use `Depends(get_current_user)`.
- `get_current_user` validates token, validates user from `users.json`, and ensures matching SQLite user row exists.

## 4) Data storage model

## 4.1 `data/users.json`

Stores auth and settings per username. Structure per user:

```json
{
  "hashed_password": "<bcrypt hash>",
  "created_at": "2026-03-03T10:00:00.000000",
  "settings": {
    "temperature": 0.7,
    "max_tokens": 32768,
    "system_prompt": "",
    "theme": "dark"
  }
}
```

## 4.2 SQLite schema (`data/chatdat.db`)

Initialized in `database.py` on module import.

### `users`

- `id INTEGER PRIMARY KEY AUTOINCREMENT`
- `username TEXT UNIQUE NOT NULL`
- `created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP`

### `conversations`

- `id INTEGER PRIMARY KEY AUTOINCREMENT`
- `user_id INTEGER NOT NULL` -> FK to `users.id`
- `title TEXT DEFAULT 'New Chat'`
- `created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP`
- `updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP`

### `messages`

- `id INTEGER PRIMARY KEY AUTOINCREMENT`
- `conversation_id INTEGER NOT NULL` -> FK to `conversations.id`
- `role TEXT NOT NULL CHECK (role IN ('user','assistant','system'))`
- `content TEXT NOT NULL`
- `files_json TEXT` (JSON array of file metadata)
- `timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP`

Indexes:

- `idx_messages_conversation`
- `idx_conversations_user`

## 5) File upload and extraction pipeline

## 5.1 Limits and validation

- Max file size: `10 MB` per file (`MAX_FILE_SIZE`)
- Allowed extensions include:
  - text/code/config (`.txt .md .py .js .ts .html .css ...`)
  - docs (`.pdf .docx`)
  - sheets/data (`.csv .xlsx .xls .json .xml .yaml ...`)
  - images (`.jpg .jpeg .png .webp .bmp .gif`)
  - logs/scripts (`.sql .log .sh .bat .ps1`)

## 5.2 Processing behavior

For each uploaded file:

- saved to `data/uploads/` with unique filename
- parsed into text by file type
- truncated to max `80,000` chars per file content
- returned in message metadata with `success`, `mime_type`, `content`, and save info

Extraction libraries:

- PDF: `PyPDF2`
- DOCX: `python-docx`
- XLSX: `openpyxl`
- XLS: `xlrd` (optional)
- Images: `Pillow` (+ OCR via `pytesseract` if available)

## 6) Model and generation pipeline

## 6.1 Model manager behavior

- Loads local model from `models/autobot-instruct`
- Resolves runtime device (`cuda` if available, else `cpu`)
- Tracks runtime context window (`N_CTX`, default `32768`)
- Enforces generation cap: `max_generation_tokens_limit = N_CTX - 128`

## 6.2 Prompt building

- Uses default system prompt from `config.py`
- Includes optional user custom instructions
- Includes formatted attached file content
- Includes conversation history (`user` + `assistant`)
- Truncates history/content/query dynamically to fit context

## 6.3 Streaming generation

- `/api/chat/stream` returns `text/event-stream`
- Model output is chunked into token-like pieces and sent as SSE JSON payloads
- Final SSE event is emitted only after assistant message persistence completes

## 6.4 Tool loop (web search)

- `model_manager` can detect tool-call output and execute `web_search` via `ToolRegistry`
- Tool results are normalized and fed back into generation
- Maximum tool rounds per response: `3`

## 7) API conventions

## 7.1 Base and auth

- Base URL: same origin (examples: `/api/...`)
- Auth header for protected routes:
  - `Authorization: Bearer <token>`

## 7.2 Content types

- JSON for most APIs
- `multipart/form-data` for:
  - `/api/chat/stream`
  - `/api/chat/stop`
  - `/api/user/change-password`

## 7.3 Error contract

Custom handlers return:

```json
{
  "error": true,
  "detail": "message"
}
```

- `HTTPException` keeps original status code
- uncaught exceptions return HTTP `500`

## 8) Request/response models used in `main.py`

### `UserRegister`

```json
{
  "username": "3-30 chars",
  "password": "min 6 chars"
}
```

### `UserLogin`

```json
{
  "username": "string",
  "password": "string"
}
```

### `SettingsUpdate` (all optional)

```json
{
  "temperature": 0.7,
  "max_tokens": 4096,
  "system_prompt": "custom instructions",
  "theme": "dark"
}
```

Server clamps:

- `temperature` to `0.0..2.0`
- `max_tokens` to `1..max_generation_tokens_limit`

### `TitleUpdate`

```json
{
  "title": "1-100 chars"
}
```

## 9) Endpoint reference (backend)

## 9.1 Frontend-serving routes

### `GET /`

- Purpose: serve main chat UI
- Response: `frontend/index.html` file

### `GET /login`

- Purpose: serve login page
- Response: `frontend/login.html`

### `GET /register`

- Purpose: serve registration page
- Response: `frontend/register.html`

### `GET /profile`

- Purpose: serve profile page
- Response: `frontend/profile.html`

### `GET /css/{file_path:path}`

- Purpose: serve CSS assets
- Response: `frontend/css/{file_path}`

### `GET /js/{file_path:path}`

- Purpose: serve JS assets
- Response: `frontend/js/{file_path}`

## 9.2 Auth APIs

### `POST /api/auth/register`

- Auth required: No
- Request JSON:

```json
{
  "username": "my_user",
  "password": "my_password"
}
```

- Success response:

```json
{
  "success": true,
  "message": "User registered successfully",
  "access_token": "<jwt>",
  "token_type": "bearer",
  "username": "my_user"
}
```

- Errors:
  - `400`: validation/duplicate username
  - `500`: registration failure

### `POST /api/auth/login`

- Auth required: No
- Request JSON:

```json
{
  "username": "my_user",
  "password": "my_password"
}
```

- Success response:

```json
{
  "success": true,
  "access_token": "<jwt>",
  "token_type": "bearer",
  "username": "my_user"
}
```

- Errors:
  - `401`: invalid username or password

### `POST /api/auth/logout`

- Auth required: Yes
- Request body: none
- Success response:

```json
{
  "success": true,
  "message": "Logged out successfully"
}
```

### `GET /api/auth/me`

- Auth required: Yes
- Success response:

```json
{
  "username": "my_user",
  "created_at": "2026-03-03T10:00:00.000000",
  "settings": {
    "temperature": 0.7,
    "max_tokens": 32768,
    "system_prompt": "",
    "theme": "dark"
  }
}
```

## 9.3 User APIs

### `GET /api/user/profile`

- Auth required: Yes
- Purpose: profile + settings + usage statistics
- Success response:

```json
{
  "username": "my_user",
  "created_at": "2026-03-03T10:00:00.000000",
  "settings": {
    "temperature": 0.7,
    "max_tokens": 32768,
    "system_prompt": "",
    "theme": "dark"
  },
  "stats": {
    "total_conversations": 4,
    "total_messages": 39
  }
}
```

### `PUT /api/user/settings`

- Auth required: Yes
- Request JSON (any subset):

```json
{
  "temperature": 0.9,
  "max_tokens": 4096,
  "system_prompt": "Be concise",
  "theme": "light"
}
```

- Success response:

```json
{
  "success": true,
  "settings": {
    "temperature": 0.9,
    "max_tokens": 4096,
    "system_prompt": "Be concise",
    "theme": "light"
  }
}
```

- Errors:
  - `400`: failed to persist settings

### `POST /api/user/change-password`

- Auth required: Yes
- Content type: `multipart/form-data`
- Form fields:
  - `old_password` (required)
  - `new_password` (required, min length 6)
- Success response:

```json
{
  "success": true,
  "message": "Password changed successfully"
}
```

- Errors:
  - `400`: invalid current password or invalid new password

## 9.4 Conversation APIs

### `GET /api/conversations`

- Auth required: Yes
- Success response:

```json
{
  "conversations": [
    {
      "id": 1,
      "user_id": 1,
      "title": "New Chat",
      "created_at": "2026-03-03 10:00:00",
      "updated_at": "2026-03-03 10:05:00"
    }
  ]
}
```

### `POST /api/conversations`

- Auth required: Yes
- Request body: none
- Success response:

```json
{
  "success": true,
  "conversation_id": 123,
  "title": "New Chat"
}
```

### `GET /api/conversations/{conversation_id}/messages`

- Auth required: Yes
- Path param:
  - `conversation_id` (int)
- Success response:

```json
{
  "conversation": {
    "id": 1,
    "user_id": 1,
    "title": "My chat",
    "created_at": "2026-03-03 10:00:00",
    "updated_at": "2026-03-03 10:05:00"
  },
  "messages": [
    {
      "id": 10,
      "conversation_id": 1,
      "role": "user",
      "content": "Hello",
      "files": [],
      "timestamp": "2026-03-03 10:00:01"
    }
  ]
}
```

- Errors:
  - `404`: conversation not found / not owned by user

### `PUT /api/conversations/{conversation_id}/title`

- Auth required: Yes
- Request JSON:

```json
{
  "title": "New title"
}
```

- Success response:

```json
{
  "success": true,
  "title": "New title"
}
```

- Errors:
  - `400`: failed update

### `DELETE /api/conversations/{conversation_id}`

- Auth required: Yes
- Success response:

```json
{
  "success": true,
  "message": "Conversation deleted"
}
```

- Errors:
  - `400`: delete failed

## 9.5 Chat APIs

### `POST /api/chat/stop`

- Auth required: Yes
- Content type: `multipart/form-data`
- Form fields:
  - `request_id` (required string)
- Success response:

```json
{
  "success": true,
  "request_id": "abc123",
  "stopped": true
}
```

Behavior:

- Returns `stopped=false` when request id is unknown/already ended.
- Returns `403` if request id belongs to another user.

### `POST /api/chat/stream`

- Auth required: Yes
- Content type: `multipart/form-data`
- Form fields:
  - `message` (required string)
  - `conversation_id` (optional int)
  - `temperature` (optional float)
  - `max_tokens` (optional int)
  - `request_id` (optional string)
  - `persist_user_message` (optional bool, default `true`)
  - `continuation_mode` (optional bool, default `false`)
  - `continuation_prefix` (optional string)
  - `continuation_message_id` (optional int)
  - `files` (optional list of files)

Response:

- `200` with `text/event-stream`
- Headers:
  - `X-Conversation-Id`
  - `X-Request-Id`
  - `Cache-Control: no-cache`
  - `Connection: keep-alive`
  - `X-Accel-Buffering: no`

SSE payload shapes (`data: <json>`):

1. Token chunk:

```json
{
  "token": "partial text",
  "finish_reason": null
}
```

2. Mid-stream error:

```json
{
  "error": "error text"
}
```

3. Final done:

```json
{
  "done": true,
  "assistant_message_id": 101,
  "error": null
}
```

4. Final stopped:

```json
{
  "stopped": true,
  "done": true,
  "assistant_message_id": 101,
  "error": null
}
```

Server behavior details:

- Creates conversation if `conversation_id` not provided.
- Persists user message unless `persist_user_message=false` or continuation path.
- In continuation mode, appends generated tail to existing assistant message where possible.
- If model runtime unavailable or model load fails, emits SSE error + done events.

Common errors before stream starts:

- `401`: not authenticated / invalid token
- `404`: invalid conversation id for user

## 9.6 Model and health APIs

### `GET /api/model/info`

- Auth required: No
- Success response includes:
  - model existence/load/availability flags
  - context window and max generation limit
  - runtime device, transformer/torch versions
  - model size in MB

Example:

```json
{
  "model_path": "C:/.../models/autobot-instruct",
  "model_exists": true,
  "is_loaded": false,
  "is_available": true,
  "last_error": null,
  "transformers_version": "4.57.0",
  "torch_version": "2.2.0",
  "context_window": 32768,
  "model_train_context_window": 32768,
  "max_generation_tokens_limit": 32640,
  "threads": 4,
  "default_temperature": 0.7,
  "default_max_tokens": 32768,
  "runtime_device": "cpu",
  "model_size_mb": 2048.5
}
```

### `GET /api/health`

- Auth required: No
- Success response:

```json
{
  "status": "healthy",
  "model_available": true,
  "model_loaded": false,
  "model_exists": true
}
```

## 10) Chat continuation behavior

Continuation mode is built for interrupted outputs:

- Frontend sends:
  - `continuation_mode=true`
  - assistant prefix text via `continuation_prefix`
  - optional `continuation_message_id`
- Backend:
  - builds continuation prompt ending with existing assistant prefix
  - strips repeated prefixes like "Let's continue..."
  - appends continuation to target assistant message when possible

## 11) Configuration reference (`backend/config.py`)

Important settings (defaults):

- `HOST=0.0.0.0`
- `PORT=8000`
- `DEBUG=false`
- `SECRET_KEY=...` (must be changed in production)
- `ACCESS_TOKEN_EXPIRE_MINUTES=10080` (7 days)
- `PRELOAD_MODEL=true`
- `N_CTX=32768`
- `N_THREADS=4`
- `TEMPERATURE=0.7`
- `MAX_TOKENS=32768`
- `TOP_P=0.9`
- `TOP_K=40`
- `DATABASE_PATH=data/chatdat.db`
- `USERS_JSON_PATH=data/users.json`
- `UPLOAD_DIR=data/uploads`
- `MAX_FILE_SIZE=10MB`

## 12) Dependencies (`backend/requirements.txt`)

Core:

- FastAPI, Uvicorn, Starlette
- python-jose, bcrypt, python-multipart
- Pydantic + pydantic-settings
- python-dotenv

Model/inference:

- torch
- transformers
- accelerate

File processing:

- PyPDF2
- python-docx
- Pillow
- openpyxl
- xlrd
- pytesseract

## 13) Logging and observability

- Chat flow emits structured logs via `log_chat_status(...)` with stages like:
  - `request_received`
  - `conversation_ready`
  - `files_processed`
  - `history_loaded`
  - `prompt_built`
  - `generation_started`
  - `generation_completed` / `generation_stopped`
  - `generation_error`

These logs include request id, conversation id, user, timing, and size metrics.

## 14) Current implementation notes

- Auth records and chat records are split across JSON + SQLite; `ensure_db_user()` keeps them aligned.
- `ChatRequest` pydantic model exists but stream endpoint currently uses form fields directly.
- CORS is fully open by default (`*`); tighten for production.
- `/api/model/info` and `/api/health` are public (no auth).
