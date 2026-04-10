# Markdown Editor — Tasklet Instant App

In-browser file editor for markdown and text files in the agent filesystem. Browse, edit, and save files without download/upload cycles.

## Setup

1. Create the app: tell your Tasklet agent to create an instant app called `markdown-editor`
2. Replace each generated file with the contents below (matching file paths)
3. Show the app preview

Or just paste this gist URL and say "build this instant app from the gist."

---

## `tasklet.config.json`

```json
{
  "name": "markdown-editor",
  "displayName": "Markdown Editor",
  "description": "In-browser file editor for markdown and text files in the agent filesystem. Browse, edit, and save files without download/upload cycles."
}
```

## `types.ts`

```ts
export interface FileEntry {
  name: string;
  path: string;
  isDirectory: boolean;
}

export type SaveStatus = "saved" | "unsaved" | "saving" | "error";
```

## `styles.css`

```css
/* Editor textarea: match line-number height precisely */
textarea {
  font-family: ui-monospace, SFMono-Regular, "SF Mono", Menlo, Consolas, monospace;
  tab-size: 2;
}

/* Hide scrollbar on line numbers (synced via JS) */
.overflow-hidden::-webkit-scrollbar {
  display: none;
}

/* Thin scrollbar on file browser */
.overflow-y-auto::-webkit-scrollbar {
  width: 4px;
}
.overflow-y-auto::-webkit-scrollbar-thumb {
  background: oklch(var(--bc) / 0.15);
  border-radius: 2px;
}
```

## `components/FileBrowser.tsx`

