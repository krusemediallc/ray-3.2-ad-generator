# Ray 3.2 Ad Generator

A Claude Code workbench that generates finished ad creatives end-to-end with **Luma's ray-3.2** (video) and **uni-1** (images), plus **ElevenLabs** (VO / SFX / music) and **HyperFrames** (motion-graphic supers). Five complementary skills:

| Skill | What it makes |
|---|---|
| `uni1-image-ad` | uni-1 image ads, uploaded to Meta as paused ads |
| `image-ad-clone` | reverse-engineers reference ads into reusable prompt templates |
| `ray3-video-ad` | single ray-3.2 video clips — b-roll, i2v, restyles, reframes |
| `claymation-ad` | multi-beat claymation story films with consistent characters + full audio ("The Lab", 45.7s, ~$5.50 in Luma credits) |
| `cinematic-ad` | fast-cut trailer-style product ads with synced supers ("Stitched", 15s, ~$4.15 all-in) |

Start with [`CLAUDE.md`](./CLAUDE.md) for the agent-facing map of the whole workbench. The sections below document the original image workflow; the video skills follow the same conventions (`.env` keys, cost gates, paused-only Meta mutations).

> ### 👉 Sign up for Luma here
> **Everything in this repo runs on Luma (uni-1 + ray-3.2). Create your account and get your API key using this link:**
>
> ### **[➡️ lumalabs.ai — sign up](https://lumalabs.ai/app?utm_source=influencer&utm_medium=linkedin&utm_campaign=mrpaidsocial-ray-3_2&utm_content=faved&from_faved=true)**
>
> Then grab your key from the dashboard → API keys and drop it into `.env` as `LUMA_API_KEY`.

---

The image skill clones an existing ad's structure (page, ad set, link, CTA), generates a new uni-1 image (optionally grounded on your brand's product photo), writes fresh ad copy informed by your account's top-spending ads, and creates the ad **paused** for you to review and launch.

The project is built around Luma's models specifically — uni-1 for images, ray-3.2 for video. Substituting other models would silently break the prompt library, reference-grounding behavior, and aspect-ratio support. The helper scripts enforce the model locks at the CLI layer.

## What's in the box

Five complementary Claude Code skills.

**Image skills:**

- **`./skills/uni1-image-ad/`** — the **template-using** skill. Generates a uni-1 image and uploads it as a paused Meta ad creative. Clones an existing ad's structure (page, ad set, link, CTA), generates the new image (optionally grounded on your brand's product photo), writes fresh ad copy informed by your account's top spenders, and creates the ad. Includes 4 helper scripts and 3 reference docs (7-template prompt library, 15 ad-copy frameworks, Meta CLI flag reference).
- **`./skills/image-ad-clone/`** — the **template-creating** skill. Take any existing image ad → reverse-engineer it into a parameterizable prompt template → append it to the prompt library so `uni1-image-ad` can reuse the format with any brand. Iterates against the original to validate, then tests the generalized version with a different brand before saving.

**Video skills (ray-3.2, same `.env` + cost-gate conventions):**

- **`./skills/ray3-video-ad/`** — single ray-3.2 clips: product b-roll, image-to-video, restyles, and aspect-ratio reframes. Cost-gated (`--dry-run` first; clips cost $0.15–$3.60). Generation/review only — Meta upload for video isn't wired yet. Needs only `LUMA_API_KEY`.
- **`./skills/claymation-ad/`** — multi-beat claymation story films with consistent characters and full ElevenLabs audio (VO + SFX + score), assembled with ffmpeg. Needs `ELEVENLABS_API_KEY` + `ffmpeg`. Validated by "The Lab" (45.7s, 12 beats, ~$5.50 in Luma credits).
- **`./skills/cinematic-ad/`** — fast-cut trailer-style product ads with movie-trailer VO, sound design, original score, and motion-graphic supers. Needs `ELEVENLABS_API_KEY` + `ffmpeg` + Node 22+ (HyperFrames supers). Validated by "Stitched" (15s, 7 shots, ~$4.15 all-in).

