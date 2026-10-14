# PLAN.md — INFLUENCER-VERSE

**Status:** build phase · **Date:** 2026-10-11
**Concept:** The player is a social-media influencer inside ANY fictional universe they choose
(free text). The game resolves the correct character cast, populates them as AI-powered NPC
influencers who post, comment, repost, follow/unfollow, start drama, and compete. Win: become
the #1 influencer. Lose: get canceled.
**Source blueprint:** `~/workspace/research_notes/influencer-game-design-blueprint-20261011-2004/report.md`
(followed decisively — see §1).

---

## 1. Headline architecture decisions (from blueprint, locked)

| Decision | Choice |
|---|---|
| NPC AI | **Hybrid:** deterministic sim owns ALL game state (traits, moods, relationships, factions, memory ledger). Utility-AI layer picks each NPC's *action*. LLM only renders *wording* for the chosen intent. LLM never touches game state. |
| LLM cascade | **Puter.js** (`https://js.puter.com/v2/`, `puter.ai.chat`, no key) → player-pasted **Groq** key → player-pasted **Gemini** key → **WebLLM** opt-in → **pure sim** (template banks). Fully playable at every tier, incl. zero-AI. |
| Universe→cast | **Curated parody packs** for popular universes (shipped, IP-safe, offline) + **AI-generated cast** for free-text universes at runtime (player-requested, never shipped in repo). **Narnia test:** 4-layer isolation — constrained generation, wiki-allowlist validation, verification pass, runtime roster/blocklist validator. Crossover syntax (`x`/`vs`) → explicit opt-in dialog, never silent mixing. |
| IP stance | **Zero copyrighted names/assets in the repo.** Curated packs are original parody universes. Runtime AI casts are generated at player request only. Default-on parody filter. Never market with real franchise names. |
| Tech | Vanilla JS + ES modules, **single `index.html`**, no build step. Pub/sub store. Versioned localStorage saves. Canvas particle VFX layer. WebAudio-synthesized SFX (no audio files). CSS neon, mobile-first. |

---

## 2. Game systems

### 2.1 Core loop
`Compose post (pick 1 of 6 types) → NPCs react (utility-scored) → engagement accrues →
rumors spread NPC→NPC → day ticks → rivals post & scheme → drama events fire →
leaderboard moves → win (#1) / lose (canceled)`.

### 2.2 Post types (6)
1. **Hot Take** — high spice, high risk/reward. 2. **Rumor** — seeds SIR propagation; can backfire if debunked. 3. **Photo** — safe, moderate reach (procedural emoji-art visual). 4. **Callout** — names a target; damages/boosts relationships, feud fuel. 5. **Collab** — invites an NPC; success depends on relationship; big mutual boost. 6. **Poll** — engagement bait, low risk, reveals NPC stances.

### 2.3 Engagement scoring
`score = reach × (timing × spice × credibility × audienceFit) + networkCascade`
- **reach** = followers × activity factor; **timing** = daypart bonus; **spice** = post-type × archetype match;
  **credibility** = player rep (rumors cost credibility if debunked); **audienceFit** = faction alignment of followers.
- **networkCascade**: each NPC engagement exposes the post to that NPC's followers (SIR-lite), with diminishing returns.
- **Virality**: if engagement velocity crosses threshold → 🔥 TRENDING → follower burst + confetti VFX + fanfare SFX.

### 2.4 Rumor propagation (SIR with mutation)
NPCs are **S**usceptible → **I**nfected (spreading) → **R**ecovered (skeptic/bored). Each day-tick,
spreaders infect contacts with p = rumorHeat × gullibility × relationship. Rumor text **mutates**
slightly per hop (template slot-swap). Heat decays; debunk events (skeptic NPCs) create immunity.
Player can ride or kill their own rumors.

### 2.5 NPC model
Each NPC: `traits {spice, loyalty, chaos, clout} (0–1)`, `mood`, `faction`, `relationships {-100..+100}`
to player + 3–5 NPCs, `memoryLedger[]` (salient events only), `saidCache` (anti-repetition),
`cooldowns{}`, follower count, archetype, voice notes. 12 NPCs per universe.

### 2.6 Utility-AI action selection (per NPC, per event)
`score(action) = Π considerations(0..1)` over: wasNamed, relationship, factionAlign, traitFit,
dramaState, cooldown, trendiness. Actions: `like, comment(support|attack|mock), repost, quotePost,
subtweet, startRumor, callout, defend, collab, unfollow, feud, apologize, ignore`.
Top-3 utility-weighted random pick (variety). **LLM renders only the winning intent** with a context
bundle (post text, computed stance, memory summary, drama flags, length cap). Sim fallback: trait-driven
template banks with slot filling.

