# friendlyfriends: an AI system that costs nothing to run

**[Source](https://github.com/humbertowgw-maker/friendlyfriends) · [Postmortem case study](./friendlyfriends.md) · [Back to profile](../README.md)**

The [first friendlyfriends case study](./friendlyfriends.md) covers a production
incident — a feature that broke twice and got cut. This one is about the part of the
project that didn't break: the actual AI system underneath it, built to a hard
constraint that most AI tooling ignores completely — it has to work with zero paid API
keys configured. Every piece below is quoted from the real, current source, not
paraphrased from the README.

## A cost-router that scores, not just picks a default

Most "route to the cheapest model" tooling is a lookup table. `SmartRouter` instead
scores providers from real usage history, and the scoring formula changes depending on
what you're optimizing for:

```js
switch (taskType) {
  case 'cost-sensitive':
    score = costScore * 0.7 + latencyScore * 0.1 + familiarityScore * 0.2;
    break;
  case 'speed':
    score = latencyScore * 0.7 + costScore * 0.1 + familiarityScore * 0.2;
    break;
  case 'quality':
    score = familiarityScore * 0.5 + latencyScore * 0.3 + costScore * 0.2;
    break;
}
```

`costScore` and `latencyScore` come from a real SQL aggregate over the last 30 days of
usage (`AVG(cost_usd / NULLIF(input_tokens + output_tokens, 0))`, `AVG(latency_ms)`),
not a hardcoded price sheet — so the recommendation reflects what a provider has
actually cost and how fast it's actually responded, not its list price.
`familiarityScore` weights toward providers with more usage history
(`min(usage_count / 20, 1)`), so a provider tried twice doesn't outrank one proven over
fifty calls. With no usage data yet, it returns `recommendation: null` and says so
plainly — a fabricated "best" pick with no evidence behind it would be worse than an
honest "not enough data."

## A generation pipeline that degrades instead of failing

`GeneratorManager` tries image-generation adapters in priority order and falls through
on failure — three free providers (a local Stable Diffusion install, Pollinations.ai,
HuggingFace) chained so the pipeline still produces output even if one is down or
unconfigured:

```js
for (const adapter of enabledAdapters) {
  try {
    const result = await adapter.generate({ ... });
    return { ...result, generator: adapter.name };
  } catch (e) {
    lastError = e;
    // Try next adapter
  }
}
throw new Error(`All generators failed. Last error: ${lastError?.message}`);
```

Reading the actual adapter registration while writing this case study turned up a real,
small discrepancy: the code comment states the intended fallback order as "local SD
first, then Pollinations, then HuggingFace," but the registered priority numbers had
Pollinations behind HuggingFace — the opposite of what the comment says. The test suite
didn't catch it because it verifies the *sorting logic* against synthetic priorities,
not the *real* registration against the stated intent — a gap between "the mechanism is
correct" and "the mechanism is configured correctly," which are different claims. Fixed
as part of writing this up (`PollinationsAdapter` priority 50 → 30), verified the
existing 8-test suite still passes. Small bug, but the kind that's invisible until
someone actually reads the source against its own comments instead of trusting either
one alone.

## Not regenerating what you already have

Every asset request goes through `PipelineHook.resolveAsset()` first, which checks
inventory before calling any generator at all:

```js
const existing = this.inventory.lookupAsset(characterId, label, type);
if (existing) {
  const updated = this.inventory.incrementAssetUse(existing.id);
  return { asset: updated, reused: true };
}
// Not found — generate it
```

Generation is the slow, rate-limited, sometimes-costs-something path; the system
defaults to not doing it. A second effect of the same function: when a new asset gets
generated, it checks for any *pending gap* records requesting that exact label/type
combination and auto-resolves them — so a manually-flagged "we need a sitting pose for
Athena" request clears itself the next time that asset actually gets generated, without
a separate reconciliation step.

## Turning three free tools into a real video pipeline

The episode assembly isn't a diagram — it's a real `ffmpeg` shell-out with a working Ken
Burns zoom/pan filter, run against actual generated images and TTS audio:

```js
const cmd = `ffmpeg -y -loop 1 -i "${imagePath}" -i "${audioPath}" -c:v libx264 ` +
  `-tune stillimage -c:a aac -b:a 192k -vf "scale=1280:720:force_original_aspect_ratio=decrease,` +
  `pad=1280:720:(ow-iw)/2:(oh-ih)/2,zoompan=z='min(zoom+0.0005,1.1)':...` +
  `-pix_fmt yuv420p -shortest "${outputPath}"`;
```

Scene clips get concatenated into a full episode via `ffmpeg`'s concat demuxer, and
`VideoAssembler` checks real availability (`ffmpeg -version`) at startup rather than
assuming the binary exists — `healthCheck()` reports `not_installed` with an install
link instead of failing opaquely mid-pipeline. The narration side maps each of the six
named cast members to a distinct free Edge TTS neural voice
(`achilles: 'en-US-ChristopherNeural'`, `athena: 'en-US-JennyNeural'`, and so on) with
its own speaking style, and checks for the `edge-tts` binary across the actual paths
Python installs it to on Windows before falling back to unavailable.

## Where it stands

The scoring, fallback, inventory, and assembly logic here are all real and covered by
the project's test suite. What's still genuinely unverified, same caveat as the first
case study: whether this is actually running anywhere continuously — `DEPLOYMENT.md`
is explicit that the live-deploy question (is Railway's auto-deploy actually connected,
is the latest commit actually live) can't be answered from the repo alone. The code
works; whether it's currently *running* is a different, still-open question.

---

**[← Back to profile](../README.md)**
