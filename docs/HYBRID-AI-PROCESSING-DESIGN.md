# Hybrid AI + Vision Video Processing — Design & API Contracts

- **Status**: Approved (Codex-reviewed on `docs/hybrid-ai-processing-design`)
- **Examples**: All recording names, digests, models, worker IDs, URLs and timings are synthetic.
- **Owning Components**:
  - `tennis-processing-server`: Strategy/tuning/per-task model validation, worker AI model catalog registration, claim/scheduler capability matching, and Worker Portal UI controls.
  - `tennis-processing`: Worker AI provider/model discovery, two-stage `hybrid_ai_vision` candidate proposal + contact-sheet AI refinement, guarded provider cancellation, bounded fallback chain, and provenance diagnostics.
  - `courtside`: Cross-component design and contract hub (documentation only).

---

## 1. Motivation & Processing Strategy Enum (`processing_strategy`)

Today `tennis-processing` (`detector.py` `practice-pose-v1`) runs `YOLOv8n-pose` at `8 fps` over a normalized SDR proxy (`analysis.mp4`) and triggers whenever the largest near-camera person's wrist velocity exceeds `swing_speed` (`2.0`) and arm extension exceeds `arm_reach` (`0.85`). While fast and sub-second accurate on local Mac CoreML hardware, pose thresholds alone cannot distinguish real tennis strokes/rallies from non-play activity (ball pickup tosses, walking with a racket, stretching, shadow swings), nor inspect far-court context.

Conversely, sending an entire 30–60 minute unedited video directly to a remote multimodal LLM introduces boundary jitter (`±1–2s` at low FPS) and unnecessary quota/upload cost on idle stretches.

We formalize a **Processing Strategy** enum on video processing tasks and watchers:

```python
PROCESSING_STRATEGIES = (
    "vision_only",       # Strategy 1: current local CoreML YOLOv8n-pose detector (default)
    "hybrid_ai_vision",  # Strategy 2: local 8 fps pose proposes candidates -> AI classifies/trims contact sheets
    "ai_only",           # Strategy 3: pure AI video/chunk scanning without local pose (reserved, unimplemented)
)
```

| Enum value | Description | Status in this milestone | Default |
| --- | --- | --- | --- |
| `vision_only` | Current local `YOLOv8n-pose` @ `8 fps` (`practice-pose-v1`), zero external AI calls. | Supported (existing behavior preserved verbatim). | Yes (when `processing_strategy` is omitted) |
| `hybrid_ai_vision` | Local `8 fps` CoreML pose proposes candidate intervals and `pose-observations.json`; AI model classifies/filters/refines candidates using compact burned-timecode contact sheets. | **Target implementation in this milestone**. | No (explicit per task or watcher) |
| `ai_only` | Pure AI multimodal video scanning without local pose extraction *(noted informally as "pure AI" in contrast to Strategy 1 vision-only and Strategy 2 hybrid)*. | Defined in enum schema; rejected at API/worker validation with `unsupported_processing_strategy` until a future milestone. | No |

### Key Design Principles

1. **Per-task AI model selection (never worker-level configuration)**: Unlike AI coaching's global `worker_default` selector in `tennis-web`, video processing workers advertise **what models they can run**, and the server sets **`ai_model` + `allow_model_fallback` per task/request** (`task.parameters`). Different tasks queued simultaneously can target different strategies, tunings, and models without mutating global worker state.
2. **Preserve `tennis-web` coaching queue separation**: `tennis-web`'s `coaching.sqlite3` queues (`ai_coach`, `learning_plan`, `clip_interval_export`, `coaching_excerpt_render`) remain completely independent per project invariants.
3. **Zero downgrade deception**: If `allow_model_fallback` is `false` and the requested model fails or is unavailable, the attempt fails explicitly rather than switching models or silently reverting to `vision_only`. Even when `allow_model_fallback` is `true`, fallback stays within the worker's AI model chain (never silently disguising a `vision_only` cut as `hybrid_ai_vision`).
4. **Honest heuristics & privacy**: Neither pose thresholds nor AI classifications claim human ground truth (`FORBIDDEN_TRUTH_CLAIMS` remain prohibited). Contact sheets live exclusively in worker-local `scratch/ai-work/` and are never published to NAS storage. Worker errors and provenance metadata redact secrets and local filesystem paths.

---

## 2. Task Parameter & AI Tuning Contract (`task.parameters`)

### 2.1 Allowed Task Parameter Keys

Existing numeric and spatial pose parameters remain valid for both `vision_only` and `hybrid_ai_vision`:
- `target_region`: `[x1, y1, x2, y2]` (`0 <= x1 < x2 <= 1`, `0 <= y1 < y2 <= 1`)
- `min_player_height`: finite number in `[0, 10000]`
- `swing_speed`: finite number in `[0, 10000]`
- `arm_reach`: finite number in `[0, 10000]`
- `padding_before`: finite number in `[0, 10000]`
- `padding_after`: finite number in `[0, 10000]`
- `merge_gap`: finite number in `[0, 10000]`
- `cut_constraints_ref`: verified server snapshot reference (`storage_id="processing"`, task-only — forbidden in watchers)

New strategy, model-selection, and tuning keys in `parameters`:

