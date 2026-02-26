# Bug Analysis: Spurious `chmod +x` on Python Library Files in `ziextract`

## Summary

When zinit installs a plugin via `from'gh-r'` (GitHub releases) or any tarball download, it
unconditionally calls `ziextract`, which uses the `file(1)` utility to detect executables and
applies `chmod a+x` to all matches. Due to a quirk in the `file` magic database, **every Python
source file** containing ordinary Python syntax (imports, class/function definitions, etc.) is
labelled `"Python script text executable"` — regardless of whether it has a shebang line or the
execute permission bit set. Zinit's filter catches all of them and marks them executable.

---

## Reproduction

Install a plugin from a GitHub release that ships a Python package:

```zsh
zinit lucid wait depth=1 as'null' from'gh' nocompile'!' from'gh-r' \
    id-as'tmux-plugins/tmux-fzf-links' extract'!' atpull'%atclone' \
    for @alberti42/tmux-fzf-links
```

Observed output:

```
Successfully extracted and marked executable the appropriate files
(fzf-links.tmux, setup.py, __main__.py, colors.py, configs.py,…)
contained in `tmux-fzf-links-1.4.13.zip'.
All the extracted 13 executables are available in the INSTALLED_EXECS array.
```

The zip archive contains **only 2 files with the execute bit set**
(`fzf-links.tmux` and `__main__.py`), yet zinit reports 13 executables and
applies `chmod a+x` to all of them — including pure library modules such as
`colors.py`, `configs.py`, `errors_types.py`, etc.

---

## Root Cause

### 1. The `file(1)` magic database labels all Python source as "executable"

The system magic database `/usr/share/file/magic/python` contains rules that emit
`"Python script text executable"` for **any** file matching common Python syntax patterns,
regardless of shebangs or filesystem permissions:

```
# from module.submodule import func1, func2
0   search/8192   import
>0  regex   \^from[\040\t]+([A-Za-z0-9_]|\\.)+[\040\t]+import.*$   Python script text executable

# class name[(base classes,)]:
0   search/8192   class
>0  regex   \^class\ [_[:alpha:]]+(\\(.*\\))?(\ )*:([\ \t]+pass)?$   Python script text executable

# def name(*args, **kwargs):
0   search/8192   def\
>0  regex    \^[[:space:]]{0,50}def\ {1,50}[_a-zA-Z]{1,100}
>>&0 regex   \\(([[:alpha:]*_,\ ]){0,255}\\):$   Python script text executable

# try: / except:
0   search/4096   try:
>&0 regex   \^[[:space:]]*except.*:$   Python script text executable
```

Verified by fresh extraction — `colors.py` (no shebang, `644` permissions in the zip):

```
$ file /tmp/zinit-test-extract/.../colors.py
colors.py: Python script text executable, ASCII text
```

This behaviour is **consistent across macOS and Linux** (both `file` 5.x). The word
`"executable"` here means *"this is a recognised executable scripting language"*, not
*"this file has the execute permission bit"* or *"this file has a shebang"*.

### 2. Zinit's filter catches every Python file

In `ziextract()` (`zinit-install.zsh:1754–1764`):

```zsh
local -aU execs
# Step 1 – collect ALL regular files in the extracted tree
execs=( **/*~(._zinit(|/*)|.git(|/*)|.svn(|/*)|.hg(|/*)|._backup(|/*))(DN-.) )

if [[ ${#execs} -gt 0 && -n $execs ]] {
    # Step 2 – run `file` on every collected file
    execs=( ${(@f)"$( file ${execs[@]} )"} )
    # Step 3 – keep only lines whose `file` description contains "executable"
    execs=( "${(M)execs[@]:#[^(:]##:*executable*}" )
    # Step 4 – strip the description, keep paths only
    execs=( "${execs[@]/(#b)([^(:]##):*/${match[1]}}" )
}

