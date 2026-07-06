# SHERPA Reality Lattice

**One line:** Deterministic constraint validator that checks every sketch against hard travel facts before it is shown — no LLM calls, pure rule logic.

## What it does
- `evaluate()` checks a sketch against 8 hard rules and returns `LatticeResult` with status OK / ADJUSTED / CONFIRM / BLOCKED
- Auto-fixes are applied automatically (e.g. shift dates, upgrade hotel style)
- Confirms are surfaced to the user as questions they must answer before proceeding
- Rules are pure Python — no external calls, no LLM. Data (closures, visa lead times) is embedded in the module

## 8 Hard Rules
| Rule | Category | Severity |
|------|----------|----------|
| Dates in the past | Auto-fix: next available weekend | block |
| Visa lead time too short (CN:14d, IN:14d, RU:21d, NG:21d for FR/IT) | Offer date shift | confirm |
| Seasonal closure or peak event (Bastille Day, Ferragosto, Fashion Week, Easter) | Advisory or confirm | warn / confirm |
| Monthly weather warning | Advisory note | warn |
| Vatican / Colosseum booking lead time | Add pre-book reminder | confirm / warn |
| Large party (>8 pax) → suggest apartment | User must decide | confirm |
| Hostel + children → block | Auto-upgrade to boutique | block |
| Booking window < 48h | Availability caveat | confirm |
| Rome dress code for sacred sites | Advisory | warn |

## Logical flow
1. `evaluate(city, check_in, check_out, pax_adults, pax_children, nationalities, hotel_style, required_sites)` called per sketch
2. Each rule appended to `issues` list if triggered
3. Status determined by highest severity: BLOCKED > CONFIRM > ADJUSTED(warn) > OK
4. `LatticeResult.to_dict()` serialised into `SketchCard.lattice_fixes` and `lattice_confirms`

## Key files
| File | Function | Purpose |
|------|----------|---------|
| `lambda/shared/lattice.py` | `evaluate` | Main evaluator — 8 rules, no LLM |
| `lambda/shared/lattice.py` | `LatticeResult` | Output dataclass with status, issues, summary |
| `lambda/shared/lattice.py` | `LatticeStatus` | Enum: OK / ADJUSTED / CONFIRM / BLOCKED |

## Critical code
```python
# lattice.py — status rollup from severity set
severities = {i.severity for i in issues}
if "block" in severities:
    status = LatticeStatus.BLOCKED
elif "confirm" in severities:
    status = LatticeStatus.CONFIRM
elif issues:
    status = LatticeStatus.ADJUSTED    # warn-only issues
else:
    status = LatticeStatus.OK
```

## Tests
| Test file | What it covers |
|-----------|---------------|
| No dedicated lattice test yet | — |

## Gotchas
- Cities are hardcoded to Paris and Rome — closures and weather data for other cities are not implemented. Adding a new city requires adding `_<CITY>_CLOSURES` and `_weather_warnings[city]` data
- Visa lead times are approximate — they reflect the typical processing time, not the official consulate SLA. Users with existing visas should answer the confirm to proceed
- Lattice runs after hotel selection — it may BLOCK a sketch that was already partially rendered. SHERPA should run lattice early (before copy generation) to avoid wasted Bedrock calls
