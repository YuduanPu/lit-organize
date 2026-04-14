---
name: lit-organize
description: >
  Organize academic literature PDFs into a structured research library. Use this skill whenever the user wants to:
  organize papers, sort PDFs into folders by topic/stream, create a literature spreadsheet, rename academic papers,
  build a bibliography, manage research literature, track readings, deduplicate papers, or maintain a .bib file.
  Also trigger when the user mentions "literature organization", "paper management", "reading list", or asks to
  sort/categorize/rename PDFs in a research context. Works for any field but is optimized for economics and
  economic history research.
---

# Literature Organizer

Organize academic literature PDFs into a structured, maintainable research library with spreadsheet tracking, BibTeX management, and incremental updates.

## Workflow overview

1. Scan PDFs, extract metadata, rename files with numbering
2. For each paper, fill in key points (methodology, findings, data sources, etc.)
3. Write everything to the xlsx spreadsheet and .bib file
4. Handle stream folders (detect existing or ask user to provide categories)

## Step 1: Gather inputs

Ask the user for:
- **Folder path**: where the PDFs are
- **Project name** (optional): for naming output files (default: "research")

## Step 2: Scan for new PDFs

Run:
```bash
python3 "<skill-dir>/scripts/lit_organize.py" --folder "<folder_path>" --project "<project_name>" --scan 2>/dev/null
```

This outputs JSON with:
- `new_files`: metadata for each unprocessed PDF (title, authors, year, source, raw_text_preview)
- `duplicates`: detected duplicates (exact hash match or >85% title similarity)

## Step 3: Review metadata and fill key points

For each new PDF, using the scan output:
1. **Correct metadata** if the script got it wrong (title, authors, year, source). The `raw_text_preview` field contains the first ~800 chars of extracted text — use this to verify and fix.
2. **Determine type**: Journal Article / Working Paper / Primary Source / Book / Book Chapter / Government Report / Other. For primary sources (archival documents, census records, government reports, first-hand materials), set Source to the archive or institution name.
3. **Fill key points** by reading the text preview:
   - Research Question / Topic
   - Methodology (e.g., DID, IV, RDD, descriptive, archival)
   - Key Findings
   - Data Sources Used
   - Theory/Framework
   - Geographic Focus
   - Time Period

**When the `raw_text_preview` is insufficient** (empty, garbled, or missing key details like methodology/findings), use the `anthropic-skills:pdf-reading` skill to read the PDF efficiently rather than loading the entire file into context. This is common with older journals, scanned documents, or PDFs without embedded text. Only invoke the pdf-reading skill for the specific PDFs that need it — most papers will have enough in the preview.

Present all entries to the user for review before proceeding.

## Step 4: Apply to spreadsheet and .bib

Write assignments to a temp JSON file and run:
```bash
python3 "<skill-dir>/scripts/lit_organize.py" --folder "<folder_path>" --project "<project_name>" --apply-assignments "<assignments_json_path>" 2>/dev/null
```

The assignments JSON format:
```json
{
  "duplicates_to_delete": ["filename1.pdf"],
  "working_paper_replacements": [{"delete": "wp.pdf", "keep": "published.pdf"}],
  "entries": [
    {
      "original_filename": "original.pdf",
      "title": "Full Title",
      "authors": "Last, First and Last2, First2",
      "year": "2023",
      "source": "Journal of Example Studies",
      "type": "Journal Article",
      "streams": [],
      "research_question": "...",
      "methodology": "...",
      "key_findings": "...",
      "data_sources": "...",
      "theory": "...",
      "geographic_focus": "...",
      "time_period": "...",
      "notes": ""
    }
  ]
}
```

This renames files (with numbering prefix and spaces in title/author/source parts), writes the spreadsheet, and updates the .bib file.

**File naming format**: `N_Title Here_Author_Year.pdf`
- `N` is a sequential number (1, 2, 3, ...) continuing from the highest existing number in the folder
- Underscores ONLY separate the categories (number, title, author, year)
- Spaces are preserved within each category (e.g., "Lift the Ban" not "Lift_the_Ban")
- Journal/source name is NOT included in the filename (it's in the spreadsheet)

## Step 5: Handle stream folders

Do NOT automatically classify papers into streams. Instead:

1. **Check for existing stream subfolders** in the literature folder. If subfolders already exist (from prior runs or user-created), list them and ask the user which papers go into which folders.
2. **If no subfolders exist**, ask the user to provide their stream/category names. The user categorizes papers manually.
3. Once the user provides assignments, run:
```bash
python3 "<skill-dir>/scripts/lit_organize.py" --folder "<folder_path>" --project "<project_name>" --apply-streams "<streams_json_path>" 2>/dev/null
```

The streams JSON format:
```json
{
  "stream_assignments": [
    {"file_path": "1_Title_Author_Year_Source.pdf", "streams": ["Stream A", "Stream B"]},
    {"file_path": "2_Title_Author_Year_Source.pdf", "streams": ["Stream C"]}
  ]
}
```

A paper can belong to multiple streams (symlinked into each folder).

## Incremental updates

When the user adds more PDFs and re-runs:
- The script only processes files NOT in `.lit-organize-state.json`
- Existing spreadsheet rows are never overwritten (preserving user edits to "Why Included", etc.)
- New entries are appended; numbering continues from the last number
- Duplicates are detected and handled (exact dupes deleted; working paper deleted if published version exists)

Do NOT re-process previously organized files. Do NOT rename files that are already in the state file.

**Deletion detection**: Every scan automatically checks whether previously processed PDFs still exist on disk. If a file was deleted:
- Its spreadsheet row is marked "DELETED" in the Notes column (preserving all other data)
- Its entry is removed from the state file so the number gap remains
- Dangling symlinks in stream folders are cleaned up
- Numbering is NOT renumbered — gaps are left as-is

## Spreadsheet columns

| Column                    | Description                                                                                        |
| ------------------------- | -------------------------------------------------------------------------------------------------- |
| Title                     | Full paper title                                                                                   |
| Author(s)                 | All authors                                                                                        |
| Year                      | Publication year                                                                                   |
| Source                    | Journal, archive, or working paper series                                                          |
| Type                      | Journal Article / Working Paper / Primary Source / Book / Book Chapter / Government Report / Other |
| Literature Stream(s)      | Comma-separated streams                                                                            |
| Research Question / Topic | Main research question                                                                             |
| Methodology               | Empirical strategy (DID, IV, RDD, descriptive, archival, etc.)                                     |
| Key Findings              | Summary of main results                                                                            |
| Data Sources Used         | Datasets or archives used                                                                          |
| Theory/Framework          | Theoretical framework                                                                              |
| Geographic Focus          | Country/region studied                                                                             |
| Time Period               | Historical period covered                                                                          |
| Why Included              | *Left blank for user to fill*                                                                      |
| BibTeX Key                | e.g., acemoglu2001colonial                                                                         |
| APA Citation              | Full APA 7th edition citation                                                                      |
| In-text Citation          | (Author, Year) format                                                                              |
| File Path                 | Relative path to renamed file                                                                      |
| Date Added                | Date the entry was created                                                                         |
| Notes                     | Additional notes                                                                                   |

## Dependencies

```bash
pip3 install pdfplumber openpyxl bibtexparser
```
