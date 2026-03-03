# Frontend Technical Reference

Last updated: 2026-03-03

## 1) Frontend scope

This frontend is a vanilla HTML/CSS/JavaScript client served by FastAPI. It includes:

- Auth pages: login and register
- Main chat UI with streaming assistant responses
- Profile and settings page
- Conversation history sidebar
- File attachment support (multipart uploads)
- Theme management (dark/light)

## 2) Frontend file map

- `frontend/index.html`: main chat page shell
- `frontend/login.html`: login form + inline auth fetch logic
- `frontend/register.html`: registration form + inline auth fetch logic
- `frontend/profile.html`: profile/settings UI + inline API logic
- `frontend/js/api.js`: shared API client wrappers
- `frontend/js/auth.js`: auth utility helpers (localStorage checks/validation)
- `frontend/js/chat.js`: chat state, rendering, SSE stream parsing, stop/continue flow
- `frontend/css/style.css`: app styles/layout/components
- `frontend/css/themes.css`: theme tokens/variables

## 3) Runtime behavior and state

### 3.1 Auth/session storage

The frontend stores session data in `localStorage`:

- `access_token`: JWT bearer token
- `username`: current username
- `theme`: local UI theme preference (set in chat page)

Token behavior:

- In `api.js`, protected requests include `Authorization: Bearer <token>`.
- If a protected request returns `401`, the client clears token data and redirects to `/login`.
- Login/register pages also do a startup token check with `GET /api/auth/me`.

### 3.2 Page routing

Server routes serving frontend pages:

- `GET /` -> `index.html`
- `GET /login` -> `login.html`
- `GET /register` -> `register.html`
- `GET /profile` -> `profile.html`
- `GET /css/{file_path}` -> static CSS
- `GET /js/{file_path}` -> static JS

### 3.3 Chat page state (`chat.js`)

Important in-memory state:

- `currentConversationId`: selected conversation id
- `attachedFiles`: pending files selected before send
- `isGenerating`: blocks concurrent sends
- `activeRequestId`: request id used for stop requests
- `activeRequestController`: `AbortController` for active request
- `activeStreamReader`: stream reader for SSE body
- `generationStoppedByUser`: stop flag for UI status

### 3.4 Chat send flow

1. User submits message (and optional files).
2. Frontend appends local user message to UI immediately.
3. Frontend `POST`s multipart form data to `/api/chat/stream`.
4. Response is `text/event-stream`; frontend reads body stream and parses `data: {json}` SSE events.
5. Assistant message is built incrementally from `token` events.
6. Terminal event (`done` or `stopped`) finalizes state and stores `assistant_message_id` on bubble element.

### 3.5 Stop and continue flow

Stop:

- Frontend calls `POST /api/chat/stop` with `request_id`.
- Backend sets stop event for that stream.
- Frontend marks message as stopped and shows `Continue` button.

Continue:

- Frontend sends `/api/chat/stream` again with:
  - internal prompt placeholder message
  - `persist_user_message=false`
  - `continuation_mode=true`
  - `continuation_prefix=<existing assistant text>`
  - `continuation_message_id=<assistant message id if available>`
- New stream tokens are appended to the existing assistant bubble.

### 3.6 File upload behavior

- Up to 5 files can be attached client-side in one send action.
- Files are sent as repeated `files` multipart fields.
- Backend processes file content and stores file metadata with user message.

## 4) API usage conventions

### 4.1 Base URL

- `API_BASE` in `frontend/js/api.js` is currently empty string `''`.
- Requests are same-origin (e.g., `/api/...`).

### 4.2 Content types used by frontend

- JSON (`application/json`) for most auth/settings/conversation endpoints
- Multipart form-data for:
  - `/api/chat/stream`
  - `/api/chat/stop`
  - `/api/user/change-password`

### 4.3 Common error shape from backend

FastAPI exception handlers return:

```json
{
  "error": true,
  "detail": "Error message"
}
```

The frontend commonly reads `detail`.

## 5) Data contracts used by frontend

### 5.1 User settings object

```json
{
  "temperature": 0.7,
  "max_tokens": 32768,
  "system_prompt": "",
  "theme": "dark"
}
```

### 5.2 Conversation object

```json
{
  "id": 1,
  "user_id": 1,
  "title": "New Chat",
  "created_at": "2026-03-03 12:34:56",
  "updated_at": "2026-03-03 12:35:01"
}
```

### 5.3 Message object

