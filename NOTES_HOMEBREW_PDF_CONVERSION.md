# Converting a Sourcebook PDF into 5etools Homebrew JSON

Architectural guidance for turning a full 5e-compatible sourcebook (delivered as a
PDF, usually in page-range chunks) into a single 5etools homebrew data file. Written
after converting *Star Shaman's Song of Planegea* (`homebrew/Star Shaman's Song of
Planegea.json`); use that file as the worked reference example.

---

## 1. The mental model

A homebrew file is **two parallel products in one JSON document**, and every page of
source feeds one or both:

1. **The renderable book** — `book` (the table of contents / metadata) plus `bookData`
   (the actual chapter prose). This is what a user reads on `book.html`. It holds
   *all* narrative: flavor text, sidebars, quotes, tables, lore.
2. **The mechanical database** — the top-level arrays (`spell`, `item`, `monster`,
   `race`, `subrace`, `subclass`, `subclassFeature`, `background`, `baseitem`,
   `optionalfeature`). These are what appears on `spells.html`, `bestiary.html`,
   `items.html`, etc., with full stat blocks, filters, and cross-linking.

**The core division of labor:** narrative/flavor goes in `bookData`; mechanical stat
blocks go in the dedicated arrays. For content that has both (a monster's flavor
paragraphs *and* its stat block), put the flavor in `bookData` and the stat block in
the array — **do not duplicate the full stat block text into `bookData`.** This
mirrors how the official Spells chapter works (bookData intro + separate `spell`
array) and keeps the file lean. Earlier chapters (Backgrounds, Equipment) put fuller
prose in bookData because those had little separate mechanical data; use judgment, but
default to "flavor in book, mechanics in array."

---

## 2. Workflow per chunk

The user uploads the PDF in sequential page ranges (e.g. "pages 337–345"). For each:

1. **Extract text.** `pdftotext -layout <file.pdf> <out.txt>`. The `-layout` flag is
   **mandatory for two-column pages** (stat blocks side by side, multi-column tables,
   god/faction lists) — without it the reading order scrambles across columns. Plain
   `pdftotext` is fine for single-column prose but `-layout` is a safe default.
2. **Read the extraction fully** before writing anything. Two-column stat blocks
   interleave in the text file — read carefully to keep each block's lines together.
3. **Write a disposable Python script** (see §5) that loads the JSON, appends the new
   `bookData` entries and array entries, and writes it back.
4. **Validate** (see §6).
5. **Commit and push**, and in the commit message **state exactly where the chunk cut
   off** so the next chunk's continuation point is unambiguous. Note any content that
   has flavor-only-so-far (stat block presumably in a later chunk).
6. **Flag page-range gaps.** If a chunk starts at 337 but the last ended at 327, say so
   — don't silently proceed. (In this project pages 328–336 really were a missing
   "Part16" the user later supplied.)

Keep the work **additive and idempotent-ish**: each script appends to the existing file;
never rewrite from scratch.

---

## 3. Schema conventions (as used, internally consistent)

Every entry gets `"source": "<ABBREV>"` (e.g. `"SSSOP"`) and a `"page"` number.

### bookData entry types
- `entries` — a named or unnamed block: `{"type":"entries","name":"...","entries":[...]}`
- `inset` — sidebar box (`{"type":"inset","name":"...","entries":[...]}`)
- `quote` — `{"type":"quote","entries":[...],"by":"..."}` (`by` optional)
- `table` — `caption`, `colLabels`, `colStyles` (e.g. `"col-2 text-center"`, `"col-10"`),
  `rows` (array of arrays)
- `list` — `{"type":"list","style":"list"|"list-hang-notitle","items":[...]}`; items may
  be strings or `{"type":"item","name":"X.","entry":"..."}`
- `flowchart` — **uses `blocks`, not `entries`**, each an `{"type":"flowBlock",...}`
- Each top-level **chapter** wrapper is `{"type":"section","name":"...","page":N,"entries":[...]}`.

### `book[0].contents` (the TOC) — critical invariant
The renderer maps `contents[i]` to `bookData.data[i]` **positionally**. Therefore:
- `contents.length` MUST equal `bookData.data.length`.
- `contents[i].name` should match `bookData.data[i].name` (browser tab title vs in-page
  heading; a mismatch is a visible bug).
- Each chapter's `headers` array should list its top-level named sections (used for the
  collapsible TOC sub-nav). Keep it current when you add sections.
- Do **not** leave a TOC entry with no matching `bookData` chapter (an orphan "Index"
  entry caused blank-page navigation here).

### spell
`level` (int 0–9), `school` (single-letter code: `A`bjuration `V`(evocation) `E`nchantment
`I`llusion `D`ivination `N`ecromancy `T`ransmutation `C`onjuration `P`sionic), `time`,
`range`, `components` (`v`/`s`/`m`), `duration`, `entries`, optional `entriesHigherLevel`,
and `classes.fromClassList` = `[{"name":"Cleric","source":"PHB"}, ...]`. In-book spell
lists reference each spell with `{@spell Name|SSSOP}`.

### monster (stat block)
- `size` array of size codes (`T`/`S`/`M`/`L`/`H`/`G`).
- `type` either a string (`"beast"`) or `{"type":"beast","tags":["defiant"]}`.
- `alignment` array of codes (`["N","G"]`, `["C","E"]`); omit/`None` for unaligned.
- `ac` array of `{"ac":N,"from":[...]}` or `{"ac":N,"condition":"while prone"}`.
- `hp` = `{"average":N,"formula":"NdM + K"}`.
- `speed` object: `walk`/`fly`/`swim`/`climb`/`burrow` + `canHover:true` for hover.
- **Ability scores (`str`…`cha`) are RAW SCORES 1–30, not modifiers.** This was the most
  common self-inflicted bug — double check.
- `save`/`skill` objects map ability/skill → modifier string (`"+5"`).
- `resist`/`immune`/`conditionImmune` arrays; a nonmagical-weapon clause is an object
  mixed into the array: `{"resist":[...],"note":"from nonmagical weapons"}`.
- `senses` array of strings; `passive` number; `languages` array; `cr` **string**.
- `trait`/`action`/`bonus`/`reaction`/`legendary` = arrays of `{"name":...,"entries":[...]}`.
- `legendaryHeader` = one intro string in an array.
- **Only real stat blocks belong here.** "Template" entries that just describe changes to
  another creature's block (no ac/hp/speed) are prose — put them in `bookData`, not the
  `monster` array, or they break the bestiary renderer.

### item / baseitem
- Magic items: `rarity` (must be a valid enum: `common`/`uncommon`/`rare`/`very rare`/
  `legendary`/`artifact`/`varies`/`none`/`unknown` — a template with per-instance rarity
  uses `"varies"`, **not** free text), `reqAttune`, `wondrous`/`weapon`/`armor`,
  `type` (`RG` ring, `WD` wand, `M`/`R` weapon melee/ranged), `entries`.
- `baseitem`: `type`, `weaponCategory` (`simple`/`martial`).

### race / subrace
- `race`: `size` array, `speed`, `ability` array (`[{"wis":2,"cha":1}]`), `age`,
  `alignment`, `languageProficiencies`, `entries`. **`age` is a structured object**
  (`{"mature":20,"max":100}`) — use **`{}`** when the source gives no fixed lifespan
  (variable/open-ended), matching Starling and Half-Ooze here. The prose "Age" trait in
  `entries[]` is separate and additional.
- `subrace`: `raceName` + `raceSource` must reference an existing `race.name`; carries its
  own `ability`/`entries`.

### subclass / subclassFeature
- `subclass`: `className` + `classSource` (usually `"PHB"`), `shortName`, `name`.
- `subclassFeature`: `subclassShortName` + `className` + `classSource` + `level` must
  match a real subclass. One feature per level-grant.

### optionalfeature / background
- `optionalfeature`: `featureType` array (`["OTH"]`, `["EI"]` eldritch invocation, etc.).
- `background`: `skillProficiencies` + `entries`.

---

## 4. Renderer gotchas that cause silent blank pages

These crash the client-side renderer (a JS exception aborts the *whole chapter/entry*,
so the symptom is a blank page, not a partial one). All were hit and fixed here:

- **`{@filter ...}` clauses use `=`, not `:`** — `{@filter x|spells.html|source=SSSOP}`.
  A colon makes `fVals` undefined and throws in `getFilterSubhashes`.
- **`flowchart` entries use `blocks`, not `entries`** — the wrong key throws on
  `entry.blocks.length`.
- Tag syntax generally: `{@spell Name|SOURCE}`, `{@i italic}`, `{@b bold}`,
  `{@damage 1d6}`, `{@dice 1d4}`, `{@creature Name|SOURCE}`, `{@item Name|SOURCE}`.
  A malformed tag can abort rendering — prefer copying a known-good tag's exact shape.

---

## 5. The Python scripting pattern

Write each chunk's script to the scratchpad dir, run once, leave it (disposable). Shape:

```python
import json
path = "homebrew/<Book>.json"
d = json.load(open(path, encoding="utf-8"))

