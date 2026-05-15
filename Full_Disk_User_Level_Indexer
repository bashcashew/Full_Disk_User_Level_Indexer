#!/usr/bin/env python3


import argparse
import os
from __future__ import annotations
from pathlib import Path
from urllib.parse import quote


## Sets up the .gitignores/ ignores
DEFAULT_IGNORES = {
    ".git", "node_modules", "__pycache__", ".idea", ".vscode",
    ".DS_Store", ".venv", "venv"
}


def escape_md(text: str) -> str:
    return text.replace("[", "\\[").replace("]", "\\]")


def rel_href(target: Path, start_dir: Path) -> str:
    rel = os.path.relpath(target, start_dir)
    rel_posix = rel.replace(os.sep, "/")
    return quote(rel_posix, safe="/")


def iter_children(directory: Path, include_hidden: bool, ignores: set[str], follow_symlinks: bool):
    try:
        with os.scandir(directory) as it:
            entries = []
            for e in it:
                name = e.name
                if (not include_hidden) and name.startswith("."):
                    continue
                if name in ignores:
                    continue
                try:
                    is_dir = e.is_dir(follow_symlinks=follow_symlinks)
                except OSError:
                    continue
                entries.append((Path(e.path), is_dir))
    except PermissionError:
        return []

    entries.sort(key=lambda t: (0 if t[1] else 1, t[0].name.casefold()))
    return entries


def build_tree_lines(
    root: Path,
    out_parent: Path,
    include_hidden: bool,
    ignores: set[str],
    max_depth: int,
    follow_symlinks: bool,
    show_root: bool,
) -> list[str]:
    lines: list[str] = []
    root_display = root.name or "/"

    if show_root:
        href = rel_href(root, out_parent)
        lines.append(f"[{escape_md(root_display)}/]({href})")

    def walk(dir_path: Path, prefixes: list[bool], depth: int):
        if max_depth and depth > max_depth:
            return
        children = iter_children(dir_path, include_hidden, ignores, follow_symlinks)
        n = len(children)
        for idx, (path, is_dir) in enumerate(children):
            last = (idx == n - 1)
            branch = "└── " if last else "├── "
            guide = "".join("│   " if cont else "    " for cont in prefixes)
            display = path.name + ("/" if is_dir else "")
            href = rel_href(path, out_parent)
            line = f"{guide}{branch}[{escape_md(display)}]({href})"
            lines.append(line)
            if is_dir:
                walk(path, prefixes + [not last], depth + 1)

    walk(root, prefixes=[] if show_root else [], depth=1)
    return lines


def _is_within(child: Path, parent: Path) -> bool:
    try:
        child.resolve().relative_to(parent.resolve())
        return True
    except Exception:
        return False


def main():
    parser = argparse.ArgumentParser(description="Generate a Markdown, clickable repo scaffold for a directory (or subfolder).")
    parser.add_argument("directory", type=Path, help="Directory to index (repo root)")
    parser.add_argument("--out", type=Path, default=None,
                        help="Output Markdown file path (default: TREE.md inside the target directory)")
    parser.add_argument("--include-hidden", action="store_true", help="Include hidden files and directories (dotfiles)")
    parser.add_argument("--ignore", action="append", default=[],
                        help="Basename to ignore (can be repeated). Example: --ignore .git --ignore node_modules")
    parser.add_argument("--max-depth", type=int, default=0,
                        help="Max depth to traverse (0 = unlimited)")
    parser.add_argument("--follow-symlinks", action="store_true",
                        help="Follow directory symlinks while walking")
    parser.add_argument("--show-root", action="store_true",
                        help="Include a top line linking to the traversal root itself")
    parser.add_argument("--extract-from", type=Path, default=None,
                        help="Only list items under this subfolder. "
                             "If relative, it is resolved inside the main DIRECTORY. "
                             "If absolute, it must be within DIRECTORY.")
    args = parser.parse_args()

    repo_root = args.directory.resolve()
    if not repo_root.exists():
        raise SystemExit(f"Error: directory not found: {repo_root}")
    if not repo_root.is_dir():
        raise SystemExit(f"Error: not a directory: {repo_root}")

    # Decide traversal root (either repo_root or a chosen subfolder)
    if args.extract_from is None:
        walk_root = repo_root
        extract_note = None
    else:
        sub = args.extract_from
        sub = (repo_root / sub).resolve() if not sub.is_absolute() else sub.resolve()
        if not sub.exists() or not sub.is_dir():
            raise SystemExit(f"Error: --extract-from path is not a directory: {sub}")
        if not _is_within(sub, repo_root):
            raise SystemExit(f"Error: --extract-from must be within the main DIRECTORY.\n  DIRECTORY: {repo_root}\n  extract-from: {sub}")
        walk_root = sub
        try:
            rel = sub.relative_to(repo_root)
            extract_note = f"(extract from `{rel.as_posix()}`)"
        except Exception:
            extract_note = f"(extract from `{sub}`)"

    # Decide output file
    out_path: Path
    if args.out is None:
        # default lives at the main directory (not the subfolder), which keeps links convenient
        out_path = (repo_root / "TREE.md").resolve()
    else:
        out_path = args.out.resolve()

    out_parent = out_path.parent

    ignores = set(DEFAULT_IGNORES)
    ignores.update(args.ignore or [])

    lines = []
    title = f"# Repository Scaffold for `{repo_root.name or '/'}`"
    if extract_note:
        title += f" {extract_note}"
    lines.append(title + "\n")
    lines.append("_Each entry below is a clickable relative link._\n\n")

    tree_lines = build_tree_lines(
        root=walk_root,
        out_parent=out_parent,
        include_hidden=args.include_hidden,
        ignores=ignores,
        max_depth=max(0, args.max_depth),
        follow_symlinks=args.follow_symlinks,
        show_root=args.show_root,
    )

    if not tree_lines:
        tree_lines.append("_(directory is empty or fully ignored)_")

    lines.append("<pre>")
    lines.extend(tree_lines)
    lines.append("</pre>\n")

    out_path.write_text("\n".join(lines), encoding="utf-8")

    print(f"Wrote scaffold to: {out_path}")
    print("\n".join(tree_lines))


if __name__ == "__main__":
    main()
