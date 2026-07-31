# KICKUP

**Three taps. Ten seconds. One goal.**

A 1v1 trivia penalty shootout for mobile. Pick a category, get matched with a
stranger, and take nine penalties at concentric targets in an empty goal while
wind shears the ball and flares fill the box with smoke. Before every shot you
both get five seconds and one football question — **get it right and your
target grows 10%.**

QuizUp's structure, Neo Turf Masters' shot mechanic, Pokémon's collection hook.

---

## Run it

```bash
git clone <your-remote> kickup && cd kickup

open index.html          # playable immediately. no build, no install.
npm test                 # 36 tests, zero dependencies
npm run check            # validate content + run tests (what CI runs)
npm run sim:curve        # simulate 200k shots and print the scoring curve
npm run dev              # static server on :5173, if you prefer http
```

There are **no dependencies**. Not "few" — none. `package.json` has empty
`dependencies` and `devDependencies` and it should stay that way for as long as
possible. Node's built-in test runner covers testing; the game is Canvas 2D and
WebAudio.

Requires Node 20+.

---

## What's here

```
kickup/
├── index.html              ← the whole playable game. one file, no build.
├── sim/                    ← deterministic core. ZERO engine deps. runs in a terminal.
│   ├── prng.js               seeded mulberry32 — Math.random() is banned in here
│   ├── ballistics.js         three taps → impact point. flight, curl, wind.
│   ├── targets.js            target motion, keeper pacing, blocking
│   ├── scoring.js            rings → points → grade
│   └── match.js              seed → whole match; inputs → authoritative result
├── content/
│   ├── balance/balance.js    every tunable number. data, no logic.
│   ├── questions/*.json      one file per category + schema.md
│   └── cards/                150 legends (not yet written — see docs)
├── test/                     36 tests incl. golden replays
├── scripts/
│   ├── validate-questions.mjs
│   ├── simulate.mjs          balance answers as numbers, not arguments
│   └── gen-golden.mjs        regenerate golden replays (read the warning first)
├── server/README.md          what goes there, and why it's nearly nothing
└── docs/design/              the full design document
```

---

## The one rule

**`sim/` has no DOM, no canvas, no engine, and no `Math.random()`.**

Three timestamps in, trajectory and score out. It must run under Node in a
terminal. That constraint buys three things that are otherwise very expensive:

1. **Balance you can measure.** `npm run sim:curve` runs 200,000 shots across
   three skill bands and prints the grade distribution. Balance arguments end
   with a number instead of a vibe.
2. **Server-side validation for free.** The server replays submitted inputs
   through the exact same module the client used. The client never reports its
   own score, so it can't lie about it.
3. **Ghost replays.** A whole match is a seed, nine input triples, and nine
   answers — **330 bytes on the wire.** That solves matchmaking at low
   population, offline play, and friend challenges in one stroke.

If you find yourself wanting to `import` something from `client/` into `sim/`,
the thing you want belongs in `sim/`.

### Corollary: quantise at capture, not at transmission

Gauge values are rounded to three decimals **at the moment of the tap**, before
anything scores them. Three decimals is finer than a pixel at 288px, so it costs
the player nothing.

Round only when serialising, and a 0.001 shift near a ring boundary silently
flips the ring. The server then disagrees with the client, and the ghost you
serve is not the match that was played — a desync rare enough to be genuinely
horrible to debug. `test/golden/golden.test.js` enforces this.

---

## Golden replays

`test/golden/replays.json` holds four recorded matches — expert, decent, novice,
and someone panic-tapping near expiry — each paired with the score the sim
produced. The tests assert today's sim still produces the same score.

**If a golden test fails, you changed the scoring maths.** That may be correct.
But it also means every stored ghost replay and every leaderboard entry in
production just changed. Decide that deliberately before running
`npm run golden:regen`, and say so in the commit message. Quietly regenerating
defeats the entire point of having them.

---

## Adding questions

One JSON file per category in `content/questions/`. Four options, correct answer
first (`"answer": 0` — options are shuffled per match from the seed). Full rules
in `content/questions/schema.md`.

```bash
npm run validate:questions
```

runs in CI on every push, so malformed contributions fail before a human reads
them. It checks IDs are unique repo-wide, options are distinct and four in
number, text fits the 288px layout, and flags near-duplicates and time-sensitive
phrasing that will rot.

**Text only. Never images.** See the legal note below.

---

## Legal — read before writing any content

Two different situations, often confused:

**Fine:** factual trivia text naming real players. Facts aren't copyrightable,
and naming someone to state a true fact about them is nominative use. Trivia has
worked this way for decades. Country names as categories are fine too.

**Not fine:**

- **Photographs of players.** Agency copyright *plus* likeness. This is why the
  schema forbids images.
- **Cards depicting real players.** Name, face, voice, signature celebration,
  distinctive tattoos, recognisable silhouette — right of publicity in the US,
  image rights and passing off in the UK and EU. Being famous *creates* the
  right; it doesn't waive it. Rights survive death in many jurisdictions.
- **Club names, crests, kits, stadium names, trophies, league names.**
  Registered trademarks.
- **FIFA marks**, including "FIFA World Cup" and the trophy silhouette. Refer to
  *international tournaments*. Categories are nations, not a branded competition.

Kit colours in the game are **plain solids with no crest and no stripe geometry**
for this reason. A yellow shirt is not a kit design.

Get an IP lawyer to clear the 150 cards and a sample of 200 questions before
launch. Nothing here is legal advice.

---

## Architecture in one paragraph

There is no realtime game server, and there should not be one. Both players
answer the same questions and shoot at their own goals in parallel — there is no
player-to-player interaction anywhere, so there is nothing to synchronise. Two
stateless HTTP calls (`POST /match`, `POST /result`) cover the whole thing. No
sockets, no rooms, no P2P, no TURN bandwidth. It works at 400ms, on a train, in
a lift. Even the shared question screen, which looks like it needs live sync,
doesn't: the opponent's answers and answer *times* arrive in the match payload
before the first whistle. Full reasoning in design doc §15.3.

---

## Where to go next

1. **Play `index.html` fifty times.** Write down every moment your thumb felt
   wrong. That is the only question that matters right now: *is the shot
   satisfying 200 times in a row?*
2. **Then the trivia spike.** Does +10% feel powerful enough to make you care
   about the questions? If not, the hybrid doesn't work, and you want that
   answer in week eight rather than week thirty.
3. Migrate `index.html` to import from `sim/` rather than duplicating the maths.
   The sim is already the source of truth; the prototype hasn't caught up.

Open questions are catalogued in §17 of the design doc.

---

*Design document: `docs/design/KICKUP-design-doc.md`*
