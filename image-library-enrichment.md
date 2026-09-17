# Skill: imageLibrary enrichment

## When to use
A user provides a list of image URLs from `dyna-nutrition.com/wp-content/uploads/...`
and wants them added to product JSONs as `imageLibrary` entries so the bot can
serve them in reply (via `utils/imageLibrary.js` → `loadImageLibrary` → LLM menu).

## Rules (hard constraints)
1. Add only to the `imageLibrary` array — never touch top-level `images`, `qa`, `reference`
2. Do NOT set `default: true` on new entries
3. `format: "infographic"` for every entry
4. `topics` must not contain the literal strings `"ad"` or `"campaign"`
5. `description` is exactly **2 sentences**, English, plain prose, **no `ad` / no `campaign` / no marketing framing** (factual only — "Square product graphic for X showing …", never "promotional" or "marketing")
6. `id` is kebab-case, product-prefixed, **no `ad` substring**
7. Preserve all existing `imageLibrary` entries (append, don't overwrite)

## URL → file mapping
| Filename pattern | Action |
|---|---|
| `ad-<product-slug>-YYYY-MM-DD-<code>-(cn|my).png` | Map `<product-slug>` → `config/products/<file-basename>.json` (often the slug matches the basename, e.g. `bionatto-plus` → `bionatto.json`); `cn` → `zh`, `my` → `ms`, no suffix → `en` |
| `ad-<product-slug>-YYYY-MM-DD-<code>.png` | Same as above, no language suffix → `en` (verify with vision; the filename-implied language can be wrong) |
| `<product-slug>_chinese_N.jpg` or `<ProductName>.png` | Filename indicates product; use vision to confirm language |
| `ChatGPT-Image-<Month>-<Day>-<Year>-<Time>-<AM/PM>.png` | Filename carries NO product or language signal — **use vision** to identify product (look for product name on box), language (text overlays), and content |
| `reswell-fb-ig-graphic-YYYY-MM-DD.png` | Unusual pattern; filename has no language suffix but vision-confirm the language |

## URL filename variant markers
Drop these from `id`:
- `ad-` prefix
- `-cn`, `-my` (replaced by `-zh`, `-ms` suffix in id)
- Campaign codes: `-A`, `-B`, `-C`, `-D`, `-E`, `-BC` (and combos)
- Variant numerals: `-1`, `-2`, `-v2`, `-v3`

When the same product has multiple same-day same-language variants, disambiguate:
- Semantic suffix: `-woman`, `-man` (ashislim)
- Numeric suffix: `-1`, `-2` (menguard)

## URL → id examples
- `ad-ashislim-plus-2026-09-05-C-cn-2.png` → `ashislim-plus-2026-09-05-zh`
- `ChatGPT-Image-Aug-7-2026-03_14_02-PM.png` (AshiSlim EN, man variant) → `ashislim-plus-2026-08-07-en-man`
- `ChatGPT-Image-Sep-1-2026-06_41_24-AM.png` (HP-FloraGut ZH) → `hp-floragut-2026-09-01-zh`
- `ad-vitamune-cdz-2026-09-16-A-1.png` (Malay despite no `-my` suffix) → `vitamune-cdz-2026-09-16-A-ms`
- `hairegain_chinese_1.jpg` (no date, ZH) → `hairegain-chinese-1`
- `Nitrovar.png` (no language marker, ZH) → `nitrovar-plus-chinese`

## Workflow
1. **Probe** all URLs with `curl -sI -o /dev/null -w "%{http_code}" "$u"`; abort on any non-200.
2. **Download** to `/tmp/<short-slug>.png` in parallel. Detect WebP masquerading as PNG with `file` — `.webp` images served with `.png` extension must be copied to a `.webp` path for `read_image` to consume.
3. **Vision pass** with `read_image` for every downloaded image. View in batches of 5–8 to keep context payload sane (~25–30 MB total typical).
4. **Author** one 2-sentence factual description per image. Reference specific elements: headline text, benefit icons + labels, ingredient tiles, product box design, scene context. For Chinese/Malay images, the English description can still mention non-English text by quoting original Chinese/Malay in unicode escapes.
5. **Author** topics: `["product", "benefits", "what is <product>", "show me the product"]` + 3–6 content-specific keywords (e.g., `["bone", "calcium", "skeletal", "magnesium", "seaweed"]` for MarineCal Plus).
6. **Edit** each product JSON. Two patterns:
   - **Fresh insert** (file has no `imageLibrary` yet): insert between `images` array and `qa` array.
   - **Append** (file already has `imageLibrary`): anchor on the closing `}` of the last entry; replace `}` with `},\n{…new entries…}`.
   Use `edit` tool with the unique closing pattern as old_string. Run edits in parallel (different files = no contention).
7. **Validate**:
   - `node -e "JSON.parse(require('fs').readFileSync('<file>','utf8'))"` → OK
   - `loadImageLibrary('<slug>')` returns expected count
   - ad/campaign scan over `id`/`description`/`topics`/`format`/`language` fields → zero matches (the `url` field is allowed to contain `ad-` from source filename)
   - Spot-check: pre-existing entries preserved, top-level `images[]` and `qa[]` unchanged
8. **Cleanup** `/tmp/*.png /tmp/*.webp` files used in this batch.

## Edge cases observed
- **WebP-as-PNG**: Server returns WebP bytes for some `.png` URLs (e.g., `ad-black-elderberry-juice-...png`, `ad-menguard-capsule-2026-09-17-A-cn.png`, `ad-vitamune-cdz-2026-09-16-A-1.png`). Detect via `file`; copy to `.webp` path for `read_image`; keep the original `.png` URL in the JSON entry.
- **Filename-implied language wrong**: Some filenames have no language suffix but the content is Chinese or Malay (e.g., `reswell-fb-ig-graphic-...` was Chinese; `ad-vitamune-cdz-2026-09-16-A-1.png` was Malay). Always vision-verify language for ambiguous URLs.
- **Existing entry with `ad-` in id**: The very first vitamune-cdz entry uses id `vitamune-cdz-ad-2026-09-16-en`. Per current rules, new ids shouldn't have `ad-` — but the existing entry is preserved (no instruction from user to rename it).
- **`official-<product>-<lang>` existing entries**: Some files (`ashislim.json`, `bionatto.json`, `menguard-capsule.json`) have legacy entries using the `official-` prefix. New entries can use a different prefix convention; both coexist.

## Validation gates (full script)
A node script to run after every batch:

```js
const fs = require('fs');
const files = { '<file>.json': '<slug>', ... };
console.log('=== JSON.parse + structure ===');
for (const [f, slug] of Object.entries(files)) {
  const d = JSON.parse(fs.readFileSync('config/products/' + f, 'utf8'));
  const lib = d.imageLibrary || [];
  console.log(f, 'JSON OK | imgs:', (d.images||[]).length, '| lib:', lib.length, '| qa:', d.qa.length);
}
console.log('=== loadImageLibrary() ===');
const lib = require('./utils/imageLibrary');
for (const slug of Object.values(files)) {
  const r = lib.loadImageLibrary(slug);
  console.log(slug, 'entries:', r.entries.length, 'langs:', r.entries.map(e => e.language).join(','), 'defaultId:', r.defaultId);
}
console.log('=== ad/campaign scan ===');
for (const [f] of Object.entries(files)) {
  const d = JSON.parse(fs.readFileSync('config/products/' + f, 'utf8'));
  for (const e of (d.imageLibrary||[])) {
    for (const k of ['description','language','format']) {
      if (typeof e[k] === 'string' && (/\bad\b/i.test(e[k]) || /\bcampaign\b/i.test(e[k]))) {
        console.log(f, e.id, k, 'has ad/campaign');
      }
    }
    for (const t of (e.topics||[])) {
      if (typeof t === 'string' && (/\bad\b/i.test(t) || /\bcampaign\b/i.test(t))) {
        console.log(f, e.id, 'topics', t);
      }
    }
  }
}
```

## Push workflow (after editing)
- `git status` to confirm only the intended product JSONs are modified
- `git add config/products/<file1> config/products/<file2> ...` (explicit paths, no `git add -A`)
- Verify no `.deploy-backup-*` directory is in `git status` output (deploy-script artifact, do not commit)
- `git -c user.name="guoloong" -c user.email="guoloong@users.noreply.github.com" commit -m "..." -m "..."` (per-commit identity, not persistent)
- `git push origin main`
- Verify `git ls-remote origin main` SHA matches local HEAD
- Push may print "fatal: unable to get credential storage lock" to stderr — benign sandbox artifact, push still succeeds if exit code 0

## Commit message template
```
Add <N> product images to <product1>, <product2>, ... imageLibrary

- <product1>: <lang1>, <lang2> entries for <url-slug>
- <product2>: ... entries for <url-slug>
- 2-sentence factual descriptions, no ad/campaign phrasing in id/topics/description
- Top-level images[], qa[] unchanged in all files
```
