# Agent Setup Dashboard UI

**One line:** Admin dashboard for managing SHERPA configuration — 19 panels organised in sidebar groups, all calling the SHERPA admin API with x-admin-key auth.

## What it does
- Single-page React 19 / TypeScript / Tailwind app in `mywai-hotel-providers/ui/`
- Deployed to Amplify app `iteraxa` (d2fq0sm6efev9k) — `git push origin main` auto-deploys; no deploy.sh needed
- All admin calls go to `SHERPA_API` (env: `VITE_SHERPA_API_URL`) with `x-admin-key: VITE_SHERPA_ADMIN_KEY` header
- `_groupForPanel` assigns each panel to a sidebar group for navigation
- DnaConfigPage is a separate React route — it calls the box16 API directly (not SHERPA API)

## Panel groups and panels
| Group | Panels |
|-------|--------|
| Sketch Engine | pattern-catalogue, tool-registry, lattice-rules, destination-admin, taste-field-axes, amenity-frequency |
| Conversation | slot-schemas, model-config, prompts |
| Scoring | (none listed — DNA config is in DnaConfigPage) |
| Intelligence | safety, adversarial |
| Monitoring | costs, funnel, experiments, outcome, shadow-ml, inspector |
| Data | data-sources, destination-briefing, amenity-frequency |

## Panel type (all 19)
```typescript
type Panel = 'slot-schemas' | 'model-config' | 'prompts' | 'safety' |
  'inspector' | 'costs' | 'funnel' | 'experiments' | 'outcome' |
  'adversarial' | 'shadow-ml' | 'data-sources' | 'destination-briefing' |
  'pattern-catalogue' | 'tool-registry' | 'lattice-rules' |
  'destination-admin' | 'taste-field-axes' | 'amenity-frequency';
```

## Key files
| File | Function | Purpose |
|------|----------|---------|
| `ui/src/components/AgentSetupDashboard.tsx` | All panels | Main dashboard (3369 lines) |
| `ui/src/components/SketchpadPanels.tsx` | 5 component panels | PatternCataloguePanel, ToolRegistryPanel, LatticeRulesPanel, DestinationAdminPanel, TasteFieldAxesPanel |
| `ui/src/components/AgentSetupPage.tsx` | Page router | Sidebar + panel routing wrapper |
| `ui/src/components/DnaConfigPage.tsx` | DNA config UI | 9 tabs for box16 config service |

## Critical code
```typescript
// AgentSetupDashboard.tsx — sidebar group assignment
const _groupForPanel = (p: Panel | undefined): string | null => {
  // returns 'Sketch Engine', 'Data', 'Intelligence', etc.
  // Used to set active sidebar group on initial load via initialPanel prop
};

// Constants
const SHERPA_API = import.meta.env.VITE_SHERPA_API_URL
  || 'https://yxg506ul0a.execute-api.eu-west-3.amazonaws.com';
const ADMIN_KEY  = import.meta.env.VITE_SHERPA_ADMIN_KEY || '';
```

## Tests
| Test file | What it covers |
|-----------|---------------|
| No automated UI tests | Manual testing via Amplify preview |

## Gotchas
- All icons must be from `lucide-react` — no emoji, no other icon libraries. The component already imports a large set from lucide-react; check what's already imported before adding a new icon
- Amplify deploys on push to `main` in `mywai-hotel-providers` — UI and backend share the same repo. A backend change that also touches the UI requires one `git push`
- `DnaConfigPage.tsx` uses a different API base (`https://h9we9bv5r4.execute-api.eu-west-3.amazonaws.com`) than `AgentSetupDashboard.tsx` — don't mix them
