# jsDired — Definitive Specification v2.0

> Merged from `desing.md` (foundation), and `message.md` (Emacs fidelity refinements). The goal: a single prompt that produces a **production-grade**, **Emacs-faithful**, **beautiful** browser Dired.

---

## ROLE

Act as a **Senior JavaScript Engineer** (10+ years experience) who specialises in terminal-style UIs, async state machines, and the Chrome File System Access API.

---

## 1 — CONSTRAINTS

| Rule | Detail |
|---|---|
| Delivery | One self-contained HTML file (`dired2.html`). |
| Language | Vanilla JavaScript only. No frameworks, no npm. |
| API | Chrome File System Access API (`showDirectoryPicker`, `FileSystemDirectoryHandle`, `FileSystemFileHandle`). |
| Async | Every file-system call must use `async/await`. |
| Style | ES2022+. Class-based architecture (`class Dired`). |

---

## 2 — BOOT & STATE SYNC

1. **Landing Overlay**: Show app title and a single **"Grant Access"** button.
2. **Directory Picker**:
   ```js
   showDirectoryPicker({ startIn: 'downloads', mode: 'readwrite' })
   ```
   Fallback to plain `showDirectoryPicker({ mode: 'readwrite' })` if `startIn` fails.
3. **The "Revert-Buffer" Philosophy**: Every operation that modifies the disk (Create, Copy, Move, Delete, Rename) **MUST** conclude by calling `refreshDirectory()`. The UI must always be a direct reflection of the physical disk.
4. **Contextual Resets**: When changing directories (entering a folder or going up), the `filterRegex` must be cleared (`null`). The user starts with a "clean slate" in every new directory.

---

## 3 — VISUAL / UI

### 3.1 Theme — "Tokyo Night"

| Token         | Hex       |
|---------------|-----------|
| `--bg`        | `#1a1b26` |
| `--bg-dark`   | `#16161e` |
| `--fg`        | `#a9b1d6` |
| `--fg-dim`    | `#565f89` |
| `--blue`      | `#7aa2f7` |
| `--cyan`      | `#7dcfff` |
| `--green`     | `#9ece6a` |
| `--orange`    | `#ff9e64` |
| `--red`       | `#f7768e` |
| `--magenta`   | `#bb9af7` |
| `--cursor-bg` | `#3b4261` |
| `--border`    | `#292e42` |

Font: `JetBrains Mono` (loaded from Google Fonts), fallback `Fira Code`, `monospace`.

### 3.2 Layout (top → bottom)

| Zone | Description |
|---|---|
| **Header** | Shows the current path (`pathSegments.join(' / ')`) on the left. App name + sort mode on the right. |
| **Column Header Row** | Sticky. Columns: `M` (mark) · `Name` · `Size` · `Modified`. Styled dim. |
| **Entry Rows** | One `<div class="row">` per entry. Cursor row gets `row-cursor` class. |
| **Minibuffer** | Absolute-positioned above the footer. Hidden by default. Used for Rename, Filter, and any future prompts. |
| **Footer / Status Bar** | Left: entry count + marked count + flagged count. Right: current sort mode + yank status. |
| **Toast Container** | Fixed, top-right. Toasts slide in, auto-dismiss after 3.5 s. |
| **Loading Overlay** | Centered spinner + "Working…" text. Shown during any async op. |

**Layout Constraint**: Detail columns (**Size**, **Modified**) must have sufficient fixed widths and `white-space: nowrap` to prevent line-wrapping and maintain row alignment.

### 3.3 Entry Rendering

```
M  Name                          Size        Modified
   ./                            -           -
   ../                           -           -
*  src/                          -           -
D  old.txt                       12.0KB      21 Apr 10:22
   video.mp4                     2.0MB       18 Apr 08:00
```

