# DUEL — Implementation Plan (Security + Game Logic)

**Status:** Finalized for execution  
**Last updated:** 2026-05-22  
**Scope:** Harden trust boundaries, then ship scoped gameplay improvements without rewriting the core loop.

---

## Goals

1. **Security:** AI reward claims and energy/top-ups cannot be abused via client manipulation or serverless cold starts.
2. **Game logic:** Make picks more readable and meaningful (damage preview, hint stake, clutch turn) while keeping **3 cards / 3 turns / 100 HP**.
3. **Maintainability:** ESLint + CI, unit tests on pure game logic, dead code removed.

## Non-goals (this plan)

- Full PvP real-time sync (Phase 4 is design-only stub; implement after Phases 0–3 ship).
- New cards, deck builder, or on-chain streak NFTs.
- Treasury / reward amount changes on `DuelRewards.sol`.
- Next.js major version upgrade (patch-only if needed for CVE).

---

## Core loop (unchanged)

```
Connect wallet → start-duel (server hand) → 3 turns (ai-move) → win → claim-rewards (sign) → on-chain claim
```

Energy gates **starting** a duel (server-enforced). Progression (`totalWins`) affects **draw pool** (server-enforced).

---

## Execution order

```
Phase 0 (security hotfix)
    ↓
Phase 1 (session + wallet binding + Redis hygiene)
    ↓
Phase 2 (server energy + progression)
    ↓
Phase 3 (game logic — engine + UI)
    ↓
Phase 4 (PvP honesty — later sprint)
    ↓
Phase 5 (quality — CI, tests, cleanup)
```

Phases 0–3 are the **first shipping milestone**. Phase 4 is a separate milestone. Phase 5 runs in parallel where possible (CI in Phase 0).

---

## Phase 0 — Security hotfix

**Estimate:** 0.5–1 day  
**Blocks:** Everything else should start after 0.1.

| ID | Task | Files | Acceptance criteria |
|----|------|-------|---------------------|
| 0.1 | Delete `test-redis.cjs` or rewrite to use env vars only; **rotate Upstash token** if file ever contained real credentials | `test-redis.cjs`, `.gitignore` | No secrets in repo; token rotated in Upstash console |
| 0.2 | Add `.eslintrc.json` (`eslint-config-next`) + `.github/workflows/ci.yml` (`npm run type-check`, `npm run lint`, `npm run build`) | root, `.github/` | CI passes on `master` |
| 0.3 | Document required env vars in `Readme.md` (Upstash, operator key, treasury) | `Readme.md` | Matches `.env.local.example` |

**Phase 0 done when:** CI green; no credential files; README lists Redis as required for production.

---

## Phase 1 — Wallet-bound duel sessions

**Estimate:** 1–2 days  
**Depends on:** Phase 0.1

| ID | Task | Files | Acceptance criteria |
|----|------|-------|---------------------|
| 1.1 | Add `playerAddress: string` to `DuelSession`; set in `createAiDuelSession` | `duelSessionStore.ts` | Session JSON includes lowercase checksummed address |
| 1.2 | `POST /api/start-duel` requires `playerAddress` (valid `0x` address); reject if missing | `start-duel/route.ts` | 400 without address |
| 1.3 | Client sends `address` from `useAccount` when calling `start-duel` | `useGameState.ts` | Hand only dealt after wallet known |
| 1.4 | `POST /api/claim-rewards` requires `playerAddress === session.playerAddress` | `claim-rewards/route.ts` | 403 if mismatch |
| 1.5 | Move top-up tx dedupe to Redis (`topup:tx:{hash}`) with TTL; remove `__TOPUP_USED_TX__` global | `topup-energy/route.ts`, optional shared `src/lib/redis.ts` | Same tx cannot grant bonus twice across cold starts |
| 1.6 | Wrap `ai-move` in try/catch; consistent 500 JSON errors | `ai-move/route.ts` | Malformed body / Claude failure returns structured error |
| 1.7 | Stop returning `gameState` in `ai-move` response; client uses `state` only | `ai-move/route.ts`, `useGameState.ts` | No full internal state in API JSON |

**Phase 1 done when:** Cannot claim a duel signature for a wallet that did not start that `duelId`; top-up dedupe survives redeploy.

---

## Phase 2 — Server-side energy & progression

