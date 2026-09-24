# Migrating The Digital Piranesi Off USC's Scalar Server: A Complete Guide

**Prepared for:** the project owner, to enable running this migration independently of any single collaborator's involvement.

**What this document is:** the full story of why this migration was hard, what was tried and discarded, the tool that ultimately made it possible, and a start-to-finish workflow for running the whole process again — whether that's finishing this migration, redoing it, or migrating a different book on the same install in the future.

---

## Part 1: Background — This Problem Predates This Project

This was not the first attempt to move The Digital Piranesi off USC's server. Earlier correspondence between USC and Scalar's own development team (preserved in project notes) shows the same core problem being fought years before this document was written, using different tools each time — and it's worth understanding that history, because it explains why the eventual solution had to be custom-built rather than borrowed from official guidance.

### What USC tried first, and what Scalar's own developer said about it

USC's team first ran into the built-in Import tool hanging at a "Loading existing relationships" stage — it would succeed on the first several thousand-node batches, then freeze completely. Scalar's developer (Erik) confirmed this wasn't a configuration mistake on USC's end:

> "It's possible for a project to get so large that the import tool, which needs to call up every node in order to handle new additions, can't complete its work... I imagine it may be possible to tweak your server and MySQL settings to increase allocated memory and/or other timeouts... but I can't say for sure."

This is the same root cause this project rediscovered independently, months later: **any tool that tries to load a book's entire relationship graph in one pass will fail once the book is large enough — the built-in import tool, the built-in export tool, and the bulk `instancesof` API endpoint are all the same underlying mechanism, and all three hit the same wall.**

### The earlier attempt to scrape around it — and why it failed differently than expected

USC's team also tried scraping the RDF API directly, in R, using the `jsonlite` package:
```r
api_url = "https://scalar.usc.edu/works/piranesidigitalproject/rdf/instancesof/content?start=0&results=10&format=json"
rdf = read_json(api_url)
write_json(rdf, path = "sample_rdf.json")
```
The resulting file *looked* valid — Scalar's own Import tool validated it and showed a green light — but import still hung indefinitely with the progress bar never moving. When Scalar's developer inspected the file, the actual bug was subtle and easy to miss:

> "I think the issue is that all of the 'value' and 'type' properties in your data file are arrays instead of strings. I don't believe the API returns data this way — could the arrays have been introduced at some point as the data was scraped?"

**This is a real pitfall worth remembering if this project is ever redone in a different language:** R's `jsonlite` silently wraps single values in arrays by default unless told not to (`simplifyVector`/`auto_unbox` settings). The tool this project ultimately built avoids this specific trap because it's written in Python, using the standard `json` library, which preserves the API's native `{"value": "...", "type": "literal"}` shape exactly — but it's a lesson worth carrying forward regardless of tooling: **always spot-check that scraped values are scalars, not accidentally-wrapped single-element arrays**, before trusting an import.

### What Scalar's developer said was and wasn't possible, at the time

In that same correspondence, Scalar's developer stated plainly: *"Unfortunately the ToC will need to be recreated — it's a limitation of the API that it can't export that particular data."* This was true of the `instancesof` bulk endpoint USC was using at the time. **This project found a narrower, more capable path that wasn't part of that guidance: individual node-level queries (`rdf/node/[slug]?res=path&rec=1`) can retrieve path/structure relationships that the bulk endpoint cannot surface at scale.** This is a genuine, hard-won improvement on the state of the art as it stood in earlier correspondence, and it's the foundation of the tool described in Part 2.

Two official Scalar guides were correctly identified back then and remain the right references today:
- Bulk importing via the Transfer tool's JSON import: `https://scalar.usc.edu/works/guide2/bulk-importing-data-from-a-json-file-using-the-transfer-tool?path=advanced-topics`
- Transferring physical media separately: `https://scalar.usc.edu/works/guide2/transfer-physical-media-too?path=advanced-topics`

---

## Part 2: What This Project Discovered and Built

### The core insight

