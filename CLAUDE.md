# Squire

Squire is a native EPUB reader for iPhone, iPhone Duo and iPad. It is the native successor to Quire, a web prototype. The goal is a calm, book-like reader for a personal library of roughly 500 DRM-free EPUBs, built first and foremost for the iPhone Duo's fold: one page when folded, a two-page spread when open.

This is a personal app for one user. Favor clarity and a small, well-tested codebase over features.

## Ground rules for Claude Code

- **Do not guess Apple or Readium APIs.** The iPhone Duo and iOS 27 SDK are newer than your training data. Before using any API for fold state, posture, the cover (outer) display or the hinge, read Apple's current documentation or the SDK headers in Xcode. If you can't confirm an API exists, stop and tell me instead of inventing one. Same for Readium: check the version pinned in `Package.resolved` and read its docs or source before calling anything.
- Work in small, buildable steps. After each change, build and run the tests. Never leave the project in a state that doesn't compile.
- Ask before adding any dependency beyond those listed below.
- Ask before changing the data model once a migration-relevant version has shipped to my device.
- Keep commits small, with messages that say what changed and why.

API research done so far lives in `docs/duo-research.md`. Read it before touching fold, hinge, spread or cover-display code.

## Stack

- Swift 6 with strict concurrency, SwiftUI, `@Observable` models.
- Minimum target: iOS 27 (needed for iPhone Duo support). iPhone and iPad.
- **EPUB engine:** Readium Swift Toolkit (`ReadiumShared`, `ReadiumStreamer`, `ReadiumNavigator`, plus `ReadiumAdapterGCDWebServer` if the current version still needs it) via Swift Package Manager. Pin an exact release tag, not a branch or range. Use `EPUBNavigatorViewController` for rendering and pagination; don't build a custom renderer.
- **Persistence:** SwiftData for library metadata and reading positions.
- **Fonts:** bundle Literata and Atkinson Hyperlegible (both SIL Open Font License; include the license files). Iowan Old Style is a system font on iOS.
- **Tests:** Swift Testing (`import Testing`). UI tests with XCTest only where needed.

## Architecture

```
Squire/
  App/            SquireApp, root navigation, scene setup
  Library/        Library models, import, search/sort, views
  Reader/         Readium navigator wrapper, settings, TOC, progress
  Design/         Colors, fonts, theme tokens, shared components
  Storage/        SwiftData models, file access, bookmarks
  Resources/      Fonts and licenses, asset catalog
SquireTests/
```

- Views stay thin. Logic lives in `@Observable` model types that can be unit-tested without UI.
- Wrap Readium in one `ReaderController` type so the rest of the app never imports Readium directly. That keeps upgrades contained.
- All file I/O and EPUB parsing runs off the main actor.

## Data model (SwiftData)

`Book`: stable ID, title, author, cover thumbnail (small JPEG data), date added, last opened, progression (0–1), last Readium `Locator` stored as JSON, file reference (see below), file size.

`ReaderSettings`: theme, font, font size, line spacing, margins, page layout (auto, one page, two pages), justify, publisher styles, page-turn animation. Stored once, applies to all books.

Identify books by a content hash of the file so re-importing the same EPUB doesn't duplicate it.

## Library and import

- Import with `fileImporter` (single or multiple `.epub` files) and also support picking a whole folder.
- **Prefer reading books in place:** store security-scoped bookmarks to the user's files or folder rather than copying them. Fall back to copying into the app's container only if a bookmark can't be resolved. Handle stale bookmarks by re-resolving and, if that fails, marking the book as missing with a clear way to relocate it.
- Import runs in the background with progress ("Adding 12 of 87"), skips duplicates, and reports files it couldn't read by name.
- Accept EPUBs opened from Files, AirDrop and the share sheet (register the document type).
- Library views: cover grid (default) and list. Search across title and author. Sort by recent, title, author, date added. A "Continue reading" card at the top for the most recent unfinished book.
- Covers come from the EPUB; if there's none, draw a generated cover with the title in Literata on a muted cloth color chosen from the title's hash.
- Progress on each cover is shown as a bookmark ribbon whose length tracks how far I've read. Show "Finished" at the end.
- Swipe or context menu to remove a book from the library. Removing never deletes the user's original file.