```tsx
import React, { useState, useEffect } from "react";
import {
  Folder,
  FolderOpen,
  FileText,
  ChevronRight,
  ChevronDown,
  RefreshCw,
  Home,
} from "lucide-react";
import { FileEntry } from "../types";

interface FileBrowserProps {
  rootPath: string;
  currentFile: string | null;
  onFileSelect: (path: string) => void;
  onRootChange: (path: string) => void;
}

interface DirNode {
  entries: FileEntry[];
  expanded: boolean;
  loaded: boolean;
}

async function listDirectory(dirPath: string): Promise<FileEntry[]> {
  try {
    const result = await window.tasklet.runCommand(
      `ls -1F "${dirPath}" 2>/dev/null`,
      30
    );
    if (result.exitCode !== 0 || !result.log.trim()) return [];
    return result.log
      .trim()
      .split("\n")
      .map((line) => {
        const isDir = line.endsWith("/");
        const name = isDir ? line.slice(0, -1) : line.replace(/[*@=|]$/, "");
        const fullPath = `${dirPath}/${name}`;
        return { name, path: fullPath, isDirectory: isDir };
      })
      .filter((e) => !e.name.startsWith("."))
      .sort((a, b) => {
        if (a.isDirectory && !b.isDirectory) return -1;
        if (!a.isDirectory && b.isDirectory) return 1;
        return a.name.localeCompare(b.name);
      });
  } catch {
    return [];
  }
}

const EDITABLE_EXT = new Set([
  "md",
  "txt",
  "csv",
  "json",
  "yaml",
  "yml",
  "xml",
  "html",
  "css",
  "js",
  "ts",
  "tsx",
  "jsx",
  "py",
  "sh",
  "sql",
  "toml",
  "ini",
  "cfg",
  "log",
  "env",
  "",
]);

function isEditable(name: string): boolean {
  const ext = name.includes(".") ? name.split(".").pop()?.toLowerCase() || "" : "";
  return EDITABLE_EXT.has(ext);
}

export const FileBrowser: React.FC<FileBrowserProps> = ({
  rootPath,
  currentFile,
  onFileSelect,
  onRootChange,
}) => {
  const [dirs, setDirs] = useState<Record<string, DirNode>>({});
  const [loading, setLoading] = useState(false);
  const [rootInput, setRootInput] = useState(rootPath);

  async function loadDir(dirPath: string) {
    if (dirs[dirPath]?.loaded) return;
    try {
      const entries = await listDirectory(dirPath);
      setDirs((prev) => ({
        ...prev,
        [dirPath]: { entries, expanded: true, loaded: true },
      }));
    } catch {
      setDirs((prev) => ({
        ...prev,
        [dirPath]: { entries: [], expanded: true, loaded: true },
      }));
    }
  }

  useEffect(() => {
    setDirs({});
    setLoading(true);
    loadDir(rootPath).finally(() => setLoading(false));
  }, [rootPath]);

  useEffect(() => {
    setRootInput(rootPath);
  }, [rootPath]);

  function toggleDir(dirPath: string) {
    const node = dirs[dirPath];
    if (!node?.loaded) {
      loadDir(dirPath);
    } else {
      setDirs((prev) => ({
        ...prev,
        [dirPath]: { ...prev[dirPath], expanded: !prev[dirPath].expanded },
      }));
    }
  }

  async function refreshAll() {
    setDirs({});
    setLoading(true);
    await loadDir(rootPath);
    setLoading(false);
  }

  function renderEntries(dirPath: string, depth: number): React.ReactNode {
    const node = dirs[dirPath];
    if (!node?.loaded) return null;
    if (!node.expanded && depth > 0) return null;

    return node.entries.map((entry) => {
      const isActive = entry.path === currentFile;
      const indent = depth * 16;

      if (entry.isDirectory) {
        const childNode = dirs[entry.path];
        const isExpanded = childNode?.expanded || false;
        return (
          <div key={entry.path}>
            <button
              className={`flex items-center gap-1 w-full text-left px-2 py-1 text-sm hover:bg-base-300 rounded ${
                isExpanded ? "text-base-content" : "text-base-content/70"
              }`}
              style={{ paddingLeft: `${indent + 8}px` }}
              onClick={() => toggleDir(entry.path)}
            >
              {isExpanded ? (
                <ChevronDown size={14} className="opacity-60 shrink-0" />
              ) : (
                <ChevronRight size={14} className="opacity-60 shrink-0" />
              )}
              {isExpanded ? (
                <FolderOpen size={14} className="text-warning shrink-0" />
              ) : (
                <Folder size={14} className="text-warning shrink-0" />
              )}
              <span className="truncate">{entry.name}</span>
            </button>
            {renderEntries(entry.path, depth + 1)}
          </div>
        );
      }

      const editable = isEditable(entry.name);
      return (
        <button
          key={entry.path}
          className={`flex items-center gap-1 w-full text-left px-2 py-1 text-sm rounded truncate ${
            isActive
              ? "bg-primary/20 text-primary"
              : editable
              ? "hover:bg-base-300 text-base-content/80"
              : "text-base-content/30 cursor-not-allowed"
          }`}
          style={{ paddingLeft: `${indent + 8}px` }}
          onClick={() => editable && onFileSelect(entry.path)}
          disabled={!editable}
        >
          <FileText size={14} className="shrink-0 opacity-60" />
          <span className="truncate">{entry.name}</span>
        </button>
      );
    });
  }

  return (
    <div className="flex flex-col h-full">
      <div className="p-2 border-b border-base-300">
        <form
          className="flex gap-1"
          onSubmit={(e) => {
            e.preventDefault();
            onRootChange(rootInput);
          }}
        >
          <label className="input input-bordered input-sm flex items-center gap-1 grow">
            <Home size={14} className="opacity-50 shrink-0" />
            <input
              type="text"
              className="grow"
              value={rootInput}
              onChange={(e) => setRootInput(e.target.value)}
              placeholder="/agent/home"
            />
          </label>
          <button
            type="button"
            className="btn btn-ghost btn-sm btn-square"
            onClick={refreshAll}
            title="Refresh"
          >
            <RefreshCw size={14} className={loading ? "animate-spin" : ""} />
          </button>
        </form>
      </div>
      <div className="overflow-y-auto flex-1 py-1">
        {loading && !dirs[rootPath]?.loaded ? (
          <div className="flex justify-center py-4">
            <span className="loading loading-spinner loading-sm text-primary" />
          </div>
        ) : (
          renderEntries(rootPath, 0)
        )}
      </div>
    </div>
  );
};
```

## `components/Editor.tsx`

