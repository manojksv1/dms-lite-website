# Supported File Formats in DMS Enterprise

This document details the file formats supported by DMS Enterprise for upload, preview, text extraction, and editing.

## 1. Documents & Office

| Extension | Format | Text Extraction | OCR Support | Preview | Online Editing |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **.pdf** | Portable Document Format | ✅ Native + OCR | ✅ Hybrid | ✅ In-Browser | ❌ (View Only) |
| **.docx** | Microsoft Word | ✅ Native | ✅ Embedded Images | ✅ WOPI (Collabora) | ✅ WOPI (Collabora) |
| **.doc** | Legacy Word | ✅ Native | ❌ | ❌ (Download Only) | ❌ |
| **.xlsx** | Microsoft Excel | ✅ Native | ✅ Embedded Images | ✅ WOPI (Collabora) | ✅ WOPI (Collabora) |
| **.xls** | Legacy Excel | ✅ Native | ❌ | ❌ (Download Only) | ❌ |
| **.pptx** | Microsoft PowerPoint | ✅ Native | ✅ Embedded Images | ✅ WOPI (Collabora) | ✅ WOPI (Collabora) |
| **.ods** | OpenDocument Spreadsheet | ✅ Native | ✅ Embedded Images | ✅ WOPI (Collabora) | ✅ WOPI (Collabora) |
| **.odt** | OpenDocument Text | ✅ Native | ✅ Embedded Images | ✅ WOPI (Collabora) | ✅ WOPI (Collabora) |
| **.odp** | OpenDocument Presentation | ✅ Native | ✅ Embedded Images | ✅ WOPI (Collabora) | ✅ WOPI (Collabora) |
| **.txt** | Plain Text | ✅ Native | N/A | ✅ In-Browser | ✅ In-Browser |
| **.md** | Markdown | ✅ Native | N/A | ✅ In-Browser | ✅ In-Browser |
| **.csv** | Comma Separated Values | ✅ Native | N/A | ✅ In-Browser | ✅ In-Browser |
| **.rtf** | Rich Text Format | ✅ Native | ❌ | ❌ (Download Only) | ❌ |

## 2. Images & Graphics

| Extension | Format | Text Extraction | OCR Support | Preview | Thumbnail |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **.jpg / .jpeg** | JPEG Image | ✅ OCR | ✅ PaddleOCR | ✅ Lightbox | ✅ Generated |
| **.png** | Portable Network Graphics | ✅ OCR | ✅ PaddleOCR | ✅ Lightbox | ✅ Generated |
| **.tiff / .tif** | Tagged Image File Format | ✅ OCR | ✅ PaddleOCR | ✅ Lightbox | ✅ Generated |
| **.bmp** | Bitmap Image | ✅ OCR | ✅ PaddleOCR | ✅ Lightbox | ✅ Generated |
| **.webp** | WebP Image | ✅ OCR | ✅ PaddleOCR | ✅ Lightbox | ✅ Generated |
| **.gif** | Graphics Interchange Format | ❌ | ❌ | ✅ Lightbox | ✅ Generated |
| **.svg** | Scalable Vector Graphics | ✅ Native (XML) | ❌ | ✅ Lightbox | ✅ Generated |

## 3. Email & Communications

| Extension | Format | Text Extraction | Attachments | Preview |
| :--- | :--- | :--- | :--- | :--- |
| **.msg** | Outlook Message | ✅ Metadata + Body | ❌ | ✅ Text Preview |
| **.eml** | RFC 822 Email | ✅ Metadata + Body | ✅ Indexed | ✅ Text Preview |

## 4. Multimedia (Audio/Video)

DMS Enterprise includes a dedicated media processing pipeline (FFmpeg) to generate streaming previews and thumbnails.

| Extension | Format | Preview | Streaming | Thumbnail |
| :--- | :--- | :--- | :--- | :--- |
| **.mp4** | MPEG-4 Video | ✅ | ✅ H.264 / AAC | ✅ Frame @ 1s |
| **.webm** | WebM Video | ✅ | ✅ VP8/VP9 | ✅ Frame @ 1s |
| **.mkv** | Matroska Video | ✅ (Transcoded) | ✅ Converted to MP4 | ✅ Frame @ 1s |
| **.mov** | QuickTime Video | ✅ (Transcoded) | ✅ Converted to MP4 | ✅ Frame @ 1s |
| **.ogg** | Ogg Video | ✅ | ✅ Theora | ✅ Frame @ 1s |
| **.mp3** | MPEG-3 Audio | ✅ Player | ✅ Native | ✅ Album Art |
| **.wav** | Waveform Audio | ✅ Player | ✅ Native | ❌ |
| **.aac** | AAC Audio | ✅ Player | ✅ Native | ❌ |
| **.flac** | FLAC Audio | ✅ Player | ✅ Native | ✅ Album Art |
| **.m4a** | MPEG-4 Audio | ✅ Player | ✅ Native | ✅ Album Art |

## 5. Code & Data

The following formats are indexed as plain text and support syntax highlighting in the previewer.

-   **.json** (JSON Data)
-   **.xml** (XML Data)
-   **.html / .htm** (HTML Webpage)
-   **.css** (Cascading Style Sheets)
-   **.js / .ts** (JavaScript / TypeScript)
-   **.py** (Python Source)
-   **.java** (Java Source)
-   **.c / .cpp / .h** (C/C++ Source)
-   **.sql** (SQL Query)
-   **.sh / .bat** (Shell Scripts)

## 6. Archives

Archive contents are not indexed recursively for search, but the system supports **Structure Preview** (listing contents without downloading).

| Extension | Format | Preview | Extraction |
| :--- | :--- | :--- | :--- |
| **.zip** | ZIP Archive | ✅ List Contents | ❌ |
| **.rar** | RAR Archive | ✅ List Contents (via 7z) | ❌ |
| **.7z** | 7-Zip Archive | ✅ List Contents (via 7z) | ❌ |
| **.tar** | Tarball | ✅ List Contents | ❌ |
| **.gz** | Gzip Compressed | ✅ List Contents | ❌ |

## 7. Unsupported / Blocked Formats

For security reasons, the following executable formats are **strictly blocked** by default configuration (though they can be allowed via Admin Settings).

-   **.exe** (Windows Executable)
-   **.dll** (Dynamic Link Library)
-   **.com** (DOS Command)
-   **.bat** (Batch File - if strict mode on)
-   **.vbs** (VBScript)
-   **.ps1** (PowerShell)
-   **.jar** (Java Archive - executable)
