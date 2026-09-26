# Smart Guided Troubleshooting Engine
### Transforming Vague Galaxy Device Complaints into Deeplinked, One-Tap Troubleshooting Plans
*Samsung Electronics &bull; Device eXperience (DX) CTO / Research*

---

## 1. Executive Summary & Problem Overview
When smartphone users experience technical issues, they rarely express them in standard technical terms. Instead, support systems receive informal, imprecise descriptions:
- *"Screen flickers and the battery dies fast"*
- *"My phone got slow after the update"*
- *"Swipe gestures go the wrong way after installing an app"*

Today, resolving these complaints requires significant human intervention:
1. Customer support agents read unstructured knowledge-base articles (from internal knowledge repositories like SIIS).
2. The agent interprets the complaint, diagnoses root causes, manually selects relevant troubleshooting steps, and arranges them in order.
3. The customer receives text instructions and must manually navigate complex nested device settings (e.g., `Settings > Display > Navigation bar`).

This manual triage process takes roughly **15 minutes per scenario** across millions of customer interactions globally.

### Target State
This repository provides an automated **Smart Guided Troubleshooting Engine** that transforms unstructured natural-language complaints into clean, validated, machine-actionable troubleshooting plans returned in **under 300 ms** for previously encountered issues.

Each plan:
- Deconstructs the problem into granular, screen-specific actions (**One Action = One Screen**).
- Orders actions logically (**least disruptive troubleshooting steps first; destructive/critical operations last**).
- Attaches exact, verified in-app Settings deeplinks (`bixby://...`) to make each remediation step actionable in a single tap.

---

## 2. System Architecture & Pipeline

```
Raw Complaint (+ Optional SIIS Raw Knowledge Text)
       │
       ▼
  [0] Query Enrichment
       ├── Normalize colloquial text into a canonical technical query
       ├── Generate semantic cache key & 8-10 distinct paraphrases
       │
       ▼
  [1] Phase 1: Structure Extraction (LLM / Parsing)
       ├── Extract Goal, categorized Actions, and discrete UI Steps
       ├── Enforce strict length, phrasing, and zero-leak constraints
       │
       ▼
  [2] Phase 2: Deeplink Mapping & Action Ordering
       ├── Match target screens to deeplink catalog entries (bixby://...)
       ├── Sequence: non-invasive settings first ──► critical/destructive last
       │
       ▼
  [3] Fast-Path Semantic Cache
       ├── Cache Hit (<= 300 ms): Return validated plan without LLM invocation
       └── Cache Miss: Run pipeline [0]-[2], validate schema, write to cache
       │
       ▼
  [4] Standardized REST API Service
       └── Expose endpoints with operational metadata (latency, hit flag, cost)
```

---

## 3. Strict Contract Specifications & Rule Constraints

### 3.1 Schema Rules
- **`goal`**: Exact syntax: `Follow these steps to perform this <Topic> Troubleshooting` (or `<Topic> Configuration`).
- **`title`**: 2 to 3 words, sentence case, identifying core issue (e.g., `Swipe navigation settings`, `Battery fast drain`).
- **`score`**: Floating-point value between 0.0 and 1.0 indicating confidence.
- **`actionName`**: Title Case. Represents exactly one physical screen or feature. Multiple steps occurring on the same screen are grouped under a single action (**One Action = One Screen**).
- **`description`**: Exactly 5 to 7 words, starting with `"It will"`, explaining concrete benefit in plain language.
- **`stepGroups[].steps`**: Clear, imperative UI steps. One physical interaction per step. Absolute prohibition of web URLs.
- **`category`**:
  - `auto`: Standard configuration screens reachable via deeplink.
  - `manual`: Physical interventions (cleaning ports, replacing cables). Cannot carry an actionable deeplink.
  - `critical`: Disruptive or irreversible operations (factory reset, restart, firmware update, safe mode). **Must be ordered last**.