- **Directories**: trailing `/`, colored `--blue`.
- **`.` and `..`**: colored `--magenta`, class `type-special`.
- **Marked `*`**: colored `--orange`, class `mark-star`.
- **Flagged `D`**: colored `--red`, class `mark-flag`.
- **Cursor line**: `background: var(--cursor-bg); font-weight: bold;`.
- **Long filenames**: `overflow: hidden; text-overflow: ellipsis;` on the Name column.
- **Rigidity**: All columns must use `white-space: nowrap`. The **Modified** column specifically must be wide enough to accommodate a full timestamp (e.g., `21 Apr 10:22`) without breaking the single-line layout.

### 3.4 Special Characters in Filenames

All entry names **must be HTML-escaped** before insertion into `innerHTML` to prevent XSS and rendering bugs with `&`, `<`, `>`, `"`, `'`, emojis, and unicode.

---

## 4 — SPECIAL DIRECTORY ENTRIES

- `.` (current directory) and `..` (parent directory) are **always** the first two entries.
- `..` is only shown if `pathSegments.length > 1` (i.e. not at root).
- Pressing `Enter` on `.` → calls `refreshDirectory()`.
- Pressing `Enter` on `..` → calls `goUp()`.
- **Protected**: the user **cannot** mark, unmark, flag, rename, or delete `.` or `..`. If attempted, show a toast: "Cannot operate on special entries".

---

## 5 — MODES

| Mode | Trigger | Behaviour |
|---|---|---|
| **NORMAL** | Default, or when `marked.size === 0 && flagged.size === 0`. | Navigation and read-only operations. |
| **DIRED** | Automatically entered when `marked.size > 0 \|\| flagged.size > 0`. | Batch edit operations are active. |

The current mode is displayed in the footer. NORMAL = green pill; DIRED = orange pill.

### 3.2 Feature Modes (Toggles)

Implement three toggleable modes that persist during the session:
- **Details Mode (`(`)**: Toggle visibility of "Size" and "Modified" columns. Default: **OFF**.
- **Icons Mode (`)`)**: Display file-type icons. **Placement**: Icons must be in the "Name" column, positioned immediately to the **LEFT** of the filenames. Default: **ON**.
- **Omit Mode (`0`)**: Toggle visibility of hidden files (dotfiles, names starting with `.`). Default: **ON** (hidden).

### 3.3 Highlighting
- **hl-line-mode**: The cursor row (`row-cursor`) must be visually distinct across the entire width of the table.


---

## 6 — KEYBINDINGS

### Navigation
| Key | Action |
|---|---|
| `j` / `↓` | Cursor down |
| `k` / `↑` | Cursor up |
| `S-Enter` | **dired-jump**: Go to parent directory (`..`). |
| `g` | **revert-buffer**: Refresh current directory. |

### Marking & Selection
| Key | Action |
|---|---|
| `m` / `SPC` | **toggle-mark**: Mark item if unmarked, unmark if marked. Advance cursor. |
| `u` | Unmark + unflag current item. Advance cursor. |
| `U` | **Unmark All**: Clear all marks and flags in the current buffer. |
| `d` | Toggle flag for deletion (`D`). Advance cursor. |
| `x` | Execute: delete all **flagged** items (or marked items if none flagged). |

### Creation & Operations
| Key | Action |
|---|---|
| `+` | **Create Directory**: Prompt for name, create folder, then refresh. |
| `'` | **Create Empty File**: Prompt for name, create file, then refresh. |
| `y` / `C` | Yank (Copy) targets to clipboard. |
| `M` | Yank for **Move** (Cut) targets to clipboard. |
| `p` | Paste / Execute Move. |
| `R` | **Rename** (single) or **Move** (marked targets). |

### Utilities
| Key | Action |
|---|---|
| `(` | Toggle **Details Mode**. |
| `)` | Toggle **Icons Mode**. |
| `0` | Toggle **Omit Mode**. |
| `W` | **Browse**: Open the current file in the browser's native context. |
| `/` | Open minibuffer for regex filter. |
| `s` | Cycle sort: `name` → `size` → `mtime`. |
| `?` | Show help overlay. |
| `q` | Quit (reload page). |