| Field | Type / Values | Default (when `hybrid_ai_vision`) | Validation & Rejection Codes |
| --- | --- | --- | --- |
| `processing_strategy` | `'vision_only' \| 'hybrid_ai_vision' \| 'ai_only'` | `'vision_only'` when omitted at top level | Unknown value -> `invalid_processing_strategy` (`400`/`422`); `'ai_only'` -> `unsupported_processing_strategy` (`400`/`422`). |
| `ai_model` | `'auto'` or `^(antigravity\|gemini\|codex):[A-Za-z0-9._-]{1,64}$` | `'auto'` | Invalid format/type -> `invalid_ai_model` (`400`/`422`). Rejected with `ai_parameters_require_ai_strategy` if effective strategy is `vision_only`. |
| `allow_model_fallback` | Strict JSON `bool` (`type(v) is bool`) | `True` | Non-bool -> `invalid_allow_model_fallback` (`400`/`422`). Rejected with `ai_parameters_require_ai_strategy` if effective strategy is `vision_only`. |
| `ai_tuning` | JSON object with optional `aggressiveness` and `accuracy_tier` | `{"aggressiveness": "balanced", "accuracy_tier": "standard"}` | Non-dict or unknown keys -> `invalid_ai_tuning` (`400`/`422`). Rejected with `ai_parameters_require_ai_strategy` if effective strategy is `vision_only`. |
| `ai_tuning.aggressiveness` | `'conservative' \| 'balanced' \| 'aggressive'` | `'balanced'` | Invalid value -> `invalid_ai_tuning`. |
| `ai_tuning.accuracy_tier` | `'fast' \| 'standard' \| 'thorough'` | `'standard'` | Invalid value -> `invalid_ai_tuning`. |

### 2.2 AI Tuning Semantics & Exhaustive Deterministic Decision Contract

#### A. Local Candidate Proposal Adjustments (`aggressiveness`)

*(Applied ONLY for pose fields not explicitly specified in `task.parameters` / `snapshot.parameter_overrides`; explicit numeric parameters always take precedence.)*

| Mode | Threshold & Padding Multipliers (when not explicitly overridden) |
| --- | --- |
| **`conservative`** | `swing_speed = base_swing_speed * 0.80`, `arm_reach = base_arm_reach * 0.92`, `min_player_height = base_min_player_height * 0.85`, `merge_gap = max(base_merge_gap, 1.0)`, `padding_before = base_padding_before`, `padding_after = base_padding_after`. |
| **`balanced`** | Standard configured thresholds and padding (`1.0x`). |
| **`aggressive`** | Standard detection thresholds (`swing_speed`, `arm_reach`, `min_player_height`); tighter padding `padding_before = base_padding_before * 0.80`, `padding_after = base_padding_after * 0.80`. |

#### B. Inspection Density & Reasoning Effort (`accuracy_tier`)

| Tier | Contact-Sheet Sampling per Candidate | Tile Width | Max Candidates / Batch | Model Reasoning Effort (`agy` / Gemini) |
| --- | --- | --- | --- | --- |
| **`fast`** | `6` frames (`3x2` grid) | `320px` | `10` | `low` |
| **`standard`** | `12` frames (`4x3` grid; `16` frames `4x4` if span `> 24s`) | `384px` | `6` | `low` |
| **`thorough`** | `16` frames (`4x4` grid; `20` frames `5x4` if span `> 30s`) | `480px` | `4` | `medium` |

#### C. Exhaustive Post-AI Classification Decision Table

For each candidate `[cand.start_s, cand.end_s]` and validated model response `(action, category, confidence, trimmed_start_s?, trimmed_end_s?)`:

1. **Trim Validity Rules** *(evaluated before applying the decision table)*:
   - In JSON Schema (`gemini_compatible_schema`-safe), `trimmed_start_s` and `trimmed_end_s` are optional numbers.
   - If `action != "trim"`: ignore `trimmed_start_s`/`trimmed_end_s` even if null/omitted.
   - If `action == "trim"`:
     - Both `trimmed_start_s` and `trimmed_end_s` MUST be present, finite numbers (not `bool`), with `trimmed_start_s < trimmed_end_s`.
     - If missing, non-finite, inverted (`>=`), or extending outside `[cand.start_s - 2.0, cand.end_s + 2.0]`, or completely disjoint from `[cand.start_s, cand.end_s]`: **reject the entire batch** with `ProviderCallError("invalid_ai_trim_bounds")` so fallback (or task failure if `allow_model_fallback=false`) catches malformed model output.
     - Clamp `[trimmed_start_s, trimmed_end_s]` into `[cand.start_s, cand.end_s]`, then snap start down / end up to `0.125s` frame grid clamped to `[cand.start_s, cand.end_s]` and `[0.0, duration]`.
     - If resulting snapped span `(t_end - t_start) < 1.0`: classify as `trim_too_short = True`; otherwise `trim_too_short = False` with valid span `[t_start, t_end]`.

2. **Complete Precedence & `(aggressiveness × category × action × confidence)` Matrix**:
   Every valid `(action, category, confidence)` evaluates deterministically through this exact decision table:

| `aggressiveness` | `category` | If `action == "keep"` | If `action == "trim"` | If `action == "drop"` |
| --- | --- | --- | --- | --- |
| **`conservative`** | `rally \| serve \| drill \| warmup \| shadow_swing` | **KEEP** `[cand.start_s, cand.end_s]` | **KEEP FULL** `[cand.start_s, cand.end_s]` *(ignore trim to preserve prep/recovery)* | **KEEP FULL** `[cand.start_s, cand.end_s]` *(conservative overrides AI drop on active/borderline labels)* |
| **`conservative`** | `uncertain` | **KEEP** `[cand.start_s, cand.end_s]` | **KEEP FULL** `[cand.start_s, cand.end_s]` | **KEEP FULL** `[cand.start_s, cand.end_s]` *(explicit unknown preservation per `ALGORITHM-SPEC.md` §2)* |
| **`conservative`** | `ball_pickup \| walking \| idle` | **KEEP** `[cand.start_s, cand.end_s]` | **KEEP FULL** `[cand.start_s, cand.end_s]` | **DROP** iff `confidence >= 0.85`; else **KEEP FULL** `[cand.start_s, cand.end_s]` |
| **`balanced`** | `rally \| serve \| drill \| warmup` | **KEEP** `[cand.start_s, cand.end_s]` | **KEEP TRIMMED** `[t_start, t_end]` (or **DROP** `short_trimmed_span_dropped` if `trim_too_short`) | **DROP** iff `confidence >= 0.80`; else **KEEP FULL** `[cand.start_s, cand.end_s]` |
| **`balanced`** | `shadow_swing` | **KEEP** `[cand.start_s, cand.end_s]` iff `confidence < 0.75`; else **DROP** | **KEEP TRIMMED** `[t_start, t_end]` iff `confidence < 0.75` and not `trim_too_short`; else **DROP** | **DROP** *(both confidence `< 0.75` and `>= 0.75` drop when AI chose `drop`)* |
| **`balanced`** | `uncertain` | **KEEP** `[cand.start_s, cand.end_s]` *(explicitly retain `uncertain` when AI chooses `keep` regardless of confidence)* | **KEEP TRIMMED** `[t_start, t_end]` if not `trim_too_short`; else **DROP** | **KEEP FULL** `[cand.start_s, cand.end_s]` iff `confidence < 0.70`; else **DROP** |
| **`balanced`** | `ball_pickup \| walking \| idle` | **DROP** iff `confidence >= 0.75`; else **KEEP FULL** | **DROP** iff `confidence >= 0.75` or `trim_too_short`; else **KEEP TRIMMED** | **DROP** |
| **`aggressive`** | `rally \| serve \| drill` | **KEEP** `[cand.start_s, cand.end_s]` | **KEEP TRIMMED** `[t_start, t_end]` (or **DROP** if `trim_too_short`) | **DROP** iff `confidence >= 0.65`; else **KEEP FULL** |
| **`aggressive`** | `uncertain` | **KEEP** `[cand.start_s, cand.end_s]` iff `confidence < 0.50`; else **DROP** | **DROP** | **DROP** |
| **`aggressive`** | `warmup \| shadow_swing \| ball_pickup \| walking \| idle` | **DROP** | **DROP** | **DROP** |

- Outcome labels recorded per candidate in diagnostics: `"kept"`, `"trimmed"`, `"dropped"`, `"short_trimmed_span_dropped"`, `"conservative_retained_uncertain"`, `"conservative_retained_low_conf_drop"`, `"balanced_retained_uncertain"`.
- Retained/trimmed intervals across all candidates are sorted and merged within `effective_merge_gap`, respecting `[0.0, duration]`.

---

### 2.3 Feedback Snapshots, Precedence, Reruns & Watcher Restrictions

1. **Non-Mutating Snapshot Validation (`feedback.py` & `cut_constraints.py`)**:
   - Both `feedback.validate_parameter_map()` (server) and `cut_constraints.validate_parameter_overrides()` (worker) accept the exact same parameter schema (`NUMERIC_PARAMETERS` + `processing_strategy`, `ai_model`, `allow_model_fallback`, `ai_tuning`), using the exact same validation checks and error codes (`unsupported_processing_strategy`, `ai_parameters_require_ai_strategy`, `invalid_processing_strategy`, `invalid_ai_model`, `invalid_allow_model_fallback`, `invalid_ai_tuning`).
   - Importantly, `feedback.validate_snapshot()` validates `parameter_overrides` **without mutating** the canonical inbound JSON payload so `snapshot_sha256` remains byte-identical across snapshot creation, storage, and worker verification (`cut_constraints.load()`).
2. **Deterministic Parameter Precedence (`files.prepare_task_spec` & `app.rerun`)**:
   Let `caller_params` be the dict explicitly supplied in the request (`POST /tasks` `body['parameters']` or `POST /tasks/{id}/rerun` `body['parameters']`), and `inherited_params` be `parent_task['parameters']` (empty for `POST /tasks`).
   When `feedback_snapshot_id` (or `cut_constraints_ref`) is present:
   - **Explicit conflict check before enqueue (`409 parameter_override_mismatch`)**: Compare `caller_params` against `snap_body['parameter_overrides']`. For every key `(pk, pv)` in `snap_overrides`:
     - If `pk != 'ai_tuning'`: if `pk in caller_params` and `caller_params[pk] != pv` -> raise `Conflict('parameter_override_mismatch')` (`HTTP 409`).
     - If `pk == 'ai_tuning'` and `caller_params` specifies `'ai_tuning'`: for every subkey `(sk, sv)` present in `snap_overrides['ai_tuning']`, if `sk in caller_params['ai_tuning']` and `caller_params['ai_tuning'][sk] != sv` -> raise `Conflict('parameter_override_mismatch')` (`HTTP 409`).
   - **Precedence (`snapshot_overrides > inherited_params`)**:
     - Merge `raw_params = copy.deepcopy(inherited_params)` -> update with `caller_params` -> overlay `snap_overrides` (deep-merging `ai_tuning` subkeys rather than overwriting with `setdefault`!).
     - If `snap_overrides.get('processing_strategy') == 'vision_only'` (or `caller_params.get('processing_strategy') == 'vision_only'` when no snapshot forces `hybrid_ai_vision`), strip **inherited** `ai_model`, `allow_model_fallback`, and `ai_tuning` keys (while preserving any explicitly supplied `caller_params` / `snap_overrides` `ai_*` keys so invalid combinations still fail `ai_parameters_require_ai_strategy`).
     - Finally, run `normalize_task_parameters(raw_params, ValueError, 'invalid_parameters')` and verify snapshot invariants on the normalized `raw_params` so the server **never** queues a task whose `parameters` conflict with its attached `feedback_snapshot`.
   - **Matching worker verification (`api_worker.py`)**:
     - For `pk, pv` in `snap_overrides`: for `'ai_tuning'`, verify each subkey specified in `snap_overrides['ai_tuning']` matches `(parameters.get('ai_tuning') or {}).get(sk)`; for all other keys, verify `parameters.get(pk) == pv` (when present in `parameters`). Apply `snap_overrides` cleanly onto `cfg` (deep-merging `ai_tuning`).