# small helpers: named(), table(), li(), mon(), mk_item() ...
# locate the target: chapter section in d["bookData"][0]["data"], then nested entries
# append bookData flavor; d["monster"].extend([...]) etc.

with open(path, "w", encoding="utf-8") as f:
    json.dump(d, f, indent="\t", ensure_ascii=False)   # TABS, not spaces
    f.write("\n")                                        # trailing newline
```

- **Serialize with `indent="\t"` and `ensure_ascii=False`** (matches the repo's format;
  keeps curly quotes/em-dashes as real characters). End with a newline.
- **Never leave a stray `open(path,"w")` anywhere but the single final write** — an
  accidental early open-for-write truncates the whole file. (Caught this once mid-draft.)
- For very large chunks, split into multiple scripts writing intermediate JSON to the
  scratchpad, then a final merge script does the one real write.
- A large `deletions` count in `git diff --stat` on a pure-add is usually just JSON key
  reordering noise from inserting a new top-level key — verify array counts didn't drop
  rather than panicking.

---

## 6. Validation protocol (run after every chunk, and a full pass at the end)

**Structural (fast, every time):**
```python
python3 -c "import json; d=json.load(open('homebrew/<Book>.json')); \
  print({k:len(d[k]) for k in ['race','subrace','subclass','subclassFeature', \
  'background','spell','baseitem','optionalfeature','item','monster']}, \
  len(d['bookData'][0]['data']))"