Every tool that failed — the built-in export, the built-in import, the `instancesof` bulk API — shares one thing in common: each tries to walk an entire book's relationship graph inside a single request. Piranesi is large and densely cross-linked enough that this exceeds a resource ceiling on the server every time, regardless of which of these three tools is used.

**Individual, single-node API requests never hit this ceiling, no matter how large the book is, because each one only ever asks for one node's worth of data.** The path to a working export was to replace one giant request with thousands of small ones, then reassemble the results into the same shape a working export would have had.

### The exact API mechanics that make this work

- `rdf/node/[slug]?format=json` — a single node's own data (title, body content, media references).
- Adding `&res=path&rec=1` — that same node's direct children in the book's structural hierarchy. The `rec=1` parameter is required; omitting it silently returns no relationship data at all, with no error.
- Adding `&res=annotation&rec=1` — **but only when aimed at a media node's own slug, not the page that displays it.** Annotations (the numbered spatial regions on an image) attach to the media item itself. This was the single hardest-won discovery in the whole project — early attempts aimed this query at the page and got nothing back, for weeks, before testing the underlying media node directly confirmed the fix.
- The book's top-level table of contents is *not* reachable via `path` relationships at all — it's a separate `toc` node, linked from the book root, whose children are listed via `dcterms:references`. Everything one level below that uses `path` relationships as normal.

### The tool: `scalar_book_export_recreator.ipynb`

A parameterized, resumable Colab notebook that:
1. Fetches the book's own top-level scope to find its `toc` node and seed the top 5 main-menu sections.
2. Recursively crawls `path` relationships from those seeds downward, discovering every real page and media item in the book, capturing each one's full content along the way.
3. Goes back over every media item found and fetches its spatial annotations.
4. Merges everything into one file, in the same shape as a native Scalar export.
5. Filters out two node types before writing the final file — **this step is essential, not optional** (see Part 3).

Set `BOOK_SLUG` at the top and run every cell in order. Every crawl phase is safely re-runnable: it recomputes what's still missing from the accumulated data each time, so an interrupted run or an individual failed request never silently erases an entire branch of the book — it just gets retried automatically the next time the cell runs.

---

## Part 3: Hard-Won Fixes — Read This Before Re-Running Anything

These are bugs that were found and fixed during actual testing against a live destination install. Each one produced a distinctly different failure symptom, which is worth knowing if troubleshooting a future run:

| Symptom | Cause | Fix |
|---|---|---|
| Crawl produces an empty file, no errors | Crawl started at the book's `index` page expecting `path`-type children, but the top-level TOC uses `dcterms:references` instead | Seed the crawl from the `toc` node's references, not `index` |
| A node's request "succeeds" once, then all its children are permanently missing on future runs | A failed fetch was marked as "visited" anyway, so it was never retried and its children were never discovered | Only mark a node as done when the fetch actually succeeds; recompute what's still pending fresh from the accumulated data each run |
| `invalid JSON in response: Expecting value: line 1 column 1` on every request | The crawl was requesting plain page URLs (e.g. `.../about`) instead of the API endpoint (`.../rdf/node/about`) — the server returned a normal HTML page, which isn't valid JSON | Normalize any plain content URL to its `/rdf/node/...` equivalent before requesting it |
| Import hangs forever at 0% progress, blank response | The export file's very first entry was the book's own top-level record, typed `Book` — the importer can't add a whole Book as a page inside another book | Exclude any node typed `.../scalar-ns#Book` from the final export |
| Import fails cleanly with `"Invalid rdf:type value." / code 400` | The export included the book's `toc` page — every new Scalar book auto-generates its own, so a second one is rejected | Exclude any node typed `.../scalar-ns#Page` from the final export |

**Consequence of the last fix:** excluding the `toc`/Page node also removes the only record of which sections belong in the main menu, and in what order. That information doesn't exist anywhere else in the export. This is why "reassign the TOC" is a required manual step after import (Part 4, Step 6) — it isn't an oversight, it's a direct consequence of what the importer will and won't accept.

---

## Part 4: The Full Migration Workflow, Start to Finish

### Step 1 — Confirm the source book is live and reachable
Visit the book's public URL directly. If it 503s or is unreachable, nothing downstream will work; wait and retry, or escalate to USC.