3. **Watchers MUST NOT Carry `cut_constraints_ref` or Bypass Full Validation (`scheduler.py`)**:
   - `POST /api/v1/watchers` and `PATCH /api/v1/watchers/{id}` validate `parameters` with `normalize_task_parameters` and **explicitly forbid `cut_constraints_ref`** (`HTTP 400 cut_constraints_forbidden_in_watcher`), since watchers enqueue `publication_policy="if_no_active_result"` tasks across arbitrary recordings whereas `cut_constraints_ref` is source-bound and requires `publication_policy="never"`.
   - `scheduler.scan()` builds each queued task specification by calling **`files.prepare_task_spec(store, raw_spec)`** (rather than assembling `body` directly and bypassing `prepare_task_spec`), guaranteeing full snapshot/capability/storage/publication-policy derivation including `hybrid-ai-vision-v1`.

---

## 3. Worker AI Model Discovery, Advertisement & Claim Matching

### 3.1 Worker Local Configuration (`api-worker.json`)

Workers remain installable independently of any IDE/workspace (`courtside/AGENTS.md`). An optional `"ai"` block enables AI capabilities:

```json
{
  "name": "Mac processing worker",
  "server_url": "http://processing.example.invalid:8089",
  "registration_token": "PRIVATE_SERVER_REGISTRATION_TOKEN",
  "state_dir": "/path/to/local/worker-state",
  "storage": {
    "originals": {"path": "/path/to/mounted/originals"},
    "processing": {"path": "/path/to/mounted/library/.processing"}
  },
  "processor": {
    "rallyclip_path": "/path/to/installed/RallyClip",
    "pose_device": "coreml"
  },
  "ai": {
    "enabled": true,
    "providers": ["antigravity", "gemini", "codex"],
    "default_model": "antigravity:gemini-4-hydrogen-low",
    "fallback_models": [
      "antigravity:gemini-3.8-flash-low",
      "gemini:gemini-2.5-flash"
    ],
    "gemini_api_key_env": "GEMINI_API_KEY",
    "gemini_models": ["gemini-2.5-flash", "gemini-2.5-flash-lite"],
    "codex_models": ["default"],
    "request_timeout_seconds": 180
  }
}
```

- When the `"ai"` section is omitted or `"enabled": false`, AI discovery is disabled cleanly (no subprocess probes, no `hybrid-ai-vision-v1` capability advertised — preserving existing zero-network worker behavior for legacy/vision-only workers).
- Secrets (`gemini_api_key`, env vars, binary paths, filesystem paths) **never** leave the worker host.

### 3.2 Worker Model Probe & Advertisement Payload

