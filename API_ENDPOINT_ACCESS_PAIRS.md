# API Endpoint Access Pairs

| API Endpoint | Backend Handler | Access | Frontend Caller(s) |
|---|---|---|---|
| `POST /api/auth/register` | `backend/main.py::api_register` | `Public` | `frontend/register.html`, `frontend/js/api.js` (`authAPI.register`) |
| `POST /api/auth/login` | `backend/main.py::api_login` | `Public` | `frontend/login.html`, `frontend/js/api.js` (`authAPI.login`) |
| `POST /api/auth/logout` | `backend/main.py::api_logout` | `Protected` | `frontend/js/chat.js` (via `authAPI.logout`), `frontend/profile.html`, `frontend/js/api.js` (`authAPI.logout`) |
| `GET /api/auth/me` | `backend/main.py::api_me` | `Protected` | `frontend/login.html`, `frontend/register.html`, `frontend/js/chat.js` (via `authAPI.me`), `frontend/js/api.js` (`authAPI.me`) |
| `GET /api/user/profile` | `backend/main.py::api_profile` | `Protected` | `frontend/profile.html`, `frontend/js/api.js` (`userAPI.getProfile`) |
| `PUT /api/user/settings` | `backend/main.py::api_update_settings` | `Protected` | `frontend/profile.html`, `frontend/js/chat.js` (theme save), `frontend/js/api.js` (`userAPI.updateSettings`) |
| `POST /api/user/change-password` | `backend/main.py::api_change_password` | `Protected` | `frontend/profile.html`, `frontend/js/api.js` (`userAPI.changePassword`) |
| `GET /api/conversations` | `backend/main.py::api_get_conversations` | `Protected` | `frontend/js/chat.js` (via `conversationAPI.list`), `frontend/js/api.js` (`conversationAPI.list`) |
| `POST /api/conversations` | `backend/main.py::api_create_conversation` | `Protected` | `frontend/js/api.js` (`conversationAPI.create`) |
| `GET /api/conversations/{conversation_id}/messages` | `backend/main.py::api_get_messages` | `Protected` | `frontend/js/chat.js` (via `conversationAPI.getMessages`), `frontend/js/api.js` (`conversationAPI.getMessages`) |
| `PUT /api/conversations/{conversation_id}/title` | `backend/main.py::api_update_title` | `Protected` | `frontend/js/api.js` (`conversationAPI.updateTitle`) |
| `DELETE /api/conversations/{conversation_id}` | `backend/main.py::api_delete_conversation` | `Protected` | `frontend/js/chat.js` (via `conversationAPI.delete`), `frontend/js/api.js` (`conversationAPI.delete`) |
| `POST /api/chat/stop` | `backend/main.py::api_chat_stop` | `Protected` | `frontend/js/chat.js` (via `chatAPI.stopGeneration`), `frontend/js/api.js` (`chatAPI.stopGeneration`) |
| `POST /api/chat/stream` | `backend/main.py::api_chat_stream` | `Protected` | `frontend/js/chat.js` (via `chatAPI.sendMessage`), `frontend/js/api.js` (`chatAPI.sendMessage`) |
| `GET /api/model/info` | `backend/main.py::api_model_info` | `Public` | `frontend/profile.html`, `frontend/js/api.js` (`modelAPI.getInfo`) |
| `GET /api/health` | `backend/main.py::health_check` | `Public` | `frontend/js/api.js` (`healthCheck`) |
