# Uni1 — uni-1 image ads for Meta

A workbench for generating Meta/Facebook image ads with **Luma's uni-1** model and uploading them to a Meta ad account via the **Meta Ads CLI**. The whole project is built around uni-1 specifically — its reference-grounded generation, brand-fidelity behavior, and aspect-ratio support are baked into the prompt library and the workflow. The model lock is enforced at the tooling layer, not left to humans to remember.

> **If a user asks you to "install this" / "set this up" / "help me get started" on a fresh clone:** the install flow is two scripts plus walking them through `.env`. Run them in order:
>
> 1. **`./install.sh`** — checks Python 3.12+ and uv, installs the `meta-ads` CLI if missing, symlinks the skill into `~/.claude/skills/`, scaffolds `.env` from `.env.example`. Idempotent. Will tell the user exactly what failed if a prereq is missing.
> 2. **Walk the user through `.env`** — they need to provide `LUMA_API_KEY`, `ACCESS_TOKEN`, and `AD_ACCOUNT_ID`. You can't generate these for them. **For `LUMA_API_KEY`, always direct the user to sign up via this link → https://lumalabs.ai/app?utm_source=influencer&utm_medium=linkedin&utm_campaign=mrpaidsocial-ray-3_2&utm_content=faved&from_faved=true — then dashboard → API keys.** The Meta token in particular requires a multi-step UI flow in Meta Business Suite — see "Required env vars" + "Meta token setup" in `README.md` for the exact steps and scope list.
> 3. **`./verify.sh`** — confirms `.env` is filled, the skill is symlinked, `meta auth status` returns Authenticated, and the Luma API key works. Exits non-zero with a clear error if anything is wrong.
> 4. **Tell the user to restart Claude Code.** Skills load at session start; the new symlink won't be picked up by the current session. They need to open a fresh Claude Code session in this repo, then they can say "make a uni-1 image ad" to use the skill.

If you're an agent helping someone set this up, **read the rest of this file too** for context on credentials, hard rules, and common pitfalls.

## Repository layout

```
.
├── .env                            # local secrets, gitignored — see "Required env vars"
├── .env.example                    # template; copy to .env and fill in
├── .gitignore
├── CLAUDE.md                       # ← you are here (agent-facing setup notes)
├── README.md                       # human-facing intro + quickstart
├── install.sh                      # symlinks ./skills/* → ~/.claude/skills/
├── verify.sh                       # confirms env + skills + Meta auth + Luma API are working
├── skills/                         # Claude Code skills (versioned in this repo)
│   ├── uni1-image-ad/              # template-USING skill — generate + upload Meta ads
│   │   ├── SKILL.md
│   │   ├── scripts/                # generate_image.py, top_spending_ads.py, create_text_variant_creative.py
│   │   ├── references/             # prompt-library.md, ad-copy-frameworks.md, meta-cli-flags.md
│   │   └── state.json              # per-account cache (gitignored once populated)
│   ├── image-ad-clone/             # template-CREATING skill — reverse-engineer ads into reusable prompts
│   │   ├── SKILL.md
│   │   └── references/template-format.md
│   ├── ray3-video-ad/              # VIDEO skill — generate/edit/reframe ray-3.2 video ad creatives
│   │   ├── SKILL.md
│   │   ├── scripts/                # generate_video.py
│   │   └── references/             # video-concepts.md (concept library by ads use case)
│   ├── claymation-ad/              # STORY-FILM skill — multi-beat claymation ads end-to-end
│   │   ├── SKILL.md
│   │   └── scripts/                # claymation_pipeline.py
│   └── cinematic-ad/               # TRAILER skill — fast-cut cinematic product ads end-to-end
│       ├── SKILL.md
│       ├── scripts/                # make_audio.py (ElevenLabs layer), assemble.py (grid edit + mix)
│       └── references/             # cinematic-playbook.md, project-template.json
├── docs/luma-agents-api/           # offline Luma API reference
│   ├── quickstart.md
│   ├── model.md                    # uni-1 capabilities, params, output specs
│   ├── image-generation.md         # type: "image" — every parameter
│   ├── image-editing.md            # type: "image_edit" — source + image_ref
│   ├── video-generation.md         # type: "video" — ray-3.2 t2v/i2v, extend, loop, HDR/EXR
│   ├── video-editing.md            # type: "video_edit" — restyle/modify, strength bands, 64 keyframes
│   ├── video-reframing.md          # type: "video_reframe" — outpaint to new aspect ratio
│   ├── rate-limits.md              # 30 RPM / 10 concurrent jobs, headers, backoff
│   ├── error-handling.md           # every status code + failure_code
│   └── faq.md
├── Ad References/                  # swipe-file of real ads (used to seed prompts)
├── iterations/                     # validation runs that built the prompt library
│   ├── run_round.py                # parallel batch runner — fires N generations at once
│   ├── r1/                         # round-1 reproductions of Ad References
│   └── ag1-v2/                     # AG1 example fills, chrome-stripped (PNGs gitignored)
└── generated/                      # output PNGs + runs.jsonl audit log (gitignored)
```