When `"ai"` is enabled (`Agent.__init__`), the worker discovers currently usable providers/models:
1. **`antigravity`**: Resolves `ai.antigravity_bin` or `agy` (`/opt/homebrew/bin/agy`, `/usr/local/bin/agy`) -> runs `agy models` (`timeout=8`) -> parses model IDs (`^[a-z0-9][a-z0-9._-]{1,63}\s{2,}(.+)$`), stripping ANSI/CR escapes and rejecting strings containing `/` or `\`. Adds `{id: "antigravity:<model>", provider: "antigravity", model: "<model>", label: "Antigravity · <display>"}`.
2. **`gemini`**: Checks non-empty API key in `ai.gemini_api_key` or `os.environ[gemini_api_key_env]` / `GOOGLE_API_KEY`. Exposes configured `gemini_models` (default `["gemini-2.5-flash", "gemini-2.5-flash-lite"]`) as `gemini:<model>`.
3. **`codex`**: Resolves `ai.codex_bin` or `codex` -> runs `codex login status` (`timeout=8`, verifying `returncode == 0` and `'ChatGPT'` in output). Exposes configured `codex_models` (default `["default"]`) as `codex:<model>`.

If at least one AI model is discovered:
- Worker includes `'hybrid-ai-vision-v1'` in `capabilities`.
- Worker includes sanitized fields on `/api/v1/workers/register`, `/api/v1/workers/{id}/profile`, `/api/v1/workers/{id}/capabilities`, and `/api/v1/workers/{id}/heartbeat`:
  - `ai_providers`: unique subset of `['antigravity', 'gemini', 'codex']`
  - `ai_models`: list (`<= 50` entries) of `{id, provider, model, label, is_default}` (max 100 chars label, zero paths/secrets)
  - `default_ai_model`: `None` or one `id` from `ai_models`

### 3.3 Server Storage, Aggregation & Claim Gating (`tennis-processing-server`)

1. **Validation & Persistence (`store.py`)**:
   - `register()`, `refresh_worker_profile()`, and `heartbeat()` validate `ai_providers`, `ai_models`, and `default_ai_model`, storing them in the worker `public` JSON body and worker-state mirror (`worker-state/<id>.json`). Older workers omitting these fields continue working unchanged.
   - `GET /api/v1/workers` returns:
     - `items`: video workers (including each worker's `ai_providers`, `ai_models`, `default_ai_model`)
     - `external`: mirrored coaching/export workers (unchanged)
     - `available_ai_models`: aggregated deduplicated catalog across all registered video workers:
       `[{"id": "antigravity:gemini-4-hydrogen-low", "provider": "antigravity", "model": "gemini-4-hydrogen-low", "label": "...", "worker_ids": [...], "online_worker_count": 1}, ...]`
2. **Task Enqueue (`files.py`)**:
   - `files.prepare_task_spec()` validates/normalizes `parameters`. When `processing_strategy == 'hybrid_ai_vision'`, it automatically appends `'hybrid-ai-vision-v1'` to `spec['required_capabilities']`.
3. **Claim Matching (`store.claim`)**:
   - When evaluating queued task `spec` for polling worker `caps`:
     1. If `strategy == 'ai_only'`: skip (`continue` — defense-in-depth).
     2. Existing checks remain (`runtime_requirements`, `required_capabilities`, `required_storage_ids`). Additionally, if `strategy == 'hybrid_ai_vision'`:
        - Enforce `'hybrid-ai-vision-v1'` in `worker_caps` even if `required_capabilities` wasn't explicitly set.
        - Require non-empty `worker_models = [m['id'] for m in (caps.get('ai_models') or [])]`.
        - Strict non-fallback pin: if `parameters.get('allow_model_fallback', True) is False` and `req_model not in (None, 'auto')` and `req_model not in worker_models`: skip (`continue`) so only a worker supporting that exact requested model can lease the task.
        - When `allow_model_fallback is True` or `req_model == 'auto'`: any online worker with `'hybrid-ai-vision-v1'` and non-empty `ai_models` can claim the task.

---

## 4. Hybrid AI + Vision Worker Execution Pipeline (`tennis-processing`)

The top-level server-tracked stage graph stays completely compatible:
`acquire_source -> verify_source -> prepare_proxy -> analyze -> process -> package_result -> upload_result`

### 4.1 Stage `analyze` Architecture for `hybrid_ai_vision`

```
analysis.mp4 (8 fps SDR proxy)
   │
   ▼ [Substep A: Raw Pose Extraction — atomic/checksummed pose-records-cache.json]
YOLOv8n-pose CoreML @ 8 fps ──► raw pose records (boxes, keypoints, conf, proxy_meta)
   │
   ▼ [Substep B: Derived Scoring & Candidate Subdivision — always recomputed from effective config]
score_poses() + pose-observations.json + valley split + <=45.0s hard-cap subdivision
   │   └─► if 0 local pose candidates: skip AI calls cleanly -> return [] (subject to empty policy)
   ▼ [Substep C: Local Contact-Sheet Rendering (scratch/ai-work/cand-NNN.jpg)]
PyAV decode analysis.mp4 + Pillow JPEG grid w/ burned header + per-tile timestamps (EXIF-free RGB)
   │
   ▼ [Substep D: Guarded Batched AI Verification + Deterministic Fallback]
Model chain (exact only if allow_model_fallback=false; ordered chain if true; 1s guard polling + killpg)
   │   └─► STRICT JSON schema + finite/bounded trim validation per candidate
   ▼ [Substep E: Exhaustive Post-Filter & Re-Merge]
