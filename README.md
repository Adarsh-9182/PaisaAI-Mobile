# Paisa — AI CFO (Mobile)

An AI-native personal finance app for India. Paisa answers money questions with **exact numbers from your own data**, not generic advice. A deterministic TypeScript engine computes every figure in integer paise (₹1 = 100 paise); the AI layer only *narrates* engine output and is structurally prevented from inventing numbers.

**Stack:** Expo SDK 56 · React Native 0.85 · React 19 · expo-router · Zustand · react-native-svg · a typed `StyleSheet` design system ("Ledger Noir"). Runs in **Expo Go** — no native build step required for development.

---

## Quick start

```bash
npm install
npx expo install --fix     # aligns native module versions to your Expo Go (good hygiene on a fresh machine)
npx expo start
```

Then press **i** (iOS simulator), **a** (Android emulator), or scan the QR with **Expo Go**.

> Versions in `package.json` are already pinned to the exact SDK 56 set, so `expo install --fix` is a safety net rather than a fix. It is still recommended after a fresh clone.

### Verify

```bash
npm run typecheck       # tsc --noEmit — 0 errors
npm run test:engine     # 18/18 — money math, formatting, affordability, forecasts
npm run doctor          # expo-doctor — environment + dependency validation
```

---

## What's verified

| Check | Status |
|---|---|
| `npm install` (no flags), lockfile committed | clean, zero peer conflicts |
| `tsc --noEmit` | 0 errors (strict) |
| Engine unit tests | 18/18 |
| `expo-doctor` | dependency/SDK checks pass |
| `expo export` (iOS + web) | bundles cleanly, all routes generated |

A device/emulator smoke test (interactive navigation, gestures, secure-store round-trips) and native EAS builds are the expected final step before release — they require hardware/build infra beyond the dev environment.

---

## Architecture (short)

Money is **integer paise** end to end (no floats), computed by a pure, tested engine. The AI is a *narrator* over engine output — it never calculates. Data, auth, AI, and persistence all sit behind typed interfaces, so swapping a mock for a real backend is a one-file change.

```
app/                 expo-router routes — thin files that render a screen
src/
  screens/           screen components (the real UI)
  navigation/        typed tab config
  api/               FinanceSource / DocumentSource contracts + mock adapters  ← backend seam
  database/          secureStorage (typed keychain persistence)
  models/            domain entities (finance, chat, document, user)
  features/          ai · auth · documents · finance (engine + tests)
  store/             Zustand stores
  components/         ui/ design system + feature components
  hooks/ theme/ types/ utils/ data/ constants/
```

See **ARCHITECTURE.md** for the full rationale, the engine invariant, the integration seams, and the libraries deferred to a dev-client phase.

### Folder responsibilities

- **`app/`** — expo-router owns this. Each route file is a 3-line delegate to a `src/screens` component. Layouts (`_layout.tsx`) and the auth gate (`index.tsx`) live here because the router requires them to.
- **`src/screens/`** — one component per screen. All real screen logic.
- **`src/api/`** — the external-data boundary. `FinanceSource`/`DocumentSource` interfaces with mock adapters today; real adapters (Account Aggregator, OCR) slot into `src/api/index.ts`.
- **`src/database/`** — the only place that touches `expo-secure-store`. Swap for MMKV here without touching features.
- **`src/models/`** — pure domain types. `@/types` re-exports these plus cross-cutting UI types.
- **`src/features/finance/`** — the deterministic engine and its tests. Pure functions, no React, no I/O.

---

## Environment variables

The app runs fully on mock data with **no environment variables required**. The integration points (all currently mocked) are:

| Concern | Flag / seam | To go live |
|---|---|---|
| Global mock switch | `USE_MOCK` in `src/constants/config.ts` | set `false` |
| AI provider | `src/features/ai/client.ts` | implement an `AIModel` (e.g. Vercel AI SDK + Anthropic/OpenAI) and provide its key via `app.config.ts` `extra` / EAS secrets |
| Bank data | `src/api/index.ts` → `financeSource` | RBI Account Aggregator adapter (Setu/Finvu) |
| Documents/OCR | `src/api/index.ts` → `documentSource` | storage + OCR pipeline |

When you add secrets, prefer EAS environment variables / `expo-constants` `extra` over committing them. `.env*.local` is git-ignored.

---

## Development workflow

1. `npx expo start`, open in Expo Go or a simulator.
2. Edit under `src/` — Fast Refresh applies changes live.
3. Add a screen: create `src/screens/XScreen.tsx`, then a thin `app/.../x.tsx` route that renders it. Add a tab in `src/navigation/tabs.ts` if needed.
4. Before committing: `npm run typecheck && npm run test:engine`.

---

## Production build & deployment

Native builds use **EAS Build** (no local Xcode/Android Studio required):

```bash
npm i -g eas-cli
eas login
eas build --platform ios        # or android, or all
eas submit --platform ios        # to App Store / Play Console
```

Web (static, expo-router output):

```bash
npx expo export --platform web   # outputs to dist/
```

App identifiers are set in `app.json` (`ios.bundleIdentifier` / `android.package` = `in.paisa.app`).

---

## Troubleshooting

- **"Native module mismatch" in Expo Go** → run `npx expo install --fix`, then restart with `npx expo start -c` (clears the Metro cache).
- **`expo-doctor` shows 2 network-related failures behind a restricted network** → the Expo config-schema and React Native Directory checks call external services; failures there are environmental, not project issues.
- **Stale bundle / weird resolution errors** → `npx expo start -c`.
- **Type errors after pulling** → `npm install` (a dependency may have changed), then `npm run typecheck`.
- **Web build can't find a native-only module** → that feature is mobile-only; guard it with `Platform.OS !== 'web'`.