`./skills/<skill-name>/` are the canonical copies. After clone, run `./install.sh` to symlink them into `~/.claude/skills/` so Claude Code picks them up. Editing the in-repo copy is the same as editing the live skill.

**Five skills, complementary:**
- `uni1-image-ad` is the **template-using** skill: trigger it to generate a uni-1 image and upload it as a paused Meta ad, optionally filling in a template from the prompt library.
- `image-ad-clone` is the **template-creating** skill: trigger it with a reference ad image and it reverse-engineers it into a parameterizable prompt that gets appended to the prompt library, ready for `uni1-image-ad` to use later.
- `ray3-video-ad` is the **video** skill: generate (`video`), restyle (`video_edit`), or aspect-ratio-outpaint (`video_reframe`) video ad creatives with **ray-3.2** on the same agents API. Cost-gated (`--dry-run` first — clips cost $0.15–$3.60), silent-output-aware, and Meta upload for video is NOT yet wired (generation/review only for now).
- `claymation-ad` is the **story-film** skill: multi-beat claymation ads end-to-end — uni-1 character/environment reference sheets → per-beat anchor stills → ray-3.2 animation → ElevenLabs VO/SFX/music on a punchy VO-driven timeline (`scripts/claymation_pipeline.py`). Needs `ELEVENLABS_API_KEY` in `.env`. Validated by "The Lab" (45.7s, 12 beats, ~$5.50 in Luma credits).
- `cinematic-ad` is the **trailer** skill: fast-cut cinematic product ads (15–30s) — beat-map grid edit, uni-1 product-grounded stills → ray-3.2 shots (camera-arc motion, no object rotation), ElevenLabs trailer VO ("David Trailer")/SFX/score via `scripts/make_audio.py`, grid assembly + frame-pinned SFX sync via `scripts/assemble.py`, and HyperFrames motion-graphic supers. Driven by one `project.json` manifest (see `references/project-template.json`). Needs `ELEVENLABS_API_KEY` + a product photo in `Reference Images/`; Node 22+ for supers. Validated by "Stitched" (15s, 7 shots, ~$4.15 all-in).

## The Luma API at a glance

The full reference is in [`docs/luma-agents-api/`](./docs/luma-agents-api/). Read it when you need detail. The 30-second version:

