# Migrating a Large Scalar Book to a New Install

A complete, self-contained guide (plus two Colab notebooks) for moving a large, densely-linked [Scalar](https://scalar.me/) book off one install and onto another — written for the specific case where the book is too big for Scalar's own built-in export/import tools to handle at all.

**If your book is small enough that the built-in Transfer tool just works, you don't need any of this** — use that instead. This repo exists for the case where it doesn't: where the built-in export hangs indefinitely, or the bulk API endpoint 503s, because the book has grown large enough that no single request can fetch its whole relationship graph anymore.

## Contents

- [Part 1: Why This Is Hard](#part-1-why-this-is-hard)
- [Part 2: How This Approach Works](#part-2-how-this-approach-works)
- [Part 3: Known Bugs — Read This First](#part-3-known-bugs--read-this-first)
- [Part 4: The Full Migration Workflow](#part-4-the-full-migration-workflow-start-to-finish)
- [Part 5: Known Limitations](#part-5-known-limitations)
- [Part 6: Shared (Multi-Book) Destinations](#part-6-if-the-destination-has-other-books-already)
- [Part 7: What's in This Repo](#part-7-whats-in-this-repo)

---

## Part 1: Why This Is Hard

Scalar's built-in Import/Export tool and its bulk `instancesof` API endpoint all share one thing in common: each tries to walk a book's **entire** relationship graph — every page, every piece of media, every annotation, every cross-reference — inside a single request. That works fine for a small or moderate book. It breaks down once a book is large and densely cross-linked enough, because the request itself exceeds a resource ceiling on the server (memory, execution time, or both) and simply fails, typically as an HTTP 503 or a progress bar that hangs forever with no error message at all.

This isn't a bug specific to one install or one book — it's inherent to how those three tools work, and it's been reported by real Scalar users well before this repo existed. One account, from a project that hit exactly this wall, is worth quoting because it confirms the mechanism directly (paraphrased from correspondence with Scalar's own development team at the time):

> "It's possible for a project to get so large that the import tool, which needs to call up every node in order to handle new additions, can't complete its work... it may be possible to tweak server and MySQL settings to increase allocated memory and other timeouts, but there's no guarantee."

That same project also tried scraping the RDF API directly in R, using the `jsonlite` package, to work around the timeout. The resulting file *looked* valid — Scalar's own import tool validated it and showed a green light — but import still hung indefinitely. The actual bug was subtle: R's `jsonlite` silently wraps single values in arrays by default (`{"value": ["x"]}` instead of `{"value": "x"}`), and Scalar's API doesn't return data that way. **If you ever build a scraper for this in a language other than the one used in this repo, check that your parsed values come out as plain strings, not accidentally-wrapped single-element arrays** — this exact mistake produced a file that looked correct and still failed silently.

At the time, Scalar's own guidance was that a book's table-of-contents structure "will need to be recreated by hand — it's a limitation of the API that it can't export that particular data." **This repo's approach found a narrower, more capable path that guidance didn't cover:** individual node-level API queries can retrieve structure and annotation data that the bulk endpoint cannot surface at scale (see Part 2). This is a genuine improvement on where official guidance stood previously, not just a workaround for the timeout itself.

Two official Scalar guides are the right reference for parts of this process that this repo doesn't try to replace:
- Bulk importing via the Transfer tool's JSON import: `https://scalar.usc.edu/works/guide2/bulk-importing-data-from-a-json-file-using-the-transfer-tool?path=advanced-topics`
- Transferring physical media separately: `https://scalar.usc.edu/works/guide2/transfer-physical-media-too?path=advanced-topics`

---

## Part 2: How This Approach Works

### The core insight

Every tool that fails at scale — built-in export, built-in import, the `instancesof` bulk API — fails because it tries to fetch a book's whole relationship graph in one request. **A request for a single node's data never hits that ceiling, no matter how large the book is**, because it only ever asks for one node's worth of information. The fix is to replace one giant request with thousands of small ones, then reassemble the results into the same shape a working export would have had.

### The exact API mechanics this depends on

- `rdf/node/[slug]?format=json` — a single node's own data (title, body content, media references).
- Adding `&res=path&rec=1` — that same node's direct children in the book's structural hierarchy. **The `rec=1` parameter is required** — omitting it silently returns no relationship data at all, with no error to signal why.
- Adding `&res=annotation&rec=1` — **but only when aimed at a media item's own slug, not the page that displays it.** Spatial annotations (numbered regions on an image) attach to the media node itself, not to whatever page embeds it. This is the single easiest thing to get wrong with this API.
- A book's top-level table of contents is *not* reachable via `path` relationships — it's a separate `toc` node, linked from the book root, whose children are listed via `dcterms:references` instead. Everything one level below that uses `path` relationships normally.
- Some media items are **only** ever referenced by an inline embed inside another page's own HTML body (`<a name="scalar-inline-media" resource="[slug]">`), with no `path` relationship of their own at all. A path-only crawl will silently miss these; scanning every page's own content for this exact tag pattern is the only way to catch them.

### The tool

`scalar_book_export_recreator.ipynb` (Part 7) is a parameterized, resumable Colab notebook that:
1. Fetches the book's own top-level scope to find its `toc` node and seed the top-level sections.
2. Recursively crawls `path` relationships from those seeds downward, discovering every real page and media item, capturing each one's full content along the way.
3. Scans every discovered page's content for inline media embeds that a path-only crawl would otherwise miss.
4. Goes back over every media item found and fetches its spatial annotations.
5. Merges everything into one file, in the same shape as a native Scalar export.
6. Filters out two node types before writing the final file — **this step is essential, not optional** (Part 3).

Every crawl phase is safely re-runnable: it recomputes what's still missing from the accumulated data each time, so an interrupted run or an individual failed request never silently erases an entire branch of the book — it just gets retried automatically the next time the cell runs. **Expect a real run to take an hour, sometimes two**, for a book large enough that this approach is necessary at all — that's inherent to deliberately making thousands of small, paced requests instead of one large one, not a sign anything is stuck.

---

## Part 3: Known Bugs — Read This First

These are real bugs, found and fixed during actual testing against a live destination install. Each produced a distinctly different failure symptom, worth recognizing if you hit something similar:

| Symptom | Cause | Fix |
|---|---|---|
| Crawl produces an empty file, no errors | Crawl started at the book's `index` page expecting `path`-type children, but the top-level TOC uses `dcterms:references` instead | Seed the crawl from the `toc` node's references, not `index` |
| A node's request "succeeds" once, then all its children are permanently missing on future runs | A failed fetch was marked "visited" anyway, so it was never retried and its children were never discovered | Only mark a node done when the fetch actually succeeds; recompute what's still pending fresh from the accumulated data each run |
| `invalid JSON in response: Expecting value: line 1 column 1` on every request | The crawl was requesting plain page URLs (e.g. `.../about`) instead of the API endpoint (`.../rdf/node/about`) — the server returned a normal HTML page, which isn't valid JSON | Normalize any plain content URL to its `/rdf/node/...` equivalent before requesting it |
| Import hangs forever at 0% progress, blank response | The export file's very first entry was the book's own top-level record, typed `Book` — the importer can't add a whole Book as a page inside another book | Exclude any node typed `.../scalar-ns#Book` from the final export |
| Import fails cleanly with `"Invalid rdf:type value." / code 400` | The export included the book's `toc` page — every new Scalar book auto-generates its own, so a second one is rejected | Exclude any node typed `.../scalar-ns#Page` from the final export |
| A page displays without an image/annotation that works fine on the source | The media item is only referenced via an inline embed, with no `path` relationship of its own — a path-only crawl never discovers it | Also scan every page's HTML content for `scalar-inline-media` embeds and crawl those slugs too |

**Consequence of the `Book`/`Page` exclusion fix:** it also removes the only record of which sections belong in the book's main menu, and in what order — that information doesn't exist anywhere else in the export. This is why reassigning the table of contents is a required manual step after import (Part 4, Step 6), not an oversight.

---

## Part 4: The Full Migration Workflow, Start to Finish

### Step 1 — Confirm the source book is live and reachable
Visit the book's public URL directly. If it 503s or is unreachable, nothing downstream will work; wait and retry, or contact whoever administers the source install.

### Step 2 — Run the export tool
Open `scalar_book_export_recreator.ipynb` in Colab, set `BOOK_SLUG` and `INSTALL_BASE` at the top to match your own book, and run every cell top to bottom. If any cell reports nodes still pending after hitting its round limit, just re-run that cell — it resumes automatically. The final cell downloads one JSON file with `Book`- and `Page`-typed nodes already excluded.

It's safe to stop partway through: every node's result is written to disk immediately after that node's own fetch completes, not just at the end of a cell, so interrupting a running cell (rather than restarting the whole runtime) never loses work already gathered.

### Step 3 — Prepare the destination
Create a new, empty book on the destination Scalar install. Confirm the destination is running the same Scalar version as the source (check the footer of any page, or ask the install's administrator) — a schema mismatch between versions can cause missing-column errors on import that have nothing to do with the export file itself.

### Step 4 — Import the JSON
Destination book's Dashboard → **Utilities → Transfer/Import** → paste or upload the exported JSON → Import. Watch the **Content** and **Relation** progress bars — if progress never starts moving at all after a minute, stop and check the browser's DevTools Network tab for the actual `add` request's Response, rather than waiting further; a blank response with no progress is the signature of a node the importer can't process (Part 3).

### Step 5 — Add physical media
Media files themselves are never included in the JSON export — only their filenames/URLs. Use `scalar_media_downloader.ipynb` (Part 7) to pull every locally-hosted file from the source book automatically.

That notebook downloads everything into a folder literally named `media`, which matters: **do not rename the individual files, but do make sure the folder itself is named exactly `media`** before uploading. Every page already imported in Step 4 references its media using a relative path like `media/[filename]`, resolved by Scalar relative to the book's own media directory — so the uploaded folder has to be named `media` for those existing references to resolve automatically, with no further link-editing needed. This is a simpler situation than the absolute old-domain URLs handled in Step 9 below — those need an explicit find-and-replace because they're hardcoded to the old domain; these relative references just need the right file sitting in the right place under the right folder name.

For the mechanics of uploading a prepared media folder into a destination Scalar install, Scalar's own guide remains the right reference: `https://scalar.usc.edu/works/guide2/transfer-physical-media-too?path=advanced-topics`.

### Step 6 — Reassign the table of contents
In the destination book's Dashboard, manually rebuild the main-menu structure (which top-level sections appear, in what order) — this data was deliberately excluded from the import (Part 3) and has to be recreated by hand. Use the source book's live site as a visual reference while doing this.

### Step 7 — Restore book-level appearance settings (custom CSS)
Excluding the `Book`-typed node (Step 4 / Part 3) also strips out the book's custom CSS. Depending on what that CSS actually does on your source book, this can be purely cosmetic — or it can affect real functionality; on one real migration, it turned out to be the only reason the page-description field displayed at all, since Scalar's default theme only shows descriptions in edit mode otherwise.

Before you import, check your source book's own custom CSS (Dashboard → book-level design/appearance settings, usually labeled "Custom CSS" or "Custom Style") and save a copy of it somewhere. After import, paste that same block into the destination book's equivalent field. If the source book is ever redesigned in the future, re-check this field for changes before repeating the migration — it's a point-in-time snapshot, not something that stays in sync automatically.

### Step 8 — Add users and assign the credited author
Add the necessary collaborators to the destination book and designate whichever author's name should display publicly on the book, via the Dashboard's user-management screen.

### Step 9 — Fix internal links and media references pointing at the old URL
Even after media files and pages are both in place, some internal hyperlinks and media embeds inside the book's own page content are still hardcoded to the *old* project's absolute URL rather than the new one — this happens whenever a link was created by pasting a full URL rather than using Scalar's relative-link picker. This is different from Step 5's media files: those use relative paths and resolve automatically once the `media` folder is in place; this step is specifically about absolute links baked into page text as literal `https://old-domain/...` strings.

Editing every affected page by hand is impractical on a book large enough to need this repo in the first place. The fix below is Scalar's own official documented method for exactly this situation (`https://scalar.usc.edu/works/guide2/transfer-physical-media-too?path=advanced-topics`), reproduced here in full so this guide is self-contained.

**This method is scoped to one specific book by its `book_id`, which matters on a shared destination install** (Part 6): every book on one Scalar install shares the same tables, so scoping every query to `book_id = @book_id` guarantees this can never touch another book's content, even by coincidence.

1. **Back up the destination database first.** In phpMyAdmin, select the destination database → **Export** tab → use the default "Quick" export method → **Go**. Save the resulting `.sql` file somewhere safe before touching anything.

2. **Find the destination book's `book_id`.** View the page source (not the rendered page) of any page in the destination book, and search for `book_id` or `book-id` — it appears as either `<link id="book_id" href="[number]" />` or `<span class="metadata" inert id="book-id">[number]</span>`. Note that number.

3. **Confirm the exact old and new base URLs**, including protocol and no trailing slash — for example:
   - Old: `https://your-old-scalar-host.edu/works/your-book-slug`
   - New: `https://your-new-domain.example/scalar/your-book-slug`

4. **Open the SQL tab** on the destination database in phpMyAdmin, and run the following as one batch — replace the placeholder values in the first three lines with your own from steps 2–3:

   ```sql
   SET @book_id = <your destination book's ID>;
   SET @old_url = "https://your-old-scalar-host.edu/works/your-book-slug";
   SET @new_url = "https://your-new-domain.example/scalar/your-book-slug";

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

5. **Also check the book-level custom CSS you pasted in manually during Step 7**, since a background-image URL declared there could independently reference the old domain too — the two queries above don't touch it, since it lives on the book-level record this migration deliberately excludes from JSON import (Part 3).

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
Compare the destination book's table of contents and a sample of annotated images directly against the live source book. Pay particular attention to any page flagged as an exception during the export (Part 5).

---

## Part 5: Known Limitations

- **A page that's hidden/unpublished on the source book can fail during export with a server-side error** rather than a clean 404. This is a legitimate exclusion to accept, not something to keep chasing — hidden pages aren't meant to be reachable this way in the first place.
- **Not every raw content row in a Scalar book is a real page.** On one real migration, a large fraction of non-media content rows turned out to be single-letter-titled annotation label fragments (a numbered key like "4" whose body text is just a caption) rather than standalone reader-facing pages. These are correctly captured as annotation data during the crawl, not as pages — don't import them as if they were pages in their own right.
- **Media files and the top-level table of contents are permanent gaps in any JSON-based import**, not bugs to keep trying to fix — Scalar's own documentation confirms media transfer has always been a separate, manual step, and the TOC exclusion is a direct, unavoidable consequence of the importer's validation rules (Part 3).
- **This method requires the source book's live API to be reachable.** If the source site is ever taken offline before a migration is attempted, this whole approach is unavailable, and any previously-saved raw SQL exports would become the only remaining option (with the multi-tenant-database cautions in Part 6).
- **A page can display an unexpected "This path is a commentary..." banner** after import, even if the source book shows no such banner for that page. This isn't necessarily an import error — Scalar tracks an editorial `category` field per page version (values like `draft`, `clean`, `commentary`) independent of the book's structural `path` relationships, and different installs/themes can render that same field differently. If you see this, check whether the same metadata already exists on the source (it may simply not be visibly rendered there); the direct fix, regardless of root cause, is changing that page's type away from "Commentary" in the Dashboard's edit view.

---

## Part 6: If the Destination Has Other Books Already

Everything above works identically whether the destination is a brand-new empty install or one with other books on it already, **except for one category of risk that only applies to a shared destination**: Scalar stores all books on one install in the same database tables, distinguished only by an internal book ID. A raw SQL dump-and-restore (as opposed to the JSON import method this guide uses) risks wiping every other book on a shared destination, since it typically drops and recreates entire tables. **The JSON import method in this guide does not have this risk** — it only ever adds content, and never touches anything belonging to another book on the same install.

---

## Part 7: What's in This Repo

- **`scalar_book_export_recreator.ipynb`** — the export notebook described in Part 2, with all fixes from Part 3 already applied. Set `BOOK_SLUG` and `INSTALL_BASE` at the top to point it at your own book before running.
- **`scalar_media_downloader.ipynb`** — downloads every locally-hosted media file referenced by an exported book (used in Step 5). Deliberately excludes externally-hosted media (e.g. content hosted on an external archive) — those are just links, and keep working wherever the book ends up. Works from either the full unfiltered graph or the final filtered export, and also re-fetches the book's own top-level scope directly, since that's where a book's own background/thumbnail images often live.
- This README — intended to be kept alongside both notebooks as their operating manual.

## Contributing

If you use this on your own migration and find something that doesn't hold up — a new failure mode, a Scalar version where one of these fixes no longer applies, an edge case Part 5 doesn't cover — issues and pull requests are welcome. The table in Part 3 in particular is meant to grow over time, not stay frozen at whatever it covers today.

## License

MIT (or your preferred license — update this section before publishing).