Decision matrix (§2.2.C) -> clamp/snap -> merge_gap merge -> [cut_constraints.apply later in process]
```

### 4.2 Raw Pose Cache vs. Derived Observations Cache

1. **Strict Separation (`pose-records-cache.json` + Shared Worker Pose Cache vs. `pose-observations.json`)**:
   - **Raw Extraction Cache**: Caches **only** raw per-frame `YOLOv8n-pose` outputs (`t`, `boxes`, `k`, `kc`, `proxy_meta`):
     - Written to both `scratch/pose-records-cache.json` (for same-task retries) **and** a bounded worker-level content-addressed cache at `state_dir/cache/pose/<pose_cache_key>.json` (atomic temp-file + `os.replace`, bounded LRU eviction retaining up to `16` newest entries and skipping cache writes above `64 MiB`), so both same-task retries and cross-task reruns on the same source/proxy/model skip redundant CoreML inference while keeping task scratch isolated.
   - **Raw Extraction Cache Key**: `pose_cache_key = sha256(canonical_json)` covering every dimension affecting raw pose inference (which runs without ROI/threshold/tuning filters):
     ```json
     {
       "cache_schema": "pose_records_v1",
       "source_sha256": "...",
       "proxy_sha256": "...",
       "model_sha256": "...",
       "backend": "coreml",
       "extractor_code_digest": "..."
     }
     ```
     Each cache file stores `payload_sha256 = sha256(canonical_records)` in its envelope and verifies key equality + SHA-256 checksum on load (looking up `scratch/pose-records-cache.json` first, then `state_dir/cache/pose/<pose_cache_key>.json`; any mismatch, missing entry, or corrupt JSON falls back cleanly to running raw pose extraction).
   - **Derived Features & Observations (`pose-observations.json`) — ALWAYS Recomputed**:
     `detector.detect_from_records(records, proxy_meta, effective_cfg, observations_path=...)` **always recomputes** `score_poses`, `events`, and `pose-observations.json` inside `state_dir/tasks/<task_id>` from the raw records using the run's effective ROI, `min_player_height`, scaled `swing_speed`/`arm_reach`, and `target_selection_policy_version`. Downstream tuning/AI/padding/strategy changes across retries or reruns therefore never serve stale derived `pose-observations.json`, while avoiding redundant CoreML decoding.
   - `diagnostics.py` receives effective `cfg` (with mode-scaled detection/padding settings) and records `software.detector_version = "hybrid-ai-vision-v1"` (while preserving `detector_spec` over the `yolov8n-pose` extractor tuple so `extraction_cache_key` continues invalidating on backend/model/ROI/threshold changes).

### 4.3 Candidate Splitting & Hard `<= 45.0s` Bound

To ensure contact sheets retain sufficient temporal density even for long rallies:
1. Build initial merged candidate spans from scaled `events` using effective `padding_before`, `padding_after`, and `merge_gap`.
2. **Pass 1 — Low-activity valley split**: For any span with `end_s - start_s > 45.0`, inspect sorted raw pose `events` strictly inside `[start_s, end_s]`. Where consecutive event intervals `[e_{k}.end, e_{k+1}.start]` have an internal gap `>= 2.0s`, split at the gap midpoint `m = (e_k.end + e_{k+1}.start) / 2` (snapped to `0.125s`), yielding non-overlapping touching spans `[start_s, m]` and `[m, end_s]`.
3. **Pass 2 — Hard `<= 45.0s` subdivision fallback**: If any sub-span still satisfies `(sub_end - sub_start) > 45.0` (e.g., continuous 2-minute baseline rally with no `>= 2.0s` pause), scan `[sub_start + 25.0, min(sub_end - 10.0, sub_start + 45.0)]` in `records` for the lowest 1-second rolling wrist speed timestamp `t_min` (snapped to `0.125s`), or fall back to uniform `<= 45.0s` cuts `[sub_start, sub_start + min(45.0, ...)]` so **every candidate sent to AI strictly satisfies `0 < duration_s <= 45.0` with zero overlap**.
4. Adjacent segments kept by AI that touch (`next.start_s <= prev.end_s + effective_merge_gap`) are merged seamlessly back together in Substep E, so continuous rallies are not fragmented in the exported `Practice.mp4`!

### 4.4 Guarded Provider Execution, Timeouts & Lease Cancellation

1. **Unified Terminable Subprocess Isolation for In-Flight Lease / Cancel Interruption (matches `pipeline.py:83-97`)**:
   - Background heartbeat thread updates `deadline` and sets `Cancelled`/`LeaseLost` when the server cancels or revokes the lease.
   - **All AI provider calls — both CLI (`antigravity`, `codex`) and HTTPS (`gemini`) — run inside isolated terminable subprocesses launched with `subprocess.Popen(..., start_new_session=True)`**:
     - **CLI providers (`antigravity`, `codex`)**: Launched in an isolated temp `batch_dir` containing only `cand-*.jpg` and schema JSON (zero workspace/NAS/secret paths).
     - **HTTPS provider (`gemini`)**: Runs a dedicated short-lived Python HTTP subprocess (`[sys.executable, str(hybrid_ai_path), "--gemini-http"]`) reading `{url, api_key, payload, timeout}` strictly over `stdin` (never via argv, environment variables, or disk files) and executing `urllib.request.urlopen(..., timeout=bounded_timeout)`. Running HTTPS inside a dedicated process-group subprocess guarantees OS-level teardown of TCP connect, TLS handshake, header wait, and response reads — closing a gap where daemon threads lack an open socket handle during connect or header wait.
   - **Unconditional Process-Group Cleanup & Non-Blocking Pipe I/O Before Return, Error, or Fallback**:
     - Immediately after `subprocess.Popen(..., start_new_session=True)`, record `pgid = process.pid` (before the leader can exit) and enter `try: ... finally:`.
     - **Non-blocking stdin/stdout/stderr I/O**: Feed any `stdin` payload (e.g., Gemini HTTP JSON containing base64 sheets + API key) via a daemon writer thread (`write` + `close`, ignoring `BrokenPipeError`/`OSError`/`ValueError`) and drain `stdout`/`stderr` concurrently via daemon reader threads (retaining up to `4 MiB` per stream) so large pipe writes or full output buffers **never block** the caller thread from polling `guard()`.
     - The caller thread polls child completion every `<= 1.0s` (calling `guard()` **before** the poll loop and on every iteration, and checking `time.monotonic() - started > request_timeout_seconds`).
     - In the unconditional `finally:` block covering **all** exit paths (normal exit, non-zero exit / `ProviderCallError`, timeout, or `Cancelled` / `LeaseLost`), executed **in this strict order before returning, raising, or attempting any fallback model**:
       1. `_terminate_process_group(pgid, process)`: terminate any live process in `pgid` (`os.killpg(pgid, signal.SIGTERM)`, wait up to `5s` when actively aborting or brief grace check for stray descendants, then `os.killpg(pgid, signal.SIGKILL)` — ignoring `ProcessLookupError`/`OSError` if already gone) and call `process.wait()`.
       2. Close `stdin`/`stdout`/`stderr` pipes safely (ignoring `OSError`/`ValueError`).
       3. Join the daemon stdin/stdout/stderr I/O threads with bounded timeout (`1.0s`) **strictly after** process-group termination so descendants holding pipes open cannot hang thread joins.
       4. Preserve any active `Cancelled` / `LeaseLost` exception cleanly (never swallowed by cleanup errors or pipe errors).
   - **Strict Propagation**: `Cancelled` and `LeaseLost` propagate **immediately** out of `_run_provider_batch` -> `run_hybrid_ai_analysis` -> `Agent.execute`. They are **never** treated as fallback-eligible `ProviderCallError` failures and **never** trigger a second provider/model!
2. **Structured Output Schema**:
   - Batch prompt attaches each candidate's `cand-NNN.jpg` plus compact JSON summary `{candidate_id, start_s, end_s, duration_s, event_count, peak_wrist_speed, active_ratio}`.
   - Required JSON response keys per decision: `candidate_id`, `action` (`keep|trim|drop`), `category` (`rally|serve|drill|warmup|shadow_swing|ball_pickup|walking|idle|uncertain`), `confidence` (`0..1`), `reason`, and optional `trimmed_start_s`, `trimmed_end_s` (validated strictly in code per §2.2.C).
   - Exact 1-to-1 `candidate_id` coverage verified (no duplicates, no missing IDs, no unknown IDs).

### 4.5 Model Chain, Fallback, Empty-Clip Policy & Provenance

1. **Model Chain & Fallback Policy**:
   - When `allow_model_fallback == False`:
     - Target is `[requested_model]` (or `[worker_default_model]` if `ai_model == 'auto'`).
     - If missing from discovered `ai_models`, fail immediately with `error_code="ai_model_unavailable"`, `retryable=False`.
     - If provider call fails or times out, fail immediately with `error_code="ai_provider_failed"`, `retryable=False`.
   - When `allow_model_fallback == True`:
     - Chain = unique ordered `[requested_model (if != 'auto' and available)] + [worker_default_model] + [configured fallback_models] + [remaining discovered models ordered by flash/low preference]`.
     - If model $M_k$ raises `ProviderCallError` (non-lease, non-cancel) on batch $i$, worker records the attempt failure and retries batch $i$ on $M_{k+1}$ (and uses $M_{k+1}$ for subsequent batches).
     - If **every** model in the chain fails, raise `ProcessingTaskError("ai_all_models_failed", ..., retryable=False)`. **Never silently fall back to `vision_only`**.
2. **Zero-Clip Post-AI Policy (`files.py` compatibility)**:
   - When post-AI intervals are empty (and no `cut_constraints_ref` adds keep/override intervals):
     - If `task['publication_policy'] == 'never'` (manual/rerun/experiment/dryrun): set `cfg['_allow_empty_analysis'] = True` -> metadata-only empty session succeeds cleanly.
     - If `task['publication_policy'] == 'if_no_active_result'` (watcher auto-publish): raise `ProcessingTaskError('no_play_detected_after_ai_filter', 'AI filter retained zero play clips; original preserved.', retryable=False)`.
3. **Batch, Candidate & Prompt/Input Provenance (`report['analysis']['ai_refinement']` & `findings['ai_refinement']`)**:
   - Satisfies `tennis-algorithm-lab/docs/ALGORITHM-SPEC.md` (`model/prompt/input provenance`) while keeping raw prompts/JPEGs off NAS:
   - Aggregate `actual_ai_model` semantics:
     - Zero AI calls (`candidate_count == 0`): `actual_ai_model: null`, `actual_ai_models: []`, `mixed_models_used: false`, `ai_call_count: 0`.
     - Uniform accepted model across all batches: `actual_ai_model: "<provider>:<model>"`, `actual_ai_models: ["<provider>:<model>"]`, `mixed_models_used: false`.
     - Mixed accepted models across batches (mid-task fallback): `actual_ai_model: "mixed"`, `actual_ai_models: ["m1", "m2"]`, `mixed_models_used: true`.
   - Detailed `batch_provenance`:
     ```json
     {
       "detector": "hybrid-ai-vision-v1",
       "pose_backbone": "practice-pose-v1",
       "processing_strategy": "hybrid_ai_vision",
       "requested_ai_model": "antigravity:gemini-4-hydrogen-low",
       "actual_ai_model": "mixed",
       "actual_ai_models": [
         "antigravity:gemini-4-hydrogen-low",
         "antigravity:gemini-3.8-flash-low"
       ],
       "mixed_models_used": true,
       "allow_model_fallback": true,
       "fallback_used": true,
       "ai_tuning": {"aggressiveness": "balanced", "accuracy_tier": "standard"},
       "ai_call_count": 3,
       "candidate_count": 14,
       "kept_count": 8,
       "trimmed_count": 2,
       "dropped_count": 4,
       "uncertain_retained_count": 1,
       "uncertain_dropped_count": 0,
       "schema_sha256": "<64-hex>",
       "provenance_artifact": "ai-provenance.json",
       "provenance_sha256": "<64-hex>",
       "provenance_truncated": false,
       "batch_count_total": 2,
       "batch_provenance": [
         {
           "batch_id": "batch-001",
           "candidate_ids": ["C001", "C002"],
           "accepted_model": "antigravity:gemini-4-hydrogen-low",
           "attempt_count": 1,
           "prompt_sha256": "<64-hex>",
           "input_manifest_sha256": "<64-hex>"
         },
         {
           "batch_id": "batch-002",
           "candidate_ids": ["C003", "C004"],
           "accepted_model": "antigravity:gemini-3.8-flash-low",
           "attempt_count": 2,
           "prompt_sha256": "<64-hex>",
           "input_manifest_sha256": "<64-hex>"
         }
       ],
       "attempted_models": [
         {"model": "antigravity:gemini-4-hydrogen-low", "batch_id": "batch-002", "status": "failed", "error_code": "provider_timeout"},
         {"model": "antigravity:gemini-3.8-flash-low", "batch_id": "batch-002", "status": "succeeded"}
       ]
     }
     ```
   - **Bounded Completion Payload vs. Lossless Artifact Provenance**:
     - `tennis-processing-server/app.py` enforces `MAX_CONTENT_LENGTH = 262144` (256 KiB) on `/attempts/{id}/complete`, whereas `files.py:validate_result` does not cap individual `metadata/` artifact sizes (`8 MiB` applies only to the `result.json` manifest itself). Therefore `package` writes **lossless, untruncated** provenance (complete `batch_provenance`, `attempted_models`, and all per-candidate `candidate_decisions` with `action`, `category`, `confidence`, `applied_action`, `trimmed_start_s`, `trimmed_end_s`, `reason_code`, `accepted_model`) into `metadata/ai-provenance.json` (declared in `session.artifacts` and checksummed in `session.metadata_checksums`), and includes `"provenance_artifact": "ai-provenance.json"` + `"provenance_sha256"` in `findings['ai_refinement']`. No evidence is truncated in `metadata/ai-provenance.json`.
     - `findings['ai_refinement']` sent in `/attempts/{id}/complete` enforces a strict `MAX_AI_REFINEMENT_FINDINGS_BYTES = 32768` (32 KiB) serialized UTF-8 JSON budget:
       1. Summary fields (`detector`, `pose_backbone`, `processing_strategy`, `requested_ai_model`, `actual_ai_model`, `actual_ai_models`, `mixed_models_used`, `allow_model_fallback`, `fallback_used`, `ai_tuning`, `ai_call_count`, `candidate_count`, `kept_count`, `trimmed_count`, `dropped_count`, `uncertain_retained_count`, `uncertain_dropped_count`, `schema_sha256`, `provenance_artifact`, `provenance_sha256`, `batch_count_total`, `provenance_truncated`) are always preserved.
       2. Inline `findings['ai_refinement']` omits verbose `candidate_decisions` (keeping them in `metadata/ai-provenance.json`).
       3. If compact `batch_provenance` + `attempted_models` would push serialized `findings['ai_refinement']` over `32768` bytes, overflow trimming first truncates `attempted_models` tail entries and then truncates `batch_provenance` tail entries until `<= 32768` bytes, setting `provenance_truncated: true`.
     - `heuristic-diagnostics.json` preserves `truth_status: "heuristic_only_not_play_or_swing_truth"` and zero `FORBIDDEN_TRUTH_CLAIMS`.
4. **Sanitized Failure Reporting**:
   - Worker `/attempts/{id}/fail` sends `{task_id, attempt_id, lease_token, final_event_sequence, error_code, message, retryable, timing}` with secrets and local filesystem paths stripped (`<= 400` chars).

---

## 5. Worker Portal UI (`tennis-processing-server`)

1. **Workers Section (Read-Only Capability Display)**:
   - Video worker cards keep showing online/paused/stage/timestamps and add a read-only capability summary line (e.g., `Vision + Hybrid AI · 3 models · Default: gemini-4-hydrogen-low` or `Vision model only`). **No** global worker-level model dropdown is rendered.
2. **Create / Rerun Task Dialog (`#task-form`)**:
   - **Processing strategy** select: `vision_only` ("Vision model · local pose (default)"), `hybrid_ai_vision` ("Hybrid AI + Vision · pose + AI filter"), `ai_only` ("Pure AI · coming soon", `disabled`).
   - **AI Options Panel** (shown when `hybrid_ai_vision` is selected):
     - **AI model** select: `auto` ("Worker default (auto)") plus options grouped by provider from `GET /api/v1/workers` `available_ai_models` (preserving any `previous.parameters.ai_model` on Rerun).
     - **Auto-fallback** checkbox (`allow_model_fallback`, default checked).
     - **Cut aggressiveness** select (`conservative`, `balanced`, `aggressive`).
     - **Inspection accuracy** select (`fast`, `standard`, `thorough`).
   - **Advanced pose parameters (JSON)** collapsed `<details>` containing only numeric/ROI overrides (stripped of strategy/AI keys so UI fields remain single source of truth, merged cleanly on submit).
3. **Input Folders / Watchers (`#watcher-form` & `#watchers`)**:
   - Allow configuring watcher default `processing_strategy` (`vision_only` / `hybrid_ai_vision`) + AI tuning/model/fallback when enabled (never `cut_constraints_ref`), and display watcher strategy badge on watcher cards.
4. **Task Cards & Inspect Run Dialog**:
   - Task cards display a compact strategy/model badge (e.g. `hybrid · gemini-4-hydrogen-low`, `fallback` or `mixed` indicator, and `balanced / standard` tuning summary) alongside status and errors.
   - `#detail` modal displays full `ai_refinement` breakdown including `batch_provenance`, `actual_ai_model`, `ai_tuning`, and candidate keep/trim/drop/uncertain counts.
