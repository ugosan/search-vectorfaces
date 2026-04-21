# Image Vector Search App - LLM Generation Boilerplate

Use this document as the canonical specification to generate a complete application similar to this demo.
The output should be production-like demo code, runnable with Docker Compose, with clear module boundaries.

## 1. Product Goal

Build a real-time image similarity search web app that:

- captures user input from camera or uploaded image,
- detects one or more faces,
- extracts embeddings,
- runs vector similarity search in Elasticsearch,
- shows top matches in a visual grid,
- allows indexing user-provided faces into a dedicated uploads index.

The app is an interactive demo for vector search and quantization comparison, not a full authentication-enabled product.

## 2. Core User Flows

Implement all flows below.

### 2.1 Live camera analysis

1. User opens app in browser.
2. Browser initializes camera.
3. Frontend sends a base64 frame to backend over WebSocket every fixed interval (10s default).
4. Backend detects faces, extracts embeddings, performs vector search, and returns response with timing data.
5. Frontend updates result grid and status panel.

### 2.2 Single image upload analysis

1. User uploads image file (jpeg/png/webp).
2. Frontend posts image to backend `/api/upload`.
3. Backend returns face analysis result.
4. Frontend displays modal with face boxes and metadata.

### 2.3 Index face from uploaded image

1. In analysis modal, user inputs the names of each detected face.
2. Frontend crops/expands region around faces and send them to `/api/index`.
3. Backend extracts embedding and indexes documents into `uploads` index.
4. UI confirms success or failure.

### 2.4 Search settings tuning

User can change runtime settings from sidebar:

- enabled indices,
- `k`,
- `num_candidates`,
- `size`,
- visual toggle for facial features.

Updated settings must be included in subsequent WebSocket frame messages.

## 3. Required Architecture

Generate two services behind Docker Compose:

- `frontend`: static SPA served by Nginx.
- `backend`: FastAPI service for REST + WebSocket.

### 3.1 High-level system design

- Frontend connects to backend WebSocket endpoint `/ws`.
- Frontend calls REST endpoints under `/api/*`.
- Nginx proxies `/api/*` and `/ws` to backend container.
- Elasticsearch is external (Elastic Cloud or local start-local), configured through environment variables.

### 3.2 Backend processing pattern

Use Chain of Responsibility for frame processing:

1. `FaceAnalysisHandler`:
	 - decode image,
	 - detect faces and embeddings,
	 - collect face-analysis timing.
2. `VectorSearchHandler`:
	 - for each face embedding, run KNN search,
	 - apply settings (`k`, `num_candidates`, `size`, selected indices),
	 - collect Elasticsearch timing and combine total timing.
3. `ResponseBuilder`:
	 - normalize final payload (`analysis`, `not_found`, `error`).

This pattern is mandatory so processing stages remain isolated and extensible.

## 4. Technology Requirements

### 4.1 Backend

- Python 3.11+
- FastAPI + Uvicorn
- insightface + onnxruntime (CPU provider by default)
- OpenCV + Pillow + NumPy
- elasticsearch Python client
- python-dotenv

### 4.2 Frontend

- Plain ES modules (no framework required)
- Bulma and FontAwesome via CDN
- face-api.js for optional local landmarks overlay
- html2canvas for screenshot/export feature support

### 4.3 Infrastructure

- Docker and Docker Compose
- Nginx as static server + reverse proxy

## 5. File and Module Layout

Use this structure (or equivalent with same boundaries):

```text
app/
	docker-compose.yml
	env
	env.local
	backend/
		Dockerfile
		requirements.txt
		server.py
		vectorfaces/
			__init__.py
			face_analyzer.py
			vector_search.py
		chain/
			__init__.py
			handler.py
			face_analysis_handler.py
			vector_search_handler.py
			response_builder.py
	frontend/
		Dockerfile
		nginx/
			nginx.conf
		public/
			index.html
			css/
			js/
				app.js
				controllers/
					usermedia.js
					websocket.js
					upload.js
					sidebar.js
					grid.js
					faceapi.js
					screenshot.js
					drag.js
				mediasource/
					MediaSource.js
					CameraMedia.js
					PictureMedia.js
	tmp/
		uploads/
```