```json
{
  "id": 11,
  "conversation_id": 1,
  "role": "user",
  "content": "Hello",
  "files": [],
  "timestamp": "2026-03-03 12:35:01"
}
```

`files` can include processing metadata for uploaded files, for example:

```json
{
  "filename": "sample.txt",
  "mime_type": "text/plain",
  "size": 1234,
  "content": "extracted text...",
  "success": true,
  "saved_permanently": true,
  "saved_path": "C:/.../data/uploads/...",
  "saved_filename": "1772471627093_6fde6962_sample.txt"
}
```

## 6) Endpoint-by-endpoint reference

## Auth APIs

### `POST /api/auth/register`

- Purpose: create new user account and return JWT session token
- Called from: `register.html` inline script and `authAPI.register()`
- Auth required: No
- Request body:

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

- Common errors:
  - `400`: username exists or validation issue
  - `500`: registration failure

### `POST /api/auth/login`

- Purpose: authenticate existing user and return JWT
- Called from: `login.html` inline script and `authAPI.login()`
- Auth required: No
- Request body:

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

- Common errors:
  - `401`: invalid username/password

### `POST /api/auth/logout`

- Purpose: logout acknowledgment (client clears local token)
- Called from: `authAPI.logout()`, `chat.js`, `profile.html`
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

- Purpose: fetch current authenticated user + settings
- Called from: token validation checks and `chat.js` startup
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

## User APIs

### `GET /api/user/profile`

