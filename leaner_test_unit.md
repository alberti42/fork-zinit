# Leaner CI Test Suite

## Current State

The CI runs **14 parallel jobs** (7 test files × 2 OS: macOS + Ubuntu).
Every single job — regardless of what it tests — runs the same setup:

```yaml
- name: install homebrew
  uses: Homebrew/actions/setup-homebrew@master

- name: install dependencies
  run: brew install --force --overwrite autoconf automake binutils byacc
       coreutils curl gettext gnu-sed libevent libtool libuv lua lua@5.4
       make ncurses ninja parallel pkg-config texinfo unzip xz zsh

- name: install musl          # Linux only
  run: apt-get install autoconf automake autotools-dev build-essential
       byacc gcc libc6-dev libevent-dev libncurses5-dev ninja-build ...

- name: install zunit
  run: git clone zunit && ./configure && make install   # compiled from source every time
```

This means a C compiler, Lua, cmake, ninja, and 20+ other packages are
installed 14 times in parallel on every PR, even for jobs that only need `zsh`.

**Observed CI time: 10+ minutes per PR.**

---

## Root Cause Analysis by Test File

### `commands.zunit` — needs: `zsh` only
Tests zinit subcommands: `self-update`, `plugins`, `delete`, etc.
No network downloads of binaries, no compilation, no heavy dependencies.
Currently wastes time installing a full C toolchain it never uses.

### `compile.zunit` — needs: `zsh`, `git`, `perl`
Tests zinit's `zcompile` ice. Clones a small zsh plugin from GitHub and
compiles it with `zcompile`. No C compiler, no Lua, no cmake.

### `ices.zunit` — needs: `zsh`, `git`, `make` (minimal)
Tests `mv`, `cp`, `atclone`, `make`, `completions` ices.
**Already well designed**: the `make` ice test uses a synthetic inline
`Makefile` (`printf 'all:\n\ttouch whatever\n'`) — no real project cloned.
No C compiler needed beyond what ships with the OS.

### `annexes.zunit` — needs: `zsh`, `git`
Tests zinit annexes. No source compilation observed.

### `gh-r.zunit` — needs: `zsh`, `git`, network
Downloads **pre-built binaries** from GitHub Releases for ~50 tools
(bat, atuin, fd, ripgrep, etc.). No compilation at all.
A C compiler is completely irrelevant here.

### `snippets.zunit` — needs: `zsh`, `git`, network
Tests zinit snippet loading. No compilation.

### `plugins.zunit` — needs: `zsh`, `git`, C compiler, cmake, Lua, ninja, ...
**The problematic file.** Builds the following from source on every CI run:

| Project | Build system | Why it is a problem |
|---------|-------------|---------------------|
| `@htop-dev/htop` | autoconf + make | htop has nothing to do with zinit |
| `@neovim/neovim` | cmake + ninja | one of the heaviest C builds in open source |
| `@vim/vim` | autoconf + make | same |
| `@tmux/tmux` | autoconf + make | same |
| `@mirror/ncurses` | configure + make | same |
| `@jqlang/jq` | autoconf + make | same |
| `@Koihik/LuaFormatter` | cmake + **Lua** | only reason Lua is installed |
| `@universal-ctags/ctags` | autoconf + make | same pattern |
| `@cmatsuoka/figlet` | make | lighter but still unnecessary |
| `@zsh-users/zsh` | configure + make | building zsh to test a zsh plugin manager |

**What these tests actually verify:** that zinit correctly invokes
`make`/`cmake`/`configure` ices. They do NOT need to build real projects
to prove this. A trivial synthetic project would serve the same purpose.

**Additional risk:** if neovim, vim, or any upstream project has a broken
commit or a flaky build, it blocks your PR even though your change has
nothing to do with that project.

---

## Proposed Improvements

### 1. Per-job dependency installation

Move dependency installation inside conditional steps based on which
test is running. Only `plugins.zunit` needs the heavy toolchain.

