# friendlyfriends: cutting a feature instead of patching around it

**[Source](https://github.com/humbertowgw-maker/friendlyfriends) · [Architecture case study](./friendlyfriends-architecture.md) · [Back to profile](../README.md)**

friendlyfriends does two unrelated jobs in one codebase: it's an AI-provider
cost and rate-limit dashboard (tracking usage across OpenAI, Anthropic,
Gemini, and local models to recommend which one to route a task to), and a
from-scratch animated-episode pipeline that turns short scripts into
character art, TTS dialogue, and assembled video — for a real cast of
pets. Both run under a deliberate constraint: zero paid API keys. Every
generator, every voice, every video frame comes from free tools
(Pollinations.ai for images, Microsoft's Edge TTS, FFmpeg for assembly)
with an adapter pattern that falls back automatically if one is
unavailable. That constraint is the actual point of the project, not an
afterthought — the pipeline runs the same whether zero API keys are
configured or five are.

This case study isn't about that part, though. It's about a feature that
shipped, broke production twice, and what happened next — because that's
the part that actually shows how the project is engineered.

## A shipped feature that broke production — twice

At some point a `fleet/` module was scoped: `FleetManager`, `WorkerNode`,
`TaskQueue`, distributed-worker routing referenced from `server/index.js`
and surfaced as a tab in the client. The source files for it were never
actually committed. That's a real gap between "referenced" and "exists" —
and it meant the app tried to import modules that didn't exist in the repo
at all, failing to build in production.

Commit [`fb96762`](https://github.com/humbertowgw-maker/friendlyfriends/commit/fb96762)
("Fix broken production build and inventory tab") pulled every reference to
`fleet/` out of `server/index.js` and the client's fleet tab. That cleanup
missed two call sites — `workerNode.start()` and `taskQueue.startPolling()`
— still invoked on server boot. Those kept crashing every server start,
including Railway's, until commit
[`54cff34`](https://github.com/humbertowgw-maker/friendlyfriends/commit/54cff34)
("Fix production crash: remove dead workerNode/taskQueue references") found
and removed them too.

The decision that matters here isn't the bug — missing a call site during a
cleanup is ordinary. It's what happened after: rather than stub out
`FleetManager` or patch the boot sequence to tolerate a missing module, the
half-shipped feature got ripped out completely, in both passes. A feature
that's half-built and referenced everywhere is a worse state than no
feature at all, and the fix treated it that way instead of papering over it.

## Deploy automation that didn't survive contact with reality

A GitHub Actions workflow to run `railway up --ci` on every push was added,
adjusted, and then deleted — all within a few commits of each other:
[`5ad96d0`](https://github.com/humbertowgw-maker/friendlyfriends/commit/5ad96d0)
added it, [`699b190`](https://github.com/humbertowgw-maker/friendlyfriends/commit/699b190)
fixed how the deploy token was passed to the CLI, then
[`c35061e`](https://github.com/humbertowgw-maker/friendlyfriends/commit/c35061e)
deleted the file outright. Net effect: no GitHub Actions deploy exists in
the repo today — the `.github` directory doesn't exist at all.

`DEPLOYMENT.md` documents this honestly rather than pretending a clean
answer exists: the most likely explanation is that Railway's own
GitHub-connected auto-deploy took over (a mechanism configured entirely on
Railway's side, invisible to the repo), but the doc explicitly lists what
it *can't* confirm without dashboard access — whether auto-deploy is
actually on, whether the latest commit has actually shipped, whether
required environment variables are set. Three independent checks against
the GitHub API (webhooks, commit statuses, the Deployments API) all came
back empty, and the doc says plainly that this rules out a webhook or
Actions-based mechanism without proving anything about Railway's own
integration either way. Reporting an inconclusive result as inconclusive,
instead of rounding it up to "deploy works," is the same instinct as the
`fleet/` removal: don't leave a false claim standing just because a cleaner
one isn't available yet.

## Proving the pipeline actually ran

Two commits carry their own test evidence in the message body instead of
just a description of intent. [`0274c51`](https://github.com/humbertowgw-maker/friendlyfriends/commit/0274c51)
("test: full end-to-end pipeline validation") records the real per-stage
result: `Pollinations background image: OK`, `Edge TTS narration audio: OK`,
`FFmpeg Ken Burns video (1280x720, 9.2s): OK`.
[`07dd43d`](https://github.com/humbertowgw-maker/friendlyfriends/commit/07dd43d)
("test: full episode build") goes further — a real named episode
("Achilles Explores the Garden"), three scenes, six TTS voice lines across
three characters, and a final measured output: a 21-second, 1280×720
concatenated MP4. Those numbers only exist if the pipeline actually ran
end to end and someone looked at what came out, which is why they're
committed as evidence rather than left as an assumption.

## Where it stands

The cost-dashboard half and the episode-pipeline half both work as
described, verified by the commits above rather than by inspection alone.
What's still genuinely open, per `DEPLOYMENT.md`: whether the live Railway
services are actually connected to auto-deploy from this repo, or whether
the last few commits are sitting on `main` waiting for a manual push. That
question doesn't have an answer here yet — checking it needs the Railway
dashboard, not more git archaeology.

---

**[← Back to profile](../README.md)**