```tsx
import React, { useEffect, useRef, useCallback, useState } from "react";
import { SaveStatus } from "../types";

interface EditorProps {
  content: string;
  onChange: (content: string) => void;
  filePath: string | null;
  saveStatus: SaveStatus;
  onSave: () => void;
  lineCount: number;
}

const BASE_LINE_HEIGHT = 24; // leading-6 = 1.5rem = 24px

export const Editor: React.FC<EditorProps> = ({
  content,
  onChange,
  filePath,
  saveStatus,
  onSave,
  lineCount,
}) => {
  const textareaRef = useRef<HTMLTextAreaElement>(null);
  const lineNumbersRef = useRef<HTMLDivElement>(null);
  const mirrorRef = useRef<HTMLDivElement>(null);
  const [wordWrap, setWordWrap] = useState(false);
  const [lineHeights, setLineHeights] = useState<number[]>([]);

  const handleKeyDown = useCallback(
    (e: KeyboardEvent) => {
      if ((e.ctrlKey || e.metaKey) && e.key === "s") {
        e.preventDefault();
        onSave();
      }
    },
    [onSave]
  );

  useEffect(() => {
    window.addEventListener("keydown", handleKeyDown);
    return () => window.removeEventListener("keydown", handleKeyDown);
  }, [handleKeyDown]);

  /* ── Measure visual height of each line when word-wrap is on ── */
  const measureLineHeights = useCallback(() => {
    if (!wordWrap || !textareaRef.current || !mirrorRef.current) {
      setLineHeights([]);
      return;
    }

    const ta = textareaRef.current;
    const mirror = mirrorRef.current;
    const cs = window.getComputedStyle(ta);

    // Make the mirror match the textarea's text-layout exactly
    Object.assign(mirror.style, {
      position: "absolute",
      visibility: "hidden",
      top: "0",
      left: "0",
      height: "auto",
      width: ta.clientWidth + "px",
      font: cs.font,
      letterSpacing: cs.letterSpacing,
      wordSpacing: cs.wordSpacing,
      paddingLeft: cs.paddingLeft,
      paddingRight: cs.paddingRight,
      paddingTop: "0",
      paddingBottom: "0",
      border: "none",
      boxSizing: "border-box",
      whiteSpace: "pre-wrap",
      overflowWrap: "break-word",
      wordBreak: "normal",
      lineHeight: cs.lineHeight,
      tabSize: (cs as any).tabSize || "2",
    });

    const textLines = content.split("\n");
    // Build all divs in a fragment for a single reflow
    const fragment = document.createDocumentFragment();
    const divs: HTMLDivElement[] = [];
    for (const line of textLines) {
      const div = document.createElement("div");
      div.textContent = line || "\u200b"; // zero-width space keeps empty-line height
      fragment.appendChild(div);
      divs.push(div);
    }
    mirror.innerHTML = "";
    mirror.appendChild(fragment);

    // Read all heights in one pass
    const heights = divs.map((d) => d.offsetHeight);
    setLineHeights(heights);
  }, [content, wordWrap]);

  useEffect(() => {
    measureLineHeights();
  }, [measureLineHeights]);

  // Re-measure when the textarea resizes (e.g. panel dragged wider/narrower)
  useEffect(() => {
    if (!wordWrap || !textareaRef.current) return;
    const obs = new ResizeObserver(() => measureLineHeights());
    obs.observe(textareaRef.current);
    return () => obs.disconnect();
  }, [wordWrap, measureLineHeights]);

  /* ── Scroll sync ── */
  function syncScroll() {
    if (textareaRef.current && lineNumbersRef.current) {
      lineNumbersRef.current.scrollTop = textareaRef.current.scrollTop;
    }
  }

  /* ── Tab key inserts 2 spaces ── */
  function handleTab(e: React.KeyboardEvent<HTMLTextAreaElement>) {
    if (e.key === "Tab") {
      e.preventDefault();
      const ta = textareaRef.current;
      if (!ta) return;
      const start = ta.selectionStart;
      const end = ta.selectionEnd;
      const value = ta.value;
      const newValue = value.substring(0, start) + "  " + value.substring(end);
      onChange(newValue);
      requestAnimationFrame(() => {
        ta.selectionStart = ta.selectionEnd = start + 2;
      });
    }
  }

  /* ── Empty state ── */
  if (!filePath) {
    return (
      <div className="flex items-center justify-center h-full text-base-content/40">
        <div className="text-center">
          <p className="text-lg">Select a file to edit</p>
          <p className="text-sm mt-1">Browse the file tree on the left</p>
        </div>
      </div>
    );
  }

  const lines = Array.from({ length: lineCount }, (_, i) => i + 1);
  const hasHeights = wordWrap && lineHeights.length === lineCount;

  return (
    <div className="flex flex-col h-full">
      {/* Hidden mirror used to measure wrapped-line heights */}
      <div ref={mirrorRef} aria-hidden="true" />

      {/* Toolbar */}
      <div className="flex items-center justify-between px-3 py-1.5 border-b border-base-300 bg-base-200">
        <span className="text-sm text-base-content/60 truncate font-mono">
          {filePath}
        </span>
        <div className="flex items-center gap-3 shrink-0">
          <label className="flex items-center gap-1.5 cursor-pointer">
            <span className="text-xs text-base-content/50">Wrap</span>
            <input
              type="checkbox"
              className="toggle toggle-xs toggle-primary"
              checked={wordWrap}
              onChange={() => setWordWrap(!wordWrap)}
            />
          </label>
          <span
            className={
              "text-xs " +
              (saveStatus === "saved"
                ? "text-success"
                : saveStatus === "unsaved"
                ? "text-warning"
                : saveStatus === "saving"
                ? "text-info"
                : "text-error")
            }
          >
            {saveStatus === "saved"
              ? "Saved"
              : saveStatus === "unsaved"
              ? "Unsaved changes"
              : saveStatus === "saving"
              ? "Saving..."
              : "Save failed"}
          </span>
          <button
            className="btn btn-primary btn-xs"
            onClick={onSave}
            disabled={saveStatus === "saved" || saveStatus === "saving"}
          >
            Save
          </button>
        </div>
      </div>

      {/* Editor body */}
      <div className="flex flex-1 overflow-hidden">
        {/* Line numbers */}
        <div
          ref={lineNumbersRef}
          className="overflow-hidden select-none text-right pr-2 pl-2 py-2 bg-base-200 border-r border-base-300 font-mono text-sm leading-6 text-base-content/30"
          style={{ minWidth: "3rem" }}
        >
          {lines.map((n, i) => (
            <div
              key={n}
              style={
                hasHeights
                  ? { height: lineHeights[i], lineHeight: BASE_LINE_HEIGHT + "px" }
                  : undefined
              }
            >
              {n}
            </div>
          ))}
        </div>

        {/* Textarea */}
        <textarea
          ref={textareaRef}
          className="flex-1 resize-none bg-base-100 text-base-content font-mono text-sm leading-6 p-2 outline-none"
          value={content}
          onChange={(e) => onChange(e.target.value)}
          onScroll={syncScroll}
          onKeyDown={handleTab}
          spellCheck={false}
          wrap={wordWrap ? "soft" : "off"}
          style={wordWrap ? { overflowX: "hidden" } : { overflowX: "auto" }}
        />
      </div>
    </div>
  );
};

```