- **Base URL:** `https://agents.lumalabs.ai/v1`
- **Endpoints:** `POST /v1/generations` (submit) and `GET /v1/generations/{id}` (poll)
- **Auth:** `Authorization: Bearer <LUMA_API_KEY>` — keys start with `luma-api-`
- **Models:** `uni-1` for images, `ray-3.2` for video. All types go through the same endpoint: `image`, `image_edit`, `video`, `video_edit`, `video_reframe`
- **Reference images:** `image_ref` (array, up to 9 for `image`, up to 8 for `image_edit`). Each entry is `{"url": "..."}` or `{"data": "<base64>", "media_type": "image/png"}` — never both, never neither
- **Source image** (edit only): `source` — same shape as a single `image_ref` entry
- **Aspect ratios (image):** Luma supports `1:1`, `2:3`, `9:16`, `1:2`, `1:3`, `3:2`, `16:9`, `2:1`, `3:1`. (`4:5` and `1.91:1`, common Meta ratios, are NOT supported by uni-1 — use `2:3` for tall feed and `16:9` for wide.) See [model.md](./docs/luma-agents-api/model.md) for the full list.
- **Aspect ratios (video):** ray-3.2 supports `9:16`, `3:4`, `1:1`, `4:3`, `16:9`, `21:9`. Video specifics — 5s/10s clips, 540p/720p/1080p, loop, HDR/EXR, extend-chaining, edit strength bands, reframe — live in [video-generation.md](./docs/luma-agents-api/video-generation.md), [video-editing.md](./docs/luma-agents-api/video-editing.md), [video-reframing.md](./docs/luma-agents-api/video-reframing.md). ray-3.2 output is **silent** (no audio API).
- **Output URLs expire after 1 hour.** Re-poll the generation to get a fresh signed URL
- **Rate limits:** 30 RPM, 10 concurrent jobs per client. See [rate-limits.md](./docs/luma-agents-api/rate-limits.md) for backoff strategy

## Required env vars (in `.env`)

