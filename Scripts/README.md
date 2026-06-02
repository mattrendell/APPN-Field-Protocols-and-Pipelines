# Scripts

Maintenance scripts for the APPN Aerial SOP repository.

| Script | Purpose |
| --- | --- |
| [`docx_to_markdown.py`](docx_to_markdown.py) | Convert the source `.docx` SOPs in `OriginalDocuments/` into the canonical Markdown + media tree under `Protocols/`. |
| [`publish_to_wiki.py`](publish_to_wiki.py) | Publish the locked revision (per `publish.yaml`) into the GitHub Wiki working copy and, optionally, render PDF snapshots into `releases/Rev<X.Y>/`. |

Run everything from the **repository root** so that relative paths
(`OriginalDocuments/`, `publish.yaml`, `Protocols/...`) resolve correctly.

---

## Conda environment

A single conda env covers both scripts and PDF rendering. Recommended:

```bash
conda create -n appn-sop -c conda-forge \
    python=3.11 \
    mammoth \
    pyyaml \
    pandoc \
    tectonic \
    librsvg
conda activate appn-sop
```

What each package is for:

- **`python` 3.11+** — both scripts use `from __future__ import annotations`
  + PEP 604 unions; 3.10+ is the minimum.
- **`mammoth`** — `.docx` → Markdown conversion (used by `docx_to_markdown.py`).
- **`pyyaml`** — reads `publish.yaml` (used by `publish_to_wiki.py`).
- **`pandoc`** — Markdown → PDF driver (used by `publish_to_wiki.py --pdf`).
- **`tectonic`** — modern self-contained LaTeX engine. Auto-detected by
  `publish_to_wiki.py` and gives the best-looking PDFs without installing the
  full TeX Live (~2 GB). First run is slow because it downloads the LaTeX
  packages it needs into a per-user cache; subsequent runs are fast.
- **`librsvg`** — provides `rsvg-convert`, required by pandoc to embed SVG
  images (e.g. EMF→SVG icons in the Validation Flight protocol) when
  rendering via a LaTeX engine. Without it the PDF build fails with
  `rsvg-convert: ... does not exist`.

### Optional extras

- **`emf2svg-conv`** — converts EMF images embedded in `.docx` files to SVG so
  they actually render in the Markdown / PDF output. Install via your system
  package manager (e.g. `sudo apt install libemf2svg-utils`); not on
  conda-forge.
- **`weasyprint`** — lightweight HTML/CSS-based PDF engine. Useful as a
  fallback or for faster (less polished) renders. Install with
  `conda install -c conda-forge weasyprint` and select with
  `--pdf-engine weasyprint`.

---

## `docx_to_markdown.py`

Converts every `.docx` under `OriginalDocuments/` (or any path you pass in)
into Markdown plus an adjacent `<basename>_media/` folder containing
embedded images.

Known SOP `.docx` files are routed automatically into the correct
`Protocols/.../` folder under the canonical (revision-less) basename — see
the `SOP_ROUTES` table at the top of the script. Unknown `.docx` files land
next to the source unless you pass `-o`.

### Usage

```bash
# Convert everything in OriginalDocuments/ into the matching Protocols/ folders
python Scripts/docx_to_markdown.py

# Convert a specific file into a custom output directory
python Scripts/docx_to_markdown.py path/to/file.docx -o /tmp/out
```

### Adding a new SOP

1. Drop the new `.docx` into `OriginalDocuments/`.
2. Add a route in `SOP_ROUTES` mapping the lowercased filename prefix to
   `(target_directory, output_basename)`.
3. Add a matching entry to [`publish.yaml`](../publish.yaml) so it gets
   published to the wiki.
4. Run the script.

---

## `publish_to_wiki.py`

Publishes the locked revision of every page listed in
[`publish.yaml`](../publish.yaml) into the wiki working copy, and optionally
renders PDF snapshots into `releases/Rev<revision>/`.

### Prerequisites

- Wiki repo cloned next to this repo:

  ```bash
  git clone https://github.com/aus-plant-phenomics-network/APPN-Field-Protocols-and-Pipelines.wiki.git \
      ../APPN-Field-Protocols-and-Pipelines.wiki
  ```

  Override the location with `--wiki-path` if you keep it elsewhere.

- The conda env above (`pyyaml` is required; `pandoc` + an engine only when
  using `--pdf`).

### Usage

```bash
# Preview what would change (no files written)
python Scripts/publish_to_wiki.py --dry-run

# Update the wiki working copy
python Scripts/publish_to_wiki.py

# Update wiki AND render PDFs into releases/Rev<rev>/
python Scripts/publish_to_wiki.py --pdf

# Force a specific PDF engine
python Scripts/publish_to_wiki.py --pdf --pdf-engine weasyprint
```

### What it does

For each page in the manifest:

- Copies the source `.md` into `<wiki>/<wiki_page>.md`.
- Injects a "Locked revision" banner directly under the H1.
- Rewrites image references that point at `*_media/...` to
  `media/<wiki_page>/...` and copies the matching media folder into
  `<wiki>/media/<wiki_page>/`.

Then it regenerates `Home.md` and `_Sidebar.md` from the manifest, grouped by
category.

With `--pdf`, each manifest page is also rendered to
`releases/Rev<revision>/<wiki_page>.pdf` via `pandoc`. The script
auto-detects an available PDF engine in this preference order:

`tectonic` → `xelatex` → `lualatex` → `pdflatex` → `weasyprint` → `wkhtmltopdf` → `prince`

Override with `--pdf-engine NAME`.

### Cutting a new revision

1. Bump `revision` (and optionally `revision_date`) in
   [`publish.yaml`](../publish.yaml).
2. Run `python Scripts/publish_to_wiki.py --pdf`.
3. Review the diff in the `.wiki` repo and push:

   ```bash
   cd ../APPN-Field-Protocols-and-Pipelines.wiki
   git add -A && git commit -m "Publish Rev X.Y" && git push
   ```

4. Commit the new `releases/Rev<X.Y>/` PDFs in this repo.
5. Tag the revision in this repo (`git tag -a vX.Y -m "Rev X.Y"; git push --tags`).

---

## Troubleshooting

- **`pdflatex not found`** — install a PDF engine (recommended:
  `conda install -c conda-forge tectonic`) or pass `--pdf-engine` to select
  one you have.
- **`Could not convert image ... .emf`** — pandoc cannot render EMF. The
  source markdown should reference the `.svg` version produced by
  `docx_to_markdown.py` (requires `emf2svg-conv` at conversion time).
- **`wiki path is not a directory`** — clone the wiki repo next to this one
  or pass `--wiki-path /path/to/wiki`.
- **`pyyaml is required`** / **`mammoth is required`** — activate the conda
  env (`conda activate appn-sop`) before running the scripts.