## `components/CreateFileModal.tsx`

```tsx
import React, { useState } from "react";
import { FilePlus } from "lucide-react";

interface CreateFileModalProps {
  rootPath: string;
  onCreated: (path: string) => void;
}

export const CreateFileModal: React.FC<CreateFileModalProps> = ({
  rootPath,
  onCreated,
}) => {
  const [open, setOpen] = useState(false);
  const [filename, setFilename] = useState("");
  const [creating, setCreating] = useState(false);

  async function handleCreate() {
    if (!filename.trim()) return;
    setCreating(true);
    const fullPath = `${rootPath}/${filename.trim()}`;
    try {
      await window.tasklet.writeFileToDisk(fullPath, "");
      onCreated(fullPath);
      setFilename("");
      setOpen(false);
    } catch (err) {
      console.error("Failed to create file:", err);
    } finally {
      setCreating(false);
    }
  }

  return (
    <>
      <button
        className="btn btn-ghost btn-sm btn-square"
        onClick={() => setOpen(true)}
        title="New file"
      >
        <FilePlus size={14} />
      </button>

      {open && (
        <div className="modal modal-open">
          <div className="modal-box bg-base-200">
            <h3 className="font-bold text-lg mb-4">Create New File</h3>
            <p className="text-sm text-base-content/60 mb-2">
              Relative to: {rootPath}
            </p>
            <input
              type="text"
              className="input input-bordered w-full"
              placeholder="filename.md"
              value={filename}
              onChange={(e) => setFilename(e.target.value)}
              onKeyDown={(e) => e.key === "Enter" && handleCreate()}
              autoFocus
            />
            <div className="modal-action">
              <button
                className="btn btn-ghost"
                onClick={() => {
                  setOpen(false);
                  setFilename("");
                }}
              >
                Cancel
              </button>
              <button
                className="btn btn-primary"
                onClick={handleCreate}
                disabled={!filename.trim() || creating}
              >
                {creating ? (
                  <span className="loading loading-spinner loading-xs" />
                ) : (
                  "Create"
                )}
              </button>
            </div>
          </div>
          <div className="modal-backdrop" onClick={() => setOpen(false)} />
        </div>
      )}
    </>
  );
};
```

## `app.tsx`