**Estimate:** 2–3 days  
**Depends on:** Phase 1 (sessions keyed by wallet)

| ID | Task | Files | Acceptance criteria |
|----|------|-------|---------------------|
| 2.1 | Add `src/lib/playerStore.ts` — Redis keys for `energy:{address}`, `wins:{address}` (memory fallback for local dev) | new module | Same Upstash pattern as `duelSessionStore` |
| 2.2 | Energy model: 5 base lives, 4h recharge per missing life, `bonusLives` from top-up; mirror current `useEnergy` rules server-side | `playerStore.ts` | `getEnergy`, `consumeLife`, `grantBonus` |
| 2.3 | `start-duel` calls `consumeLife(playerAddress)`; 402/403 if no lives | `start-duel/route.ts` | API refuses duel with 0 energy |
| 2.4 | `topup-energy` calls `grantBonus(playerAddress)` after tx verify | `topup-energy/route.ts` | Bonus reflected on next `start-duel` |
| 2.5 | `start-duel` reads `totalWins` from `playerStore`, not request body | `start-duel/route.ts`, `useGameState.ts` | Client cannot inflate tier pool |
| 2.6 | On successful claim (or on win in `ai-move` when `isOver && playerWon`): increment server `wins` | `claim-rewards` and/or `ai-move` | Wins sync with tier unlock thresholds (5, 15) |
| 2.7 | Add `GET /api/player-state?address=` returning `{ lives, bonusLives, nextRechargeAt, totalWins }` | new route, `useEnergy.ts` | UI reads server state; localStorage only as cache/fallback |
| 2.8 | Rate limit: 30 `ai-move` / 10 `start-duel` / 5 `claim-rewards` per address per hour (Redis counters) | middleware or per-route | 429 when exceeded |

**Phase 2 done when:** Clearing localStorage does not grant infinite duels; tier hands follow server win count.

**Migration note:** On first `GET /api/player-state`, optionally seed `wins` from `localStorage` once (document in PR) so existing players keep tier progress.

---

## Phase 3 — Game logic updates (Tier A)

**Estimate:** 2–3 days  
**Depends on:** Phase 1.7 (client already uses `state` from API)

All rule changes live in **`gameEngine.ts`** so server replay in `claim-rewards` stays authoritative.

| ID | Task | Files | Acceptance criteria |
|----|------|-------|---------------------|
| 3.1 | **Clutch turn:** when resolving turn 3 (`state.turn === 3` before resolve), both sides deal +10% damage (floor) | `gameEngine.ts` | Unit test: turn 3 damage > turn 2 same cards |
| 3.2 | **Hint stake:** if AI card `type` matches `aiHintType` from request, AI gets +5 shield for that turn only (applied before damage calc) | `gameEngine.ts`, `ai-move/route.ts` (pass hint through session or validate against last stored hint) | Unit test: Block hint + Block card reduces player damage |
| 3.3 | Store `lastAiHintType` on session when player enters pick phase (client reports on `ai-move`, server validates enum) | `duelSessionStore.ts`, `ai-move` | Server does not trust hint without session continuity |
| 3.4 | **Damage preview helper** `previewDamage(attacker, defenderShield)` in `gameEngine.ts` or `cards.ts` | new export | Pure function, tested |
| 3.5 | **CardTile / PlayerHand:** show “Deals ~X” and “Blocks ~Y” on hover/always | `CardTile.tsx` | Uses card stats; optional “vs hint” line when pick phase |
| 3.6 | **Turn damage ticker:** after resolve, show `−N` on HP bars using `TurnResult` fields | `HealthBarsSection.tsx` or arena | Visible for 1s after each turn |
| 3.7 | **Tier badge** on cards (I / II / III from `card.tier`) | `CardTile.tsx` | Visible on hand |
| 3.8 | **Perfect duel** (optional): win with `playerHp >= 80` → `grantBonus` +1 or client-only toast first | `gameEngine` + `useGameState` | Document in UI; no extra cUSD |

**Phase 3 done when:** `npm run type-check` passes; gameEngine unit tests cover clutch + hint stake; claim replay still validates wins.

**Hint stake flow (final):**

```
pick phase → client sets random hint → ai-move sends aiHintType
→ server stores hint on session → resolves with +5 AI shield if AI honors type
→ CIPHER prompt unchanged (may still bluff narratively)
```

---

## Phase 4 — PvP honesty (separate milestone)