## 6. Environment Contract

Read runtime config from `env.local` (fallback `env`):

```bash
ES_INDEX="faces"
ES_INDICES="faces-int4_hnsw-10.15,faces-int8_hnsw-10.15,faces-disk_bbq-10.15,faces-bbq_hnsw-10.15,faces-bbq_hnsw-uploads"
ES_UPLOADS_INDEX="faces-bbq_hnsw-uploads"
ES_HOST="https://your-es-host:443"
ES_API_KEY="your-api-key"
```

Rules:

- `ES_INDICES` is the list the UI can toggle.
- Upload indexing always targets `ES_UPLOADS_INDEX`.
- `ES_HOST` + `ES_API_KEY` are required for vector search and indexing.

## 7. API and Messaging Contracts

### 7.1 REST endpoints

- `GET /api/health`
	- Returns service health.

- `GET /api/stats`
	- Returns backend status, active websocket connections, and Elasticsearch index stats for all configured indices.

- `POST /api/analyze`
	- Body: `{ image: <base64 data URL or raw b64>, timestamp: <iso> }`
	- Returns face analysis only (no vector search).

- `POST /api/upload` (multipart)
	- Fields: `image` (file), `timestamp` (string)
	- Returns face analysis payload for uploaded image.

- `POST /api/index`
	- Body: `{ image: <base64>, timestamp: <iso>, name?: <string> }`
	- Detects faces and indexes embeddings from cropped upload image into uploads index.

### 7.2 WebSocket `/ws`

Inbound frame message:

```json
{
	"type": "frame",
	"timestamp": "2026-04-21T12:00:00.000Z",
	"image": "data:image/jpeg;base64,...",
	"settings": {
		"indices": [{ "name": "faces-int8_hnsw-10.15", "selected": true }],
		"k": "50",
		"num_candidates": "200",
		"size": "50",
		"showFacialFeatures": false
	}
}
```

Outbound analysis message:

```json
{
	"type": "analysis",
	"timestamp": "2026-04-21T12:00:00.000Z",
	"timing_stats": {
		"face_analysis_ms": 34.1,
		"elasticsearch_total_ms": 18,
		"elasticsearch_total_hits": 500,
		"total_processing_ms": 52.1
	},
	"face_analysis": {
		"success": true,
		"face_count": 1,
		"faces": [{ "bbox": [0, 0, 0, 0], "confidence": 0.99, "embedding": [0.0] }]
	},
	"matching_faces": [
		{
			"index": "faces-int8_hnsw-10.15",
			"score": 0.83,
			"metadata": {
				"name": "Example",
				"gender": "M",
				"image_path": "/faces/example.jpg"
			}
		}
	]
}
```

Alternative outputs:

- `type = "not_found"` when no faces were detected.
- `type = "error"` with `error` field when analysis/search fails.

## 8. Vector Search Behavior

Implement KNN query on `face_embeddings` vector field with cosine similarity.

Expected constraints:

- embedding dimension: 512,
- sanitize runtime params:
	- `k`: clamp to [3, 100],
	- `num_candidates`: clamp to [50, 1000],
	- `size`: clamp to [10, 100],
- use selected indices from settings; unselected indices are excluded,
- apply gender filter inferred from detected face (`M`/`F`) where available.

Search should return both matches and timing metadata from Elasticsearch (`took`, hits, timeout).

## 9. Face Analysis Behavior

`FaceAnalyzer` must:

- initialize model once at app startup,
- accept base64 data URLs and raw base64 input,
- detect all faces,
- return per-face:
	- bounding box,
	- confidence,
	- age,
	- gender,
	- embedding,
	- landmarks,
- fail gracefully when model is unavailable or image decode fails.