```tsx
import React, { useState, useCallback, useRef } from "react";
import { createRoot } from "react-dom/client";
import { FileBrowser } from "./components/FileBrowser";
import { Editor } from "./components/Editor";
import { CreateFileModal } from "./components/CreateFileModal";
import { SaveStatus } from "./types";

const DEFAULT_ROOT = "/agent/home";

function App() {
  const [rootPath, setRootPath] = useState(DEFAULT_ROOT);
  const [currentFile, setCurrentFile] = useState<string | null>(null);
  const [content, setContent] = useState("");
  const [originalContent, setOriginalContent] = useState("");
  const [saveStatus, setSaveStatus] = useState<SaveStatus>("saved");
  const [loading, setLoading] = useState(false);
  const [sidebarWidth, setSidebarWidth] = useState(260);
  const [refreshKey, setRefreshKey] = useState(0);
  const dragRef = useRef<{ startX: number; startWidth: number } | null>(null);

  async function openFile(path: string) {
    setLoading(true);
    try {
      const text = await window.tasklet.readFileFromDisk(path);
      setContent(text);
      setOriginalContent(text);
      setCurrentFile(path);
      setSaveStatus("saved");
    } catch (err) {
      console.error("Failed to read file:", err);
      setContent("Error: could not read " + path);
      setOriginalContent("");
      setCurrentFile(path);
      setSaveStatus("error");
    } finally {
      setLoading(false);
    }
  }

  function handleContentChange(newContent: string) {
    setContent(newContent);
    setSaveStatus(newContent === originalContent ? "saved" : "unsaved");
  }

  const handleSave = useCallback(async () => {
    if (!currentFile || content === originalContent) return;
    setSaveStatus("saving");
    try {
      await window.tasklet.writeFileToDisk(currentFile, content);
      setOriginalContent(content);
      setSaveStatus("saved");
    } catch (err) {
      console.error("Failed to save file:", err);
      setSaveStatus("error");
    }
  }, [currentFile, content, originalContent]);

  function handleFileCreated(path: string) {
    setRefreshKey((k) => k + 1);
    openFile(path);
  }

  function handleMouseDown(e: React.MouseEvent) {
    e.preventDefault();
    dragRef.current = { startX: e.clientX, startWidth: sidebarWidth };

    function onMouseMove(ev: MouseEvent) {
      if (!dragRef.current) return;
      const delta = ev.clientX - dragRef.current.startX;
      const newWidth = Math.max(180, Math.min(500, dragRef.current.startWidth + delta));
      setSidebarWidth(newWidth);
    }

    function onMouseUp() {
      dragRef.current = null;
      document.removeEventListener("mousemove", onMouseMove);
      document.removeEventListener("mouseup", onMouseUp);
    }

    document.addEventListener("mousemove", onMouseMove);
    document.addEventListener("mouseup", onMouseUp);
  }

  const lineCount = content.split("\n").length;

  return (
    <div className="flex h-screen bg-base-100 overflow-hidden">
      <div
        className="flex flex-col bg-base-200 border-r border-base-300 shrink-0"
        style={{ width: sidebarWidth + "px" }}
      >
        <div className="flex items-center justify-between px-2 py-1.5 border-b border-base-300">
          <span className="text-xs font-semibold text-base-content/50 uppercase tracking-wider">
            Files
          </span>
          <CreateFileModal rootPath={rootPath} onCreated={handleFileCreated} />
        </div>
        <FileBrowser
          key={refreshKey}
          rootPath={rootPath}
          currentFile={currentFile}
          onFileSelect={openFile}
          onRootChange={setRootPath}
        />
      </div>

      <div
        className="w-1 cursor-col-resize hover:bg-primary/30 active:bg-primary/50 shrink-0"
        onMouseDown={handleMouseDown}
      />

      <div className="flex-1 min-w-0">
        {loading ? (
          <div className="flex items-center justify-center h-full">
            <span className="loading loading-spinner loading-lg text-primary"></span>
          </div>
        ) : (
          <Editor
            content={content}
            onChange={handleContentChange}
            filePath={currentFile}
            saveStatus={saveStatus}
            onSave={handleSave}
            lineCount={lineCount}
          />
        )}
      </div>
    </div>
  );
}

createRoot(document.getElementById("root")!).render(<App />);
```

---

## Notes

- Runs on Alpine Linux (BusyBox). Uses `ls -1F` instead of GNU `find -printf`.
- Timeout on directory listing is 30s to handle cloud-mounted FUSE storage.
- Editable file types: md, txt, csv, json, yaml, yml, xml, html, css, js, ts, tsx, jsx, py, sh, sql, toml, ini, cfg, log, env, and extensionless files.
- Word wrap toggle in the editor toolbar. Off by default.
- Ctrl+S / Cmd+S to save. Tab key inserts two spaces.
- Sidebar is resizable by dragging the divider.
- Uses DaisyUI + Tailwind (provided by Tasklet runtime). Lucide icons for the file tree.