**Estimate:** 1–2 weeks  
**Start after:** Phases 0–3 deployed and stable.

| ID | Task | Acceptance criteria |
|----|------|---------------------|
| 4.1 | `PvpDuelSession` in Redis: hands, transcript, both addresses, on-chain `duelId` | Same replay pattern as AI |
| 4.2 | APIs: `pvp/start`, `pvp/move`, `pvp/claim-resolve` (replay before sign) | `sign-resolve` only if replay winner matches |
| 4.3 | `/pvp` page uses shared `gameEngine` | No client-declared winner |

**Not started in first milestone.**

---

## Phase 5 — Quality & cleanup

**Estimate:** 1–2 days (parallel with Phase 3)

| ID | Task | Acceptance criteria |
|----|------|---------------------|
| 5.1 | `vitest` or `jest` + tests for `gameEngine`, `drawHandWithRng`, `getTieredPool`, damage preview | `npm test` in CI |
| 5.2 | Remove legacy `turns` from `useClaimReward` | No dead API fields |
| 5.3 | `.gitignore` `artifacts/`, `cache/` if not needed for deploy | Cleaner repo |
| 5.4 | Patch Next.js 14.x to latest patch | `npm run build` still passes |

---

## API contract changes (summary)

### `POST /api/start-duel`

```json
// Request (new required field)
{ "playerAddress": "0x..." }

// Response (unchanged)
{ "duelId": "...", "hand": [...] }
```

**Errors:** `400` invalid address · `403` no energy · `429` rate limited

### `POST /api/ai-move`

```json
// Request (unchanged fields + aiHintType validated)
{ "duelId", "playerCard", "aiHintType": "attack"|"defend"|"special", ... }

// Response (removed gameState)
{ "card", "reasoning", "state": { playerHp, aiHp, turn, isOver, playerWon, turnsCount } }
```

### `POST /api/claim-rewards`

**Errors:** `403` address mismatch · `409` already claimed (unchanged)

### `GET /api/player-state?address=0x...` (new)

```json
{
  "lives": 3,
  "bonusLives": 1,
  "nextRechargeAt": 1730000000000,
  "totalWins": 7,
  "maxLives": 5
}
```

---

## Testing checklist (manual, before merge)

- [ ] Start duel without wallet → blocked
- [ ] Start duel with 0 energy → blocked
- [ ] Win duel, claim with same wallet → signature OK
- [ ] Win duel, claim with different wallet → 403
- [ ] Replay same top-up tx → 409
- [ ] Turn 3 deals more damage than turn 1 (same cards, clutch)
- [ ] AI honors hint type → extra shield observable in HP delta
- [ ] CI: type-check, lint, build, test

---

## Rollout

1. Deploy with Upstash env vars set on Vercel.
2. Rotate Redis token if Phase 0.1 applied.
3. Monitor: 429 rate, 403 claim mismatches, Anthropic errors on `ai-move`.
4. Optional: one-time wins migration banner in UI (“Progress synced to server”).

---

## Task checklist (copy for PR tracking)

```
Phase 0
[ ] 0.1 Remove/secure test-redis + rotate token
[ ] 0.2 ESLint + GitHub Actions CI
[ ] 0.3 README env docs

Phase 1
[ ] 1.1–1.4 Wallet-bound sessions + claim check
[ ] 1.5 Top-up Redis dedupe
[ ] 1.6–1.7 ai-move errors + slim response

Phase 2
[ ] 2.1–2.8 Server energy, wins, player-state API, rate limits

Phase 3
[ ] 3.1–3.8 Clutch, hint stake, preview UI, tier badge, damage ticker

Phase 5 (parallel)
[ ] 5.1–5.4 Tests + cleanup

Phase 4 (later)
[ ] 4.1–4.3 PvP session replay
```

---

## Open decisions (defaults chosen)

| Question | Decision |
|----------|----------|
| Increment wins on claim or on duel win? | **On duel win** in `ai-move` when `isOver && playerWon`; claim only signs |
| Perfect duel reward? | **+1 bonus life** via `grantBonus` (no extra cUSD) |
| Tie-breaker | **Keep player wins ties** (no change) |
| Seed localStorage wins once? | **Yes**, one-time on first `player-state` fetch |

---

*Approved for implementation. Work phase-by-phase; one PR per phase recommended.*