Plus:

- **Offline Luma API docs** — `./docs/luma-agents-api/` — the canonical reference for the Luma uni-1 API. Read this when in doubt about behavior.
- **Validation runs** — `./iterations/` — the prompt-engineering work that built the seed library (T1–T7). Each prompt was iterated until uni-1 reproduced the original ad reference, then generalized into placeholders. You can re-run any of them with `python3 iterations/run_round.py <jobs.json>`.
- **Ad references** — `./Ad References/` — the swipe-file of real ads the prompt library was patterned after. Used as input for `image-ad-clone` when extracting new templates.

## Prerequisites

**Core (every skill):**
- macOS (the helper script uses `sips` for image dimension probing; everything else is portable)
- Python 3.13 + [`uv`](https://docs.astral.sh/uv/) — `brew install uv`
- A Luma API key (uni-1 images + ray-3.2 video) — **[sign up at lumalabs.ai](https://lumalabs.ai/app?utm_source=influencer&utm_medium=linkedin&utm_campaign=mrpaidsocial-ray-3_2&utm_content=faved&from_faved=true)**, then dashboard → API keys
- Claude Code with skill support

**Image skill (`uni1-image-ad`) — for the Meta upload step:**
- A Meta system user access token (see [Meta token setup](#meta-token-setup) below)

**Video film skills (`claymation-ad`, `cinematic-ad`):**
- An ElevenLabs API key (VO / SFX / music) — [elevenlabs.io](https://elevenlabs.io) → Profile → API key
- `ffmpeg` + `ffprobe` (clip assembly + audio mux) — `brew install ffmpeg`
- Node 22+ (`cinematic-ad` motion-graphic supers via HyperFrames) — `brew install node`

`ray3-video-ad` (single clips) needs only the core deps — no Meta token, ElevenLabs key, ffmpeg, or Node. `install.sh` warns (non-fatally) if ffmpeg or Node is missing; `verify.sh` reports the ElevenLabs key and video tooling as soft checks that don't fail the run.

## Quickstart

If you're using Claude Code, the simplest path is to just clone this repo, open Claude Code in the directory, and say **"help me install this."** Claude reads `CLAUDE.md` on session start and will walk you through everything below. You'll need to provide your own Luma API key and Meta system user token (Claude can't generate those for you), but it'll handle the rest.

If you're doing it manually:

```bash
# 1. Clone
git clone <this-repo> uni1-image-ad
cd uni1-image-ad

# 2. Run the install script (checks Python + uv, installs meta-ads CLI,
#    symlinks the skill, scaffolds .env). Idempotent.
./install.sh

# 3. Edit .env with your credentials
$EDITOR .env
#    LUMA_API_KEY=luma-api-...     sign up at https://lumalabs.ai/app?utm_source=influencer&utm_medium=linkedin&utm_campaign=mrpaidsocial-ray-3_2&utm_content=faved&from_faved=true → dashboard → API keys
#    ACCESS_TOKEN=...              Meta system user token; see "Meta token setup" below
#    AD_ACCOUNT_ID=act_...         after Meta auth: meta ads adaccount list
#    ELEVENLABS_API_KEY=...        only for claymation-ad / cinematic-ad (VO/SFX/music)

# 4. Verify everything is wired correctly
./verify.sh

# 5. Restart Claude Code (open a fresh session in this repo) so the skill loads
```

Then in Claude Code, say something like:

> "Make a uni-1 image ad cloning ad `120246xxxxxxxxxxx`. Use the FORBES editorial template. Brand product image is at `~/Downloads/my-product.png`."

Claude triggers `uni1-image-ad`, clones the existing ad, generates a new uni-1 image grounded on your product photo, writes new ad copy in the style of your account's top-spending ads, shows you the previews, and (after your confirmation) uploads the new ad to your account in `PAUSED` status.

Or, to **add a new template** to your prompt library from any existing ad image you've seen and liked:

> "Reverse engineer this ad into a reusable template: `~/Downloads/some-ad-i-saved.png`."

Claude triggers `image-ad-clone`, analyzes the reference, drafts a faithful prompt, validates it against the original via uni-1, generalizes it into a parameterizable template, tests it on a different brand, and appends it to your prompt library as T8 / T9 / etc. — ready for `uni1-image-ad` to use later.

## Meta token setup

The skill needs a Meta system user access token, not a personal access token.

1. Go to [Meta Business Suite](https://business.facebook.com) → Settings → Users → System Users
2. Create a new system user with the **Admin** role (name it whatever, e.g. "uni1-image-ad")
3. Click "Assign Assets" and grant access to:
   - The ad account(s) you want to manage
   - The Facebook Page(s) you want to use as the brand identity on creatives
   - Any product catalogs / Pixels you'll reference
4. Add the system user as an **App Admin** in your Meta for Developers app (App Settings → Roles → Roles)
5. Generate a token with these scopes: `business_management`, `ads_management`, `pages_show_list`, `pages_read_engagement`, `pages_manage_ads`, `catalog_management`, `read_insights`
6. Paste the token into `.env` as `ACCESS_TOKEN` (not `META_ACCESS_TOKEN` — the CLI is strict)

System user tokens last 60 days by default. When yours expires, regenerate it the same way.

## How the skill works (high-level)

Trigger phrases like *"uni-1 ad"*, *"new image ad in `<ad set>`"*, or *"upload uni-1 image as Meta ad"* fire the skill. Then it walks through:

1. **Preflight** — verify `.env` and Meta auth
2. **Clone from existing ad** — read `meta ads ad get <AD_ID>` and `meta ads creative get <CREATIVE_ID>` to extract page, ad set, link URL, CTA, and the cloned ad's tone (for reference)
3. **Prompt rewrite** — pick a starter from `references/prompt-library.md` (T1–T7) or write fresh, fill placeholders for your brand
4. **Generate** — `scripts/generate_image.py` submits to Luma in parallel (`--n` 1–5 variants), polls, downloads, validates dimensions
5. **Variant selection** — you pick which to upload
6. **Top-spending-ads research** — `scripts/top_spending_ads.py` pulls top-10 ads by spend in your account with their copy, surfaces patterns
7. **Ad copy** — Claude writes new body+headline per variant using a framework from `references/ad-copy-frameworks.md`, matched to the patterns it just read
8. **Upload** — confirmation-gated `meta ads creative create` + `meta ads ad create`. Always `PAUSED`.
9. **Audit + report** — append to `./generated/runs.jsonl`, deep-link you to Ads Manager

See [`skills/uni1-image-ad/SKILL.md`](./skills/uni1-image-ad/SKILL.md) for the full workflow.

## The prompt library

`./skills/uni1-image-ad/references/prompt-library.md` contains 7 validated parameterizable templates:

| Tag | Format | When to use |
|---|---|---|
| **T1** | Apple Notes listicle aesthetic | Sentimental "Why I switched" voice |
| **T2** | Editorial article hero | Publication-co-signed credibility (FORBES, WIRED, Vogue…) |
| **T3** | Story tweet+UGC composite | Authority quote + UGC photo for Stories/Reels |
| **T4** | Fake Google search mosaic | "Best X for Y" with publication logos |
| **T5** | Comparison table (light) | Brand vs. category competitor with feature checklist |
| **T6** | Comparison table (dark, hooky) | Stop-the-scroll dark-mode "this RUINS X" hook |
| **T7** | Sticky-note + product flatlay | Tactile UGC-style, 30-day-test reviewer voice |

Each template has placeholder variables (e.g. `{brand.name}`, `{tagline}`, `{checklist_items}`) that Claude fills based on your brand's reference image and seed prompt.

### Examples

These were generated by uni-1 from the templates above, using a single product reference photo. **AG1 is used as a worked example here — no affiliation, no endorsement; just a well-known brand to validate the templates against.** When you use the skill, swap your own brand asset in.

<table>
<tr>
  <td align="center" width="50%">
    <img src="docs/screenshots/T1-ios-notes-ag1.jpg" alt="T1 Apple Notes listicle (AG1 example)" width="100%"><br>
    <sub><b>T1 — Apple Notes listicle</b><br>Sentimental "why I switched" voice</sub>
  </td>
  <td align="center" width="50%">
    <img src="docs/screenshots/T2-editorial-ag1.jpg" alt="T2 Editorial article hero (AG1 example)" width="100%"><br>
    <sub><b>T2 — Editorial article hero</b><br>Publication-co-signed credibility</sub>
  </td>
</tr>
<tr>
  <td align="center" width="50%">
    <img src="docs/screenshots/T5-comparison-ag1.jpg" alt="T5 Comparison table (AG1 example)" width="100%"><br>
    <sub><b>T5 — Comparison table</b><br>Brand vs. category competitor</sub>
  </td>
  <td align="center" width="50%">
    <img src="docs/screenshots/T7-sticky-note-ag1.jpg" alt="T7 Sticky-note flatlay (AG1 example)" width="100%"><br>
    <sub><b>T7 — Sticky-note + product flatlay</b><br>Tactile UGC reviewer voice</sub>
  </td>
</tr>
</table>

Every example is a standalone PNG ready to upload as a Meta ad creative — no iOS chrome, no Sponsored badge, no platform UI. The `--allow-chrome` flag is the explicit escape hatch if you want screenshot-style ads.

## Hard rules

These are enforced at the script and skill level, not optional:

1. **Model is `uni-1`** — the script refuses any other model
2. **New ads are always `PAUSED`** — never `ACTIVE`
3. **No screenshot/platform chrome** in generated images — the script auto-strips iOS bars, Sponsored badges, engagement rows, link cards. `--allow-chrome` is the explicit escape hatch.
4. **Confirmation gate before every Meta CLI mutation** — you see the exact command before it runs
5. **Audit log every creative** — `./generated/runs.jsonl` is append-only

## Common pitfalls

- **Wrong Luma endpoint.** uni-1 lives at `agents.lumalabs.ai` — `api.lumalabs.ai` is the legacy Photon endpoint. Don't mix them.
- **Wrong env var name.** The Meta CLI requires `ACCESS_TOKEN`, not `META_ACCESS_TOKEN` or `FB_ACCESS_TOKEN`.
- **Wrong PyPI package.** The official Meta CLI is `meta-ads` (published by Meta). `meta-ads-cli` is a third-party typosquat — don't install it.
- **Python 3.14 on the system.** `meta-ads` only ships wheels for cp312/cp313. Use `uv tool install meta-ads --python 3.13` to get an isolated venv.
- **Aspect ratios.** Luma supports `1:1, 2:3, 9:16, 1:2, 1:3, 3:2, 16:9, 2:1, 3:1`. Common Meta ratios `4:5` and `1.91:1` are NOT supported — use `2:3` and `16:9` respectively.
- **Image dimension floor.** Meta rejects images below 1080×1080 for feed/single-image. The script warns when output is below.
- **Output URL expiry.** Luma presigned URLs expire after 1 hour. Re-poll the generation to mint a fresh URL.

## License

MIT. Use it, fork it, ship better versions.

## Credits

Built by Caleb Kruse ([Mr. Paid Social](https://mrpaidsocial.com)) in collaboration with Claude. The `Ad References/` swipe file is sourced from public ad libraries; AG1 used as a worked-example for the prompt library is just a well-known brand to test against — no affiliation.
