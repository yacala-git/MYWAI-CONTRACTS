# PRIMA Admin UI

**One line:** AgentSetupDashboard provides 6 panels to manage SHERPA configuration — patterns, anchors, lattice rules, destination settings, TasteField axes, and amenity frequency data.

## What it does
- Single-page dashboard in `mywai-hotel-providers/ui/src/components/AgentSetupDashboard.tsx`
- All panels call SHERPA admin API (`https://yxg506ul0a.execute-api.eu-west-3.amazonaws.com`) with `x-admin-key` header
- Deployed to Amplify app `iteraxa` (d2fq0sm6efev9k) — `git push origin main` triggers auto-deploy
- DNA Config management lives in a separate page: `DnaConfigPage.tsx` (9 tabs for box16 config service)

## The 6 Admin Panels
| Panel ID | Name | What it manages |
|----------|------|----------------|
| `pattern-catalogue` | Pattern Catalogue | Trip pattern templates — view, edit render defaults, taste offsets |
| `tool-registry` | Tool Registry | Registered tool Lambdas — enable/disable, view contracts |
| `lattice-rules` | Lattice Rules | City-specific hard constraint rules — view auto (W6) + manual rules |
| `destination-admin` | Destination Admin | City config — active cities, anchor counts, lattice rule counts |
| `taste-field-axes` | TasteField Axes | 16 preference axes — view axis descriptions and valid ranges |
| `amenity-frequency` | Amenity Frequency | Per-city amenity frequency data — load on tab switch, Run Now trigger |

## Other panels (non-PRIMA)
| Panel ID | Purpose |
|----------|---------|
| `slot-schemas` | Slot MCQ schema viewer |
| `model-config` | Bedrock model tier config |
| `prompts` | System prompt viewer |
| `safety` | Safety policies + refusal stats |
| `costs` | Per-turn cost aggregations |
| `funnel` | Conversion funnel + daily trend |
| `experiments` | A/B experiment management |
| `adversarial` | Red-team attack suite |
| `shadow-ml` | Shadow ML stats + bandit |

## Sidebar grouping (`_groupForPanel`)
Panels are grouped in the sidebar: Sketch Engine (pattern-catalogue, tool-registry, lattice-rules, destination-admin, taste-field-axes, amenity-frequency), Conversation, Scoring, Intelligence, Monitoring.

## Key files
| File | Function | Purpose |
|------|----------|---------|
| `ui/src/components/AgentSetupDashboard.tsx` | All panels | Main dashboard component (3369 lines) |
| `ui/src/components/SketchpadPanels.tsx` | PatternCataloguePanel etc. | 5 panel components imported by dashboard |
| `ui/src/components/AgentSetupPage.tsx` | Router | Page wrapper + sidebar routing |
| `ui/src/components/DnaConfigPage.tsx` | DNA config | box16 config service UI (9 tabs) |

## Critical code
```typescript
// AgentSetupDashboard.tsx — panel type union
type Panel = 'slot-schemas' | 'model-config' | 'prompts' | 'safety' |
  'inspector' | 'costs' | 'funnel' | 'experiments' | 'outcome' |
  'adversarial' | 'shadow-ml' | 'data-sources' | 'destination-briefing' |
  'pattern-catalogue' | 'tool-registry' | 'lattice-rules' |
  'destination-admin' | 'taste-field-axes' | 'amenity-frequency';

// SHERPA API base — all admin calls go here
const SHERPA_API = import.meta.env.VITE_SHERPA_API_URL
  || 'https://yxg506ul0a.execute-api.eu-west-3.amazonaws.com';
```

## Tests
| Test file | What it covers |
|-----------|---------------|
| No automated UI tests | Manual testing via Amplify preview |

## Gotchas
- All icons must be from `lucide-react` — no emoji, no other icon libraries (see memory `feedback_icon_library.md`)
- Amplify deploys on `git push origin main` in `mywai-hotel-providers` — no deploy.sh step needed for UI changes
- DnaConfigPage is a separate page from AgentSetupDashboard — it accesses the box16 API directly at `https://h9we9bv5r4.execute-api.eu-west-3.amazonaws.com`, not the SHERPA API