### 2.7 Rival AI
The #1 NPC at game start is the designated **rival**. Posts 1–2×/day on a schedule; when the player
enters the top 3, rival escalates: subtweets → callouts → smear rumors → (rare) collab offer as trap.
Rival has its own follower growth; beating it requires sustained engagement, not one viral post.

### 2.8 Drama director
Event triggers on thresholds: **feuds** (mutual callouts, 3-day arc), **collabs** (mutual boost),
**cancellations** (scandal meter; 3 strikes or rep ≤ 0 = game over), **apologies** (rep recovery,
credibility cost), **milestones** (follower tiers → fanfare). Scandal meter rises from debunked rumors
and lost feuds; decays slowly with wholesome posts.

### 2.9 Follower economy & win/lose
- Followers: NPC follows + organic growth from viral posts + collab cross-pollination − unfollows from scandals.
- **WIN:** reach rank #1 on the leaderboard (overtake the rival).
- **LOSE:** canceled — scandal strikes ≥ 3 or reputation ≤ 0.
- Day cap: none (sandbox), but rival keeps growing — stalling loses.

### 2.10 Day-tick loop
`Next Day →` NPC scheduled posts (1–2 per active NPC) → rumor SIR tick → relationship decay toward
baseline → drama arc progression → scandal decay → leaderboard recompute → random world event
(10–15%: e.g. "award show", "server outage", "mysterious benefactor") → save.

---

## 3. Universe→cast resolution (runtime)

1. Player enters free text (or picks a curated pack chip / 🎲 surprise).
2. **Crossover detect:** regex for `\bx\b`, `\bvs\b`, `mashup`, `crossover`, or ≥2 known franchise names → opt-in dialog.
3. **Curated match:** if input matches a shipped pack (alias table) → load pack instantly (works offline).
4. **AI path** (needs any LLM tier): constrained draft (temp 0.2–0.3, "canonical characters only, omit if unsure")
   → chain-of-verification ("which franchise is each from?") → **Fandom wiki allowlist** (MediaWiki API,
   keyless) → bounded repair (≤2 retries, then shrink cast) → **player cast-review screen** → lock universe bible.
5. **No-AI fallback:** free text disabled with honest message → curated packs only.
6. **Runtime:** every NPC prompt carries universe bible + roster; post-generation validator scans for
   off-roster names + global cross-franchise blocklist → 1 regen → template fallback.

---

## 4. Likely bugs & mitigations

| # | Bug | Mitigation |
|---|---|---|
| 1 | Puter.js blocked (offline / `file://` / consent dismissed) | Feature-detect `window.puter`; every LLM call wrapped in try/catch with tier-fallback; sim-first means zero user-facing breakage. Test: block the script URL in headless run. |
| 2 | LLM returns garbage / non-JSON / overlong | Defensive parse (strip fences, JSON try, length clamp); 1 retry; then template-bank fallback. Renderer never throws into the sim. |
| 3 | NPC sameness (all NPCs sound alike) | Trait-divergent template banks (spicy/formal/wholesome/menace voices); said-cache + cooldowns + topic fatigue; utility-weighted top-3 randomness. |
| 4 | Runaway game state (followers → Infinity, NaN) | Clamp all numerics; invariant asserts in dev; economy tuned so rival growth ≈ player achievable growth. |
| 5 | Rumor propagation explodes or dies instantly | SIR params tuned (base p ≈ 0.25, heat decay 0.8/day); cap spreaders per tick; seeded RNG for reproducibility in tests. |
| 6 | localStorage schema drift / quota | Versioned save key (`iv_save_v1`); try/catch on parse with corrupt-save recovery; keep saves < 200KB (no LLM text persisted verbatim beyond ledger summaries). |
| 7 | Double day-tick / double rAF loops | Single `tick()` entry with `ticking` guard; one rAF loop for particles started once. |
| 8 | Mobile layout breakage | Mobile-first CSS, feed max-width 560px, bottom nav on small screens; headless test at 390×844. |
| 9 | XSS via player post text | Escape all user text before innerHTML; LLM text also escaped (never trusted). |
| 10 | Cast-gen hangs (LLM timeout) | 25s timeout per call with progress UI; cancel button; fallback to curated packs. |

---

## 5. UI/UX

**Screens:** Boot splash (VN INFLUENCER-VERSE, neon logo, Enter) → Universe picker (free text +
pack chips + 🎲 + AI-tier badge + handle/archetype) → Cast review (12 NPC cards, Enter the Verse)
→ Game (Feed / Compose / Ranks / Verse / Settings tabs) → Game over (win/lose, stats, restart).

**Feed:** familiar social-app cards — avatar (procedural canvas), handle, time, text, photo-art for
Photo posts, like/comment/repost counts, typing indicators ("💬 @rival is typing…"), trending badges.
**Compose:** modal with 6 post-type buttons, textarea, target picker for callouts/collabs, spice meter preview.
**Ranks:** leaderboard with follower bars, player highlighted, rival crowned 👑.
**Verse:** NPC grid — tap for profile (traits, relationship meter, memory highlights, follow/unfollow).
**Settings:** AI tier cascade (Puter status, Groq/Gemini key paste, WebLLM opt-in), SFX/VFX toggles, save/reset.