```
Confirm no array *shrank* vs the previous commit (guards against accidental data loss).
Then check: no duplicate names per array; every `subrace.raceName`/
`subclassFeature.subclassShortName` resolves; every monster has `ac`/`hp`/`speed`/`cr`;
ability scores are 1–30; `book.contents` length == `bookData.data` length and names align.

**Behavioral (the check that actually catches the blank-page bugs — do this at least
once at the end):** load the pages in a headless browser and render each chapter through
the real renderer. Chromium is preinstalled (`/opt/pw-browsers/chromium`,
`PLAYWRIGHT_BROWSERS_PATH=/opt/pw-browsers`). Serve the repo (`python3 -m http.server`)
and, for each chapter, call `Renderer.get().recursiveRender(entry, [], {depth:1})` in a
`try/catch` to find the *exact* entry that throws. A structurally-valid JSON can still
crash the renderer (see §4) — only a render pass finds those.

---

## 7. Git workflow

- Develop on the assigned feature branch; commit per chunk with a descriptive message
  documenting what was added and the exact cut-off page.
- `git add "homebrew/<Book>.json"` then `git push -u origin <branch>`.
- Register the file once in `homebrew/index.json` under `toImport`.
- Final delivery to `main`: if the feature branch is a clean fast-forward of `main`
  (`git merge-base --is-ancestor origin/main HEAD`), fast-forward `main` to it. Only push
  to the default branch on explicit user request.

---

## 8. Order of operations for a brand-new book

1. Seed `_meta.sources`, `book[0]` (name/id/source/group/author/contents skeleton),
   `bookData[0]` (matching chapter skeleton), and register in `homebrew/index.json`.
2. Ingest chapter by chapter, chunk by chunk, per §2.
3. Keep `book.contents` and `bookData.data` in lockstep as chapters/sections appear.
4. Run the full structural + behavioral validation pass at the end (§6) and fix any
   renderer gotchas (§4) before final delivery.