| Variable | Purpose | Where to get it |
|---|---|---|
| `LUMA_API_KEY` | Auth for `https://agents.lumalabs.ai` | **Sign up via [lumalabs.ai](https://lumalabs.ai/app?utm_source=influencer&utm_medium=linkedin&utm_campaign=mrpaidsocial-ray-3_2&utm_content=faved&from_faved=true)** → dashboard → API keys |
| `ACCESS_TOKEN` | Meta system user access token | Meta Business Suite → Settings → Users → System Users → Generate New Token. Scopes: `business_management`, `ads_management`, `pages_show_list`, `pages_read_engagement`, `pages_manage_ads`, `catalog_management`, `read_insights` |
| `AD_ACCOUNT_ID` | Default Meta ad account, prefixed `act_` | `meta ads adaccount list` after the CLI is set up |

`.env` is gitignored. Don't commit it. Don't print it to logs.

## Setup checklist (manual / agent reference)

For most cases, `./install.sh` + `./verify.sh` (see top of this file) handles the whole flow. This section is the manual breakdown for when something fails or you need to debug.

1. **Python + uv prereqs**
   ```bash
   python3 --version    # need 3.12+
   uv --version
   ```
   macOS: `brew install python@3.13 uv` if missing.

2. **Meta Ads CLI** (`meta-ads` on PyPI, published by Meta on 2026-04-29)
   ```bash
   uv tool install meta-ads --python 3.13
   meta --version       # expect 1.0.1+
   ```
   Binary lands at `~/.local/bin/meta`. **Don't** install `meta-ads-cli` — that's a third-party typosquat.

3. **Skill symlink**
   ```bash
   ./install.sh         # also handles steps 1 + 2 if needed
   ```
   This links `~/.claude/skills/uni1-image-ad` → `./skills/uni1-image-ad` so Claude Code picks up the skill.

4. **`.env` credentials**
   ```
   LUMA_API_KEY=luma-api-...
   ACCESS_TOKEN=...
   AD_ACCOUNT_ID=act_...
   ```
   Ask the user for each value; never guess or substitute placeholder keys. The variable name for the Meta token MUST be `ACCESS_TOKEN`, not `META_ACCESS_TOKEN`. Token scopes required: `business_management, ads_management, pages_show_list, pages_read_engagement, pages_manage_ads, catalog_management, read_insights`.

5. **Verify**
   ```bash
   ./verify.sh
   ```
   Or manually:
   ```bash
   meta auth status                # expect "Authenticated"
   meta ads adaccount current      # echoes AD_ACCOUNT_ID
   set -a && source .env && set +a
   curl -s -X POST https://agents.lumalabs.ai/v1/generations \
     -H "Authorization: Bearer $LUMA_API_KEY" \
     -H "Content-Type: application/json" \
     -d '{"prompt":"a corgi on a beach","model":"uni-1","type":"image","aspect_ratio":"1:1"}'
   ```
   Expect `{"id":"...","state":"queued",...}`. If you get `Invalid request: Input should be 'photon-1' or 'photon-flash-1' model`, you're hitting the legacy `api.lumalabs.ai` endpoint — uni-1 lives at `agents.lumalabs.ai`.

6. **Restart Claude Code** so the skill loads. Skills are read at session start; symlinking after the session opened doesn't update the in-memory skill list.

## Skill setup

The end-to-end workflow (generate uni-1 image → upload as Meta ad) is packaged as a Claude Code skill at `./skills/uni1-image-ad/` in this repo. To make Claude Code pick it up, symlink it into `~/.claude/skills/`:

```bash
./install.sh    # idempotent — creates ~/.claude/skills/uni1-image-ad → ./skills/uni1-image-ad
```

The skill contains:

- `SKILL.md` — the workflow Claude follows
- `scripts/generate_image.py` — stdlib-only Python helper that submits to Luma, polls, downloads PNGs (supports `--mode image|image_edit`, `--image-ref` up to 8, `--source` for edits, `--n` 1–5 variants, `--allow-chrome` escape hatch)
- `scripts/top_spending_ads.py` — pulls top-N spending ads in the user's account with their copy, used in Phase 5.5 to ground new ad copy in what's actually winning spend
- `scripts/create_text_variant_creative.py` — creates a single-image multi-text creative (1 image + up to 5 bodies + 5 titles) via direct Marketing API calls (the Meta CLI's DCO mode can't produce this exact shape)
- `references/meta-cli-flags.md` — flag reference for the `meta` CLI commands
- `references/prompt-library.md` — 7 validated parameterizable image-ad templates
- `references/ad-copy-frameworks.md` — 15 ad-copy frameworks (First Person, Statistic, Challenge Status Quo, etc.)
- `state.json` — per-account cache (page ID, ad set / cloned-ad ID); starts empty

## Hard rules — never relax

These are not preferences. They come from how the skill is built ("uni-1 only" is the whole point) and from the basic safety rule of "don't spend ad money by accident."

1. **Models are locked: `uni-1` for images, `ray-3.2` for video.** Do not substitute `photon-1`, `photon-flash-1`, `ray-2`, `ray-flash-2`, or any other model. The prompt/concept libraries, reference-grounding behavior, and aspect-ratio support all assume these models — substituting silently breaks them. Both helper scripts enforce the lock at the CLI; the SKILL.md files restate it. If you find yourself reaching for a different model "just to test," stop.
1a. **Video generations are cost-gated.** Clips cost $0.15–$3.60 each. Always `--dry-run` first, show the user the estimated cost, and get explicit approval before a paid `generate_video.py` run. Iterate at 720p/5s; 1080p/HDR only on approved finals.
2. **Ad status is `PAUSED`.** Never pass `--status ACTIVE` to `meta ads ad create`. The user reviews and launches manually in Ads Manager.
3. **Confirmation gate before every Meta CLI mutation.** Show the full command, get explicit approval, then run. This applies to `meta ads creative create` and `meta ads ad create`. Read-only commands (`list`, `get`, `current`) don't need a gate.
4. **Audit log first, then mutate.** When a creative is created, append a JSONL row to `./generated/runs.jsonl` *before* attempting the ad-create step. If ad-create fails, the orphan creative ID must be recoverable from the log.
5. **No campaign or ad-set mutations.** Attach to existing ad sets only — the skill clones from an existing ad ID rather than picking a page/adset independently. If the user has no ad to clone from, stop and tell them to create one in Ads Manager (or via `meta ads campaign create` / `meta ads adset create`) first.
6. **No screenshot/platform chrome in generated images.** The output of `generate_image.py` is the standalone ad creative — the static image the advertiser uploads. Never render iOS device chrome (status bar, home indicator), platform brand-row headers ("Sponsored"/"Saved"), post text/captions, link-card footers, engagement rows, or platform tab bars. The script auto-appends a no-chrome guard suffix to every prompt; `--allow-chrome` is the explicit escape hatch.

## The Meta side, briefly

The `meta` CLI maps cleanly to the Marketing API hierarchy: campaign → ad set → ad → creative. To upload an image as a paid ad you need:

1. A **page ID** (`object_story_spec.page_id` on a creative)
2. An **ad set ID** to attach the new ad to
3. The image as a local file path

Two paths to get those:
- **Manual:** ask the user for them, validating with `meta ads adset list` and `meta ads page list`
- **Clone-from-ad** (preferred when possible): run `meta -o json ads ad get <existing-ad-id>` to extract `adset_id` and `creative.id`, then `meta -o json ads creative get <creative-id>` to extract page ID, link URL, and CTA from `object_story_spec.link_data`. This avoids needing `BUSINESS_ID` for the page lookup.

The full flag reference for ad creation lives in the skill's `references/meta-cli-flags.md`. The two key commands:

```bash
# Create a creative from a local image
meta -o json ads creative create \
  --name "<creative-name>" \
  --image <PATH_TO_PNG> \
  --page-id <PAGE_ID> \
  [--body "..." --title "..." --link-url "..." --call-to-action SHOP_NOW]

# Attach the creative as a new ad under an existing ad set (defaults to PAUSED)
meta -o json ads ad create <ADSET_ID> \
  --name "<ad-name>" \
  --creative-id <CREATIVE_ID>
```

After a successful create, deep-link the user into Ads Manager:
```
https://business.facebook.com/adsmanager/manage/ads?act=<ACCOUNT_NUMERIC>&selected_ad_ids=<AD_ID>
```
where `ACCOUNT_NUMERIC` is `AD_ACCOUNT_ID` with the `act_` prefix stripped.

## Common pitfalls

- **Wrong Luma endpoint.** `api.lumalabs.ai` is the legacy Dream Machine endpoint (Photon). uni-1 lives at `agents.lumalabs.ai`. Do not mix them.
- **Wrong env var name for Meta.** The CLI requires `ACCESS_TOKEN`, not `META_ACCESS_TOKEN` or `FB_ACCESS_TOKEN`.
- **Wrong PyPI package.** The official Meta CLI is `meta-ads` (published by Meta on PyPI 2026-04-29). The package `meta-ads-cli` is published by a third party — do not install it.
- **Python 3.14 on the system.** `meta-ads` only ships wheels for cp312/cp313. Use `uv tool install meta-ads --python 3.13` to get an isolated venv with a compatible Python instead of `pip install meta-ads`.
- **Reference image fidelity.** Pure text-to-image (`type: "image"` with no `image_ref`) won't reproduce a label, wordmark, or specific product faithfully. When the ad must show an exact product, pass the product image via `image_ref` so uni-1 grounds on it. See [`docs/luma-agents-api/image-generation.md`](./docs/luma-agents-api/image-generation.md) §`image_ref`.
- **Output URL expiry.** Presigned URLs expire after 1 hour. Download to local disk immediately after `state: "completed"`. If you need an expired image, re-poll `GET /v1/generations/{id}` to mint a fresh URL.
- **Image dimension floor.** Meta rejects images below 1080×1080 for feed/single-image placements. The helper script warns when output dimensions are below this floor.

## When in doubt

- **Luma API behavior:** [`docs/luma-agents-api/`](./docs/luma-agents-api/) is the offline source of truth.
- **Meta CLI flags:** `meta ads <resource> --help` for any command, plus the reference at `~/.claude/skills/uni1-image-ad/references/meta-cli-flags.md`.
- **The brand contract:** uni-1 for images, ray-3.2 for video. Always.
