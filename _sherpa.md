# SHERPA

**One line:** The conversational AI travel agent — understands what users want, loads their DNA profile, assembles hotel sketches, and streams them to the UI in real time.

## What it does
- Receives a user message via POST /turns, fires the cognitive Lambda async, returns 202 immediately
- Extracts intent (destination, dates, pax, budget, amenities) using Claude Sonnet via Bedrock
- Loads the user's TravelDNA profile from DynamoDB and injects it into the system prompt for personalisation
- Calls the DNA shortlist Lambda to find matching hotels, then assembles them into sketch cards
- Streams each block (hotel cards, anchors, copy, footer) to the frontend via AppSync GraphQL subscriptions

## Logical flow (per turn)
1. API Lambda (box7) receives `POST /turns`, fires cognitive Lambda with `InvocationType=Event`, returns `{turn_id}` 202
2. Cognitive Lambda (`handler.py`) loads user DNA (DDB), runs safety hard-block check, extracts intent via Bedrock Sonnet
3. `sketch_engine.py` enriches the query (rule map + Haiku translation if foreign text), calls DNA shortlist, fetches hotel metadata from DDB
4. Reality Lattice validates the sketch against hard constraints (visa, capacity, seasonal closures)
5. Render Policy decides FORK / SWATCH / SWIPER / COMMIT and how many sketches to show
6. Each hotel block is published to AppSync; frontend receives blocks in real time

## Boxes (10 CloudFormation stacks)
| Box | Purpose |
|-----|---------|
| b0-ssm | SSM parameter registry |
| b1-control | All 15 DynamoDB tables |
| b2-iam | IAM roles for Lambda/Step Functions |
| b3-config | S3 config buckets, Bedrock budget, guardrails |
| b4-cognitive | Cognitive Lambda — intent, slots, sketch engine |
| b5-execution | Execution worker Lambda (direct invoke from cognitive) |
| b6-tools | Tool Lambdas (one folder per tool) |
| b7-api | HTTP API Gateway + AppSync streaming |
| b8-schedules | EventBridge schedulers (stub) |
| b9-observability | Dashboards, alarms, cost rollup |
| b10-shadow-ml | Shadow ML stub |

## Key files
| File | Function | Purpose |
|------|----------|---------|
| `lambda/api/handler.py` | `lambda_handler` | API Gateway entry — routes 30+ endpoints |
| `lambda/cognitive/handler.py` | `lambda_handler` | Main turn handler — orchestrates all cognitive steps |
| `lambda/cognitive/sketch_engine.py` | `_search_and_rank_hotels` | Hotel search, ranking, and sketch assembly |
| `lambda/cognitive/intent.py` | `produce_intent` | Bedrock Sonnet call to extract structured IntentPayload |
| `lambda/shared/sketch_types.py` | `SketchFrame`, `HotelBlock` | All output types for the sketchpad UI |
| `lambda/shared/dna_client.py` | `load_user_dna` | DDB read + 300s TTL cache for DNA profile |

## Live endpoints
| Endpoint | URL |
|----------|-----|
| SHERPA HTTP API | `https://bc910jrc18.execute-api.eu-west-3.amazonaws.com` |
| GET /user/dna | Same base URL — returns DNA profile for current user |
| DNA Config (box16) | `https://h9we9bv5r4.execute-api.eu-west-3.amazonaws.com` |

## Critical code
```python
# api/handler.py — async turn dispatch
_lam = boto3.client("lambda", config=Config(read_timeout=5))  # async only
_lam_sync = boto3.client("lambda", config=Config(read_timeout=90))  # eval calls

# POST /turns fires cognitive Lambda with Event invocation — never waits for result
_lam.invoke(
    FunctionName=_COGNITIVE_FN,
    InvocationType="Event",
    Payload=json.dumps({"turn_id": turn_id, "user_id": uid, ...}).encode(),
)
return _resp(202, {"turn_id": turn_id})
```

## Tests
| Test file | What it covers |
|-----------|---------------|
| `tests/cognitive/test_api_handler.py` | Route matching, auth, 202 response |
| `tests/golden/test_pipeline_golden.py` | End-to-end intent → sketch golden cases |
| `tests/smoke/test_syntax.py` | py_compile on every handler |

## Gotchas
- `ReservedConcurrentExecutions: 1` on cognitive + dna-api + dna-shortlist is demo-only — MUST remove before production traffic
- `GET /user/dna` needs 4 things: API GW route, DDB IAM for DNA tables, Lambda invoke IAM for dna-api, and 4 env vars on the cognitive Lambda. Missing any one gives a CORS-masked 404
- Lambda zip packaging: `shared/` must be a subdirectory inside the zip, not at zip root. Use temp-dir copy pattern (see CLAUDE.md)