# Step 5 – chmod a+x everything that survived the filter
if [[ ${#execs} -gt 0 ]] {
    command chmod a+x "${execs[@]}"
    ...
}
```

The filter on step 3 — `*executable*` — matches the word "executable" anywhere in the
`file` output. Because the `file` magic database unconditionally appends `executable` to the
description of every Python source file, **all Python files pass the filter** and all receive
`chmod a+x`.

The `__init__.py` file in the test package was the only exception; being nearly empty, it
contained no Python syntax patterns and was reported as plain `"ASCII text"`.

---

## Affected Code Paths

`ziextract` is called from three different sites in `zinit-install.zsh`, with very different
triggering conditions:

| Call site | Code path | Triggered |
|-----------|-----------|-----------|
| Line 272 | Tarball download (`from'url'`) | **Always**, even without `extract` ice (`--move` is the default when ice is unset) |
| Line 400 | GitHub releases (`from'gh-r'`) | **Always**, unconditionally for every downloaded release asset |
| Line 414 | Cygwin packages | **Always** |
| `∞zinit-extract-hook` → line 2275 | Regular git plugin/snippet install | **Only** when `extract` ice is explicitly set — guarded by `(( ${+ICE[extract]} )) || return 0` |

### Consequence

- **Regular git clone** (`zinit light user/plugin`, `zinit load`, etc.): `ziextract` is
  **never called** unless `extract` ice is present. The bug does not manifest.
- **`from'gh-r'`, tarball, or Cygwin downloads**: `ziextract` is **always called**. The
  bug fires on every install and update, independent of whether `extract` ice is set.

This explains why the bug was not observed on Linux without the `extract` ice: the Linux test
used a regular git clone, which never enters `ziextract`.

It also means that in the original invocation the `extract'!'` ice is **redundant** for
triggering extraction — `from'gh-r'` already calls `ziextract` unconditionally. The `!`
modifier only influences the `--move` flag passed to `ziextract`.

---

## Why This Is Not a `file` Bug

The `file(1)` man page states that the word `"executable"` in its output means *"the file
contains the result of compiling a program in a form understandable to some UNIX kernel"* (i.e.
a binary). The magic database has extended this concept to scripting languages, where
`"executable"` means *"this is a file written in a language that can be run"*. While the
overloading of the term is debatable, it is intentional and documented behaviour of the magic
database (version tracked in the file header: `$File: python,v 1.43 2021/05/25 ...`).

The assumption that `"executable"` in `file` output reliably indicates *"this file should have
`+x`"* is the incorrect assumption — and it lives in zinit, not in `file`.

---

## Proposed Fix

Instead of relying on `file` output, zinit should use one or more of the following approaches:

### Option A — Trust the archive's stored permissions (most faithful to author intent)

Replace the `file`-based detection with the `*` zsh glob qualifier, which selects files that
**already have the execute bit set** (as stored in the archive and applied by `unzip`):

```zsh
# Collect only files that already have +x from the archive
execs=( **/*~(._zinit(|/*)|.git(|/*)|.svn(|/*)|.hg(|/*)|._backup(|/*))(DN*.) )
#                                                                           ^ * = has execute bit
```

This is portable, avoids any content inspection, and reflects exactly what the package author
intended.

### Option B — Check for shebangs explicitly

Only mark executable files that begin with `#!`:

```zsh
local -aU execs all_files
all_files=( **/*~(._zinit(|/*)|.git(|/*)|.svn(|/*)|.hg(|/*)|._backup(|/*))(DN-.) )
for f ( $all_files ) {
    [[ "$(<$f[1,2])" == '#!' ]] && execs+=( $f )
}
```

This correctly identifies scripts with shebangs and ignores library modules, regardless of
existing permissions or `file` output.

### Option C — Restrict the `file` filter to non-script types

Narrow the existing filter to exclude patterns that match scripting languages:

```zsh
# Keep binaries (ELF, Mach-O, etc.) and shebang-based scripts,
# but exclude generic "Python script", "perl script", etc.
execs=( "${(M)execs[@]:#[^(:]##:*executable*~*script*}" )
```

Though more fragile, this would at minimum exclude all "X script text executable" matches while
retaining true binary executables.

---

## Files Referenced

| File | Relevance |
|------|-----------|
| `zinit-install.zsh:1754–1788` | `ziextract()` — executable detection and `chmod` logic |
| `zinit-install.zsh:272` | `ziextract` call site for tarball downloads |
| `zinit-install.zsh:400` | `ziextract` call site for `from'gh-r'` |
| `zinit-install.zsh:2264–2276` | `∞zinit-extract-hook` — gated on `extract` ice |
| `/usr/share/file/magic/python` | Magic database rules that label all Python as "executable" |