---

## 7 — STATE MODEL

```js
class Dired {
    rootHandle          // FileSystemDirectoryHandle — the initially picked dir
    currentHandle       // FileSystemDirectoryHandle — the directory we are viewing
    pathSegments        // Array<{ handle, name }> — breadcrumb stack

    entries             // Array<{ handle, name, kind, size, mtime, isDot, isDotDot }>
    cursorIndex         // Number

    marked              // Set<string> — entry names that are marked (*)
    flagged             // Set<string> — entry names flagged for deletion (D)

    clipboard           // { action: 'copy'|'move', items: Array<{handle, name, kind}>, sourceHandle }
    filterRegex         // RegExp | null
    sortBy              // 'name' | 'size' | 'mtime'

    loading             // Boolean — prevents overlapping operations
    isInputActive       // Boolean — minibuffer is focused
    inputCallback       // Function | null — called when minibuffer is accepted
}
```

**Key design decision**: `marked` and `flagged` store **entry names** (not indices), so they remain valid after a refresh reorders the list.

---

## 8 — CORE FUNCTIONS

### `async refreshDirectory()`
1. Guard: if `loading`, return.
2. Set loading = true.
3. Build `rawEntries`:
   - Push `.` entry.
   - Push `..` entry (only if not at root).
   - Iterate `currentHandle.values()`, call `getFileStats()` for each.
4. Filter: if `filterRegex`, keep only dot entries + matching names.
5. Sort via `sortItems()`.
6. Clamp `cursorIndex` to valid range.
7. Call `render()`.
8. Set loading = false (debounced: `setTimeout(() => ..., 200)`).

### `async getFileStats(handle)`
- If directory → `{ size: null, mtime: null }`.
- If file → `handle.getFile()` → `{ size: file.size, mtime: new Date(file.lastModified) }`.
- On error → `{ size: 0, mtime: null }`.

### `sortItems(items)`
- `.` always first, `..` always second.
- Then directories before files.
- Then compare by `sortBy`.

### `getOperationTargets()`
- If `marked.size > 0` → return entries matching marked names (excluding dot entries).
- Else → return `[entries[cursorIndex]]` (if not a dot entry).

### `async deleteItems()`
- Get targets from flagged set (or marked set if no flags).
- `confirm()` before proceeding.
- For each: `currentHandle.removeEntry(name, { recursive: true })`.
- Clear sets, refresh.

### `async renameItem()`
- Open minibuffer with current name.
- On accept: try `handle.move(newName)`, fallback to copy+delete for files.
- Refresh.

### `async copyItems(moveMode)`
- Store targets in `clipboard` with `action = moveMode ? 'move' : 'copy'`.
- Show toast.

### `async pasteItems()`
- For each clipboard item:
  - If same directory + copy mode → prefix name with `copy_of_`.
  - Files: `getFileHandle(name, {create}) → createWritable() → stream.pipeTo()`.
  - Directories: `recursiveCopy()`.
  - If move mode: delete source after copy.
- Refresh.

### `async recursiveCopy(srcHandle, destParent, newName?)`
- Create dest dir handle.
- Walk source entries recursively, copying files via stream.

---

## 9 — MINIBUFFER

An Emacs-style input line at the bottom of the terminal area.

- **Open**: `openMini(label, defaultValue, callback)` → show, focus input.
- **Close**: `Enter` accepts (calls callback with value), `Escape` cancels.
- While active, all other keybindings are disabled.

Used for: **Rename** (`R`), **Filter** (`/`), **New Item** (`+`, `'`).

- **Rename (R) Behavior**:
    - Pre-fills input with current filename.
    - Selects the text by default for instant replacement or editing.
    - If 1 item: performs Rename.
    - If multiple marked: switches context to **Move**.

---

## 10 — TOAST NOTIFICATIONS

- Non-blocking, top-right, slide-in animation.
- Auto-dismiss after 3.5 seconds.
- Color-coded border: green (success), red (error), blue (info).

