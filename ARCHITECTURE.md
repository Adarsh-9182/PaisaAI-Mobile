# Architecture — Paisa Mobile

## 1. The one invariant: money is integer paise

Every monetary value is an integer number of **paise** (₹1 = 100 paise), typed `Paise`. No floats in the money path. This eliminates the rounding drift that plagues finance apps.

- Conversion happens only at the boundary: `toPaise(rupees)` in, `formatINR(paise)` out.
- `formatINR` does Indian digit grouping (₹4,25,000) and compact forms (₹1.5L, ₹4.3Cr).
- The engine (`src/features/finance/engine.ts`) is **pure** — data in, numbers out, no I/O, no React — so it is unit-tested in isolation (`npm run test:engine`, 18/18) and trustworthy enough to feed the AI.

### The AI never invents numbers

```
user question
  → features/ai/router.ts        classify intent
  → features/finance/engine.ts   compute real figures (paise)
  → features/ai/mockResponder.ts compose a sentence quoting ONLY those figures
  → UI                           render answer + structured card
```

The AI is a **narrator over engine output**, not a calculator. `features/ai/client.ts` is the swap point: replace the mock `AIModel` with a real provider and it receives the same engine-computed context plus the `SYSTEM_CFO` grounding prompt. Numbers come from the engine; language comes from the model.

---

## 2. Folder structure

```
app/                          expo-router — file-based routes (thin delegates)
  _layout.tsx                 providers, fonts, splash, root Stack
  index.tsx                   auth gate → (tabs) or onboarding
  onboarding.tsx              → renders OnboardingScreen
  documents.tsx               → renders DocumentsScreen
  (tabs)/_layout.tsx          bottom tab bar (renders from navigation/tabs)
  (tabs)/{index,chat,dashboard,goals,profile}.tsx   → render *Screen components

src/
  screens/    HomeScreen, ChatScreen, DashboardScreen, GoalsScreen,
              ProfileScreen, DocumentsScreen, OnboardingScreen
  navigation/ tabs.ts                      typed tab definitions
  api/        types.ts                     FinanceSource / DocumentSource contracts
              index.ts                     active sources (the backend swap point)
              mock/                         mock adapters over src/data
  database/   secureStorage.ts             typed keychain persistence + STORAGE_KEYS
  models/     finance, chat, document, user   domain entities
  features/
    finance/  engine.ts + engine.test.ts   pure, tested
    ai/       router, mockResponder, client, prompts, types
    auth/     authService
    documents/documentService
  store/      ui, auth, finance, goals, chat   (Zustand)
  components/ ui/ (design system) + home/ chat/ dashboard/ goals/ common/
  hooks/      useTheme (provider), useGreeting
  theme/      tokens (Ledger Noir, light + dark)
  types/      index.ts (barrel re-exporting models + icon), icon.ts
  utils/      money, date, id
  data/       mock datasets
  constants/  config (USE_MOCK, plans, app meta)
```

**Layering rule:** routes → screens → stores → services (`features/*`, `api/`) → engine/database/utils. Dependencies point inward. The engine depends on nothing; UI depends on everything below it.

---

## 3. Why screens live in `src/`, routes in `app/`

expo-router is **file-based**: routes must physically exist in `app/`. So screens cannot simply move to `src/screens/` — instead each route is a three-line delegate:

```tsx
import HomeScreen from '@/screens/HomeScreen';
export default function HomeRoute() { return <HomeScreen />; }
```

This gives the conventional `src/screens/` separation (one responsibility per file, screens are unit-targetable and reusable) without fighting the router. Layouts and the auth gate stay in `app/` because the router owns navigation structure.

---

## 4. State management — Zustand

Five slices, each owning one concern. Stores call **services** and the **engine**; components read stores and dispatch actions, never calling services directly.