## 10. Frontend UX Requirements

Implement these UI elements:

- media area with camera/image toggle,
- settings sidebar with index checkboxes and sliders (`k`, `num_candidates`, `size`),
- status indicator,
- result grid showing matched faces,
- info box showing top match, similarity score, and timing,
- upload button and analysis modal,
- button to index uploaded faces.

Interaction requirements:

- auto-reconnect WebSocket on disconnect,
- preserve modular controllers (one controller per concern),
- keep all settings in global runtime state object (`window.settings`),
- trigger and listen for `settingsChanged` custom event.

## 11. Nginx Requirements

Nginx config must:

- serve static frontend,
- proxy `/api/*` to backend,
- proxy `/ws` with upgrade headers,
- optionally expose static dataset files under `/faces/*` (proxy to external storage),
- optionally expose local uploads from shared volume under `/local/uploads/*`.

## 12. Docker Compose Requirements

Compose must:

- define `frontend` and `backend` services,
- mount frontend static files,
- mount backend source for iterative development,
- mount shared `tmp/uploads` and model cache folders,
- expose frontend on port `16700` and backend on `8000`,
- include health checks and startup ordering (`frontend` waits for backend health).

## 13. Elasticsearch Indexing Expectations

Assume external setup script already creates core indices for quantization experiments.

Application responsibilities:

- connect to Elasticsearch on startup,
- collect stats for each index in `ES_INDICES`,
- ensure uploads index exists (create with dense_vector mapping if missing),
- index uploaded faces with metadata fields:
	- `name` (optional),
	- `gender`,
	- `age`,
	- `confidence`,
	- `bbox`,
	- `image_path`,
	- `uploaded_at`.

## 14. Non-Functional Requirements

- Keep code split into small, focused modules.
- Avoid monolithic files handling unrelated concerns.
- Use clear error logging in backend and browser console.
- Use explicit HTTP errors for invalid payloads.
- Avoid hardcoding secrets.
- Ensure app can run entirely through Docker Compose once env values are provided.

## 15. Acceptance Criteria (Definition of Done)

A generated app is valid only if all checks pass:

1. `docker-compose up` starts frontend and backend without manual patching.
2. Browser app loads on configured port.
3. Camera mode sends periodic frames over WebSocket.
4. Backend returns face analysis and vector matches.
5. Sidebar setting changes affect subsequent search behavior.
6. Upload mode analyzes image and opens modal with detected faces.
7. "Index to Elasticsearch" stores face document in uploads index.
8. `/api/stats` reports per-index stats from `ES_INDICES`.
9. WebSocket reconnects after forced disconnection.
10. Codebase keeps controller/service/handler boundaries described above.

## 16. Prompt Template for LLM Code Generation

Use this prompt when generating a new similar app:

```text
Generate a full Dockerized image vector search demo application using the following specification:

- Backend: FastAPI with WebSocket /ws and REST endpoints /api/health, /api/stats, /api/analyze, /api/upload, /api/index.
- Face analysis: InsightFace (CPU provider), returns bbox/confidence/age/gender/embedding/landmarks.
- Vector search: Elasticsearch dense_vector KNN over 512-d embeddings, runtime controls for k, num_candidates, size, and selected indices.
- Pattern: Chain of Responsibility with FaceAnalysisHandler -> VectorSearchHandler -> ResponseBuilder.
- Frontend: Nginx-served SPA with modular ES controllers (websocket, media, upload, sidebar, grid, faceapi).
- UI: camera/image toggle, settings sidebar, status badge, results grid, upload modal, index button.
- Env vars: ES_HOST, ES_API_KEY, ES_INDEX, ES_INDICES, ES_UPLOADS_INDEX.
- Compose: frontend and backend services, shared uploads volume, health checks.
- Keep modules separated and code readable. Do not place unrelated logic in single files.

Return all source files with complete code.
```

If there is any ambiguity during generation, prioritize preserving these contracts and module boundaries.