- **`actionableDeeplink`**: Verbatim masked URI copied from `deeplinks.json` matching the target screen.
- **`bixby://dummy_positive`**: Reserved generic placeholder used exclusively when a step opens a valid Settings screen not currently indexed in the catalog.
- **`query_variations`**: 8 to 10 distinct paraphrases across varied registers (formal, casual, keyword-only, frustrated, typo-inclusive).

### 3.2 Non-Negotiable Operational Constraints
1. **Zero URL Leaks:** Strict automated regex scrubbing eliminates extraneous web links (`http`, `https`, `www.`, markdown links).
2. **Catalog Integrity:** Deeplinks are matched from `deeplinks.json` using semantic metadata descriptions. Altering or hallucinating URIs is forbidden.
3. **No Hallucinated Steps:** If reference text contains no viable solution, the engine returns an empty list (`contexts: []`) with fallback metadata (`"fallback": "no_match"`).
4. **Pure JSON Delivery:** Responses never include markdown fences (````json ... ````) or conversational preamble.

---

## 4. API Endpoints

### 1. `POST /v1/troubleshoot`
Processes a customer complaint and returns an actionable plan.

#### Request:
```json
{
  "query": "phone swipe gestures wrong direction after app install",
  "siis_response": "<optional raw text context>"
}
```
*Note: If `siis_response` is omitted, the engine performs semantic lookup against pre-warmed cache entries.*

#### Response (200 OK):
```json
{
  "query": "The mobile phone swipe navigation moves up or down instead of left or right after downloading an app",
  "query_variations": [
    "Ever since I installed a new app, swiping on my phone scrolls up and down instead of going left or right.",
    "Why does my phone swipe vertically when I try to swipe sideways after downloading an app?",
    "phone swipe gestures wrong direction after app install",
    "I downloaded an application yesterday and now the swipe navigation on my Galaxy moves up or down when it shouldn't.",
    "Screen navigation gestures are misbehaving after an app download; horizontal swipes register as vertical movements.",
    "My phone's gesture navigation got messed up by a new app and swipes go the wrong way.",
    "what should I do when swiping left or right on my phone scrolls the screen up and down instead?",
    "Swipe navigation broken after installing app.",
    "This is so annoying - I can't swipe sideways anymore since installing that app, everything just scrolls vertically!",
    "Navigation swipes on my Samsung phone respond in the wrong axis after a recent app installation."
  ],
  "response": {
    "contexts": [
      {
        "goal": "Follow these steps to perform this Swipe Navigation Troubleshooting",
        "title": "Swipe navigation settings",
        "score": 0.93,
        "actions": [
          {
            "actionName": "Configure Navigation Bar Settings",
            "description": "It will let you choose navigation type",
            "category": "auto",
            "stepGroups": [
              {
                "steps": [
                  "Navigate to and open Settings.",
                  "Tap on Display.",
                  "Tap on Navigation bar.",
                  "Select your preferred navigation type between Buttons and Swipe gestures.",
                  "Optionally toggle on Gesture hint to display guidance lines at the bottom of the screen."
                ],
                "actionableDeeplink": {
                  "deeplink": "bixby://masked/act/display/navigation_bar",
                  "description": "Open navigation bar settings under Display",
                  "message": "choose navigation type in Display settings"
                },
                "validationDeeplink": null
              }
            ]
          }
        ]
      }
    ]
  },
  "meta": {
    "latency_ms": 0.18,
    "cache_hit": true,
    "model": "fast-path-cache-tier1_exact",
    "cost_usd": 0.0
  }
}
```

### 2. `GET /health`
Returns HTTP 200 `{"status": "ok"}` when the caching layer, model connections, and indexes are fully initialized.

---

## 5. System Performance & Evaluation Results (`metrics.md`)

Benchmark conducted with 35 requests per execution path:

| Metric | Target | Measured Value |
| :--- | :--- | :--- |
| **Exact Match Cache Hit (P95)** | &le; 300 ms | **0.002 ms** |
| **Semantic Paraphrase Cache Hit (P95)** | &le; 300 ms | **0.284 ms** |
| **Semantic Paraphrase Hit Rate** | &ge; 80% | **100.0%** |
| **Cold Query Extraction & Mapping (P95)** | &le; 8000 ms | **5.25 ms** |
| **Schema Conformance** | &ge; 99% | **100.0%** |
| **Rule Compliance (Goal/Title/Description)** | &ge; 95% | **100.0%** |
| **Absolute URL Leaks** | 0 | **0** |
| **Deeplink Catalog Validity** | 100% | **100.0%** |
| **Cache Hit Cost** | $0.00 | **$0.00** |

---

## 6. Project Structure

```
smart_troubleshooting_engine/
│
├── data/
│   ├── queries.json              # Canonical queries across Battery, Display, Camera, Performance
│   ├── siis_responses.json       # Pre-cleaned SIIS knowledge-base customer care texts
│   ├── deeplinks.json            # Deep catalog of Samsung Galaxy masked deeplinks (bixby://masked/act/...)
│   └── samples/                  # Five complete reference input-output pairs
│       ├── sample_1_navigation.json
│       ├── sample_2_battery_drain.json
│       ├── sample_3_screen_flicker.json
│       ├── sample_4_camera_lag.json
│       └── sample_5_device_slow_performance.json
│
├── app/
│   ├── __init__.py
│   ├── schema.py                 # Exact Pydantic data contract (Appendix A & B)
│   ├── config.py                 # Operational thresholds & parameters
│   ├── main.py                   # FastAPI service with /v1/troubleshoot, /health, /
│   ├── engine/
│   │   ├── __init__.py
│   │   ├── enrichment.py         # Stage 0: Query normalization & 8-10 diverse register paraphrases
│   │   ├── extractor.py          # Stage 1: Structure extraction (One Action = One Screen)
│   │   ├── deeplink_matcher.py   # Stage 2: BM25 metadata matching & dummy_positive support
│   │   ├── sequencer.py          # Action ordering (auto -> manual -> critical last)
│   │   ├── validators.py         # Programmatic guardrails (zero URL leaks, 5-7 word descriptions)
│   │   ├── pipeline.py           # Unified cold query pipeline
│   │   └── llm_service.py        # Gemini API client & deterministic fallback
│   ├── cache/
│   │   ├── __init__.py
│   │   └── fast_cache.py         # Sub-300ms multi-tier semantic cache
│   └── static/
│       ├── index.html            # Interactive troubleshooting dashboard & Galaxy phone simulator
│       ├── app.js                # One-tap settings deeplink interactive simulator
│       └── style.css             # Samsung One UI inspired dark theme
│
├── tests/
│   ├── test_schema_compliance.py # 100% adherence to Appendix A
│   ├── test_zero_url_leaks.py    # URL regex scrub verification
│   ├── test_action_ordering.py   # Hierarchy test: auto -> manual -> critical last
│   ├── test_cache_latency.py     # P95 <= 300ms & >= 80% paraphrase hit verification
│   └── test_api_endpoints.py     # FastAPI integration tests
│
├── benchmarks/
│   └── run_benchmarks.py         # Comprehensive benchmark suite generating metrics.md
│
├── metrics.md                    # Standardized evaluation report (Appendix C)
├── run_server.py                 # One-click FastAPI server starter
└── requirements.txt              # Production dependencies
```

---

## 7. How to Run

### 1. Run Tests
```bash
python -m unittest discover -s tests -p "test_*.py"
```

### 2. Run Benchmarks & Re-generate `metrics.md`
```bash
python benchmarks/run_benchmarks.py
```

### 3. Launch Interactive Server
```bash
python run_server.py
```
Open **`http://localhost:8000`** in your browser to experience:
- Live preset scenario selection (Navigation, Screen Flicker, Battery Drain, Slow Phone, Camera).
- Sub-300ms latency and cache hit badges.
- One-Tap Deeplink simulation updating the interactive Galaxy phone screen in real time.
- OpenAPI Interactive Docs at `http://localhost:8000/docs`.