```yaml
- name: install build dependencies
  if: matrix.zunit_test == 'plugins'
  run: brew install autoconf automake cmake ninja lua lua@5.4 ...

- name: install zsh (all other jobs)
  if: matrix.zunit_test != 'plugins'
  run: brew install zsh
```

This alone would make 13 out of 14 jobs significantly faster.

### 2. Cache zunit installation

zunit is compiled from source on every run. It almost never changes.
Use GitHub Actions cache:

```yaml
- name: cache zunit
  uses: actions/cache@v4
  with:
    path: ~/.local
    key: zunit-${{ runner.os }}-${{ hashFiles('**/zunit/**') }}

- name: install zunit
  if: steps.cache-zunit.outputs.cache-hit != 'true'
  run: git clone --depth 1 https://github.com/zdharma-continuum/zunit
       && cd zunit && ./configure --prefix=$HOME/.local && make install
```

### 3. Replace `plugins.zunit` real builds with synthetic targets

The goal of testing `make`/`cmake`/`configure` ices is to verify zinit
invokes the build system correctly, not to build real software.

Replace each heavy build with a minimal synthetic project hosted in
`zdharma-continuum/null` or a dedicated `zdharma-continuum/zinit-test-fixtures`
repo containing:

**Synthetic `make` target** (already done correctly in `ices.zunit`):
```zsh
run zinit as"null" id-as"test/make" \
    atclone"printf 'all:\n\ttouch build-artifact\n' > Makefile" \
    make"all" for zdharma-continuum/null
assert "$ZPLUGINS/test---make/build-artifact" is_file
```

**Synthetic `cmake` target**:
```zsh
# A repo with a 3-line CMakeLists.txt that creates a file
run zinit as"null" id-as"test/cmake" cmake for zdharma-continuum/zinit-test-fixtures
assert "$ZPLUGINS/test---cmake/build-artifact" is_file
```

**Synthetic `configure` target**:
```zsh
# A repo with a minimal autoconf setup
run zinit as"null" id-as"test/configure" configure make for zdharma-continuum/zinit-test-fixtures
assert $state equals 0
```

This would reduce `plugins.zunit` from ~8 minutes to ~10 seconds and
eliminate the need for Lua, cmake, ninja, and the full C toolchain.

### 4. Cache Homebrew packages

For the packages that are genuinely needed, use Homebrew caching:

```yaml
- name: cache homebrew
  uses: actions/cache@v4
  with:
    path: |
      ~/Library/Caches/Homebrew
      /usr/local/Cellar
    key: brew-${{ runner.os }}-${{ hashFiles('.github/workflows/tests.yaml') }}
```

### 5. Consider dropping the Linux matrix for `plugins.zunit`

Building neovim, vim, tmux on both macOS and Ubuntu doubles the cost of
the heaviest job. The build ice behaviour is OS-agnostic — zinit calls
`make` the same way on both platforms. Testing on one OS is sufficient
for this class of test.

---

## Expected Impact

| Job | Current | After improvements |
|-----|---------|-------------------|
| `commands` | ~8 min | ~1 min |
| `compile` | ~8 min | ~1 min |
| `ices` | ~8 min | ~1 min |
| `annexes` | ~8 min | ~1 min |
| `gh-r` | ~8 min | ~3 min (network bound) |
| `snippets` | ~8 min | ~1 min |
| `plugins` | ~10 min | ~2 min (synthetic builds) |

Total CI time per PR: from **10+ minutes** to **~3 minutes** (bounded by
`gh-r` network downloads).

---

## Priority Order for PRs

1. **Per-job dependency installation** — highest impact, lowest risk, pure
   workflow change, no test logic touched.
2. **Cache zunit** — trivial addition to the workflow.
3. **Replace `plugins.zunit` real builds with synthetic targets** — requires
   creating a `zinit-test-fixtures` repo or extending `zdharma-continuum/null`,
   then rewriting `plugins.zunit`. Highest impact on test correctness and
   reliability.
4. **Cache Homebrew** — marginal gains after the above, but worth doing.
5. **Drop Linux matrix for `plugins`** — optional, discuss with maintainers.