- Purpose: get profile, settings, and usage stats
- Called from: `profile.html`, `userAPI.getProfile()`
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
  },
  "stats": {
    "total_conversations": 4,
    "total_messages": 39
  }
}
```

### `PUT /api/user/settings`

- Purpose: persist user AI and theme settings
- Called from: `profile.html` forms and theme toggle in `chat.js`
- Auth required: Yes
- Request body (any subset is allowed):

```json
{
  "temperature": 0.9,
  "max_tokens": 4096,
  "system_prompt": "Be concise.",
  "theme": "light"
}
```

- Server behavior:
  - `temperature` is clamped to `0.0 .. 2.0`
  - `max_tokens` is clamped to model runtime limit

- Success response:

```json
{
  "success": true,
  "settings": {
    "temperature": 0.9,
    "max_tokens": 4096,
    "system_prompt": "Be concise.",
    "theme": "light"
  }
}
```

### `POST /api/user/change-password`

- Purpose: update current user password
- Called from: `profile.html`, `userAPI.changePassword()`
- Auth required: Yes
- Request content type: `multipart/form-data`
- Form fields:
  - `old_password` (string)
  - `new_password` (string, min length 6)
- Success response:

```json
{
  "success": true,
  "message": "Password changed successfully"
}
```

- Common errors:
  - `400`: invalid old password or weak new password

## Conversation APIs

### `GET /api/conversations`

- Purpose: list all conversations for current user
- Called from: chat startup/refresh (`loadConversations`)
- Auth required: Yes
- Success response:

```json
{
  "conversations": [
    {
      "id": 1,
      "user_id": 1,
      "title": "Hello...",
      "created_at": "2026-03-03 10:11:12",
      "updated_at": "2026-03-03 10:12:20"
    }
  ]
}
```

### `POST /api/conversations`

- Purpose: create an empty conversation
- Called from: available via `conversationAPI.create()` (not primary chat path)
- Auth required: Yes
- Request body: none
- Success response:

```json
{
  "success": true,
  "conversation_id": 12,
  "title": "New Chat"
}
```

### `GET /api/conversations/{conversation_id}/messages`

- Purpose: fetch one conversation and its full message history
- Called from: selecting conversation in sidebar
- Auth required: Yes
- Path param:
  - `conversation_id` (integer)
- Success response:

```json
{
  "conversation": {
    "id": 1,
    "user_id": 1,
    "title": "Hello...",
    "created_at": "2026-03-03 10:11:12",
    "updated_at": "2026-03-03 10:12:20"
  },
  "messages": [
    {
      "id": 100,
      "conversation_id": 1,
      "role": "user",
      "content": "Hi",
      "files": [],
      "timestamp": "2026-03-03 10:11:12"
    },
    {
      "id": 101,
      "conversation_id": 1,
      "role": "assistant",
      "content": "Hello!",
      "files": [],
      "timestamp": "2026-03-03 10:11:13"
    }
  ]
}
```

- Common errors:
  - `404`: conversation not found or not owned by user

### `PUT /api/conversations/{conversation_id}/title`

- Purpose: rename a conversation
- Called from: available via `conversationAPI.updateTitle()` wrapper
- Auth required: Yes
- Request body:

```json
{
  "title": "Project Ideas"
}
```

- Success response:

```json
{
  "success": true,
  "title": "Project Ideas"
}
```

### `DELETE /api/conversations/{conversation_id}`

- Purpose: delete a conversation and its messages
- Called from: chat sidebar delete button
- Auth required: Yes
- Success response:

```json
{
  "success": true,
  "message": "Conversation deleted"
}
```

## Chat APIs

### `POST /api/chat/stream`

- Purpose: send message and receive streaming assistant output
- Called from: `chat.js` send and continue actions
- Auth required: Yes
- Request content type: `multipart/form-data`
- Form fields:
  - `message` (string, required)
  - `conversation_id` (int, optional)
  - `temperature` (float, optional)
  - `max_tokens` (int, optional)
  - `request_id` (string, optional; frontend usually sends client-generated UUID)
  - `persist_user_message` (bool, optional; false for continuation)
  - `continuation_mode` (bool, optional)
  - `continuation_prefix` (string, optional)
  - `continuation_message_id` (int, optional)
  - `files` (0..N file parts, optional)

- Response type: `text/event-stream`
- Response headers used by frontend:
  - `X-Conversation-Id`: conversation id used/created by backend
  - `X-Request-Id`: canonical request id used by stop API

SSE event payloads (`data: <json>`):

1. Token chunk event:

```json
{
  "token": "partial text chunk",
  "finish_reason": null
}
```

2. Error event:

```json
{
  "error": "error text"
}
```

3. Done event (final):

```json
{
  "done": true,
  "assistant_message_id": 101,
  "error": null
}
```

4. Stopped event (final when cancelled/disconnected):

```json
{
  "stopped": true,
  "done": true,
  "assistant_message_id": 101,
  "error": null
}
```

Notes:

- If `conversation_id` is missing, backend creates a new conversation.
- Uploaded file contents are added to prompt context.
- Frontend appends stream tokens live, then formats markdown-like content on finalize.

### `POST /api/chat/stop`

- Purpose: request cancellation of an active streaming generation
- Called from: stop button in `chat.js`
- Auth required: Yes
- Request content type: `multipart/form-data`
- Form fields:
  - `request_id` (string, required)
- Success response:

```json
{
  "success": true,
  "request_id": "abc123",
  "stopped": true
}
```

`stopped` can be `false` if request was already finished or unknown.

## Model/health APIs

### `GET /api/model/info`

- Purpose: return runtime model metadata for profile page
- Called from: `profile.html` and `modelAPI.getInfo()`
- Auth required: No
- Success response shape:

```json
{
  "model_path": "C:/.../models/autobot-instruct",
  "model_exists": true,
  "is_loaded": true,
  "is_available": true,
  "last_error": null,
  "transformers_version": "x.y.z",
  "torch_version": "x.y.z",
  "context_window": 32768,
  "model_train_context_window": 32768,
  "max_generation_tokens_limit": 32640,
  "threads": 4,
  "default_temperature": 0.7,
  "default_max_tokens": 32768,
  "runtime_device": "cuda",
  "model_size_mb": 2048.5
}
```

### `GET /api/health`

- Purpose: health/probe endpoint
- Called from: `healthCheck()` helper in `api.js`
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

## 7) Frontend request matrix by page

### `login.html`

- `GET /api/auth/me` (startup token validity check)
- `POST /api/auth/login` (form submit)

### `register.html`

- `GET /api/auth/me` (startup token validity check)
- `POST /api/auth/register` (form submit)

### `index.html` + `chat.js`

- `GET /api/auth/me`
- `GET /api/conversations`
- `GET /api/conversations/{id}/messages`
- `DELETE /api/conversations/{id}`
- `POST /api/chat/stream`
- `POST /api/chat/stop`
- `PUT /api/user/settings` (theme save)
- `POST /api/auth/logout`

### `profile.html`

- `GET /api/model/info`
- `GET /api/user/profile`
- `PUT /api/user/settings`
- `POST /api/user/change-password`
- `POST /api/auth/logout`

## 8) Current implementation notes

- The frontend uses both shared API helpers (`frontend/js/api.js`) and direct `fetch` calls inside page HTML scripts.
- `conversationAPI.create()` and `conversationAPI.updateTitle()` exist in API client but are not currently the primary path in `chat.js`.
- Profile page defines its own `api()` helper instead of reusing shared wrappers for all actions.
- Theme is stored both server-side (`settings.theme`) and client-side (`localStorage.theme`).