## 5 — CORE LOGIC

### 5.2 Icon Implementation
- Use a simple mapping for common extensions (js, html, css, md, txt, png, jpg, pdf, zip) and folders.
- Icons are inserted into the DOM only when `Icons Mode` is active.
### 5.3 Recursive Operations
- Paste operations must handle recursive directory copying using `ReadableStream.pipeTo` for files to avoid memory exhaustion.
- Collisions should auto-rename with a `copy_of_` prefix.

### 5.4 Move Behavior (Atomic)
- **Logic**: A "Move" is a "Copy then Delete". The source is only deleted after a successful copy is verified.
- **Trigger**: Key `M` or key `R` (when items are marked).
- **Collision**: Block move if source and destination directories are identical.
- **UI**: Status bar shows `[Move Active]` when targets are staged for move.



---

## 11 — EDGE CASES (MANDATORY)

Your implementation **must** handle all of the following. Add inline comments referencing the test number.

| # | Scenario | Expected Behaviour |
|---|---|---|
| 1 | Directory with 1000+ files | UI stays responsive; no full-DOM rebuild on cursor move. |
| 2 | Filenames with `&`, `<`, `>`, `"`, `'`, emojis | HTML-escaped before render. |
| 3 | Attempt to mark/flag/rename/delete `.` or `..` | Blocked. Toast shown. |
| 4 | Permission denied mid-operation | Catch error, show toast, continue with remaining items. |
| 5 | Empty directory | Shows only `.` (and `..` if not root). |
| 6 | Hidden files (`.dotfiles`) | Visible by default. |
| 7 | Recursive directory copy/move | `recursiveCopy()` walks the tree. |
| 8 | Paste collision (same name exists) | Auto-rename to `copy_of_<name>`. |
| 9 | Refresh while operation in progress | Guarded by `loading` flag; refresh is a no-op. |
| 10 | Very long filenames | `text-overflow: ellipsis` on the Name column. |
| 11 | Rapid key presses | All state mutations are synchronous; async ops are guarded. |
| 12 | User cancels directory picker | `AbortError` caught, toast shown, overlay stays. |

---

## 12 — PERFORMANCE

- **Rendering**: Only rebuild `innerHTML` on `render()`, not on every cursor move.
- **Debounced Refresh**: `setTimeout` guard of 200ms after `setLoading(false)`.
- **Stream Copy**: Use `ReadableStream.pipeTo(WritableStream)` for file copies to avoid loading entire files into memory.
- **UI Responsiveness & Deadlock Prevention**: 
    - Heavy operations **must** yield to the main thread (e.g., via `await new Promise(r => setTimeout(r, 0))`) after setting `loading = true`.
    - **MANDATORY**: All operations that set `loading = true` must use a `try...finally` block to ensure `loading = false` is called, preventing infinite UI blocking.
- **Snappy Navigation**: 
    - Directory switching should feel instant. 
    - **Threshold Loading**: Do not show the loading overlay immediately for `refreshDirectory`. Implement a 150ms-200ms delay before the overlay becomes visible. If the refresh completes within this window, the loading indicator should not appear.

---

## 13 — FUTURE ROADMAP (from design2)

These are **NOT** required in v1 but inform architectural decisions:

1. **Dual-Pane Mode** (Midnight Commander): TAB to switch panes.
2. **File Preview Panel**: Images, text, markdown rendered on hover/delay.
3. **Web Worker directory sizing**: Background recursive size calculation.
4. **IndexedDB handle persistence**: Remember permissions across sessions.
5. **Virtual Scrolling**: For directories with 10,000+ entries.

---

## OUTPUT

Return **one complete, ready-to-run HTML file** saved as `dired_mix1.html`. It must:

- Include a version comment at the top: `<!-- jsDired Mix v1.0 -->`.
- Be clean, well-commented, production-quality code.

Do not include explanations outside the file.
