# DigiTech Markdown Converter — Web App Design Spec

**Date:** 2026-07-08
**Stack:** Flask (Python) + Vanilla JS
**Status:** Awaiting implementation

---

## 1. Overview

A local web application that lets users drag-and-drop files and convert them to Markdown using the existing `markitdown` Python library. Files are processed server-side, previewed in-browser (rendered HTML), copied, and available for download. Converted files are temporary and deleted after 30 minutes of inactivity.

---

## 2. Project Structure

```
markdown-converter/
├── app.py                  ← Flask server
├── templates/
│   └── index.html          ← Single-page UI
├── static/
│   ├── style.css           ← Styles (dark/light mode + DIGI palette)
│   ├── app.js              ← Drag & drop, upload, preview, download logic
│   ├── digiicono.png       ← Small icon / favicon
│   └── digitechsupport.png ← Header logo
├── script                  ← Existing install/reinstall helper
└── requirements.txt        ← flask, markitdown[all]
```

---

## 3. Backend — Flask (`app.py`)

### Routes

| Method | Route | Description |
|--------|-------|-------------|
| GET | `/` | Serve the HTML page |
| POST | `/convert` | Accept one or more files, convert via markitdown, return JSON |
| GET | `/preview/<file_id>` | Return raw markdown text for in-browser rendering |
| GET | `/download/<file_id>` | Serve the `.md` file; delete it from disk after serving |

### Conversion Logic

- Run Flask **inside the `~/.amigo` virtualenv** (user activates it before running `python app.py`)
- Call markitdown via **Python API** (not subprocess):
  ```python
  from markitdown import MarkItDown
  md = MarkItDown()
  result = md.convert(input_path)
  markdown_text = result.text_content
  ```
- Support **all formats markitdown accepts**: PDF, Word (.docx), Excel (.xlsx), PowerPoint (.pptx), HTML, images, audio, and more
- Each uploaded file gets a UUID: saved to `/tmp/mc-<uuid>/original_name.ext`, converted to `/tmp/mc-<uuid>/original_name.md`

### Temporary File Cleanup

- **On download:** delete the file's temp directory immediately after serving
- **Background sweep:** on every incoming request, delete any `/tmp/mc-*/` directories older than 30 minutes
- No database needed — all state lives in the temp filesystem

### POST `/convert` — Response format

```json
{
  "files": [
    {
      "id": "uuid-string",
      "original_name": "document.pdf",
      "md_name": "document.md",
      "status": "ok"
    },
    {
      "id": "uuid-string-2",
      "original_name": "bad.xyz",
      "md_name": null,
      "status": "error",
      "error": "Unsupported file type"
    }
  ]
}
```

### Error Handling

- Unsupported file types return status `"error"` per-file (not a 500)
- Max upload size: 50 MB per file (configurable via `MAX_CONTENT_LENGTH` in Flask config)
- If markitdown raises an exception, catch it and return status `"error"` with the message

### Running the Server

```bash
source ~/.amigo/bin/activate
python app.py
# → http://SERVER_IP:8082
# (port 8082 instead of 5000: macOS AirPlay Receiver occupies 5000)
```

---

## 4. Frontend — UI Design

### Branding & Theme

**Default:** Dark mode. Toggle button (sun/moon icon) top-right to switch to light mode. Preference saved in `localStorage`.

**Logos:**
- `digitechsupport.png` — displayed in the header/navbar (top of page)
- `digiicono.png` — used as browser favicon

### DIGI Color Palette

#### Dark Mode (default)
| Role | Color | Hex |
|------|-------|-----|
| Page background | Very Dark Blue | `#1B4965` |
| Surface / cards | Dark Gray | `#3F4245` |
| Primary accent | DIGI Green | `#84C361` |
| Text primary | White | `#FFFFFF` |
| Text secondary | Very Light Gray | `#EEF0F0` |
| Border / divider | Gray | `#717174` |
| Error / warning | Orange | `#D15227` |
| Success indicator | Dark Green | `#41823C` |

#### Light Mode
| Role | Color | Hex |
|------|-------|-----|
| Page background | Very Light Gray | `#EEF0F0` |
| Surface / cards | White | `#FFFFFF` |
| Primary accent | DIGI Green | `#84C361` |
| Text primary | Dark Gray | `#3F4245` |
| Text secondary | Gray | `#717174` |
| Border / divider | Medium Gray | `#DAD8D8` |
| Error / warning | Orange | `#D15227` |
| Success indicator | Dark Green | `#41823C` |