## Reader

- Open to the saved `Locator`. Save position on every page turn (debounced) and when the app backgrounds.
- **Layout:** in Auto mode, use one page when folded or in narrow windows and a two-page spread when the Duo is unfolded or on iPad. Prefer Readium's own spread setting if it can be driven by available width; otherwise switch it from the view's size class or width. Keep the reading position when the layout changes mid-book.
- In spread mode, draw a subtle fold shadow down the center gutter.
- Page turns by tapping the left or right third, swiping, or hardware keys. Tapping the middle shows the top and bottom bars.
- Top bar: back to library, book title, contents, reading settings ("Aa"). Bottom bar: current chapter name, percentage, and a scrubber to jump anywhere in the book.
- Contents sheet from the EPUB's navigation, with the current chapter highlighted.
- Settings sheet: themes (match system, paper, sepia, night, black), font (Literata, Iowan, Atkinson), text size stepper, line spacing (tight, normal, loose), margins (narrow, normal, wide), pages (auto, one, two), and toggles for justify text, publisher styles and page animation. Changes apply live.
- Respect Reduce Motion: no page-turn animation when it's on.

## iPhone Duo features

Research these against the iOS 27 SDK before building them, per the ground rules:

1. Detect folded vs. unfolded and drive the Auto page layout from it.
2. When the phone is closed mid-book, show the current book's cover on the outer display, if the SDK allows an app to do that.
3. Keep the exact reading position across every fold and unfold.

If any of these isn't possible with public APIs, tell me and propose the closest alternative.

## Design tokens

Reader themes (background / text / secondary / accent):

| Theme | Background | Text | Secondary | Accent |
|---|---|---|---|---|
| Paper | #F6F6F1 | #1F2421 | #6B726E | #2B4FD8 |
| Sepia | #EFE4CE | #3A2E21 | #7A6A55 | #8A4B1F |
| Night | #18202D | #CDD3DD | #8892A3 | #9DB2FF |
| Black | #000000 | #BDBDBD | #777777 | #9DB2FF |

Library (light / dark): background #EDEFEB / #121715, surface #FFFFFF / #1B221F, text #1D2320 / #E3E8E5, secondary #5E6A64 / #98A49E, accent #2B4FD8 / #8EA6FF.

Type: Literata for titles and reading; Atkinson Hyperlegible for interface text. Default reading size 19pt, line spacing 1.6. Use sentence case everywhere and plain verbs on buttons ("Add books", "Remove", "Continue reading"). Covers get a slightly asymmetric corner radius (tighter on the spine side) and a faint spine shading on the left edge.

## Build order

1. Project skeleton, SwiftData models, design tokens, fonts. Build passes, empty library screen shows.
2. Import from Files with bookmarks, metadata and cover extraction, duplicate detection. Tests for parsing and hashing.
3. Library grid, search, sort, continue reading, remove.
4. Reader with Readium: open, paginate, save and restore position.
5. Bars, contents sheet, scrubber, settings sheet.
6. Auto spread on unfold and the center fold shadow.
7. iPhone Duo extras (after research): cover display when closed.
8. Polish: accessibility (VoiceOver labels, Dynamic Type in the library), performance with a 500+ book library, empty and error states.

Stop at the end of each step and summarize what works, what doesn't, and what's next.

## Commands

Fill these in once the project exists:

- Build: `xcodebuild -scheme Squire -destination 'platform=iOS Simulator,name=<iPhone Duo simulator name>' build`
- Test: `xcodebuild -scheme Squire -destination '<same>' test`

## Testing

- Unit-test EPUB metadata extraction, cover fallback, content hashing, duplicate detection, search and sort, and progress math.
- Keep a few small sample EPUBs in `SquireTests/Fixtures`: one simple, one with images, one with no cover, one with a nested table of contents, and one deliberately malformed.
- Manually verify on the iPhone Duo simulator (folded and unfolded) and an iPad simulator before calling a reader step done.

## Out of scope for now

DRM-protected books, PDF, cloud sync between devices, Calibre library sync, highlights and notes, text-to-speech, and Android. Note ideas for these in `IDEAS.md` instead of building them.