| Store | Owns |
|---|---|
| `useUIStore` | theme pref, biometric/notification toggles, hydration (via `database/secureStorage`) |
| `useAuthStore` | current user, sign-in/out (via `authService`) |
| `useFinanceStore` | snapshot/accounts/intelligence; `refresh()` re-pulls through `api/financeSource` |
| `useGoalsStore` | goals; `forecast`/`progressOf`/`simulate` call the engine |
| `useChatStore` | conversations + streaming reveal |

---

## 5. Integration seams (mock → real)

Everything external sits behind a typed interface. Flip `USE_MOCK` in `constants/config.ts` and replace the implementation; the UI is untouched.

| Concern | Interface | Mock today | Live |
|---|---|---|---|
| Bank data | `api/types.ts` `FinanceSource` | `api/mock/financeSource` | Account Aggregator adapter in `api/index.ts` |
| Documents | `api/types.ts` `DocumentSource` | `api/mock/documentSource` | storage + OCR adapter |
| AI | `features/ai/types.ts` `AIModel` | `mockResponder` | real provider in `features/ai/client.ts` |
| Auth | `authService` | mock + SecureStore | OTP/OAuth, same store interface |
| Persistence | `database/secureStorage` | SecureStore | add MMKV cache behind the same wrapper |

### Adding an AI capability
1. Add an `Intent` + regex in `ai/router.ts`.
2. Add the engine computation in `features/finance/engine.ts` (+ test).
3. Compose the response quoting only engine numbers.
4. (Optional) add a structured card in `components/chat/`.

### MCP / tool-calling
`ai/client.ts` is the seam. Expose engine functions as tools and keep a `verifyNarration()` guard that rejects any number not present in tool results — the grounding contract, enforced programmatically.

---

## 6. Design system — "Ledger Noir"

A typed `StyleSheet` system (not a utility framework) for deterministic rendering and zero babel/runtime risk in Expo Go.

- **Colors:** evergreen `#15795B` (brand), emerald `#34D399` (accent/positive), brass `#C9A24B`, coral `#F4736B` (destructive), forest `#0E1A14`, cream `#F6F4EF`. Full light + dark `ColorScheme` in `theme/tokens.ts`.
- **Type:** Fraunces (serif display), Space Grotesk (UI, `tabular-nums` so money columns align).
- **Theming:** `useTheme()` resolves light/dark from `useUIStore` + system; components read colors from context.
- **Icons:** the `IconName` type (`keyof typeof Ionicons.glyphMap`) makes every icon name compile-validated.
- Charts are hand-rolled `react-native-svg` (Catmull-Rom) — no charting dependency.

---

## 7. Libraries intentionally deferred

To keep first-boot clean in Expo Go, these were left out; each has a clean slot in a dev-client/EAS phase:

| Library | Why deferred | Slot |
|---|---|---|
| NativeWind | babel/runtime setup risk | swap the `StyleSheet` theme; tokens map 1:1 |
| Reanimated | needs babel plugin + dev client | replace RN `Animated` usages |
| React Query | unneeded while data is local | wrap `api/` calls in the stores |
| MMKV | needs native build | drop-in behind `database/secureStorage` |
| FlashList | lists are short | replace `ScrollView`/`.map` in chat, documents, goals |

Animations use the built-in `Animated` API and charts use `react-native-svg`, both Expo-Go-safe.

---

## 8. Security & privacy posture

- Money math is integer-paise, pure, and tested — no client-side float drift.
- Secrets/preferences via `expo-secure-store` (Keychain/Keystore), funnelled through one `database/secureStorage` wrapper with versioned keys.
- AI is grounded: it cannot surface a number the engine didn't produce.
- Designed for India's **DPDP Act** (data minimization, on-device document encryption, explicit consent for Account-Aggregator pulls).

---

## 9. Verification status

- `npm install` (no flags) → clean, lockfile committed, zero peer conflicts.
- `tsc --noEmit` → 0 errors (strict). `npm run test:engine` → 18/18.
- `expo-doctor` → dependency/SDK checks pass. `expo export` (iOS + web) → bundles clean.
- madge → 0 circular dependencies.
- Pending hardware: interactive device smoke test + native EAS builds.