**WCAG AA compliance:** DIGI Green text on Very Dark Blue background, Dark Gray text on White — both pass AA contrast ratio per the brand guide.

### Layout

```
┌──────────────────────────────────────────┐
│  [digitechsupport.png logo]    [🌙 / ☀]  │  ← Header
├──────────────────────────────────────────┤
│                                          │
│   ┌──────────────────────────────────┐   │
│   │                                  │   │
│   │   Drag & drop files here         │   │  ← Drop zone
│   │   or click to browse             │   │
│   │                                  │   │
│   │   [Browse Files button]          │   │
│   └──────────────────────────────────┘   │
│                                          │
│   Converted Files                        │
│   ┌──────────────────────────────────┐   │
│   │ 📄 document.md     [👁] [⬇]      │   │
│   │ 📄 report.md       [👁] [⬇]      │   │  ← Results list
│   │ ❌ bad.xyz   Unsupported format   │   │
│   └──────────────────────────────────┘   │
│                                          │
└──────────────────────────────────────────┘

  ← Preview panel slides in from the right →
  ┌────────────────────────────────────────┐
  │ document.md                      [✕]  │
  │ ────────────────────────────────────── │
  │ # Document Title                       │
  │                                        │
  │ Rendered markdown content here...      │
  │                                        │
  │                      [⬇ Download]      │
  └────────────────────────────────────────┘
```

### Drag & Drop Zone

- Accepts multiple files simultaneously
- Border highlights in DIGI Green on drag-over
- Hidden `<input type="file" multiple>` triggered by "Browse Files" button or clicking the zone
- Accepts all file types (server validates and returns per-file errors)
- Shows per-file upload progress bar during conversion

### Results List

Each successfully converted file shows:
- File icon + `.md` filename
- **Eye button** `[👁]` — opens slide-in preview panel
- **Download button** `[⬇]` — downloads file and keeps the entry in the session history
- Failed files show with red `❌` and the error message inline

### Preview Panel

- Slides in from the right (CSS transition, no page reload)
- Renders markdown as HTML using `marked.js` (loaded from CDN)
- Close button `[✕]` top-right
- **Download** button at bottom of panel
- Does not block the rest of the UI (non-modal slide-over with dim backdrop)

### Theme Toggle

- Button top-right: moon icon in dark mode / sun icon in light mode
- Clicking toggles `data-theme="dark"` attribute on `<html>`
- All colors use CSS custom properties (`--color-bg`, `--color-surface`, `--color-accent`, etc.)
- Preference stored in `localStorage` under key `"digi-theme"`

---

## 5. Implementation Notes

### requirements.txt
```
flask>=3.0
markitdown[all]
```

### Startup Check
On startup, `app.py` tries to import markitdown. If it fails, it prints:
```
ERROR: markitdown not found. Run ./script -i to install dependencies.
```
and exits with code 1.

### Security
- Sanitize uploaded filenames with `werkzeug.utils.secure_filename`
- Validate that `file_id` in `/preview/<file_id>` and `/download/<file_id>` is a valid UUID before constructing any file path (prevents path traversal)
- No CORS config needed (Flask serves both frontend and API)

### marked.js CDN
```html
<script src="https://cdn.jsdelivr.net/npm/marked/marked.min.js"></script>
```

---

## 6. Supported File Formats

All formats supported by `markitdown[all]`:
- Documents: PDF, DOCX, PPTX, XLSX, XLS
- Web: HTML, HTM
- Text: TXT, CSV, JSON, XML
- Images: PNG, JPG, JPEG, GIF, BMP, TIFF (via OCR/caption)
- Audio: MP3, WAV (via transcription)
- Other: ZIP (contents), EPUB

---

## 7. Out of Scope (v1)

- User authentication
- Persistent file history across server restarts
- Cloud storage
- Batch download (ZIP of all files)
- Mobile-specific layout (basic CSS responsiveness only)

---

## 8. How to Run

```bash
# First time setup (if not done already)
./script -i

# Start the server
source ~/.amigo/bin/activate
python app.py

# Open in browser
open http://SERVER_IP:8082
```