**First-60-seconds:** picker → cast reveal → guided first post → NPC reactions roll in → follower burst.
Loop taught by doing, no tutorial text walls.

---

## 6. SFX/VFX plan (all synthesized, zero audio files)

**SFX (WebAudio):** ui blip, post whoosh, like pop, repost swish, viral fanfare (arp), trending alarm,
drama sting (dissonant hit), feud drums, scandal buzz, milestone chime, win fanfare, lose descend,
day-tick soft chime, typing ticks. Master gain + mute toggle.
**VFX (canvas overlay):** follower "+N" floaters, confetti burst (viral/milestone), ember drift (feud/
drama), spark burst (post), screen shake (cancellation), glow pulse (trending badge). Particle pool
capped; `prefers-reduced-motion` respected.

---

## 8. Teardown upgrades (2026-10-14 — REA study of 3 open-source sims, re-implemented in original code)

Comparators (all cloned from official GitHub repos; zero code/assets/strings copied):
- **virality-simulator** (nagashreeavd) — NO LICENSE file in repo → treated as unlicensed:
  ideas only. Mechanisms learned: Independent Cascade diffusion, Barabási–Albert
  preferential-attachment graph, topic-vector cosine affinity, persona traits
  (susceptibility/shareDrive/influence/fickleness), shareability levers, verdict
  classification + R-factor.
- **liveops-tycoon** (stonedhawk, MIT) — offline progress credit, delta-time cap,
  exponential upgrade costs (base × 1.15^owned), derived-stats recalculation.
- **creator-journey-simulator** (tanya-garg10, MIT) — 7-chapter creator arc,
  consistency calendar, burnout meter, pivot choice (evolve vs double-down),
  milestone timeline. REA's JS analyzer failed on this repo (exit 1, no evidence);
  mechanisms taken from direct source read of its MIT-licensed script.

Implemented (all in `index.html`, sim-first, no new dependencies):
1. **Follower-graph cascade** — `buildVerseGraph()` (hub-biased links + wild-card),
   `runCascade()` (2-round Independent Cascade: only last round's engagers get one
   chance per unreached neighbor; p from affinity × shareability × sender push).
   `npcReact(..., {force})` keeps one engagement resolver. `settlePost` uses
   delta-crediting (`post.credited`) so the cascade re-settle never double-pays.
2. **Topic affinity** — `TOPICS` (drama/wholesome/analysis/chaos/glam/lore),
   per-NPC `tastes` (archetype profile + seeded jitter), `postAffinity()` cosine.
   Post types carry topic profiles; cascade odds + verdict on-target rate use it.
3. **Post verdicts** — `verdictFor()`: Flop / Niche Hit / Slow Burn / Viral Hit
   + tip + reach% (unique NPCs reached) + on-target% + R-factor, rendered on the
   post card; tallied in `G.stats.verdicts` and shown on the game-over screen.
4. **Retention loop** — daily **streak** (+2% followers at 3+ days, HUD chip),
   **burnout** (4+ posts/day builds it, halves engagement at ≥80%, rest days cure),
   **welcome-back** bonus (away 5+ min → +1%/hr, capped at +5%, announced on Continue).
5. **Reinvention pivot** — stall 3 days at one rank → one-time choice: Evolve
   (next archetype, credibility reset to 60, burnout cleared) or Double Down
   (+15% post reach, −5 rep). `doPivot()` is one-shot (`G.stats.pivoted`).
   Old saves load cleanly: tastes/graph backfilled, stats defaults merged, no
   welcome-back without `savedAt`.

Deliberately skipped: LiveOps-style exponential upgrade shop (no currency exists
in this game — followers ARE the score, so a shop would cannibalize the win
condition); delta-time cap (turn-based, no realtime loop); seeded-RNG replay
(virality's hash-seed reproducibility — the Verse's seeded `mulberry` RNG already
covers in-run determinism; full replay is out of scope).
QA: `node --check` on extracted module script + 2 zero-dependency headless
harnesses (49 assertions total, incl. a real cascade-verdict bug caught and
fixed: reach% double-counted engagers who were also exposed — now union-based).
No browser/JS DOM available in the worker env, so no visual check was run.

## 7. VN signature & portfolio requirements
- "VN" tag in header/footer + README + description.txt.
- Maximalist neon styling: dark bg, neon gradients, glow text, animated borders.
- README.md (features, how to play, AI cascade, screenshot section), LICENSE (MIT),
  `assets/screenshot.png` (real gameplay frame), `description.txt` (one line).
- Tested headless: universe pick → post → NPC reactions → day ticks → no errors → all DOM ids resolve.