### Step 2 — Run the export tool
Open `scalar_book_export_recreator.ipynb` in Colab, set `BOOK_SLUG`, and run every cell top to bottom. If any cell reports nodes still pending after hitting its round limit, just re-run that cell — it resumes automatically. The final cell downloads one JSON file with Book- and Page-typed nodes already excluded.

**Expect this to take a while — an hour is typical, and up to two hours isn't unusual** for a book this size, since the whole point of this tool is deliberately making thousands of small, individually-paced requests instead of one large one. This is normal, expected behavior, not a sign anything is stuck. It's safe to stop partway through: every node's result is written to disk immediately after that node's own fetch completes, not just at the end of a cell, so interrupting a running cell (rather than restarting the whole runtime) never loses work already gathered — the next run picks up exactly where it left off (see the resumability note in Part 2).

### Step 3 — Prepare the destination
Create a new, empty book on the destination Scalar install. Confirm the destination is running the same Scalar version as the source (check the footer of any page, or ask the install's administrator) — a schema mismatch between versions can cause missing-column errors on import that have nothing to do with the export file itself.

### Step 4 — Import the JSON
Destination book's Dashboard → **Utilities → Transfer/Import** → paste or upload the exported JSON → Import. Watch the **Content** and **Relation** progress bars — if progress never starts moving at all after a minute, stop and check the browser's DevTools Network tab for the actual `add` request's Response, rather than waiting further; a blank response with no progress is the signature of a node the importer can't process (see Part 3's table).

### Step 5 — Add physical media
Media files themselves are never included in the JSON export — only their filenames/URLs. Use `scalar_media_downloader.ipynb` (Part 7) to pull every locally-hosted file from the source book automatically, rather than tracking them down by hand.

That notebook downloads everything into a folder literally named `media`, which is deliberate and matters: **do not rename the individual files, but do check that the folder itself is named exactly `media`** before uploading (the notebook's own zip preserves this, but if you've unzipped and re-zipped it yourself, or are uploading files individually, double check the destination folder name matches). Every page already imported in Step 4 references its media using a relative path like `media/[filename]` — Scalar resolves that path relative to the book's own media directory, so the uploaded folder has to be named `media` for those existing references to resolve automatically, with no further link-editing needed. This is a different, simpler situation than the absolute old-domain URLs handled in Step 9 below — those need an explicit find-and-replace because they're hardcoded to the old domain; these relative references just need the right file sitting in the right place under the right folder name.

For the mechanics of actually uploading a media folder into a destination Scalar install once you have it prepared this way, Scalar's own guide remains the right reference: `https://scalar.usc.edu/works/guide2/transfer-physical-media-too?path=advanced-topics`.

### Step 6 — Reassign the table of contents
In the destination book's Dashboard, manually rebuild the main-menu structure (which top-level sections appear, in what order) — this data was deliberately excluded from the import (Part 3) and has to be recreated by hand. Use the source book's live site as a visual reference while doing this.

### Step 7 — Restore book-level appearance settings (custom CSS)
Excluding the `Book`-typed node (Step 4 / Part 3) also strips out the book's custom CSS — and on this book, that CSS isn't cosmetic-only: it's the reason the description field displays on the page at all. Scalar's default theme only shows a page's description in edit mode, not on the live page; the source book overrides that with an explicit rule. Without this CSS on the destination, every page will otherwise look and behave correctly but silently drop its visible description text.

In the destination book's Dashboard, find the book-level design/appearance settings (labeled "Custom CSS" or "Custom Style" depending on Scalar version) and paste in the source book's full block:

```css
#scalarheader { 
    background-color: scarlet;
font-family: Arial, Serif;} 
/* Text content area to full width */
.body_copy {max-width:none;}
/* Display description below title */
header > h1 ~ [property="dcterms:description"] {display:block !important; color:#888888; margin:0rem 2rem 4rem 2rem;}
@media only screen and (min-width : 500px) {
  header > h1 ~ [property="dcterms:description"] {margin:-3rem 7.2rem 4rem 7.2rem;}
}
/*Text content font uniformity */
.body_font{ font-family: 'Lato', Arial, sans-serif !important; font-size: 16px !important;}
```

If the source book is ever redesigned in the future, re-check its `scalar:customStyle` field for changes before repeating this migration — this block is a snapshot as of this project, not something that updates itself.

### Step 8 — Add users and assign the credited author
Add the necessary collaborators to the destination book and designate whichever author's name should display publicly on the book, via the Dashboard's user-management screen.

### Step 9 — Fix internal links and media references pointing at the old URL
Even after media files and pages are both in place, some internal hyperlinks and media embeds inside the book's own page content are still hardcoded to the *old* project's absolute URL rather than the new one — this happens whenever a link was created by pasting a full URL rather than using Scalar's relative-link picker. This is a different problem than Step 5's media files: those use relative paths and resolve automatically once the `media` folder is in place; this step is specifically about absolute links baked into page text as literal `https://old-domain/...` strings.

The original approach to this — editing every affected page's content by hand, one link at a time — was tried once on this exact project and abandoned as impractical on a book this size. The fix below is Scalar's own official documented method for this exact situation (`https://scalar.usc.edu/works/guide2/transfer-physical-media-too?path=advanced-topics`), reproduced here in full so this guide is self-contained — the link is included only as the original authoritative source.

**This method is scoped to one specific book by its `book_id`, which matters on a shared destination install** (Part 6): every book on one Scalar install shares the same tables, so scoping every query to `book_id = @book_id` guarantees this can never touch another book's content, even by coincidence.

1. **Back up the destination database first.** In phpMyAdmin, select the destination database → **Export** tab → use the default "Quick" export method → **Go**. Save the resulting `.sql` file somewhere safe before touching anything.

2. **Find the destination book's `book_id`.** View the page source (not the rendered page) of any page in the destination book, and search for `book_id` or `book-id` — it appears as either `<link id="book_id" href="[number]" />` or `<span class="metadata" inert id="book-id">[number]</span>`. Note that number.

3. **Confirm the exact old and new base URLs.** For this project:
   - Old: `https://scalar.usc.edu/works/piranesidigitalproject`
   - New: your destination book's actual URL (e.g. `https://jmdps.com/scalar/dp-test-1`)

   Get these exactly right, including protocol and no trailing slash.

4. **Open the SQL tab** on the destination database in phpMyAdmin (the same place you ran the `ALTER TABLE` command earlier in this project, if you did that step), and run the following as one batch — replace the placeholder values in the first three lines with your own from steps 2–3:

   ```sql
   SET @book_id = 1;
   SET @old_url = "https://scalar.usc.edu/works/piranesidigitalproject";
   SET @new_url = "https://jmdps.com/scalar/dp-test-1";

   UPDATE
       scalar_db_versions
   INNER JOIN
       scalar_db_content ON scalar_db_versions.content_id = scalar_db_content.content_id
                          AND scalar_db_content.book_id = @book_id
   SET
       scalar_db_versions.url = REPLACE(scalar_db_versions.url, @old_url, @new_url),
       scalar_db_versions.content = REPLACE(scalar_db_versions.content, @old_url, @new_url);

   UPDATE
       scalar_db_content
   SET
       scalar_db_content.background = REPLACE(scalar_db_content.background, @old_url, @new_url),
       scalar_db_content.banner = REPLACE(scalar_db_content.banner, @old_url, @new_url),
       scalar_db_content.thumbnail = REPLACE(scalar_db_content.thumbnail, @old_url, @new_url)
   WHERE
       scalar_db_content.book_id = @book_id;
   ```

   The first `UPDATE` fixes both a version's own URL field and any inline links/embeds inside its actual page body (`content`), scoped to this book via the join. The second fixes background/banner/thumbnail image references at the content level.

5. **Also check the book-level custom CSS you pasted in manually during Step 7**, since a background-image URL declared there could independently reference the old domain too — the two queries above don't touch it, since it lives on the book-level record this migration deliberately excludes from JSON import (Part 3). Open that field directly in the Dashboard and confirm it uses either a relative `media/...` path or the new domain.

6. **Verify the fix** by running this afterward — it should return zero rows:
   ```sql
   SELECT v.version_id, v.title
   FROM scalar_db_versions v
   INNER JOIN scalar_db_content c ON v.content_id = c.content_id AND c.book_id = @book_id
   WHERE v.content LIKE CONCAT('%', @old_url, '%')
      OR v.url LIKE CONCAT('%', @old_url, '%');
   ```
   Then spot-check a few pages live, clicking through their internal links to confirm they land on the new site rather than the old one.

### Step 10 — Spot-check the result
Compare the destination book's table of contents and a sample of annotated images directly against the live source book. Pay particular attention to any page that was flagged as an exception during the original export (Part 5).

---

## Part 5: Known Exceptions and Permanent Limitations

- **One page failed during the original export with a server-side PHP error** (`Undefined property: stdClass::$user_id`), traced to the fact that the page is hidden/unpublished on the source book. This was accepted as a legitimate exclusion, not something to keep chasing.
- **A large fraction of a book's raw content rows are not real pages.** In this book, roughly 4,250 of ~6,900 non-media content rows were single-letter-titled annotation label fragments (e.g. a numbered key "4" whose body text is a caption), not standalone reader-facing pages. These are correctly captured as annotation data during the crawl, not as pages — they should never be imported as if they were pages in their own right.
- **Media files and the top-level table of contents are permanent gaps in any JSON-based import**, not bugs to keep trying to fix — Scalar's own documentation confirms media transfer has always been a separate, manual step, and the TOC exclusion is a direct, unavoidable consequence of the importer's validation rules (Part 3).
- **This method requires the source book's live API to be reachable.** If the source site is ever taken offline before a migration is attempted, this whole approach is unavailable, and any previously-saved raw SQL exports would become the only remaining option (with the multi-tenant-database cautions that come with that path — see below).
- **The index page may display an unexpected "This path is a commentary on the book, written by [author] on [date]" banner** after import, even though the source book's own index page shows no such banner. This is not an import error — the underlying data (an editorial `category` field set to `commentary` on the index page's current version) is genuinely present on the source book too; it was confirmed directly in this project's very first exploratory API query, long before any migration tooling existed. What differs between the two installs is only how visibly each one's theme renders that same field — likely a template/theme mismatch between the source's `cantaloupe` theme and whatever the destination book defaulted to, though this wasn't conclusively isolated. Rather than chase the theme difference, the direct fix is simpler: in the destination book's Dashboard, open the index page's edit view and change its page type away from "Commentary" to ordinary book content. In this project, this affected only the index page — nothing else in the book carried the same `category: commentary` metadata — so a systematic scan for other instances wasn't necessary here, though one is easy to write if a future migration needs it (search the exported JSON for any version node with a `category` value of `commentary`).

## Part 6: If the Destination Has Other Books Already

Everything above works identically whether the destination is a brand-new empty install or one with other books on it already, **except for one category of risk that only applies to a shared destination**: Scalar stores all books on one install in the same database tables, distinguished only by an internal book ID. A raw SQL dump-and-restore (as opposed to the JSON import method this guide uses) would risk wiping every other book on a shared destination, since it typically drops and recreates entire tables. **The JSON import method in this guide does not have this risk** — it only ever adds content, and was never going to touch anything belonging to another book on the same install.

---

## Part 7: Reusable Tools Produced by This Project

- **`scalar_book_export_recreator.ipynb`** — the parameterized export notebook described in Part 2, with all fixes from Part 3 already applied.
- **`scalar_media_downloader.ipynb`** — downloads every locally-hosted media file referenced by an exported book (used in Step 5). Deliberately excludes externally-hosted media (e.g. plates hosted on archive.org) — those are just links, and keep working wherever the book ends up. Works from either the full unfiltered graph or the final filtered export, and also re-fetches the book's own top-level scope directly, since that's where a book's own background/thumbnail images often live.
- This document — intended to be kept alongside both notebooks as their operating manual.

All three should be reusable, largely as-is, for migrating any other sufficiently large Scalar book off USC's install in the future, or for re-running this same migration if it ever needs to be redone.
