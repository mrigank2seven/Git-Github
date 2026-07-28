# Eleventy Architecture Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the single 2,700-line `index.html` with an Eleventy-built static site — same HTML/JS authoring style, same visual result, but split into maintainable, independently-loadable pages with shared layout/styles/scripts.

**Architecture:** Eleventy (`@11ty/eleventy` v3) processes plain `.html` files in `src/` using its default Liquid template engine (front matter + `{{ }}`/`{% %}` work in `.html` files with zero config — verified against the official Eleventy docs). A shared `_includes/base.html` layout provides the header/nav/footer chrome; each page supplies its own body content and declares which page-specific JS it needs via front matter.

**Tech Stack:** Node.js, `@11ty/eleventy` ^3.1.6 (only dependency), vanilla JS (no framework), plain CSS.

## Global Constraints

- **No visual, content, or behavioral changes.** The rebuilt site must look and behave identically to the current `index.html`. Every content-copying step below cites exact, verified line ranges from the current `index.html` and must be copied verbatim — no paraphrasing, no "improving" the prose.
- **No Markdown.** All content stays in HTML, using Eleventy's default Liquid processing of `.html` files.
- **No automated tests.** Verification is `npm run build` succeeding plus a manual browser pass per task (per the spec's explicit non-goal — the project has no test suite today and this is a pure restructuring).
- **No hosting/CI changes in this plan.** Local build only.
- Every reference to "the original file" below means the **current, unmodified** `index.html` at the repo root. Do not modify or delete it until the final task (Task 7) — every other task's line-number citations depend on it staying exactly as it is today.

## A note on "copy verbatim" steps

Several steps below say "copy lines A–B of `index.html` verbatim" instead of pasting thousands of characters of existing guide prose into this plan. Each such range was verified against the current file using a structural div-nesting parse (not eyeballed), so the range is exact and unambiguous — this is a mechanical extraction, not a judgment call. Steps use `sed -n 'A,Bp' index.html >> target-file` so the operation is scriptable and exactly reproducible. All **new** code (JS, config, templates) is written out in full below — nothing there is a citation.

---

### Task 1: Project scaffold and build tooling

**Files:**
- Create: `package.json`
- Create: `.eleventy.js`
- Create: `src/` directory tree (empty except structure): `src/_includes/`, `src/_data/`, `src/assets/css/`, `src/assets/js/`, `src/guide/`
- Modify: `.gitignore`

**Interfaces:**
- Produces: Eleventy config with `dir.input = "src"`, `dir.output = "_site"`, `dir.includes = "_includes"`, `dir.data = "_data"`, and passthrough copy of `src/assets` → `_site/assets`. All later tasks depend on this exact directory mapping.

- [ ] **Step 1: Create `package.json`**

```json
{
  "name": "git-github-guide",
  "private": true,
  "version": "1.0.0",
  "description": "A single-page, interactive reference guide for Git and GitHub.",
  "scripts": {
    "build": "eleventy",
    "serve": "eleventy --serve"
  },
  "devDependencies": {
    "@11ty/eleventy": "^3.1.6"
  }
}
```

- [ ] **Step 2: Install dependencies**

Run: `npm install`
Expected: `node_modules/` and `package-lock.json` created, no errors.

- [ ] **Step 3: Create `.eleventy.js`**

```js
module.exports = function (eleventyConfig) {
    eleventyConfig.addPassthroughCopy("src/assets");

    return {
        dir: {
            input: "src",
            output: "_site",
            includes: "_includes",
            data: "_data",
        },
    };
};
```

- [ ] **Step 4: Create the `src/` directory skeleton**

Run: `mkdir -p src/_includes src/_data src/assets/css src/assets/js src/guide`

- [ ] **Step 5: Update `.gitignore`**

Append to the end of the existing `.gitignore`:

```
# Eleventy
node_modules/
_site/
```

- [ ] **Step 6: Verify the build tooling works with an empty input tree**

Run: `npm run build`
Expected: Exits 0. Eleventy reports writing 0 files (there are no templates yet) — this confirms the config and dependency install are correct before any content work begins.

- [ ] **Step 7: Commit**

```bash
git add package.json package-lock.json .eleventy.js .gitignore
git commit -m "chore: scaffold Eleventy build tooling"
```

(Note: `src/` is empty except for now-untracked directories; git does not track empty directories, so nothing else to add yet.)

---

### Task 2: Shared layout, global styles/scripts, and Home (Installation) page

**Files:**
- Create: `src/assets/css/styles.css`
- Create: `src/assets/js/theme.js`
- Create: `src/assets/js/copy-code.js`
- Create: `src/assets/js/os-tabs.js`
- Create: `src/_includes/base.html`
- Create: `src/index.html`

**Interfaces:**
- Consumes: Eleventy config from Task 1.
- Produces: `_includes/base.html` layout accepting front matter `title` (string), `navActive` (one of `installation`/`commands`/`beginner`/`visual-guide`), `extraScripts` (array of root-relative script paths, optional). Every later page task must set `layout: base.html` and one of these `navActive` values. Produces global `theme.js`/`copy-code.js` loaded on every page via base.html.

- [ ] **Step 1: Extract global CSS**

Run:
```bash
sed -n '8,888p' index.html > src/assets/css/styles.css
```

This copies the full `<style>` block contents (excluding the `<style>`/`</style>` tags themselves) verbatim.

Then apply two small, necessary edits — the original CSS styled `.nav-tab` and `.filter-btn` only as `<button>` elements; both classes are now also applied to `<a>` tags (Task 2's nav links, Task 4's guide sub-nav links) which have a default underline that buttons don't. Add `text-decoration: none;` to both rules so anchors render identically to how the buttons looked:

Find in `src/assets/css/styles.css`:
```css
        .nav-tab {
            padding: 16px 24px;
            border: none;
            background: none;
            cursor: pointer;
            font-size: 15px;
            font-weight: 600;
            color: var(--text-muted);
            transition: color 0.25s ease, background 0.25s ease, transform 0.25s var(--spring);
            white-space: nowrap;
            border-bottom: 3px solid transparent;
            margin-bottom: -2px;
        }
```
Replace with (added line only):
```css
        .nav-tab {
            padding: 16px 24px;
            border: none;
            background: none;
            cursor: pointer;
            font-size: 15px;
            font-weight: 600;
            color: var(--text-muted);
            transition: color 0.25s ease, background 0.25s ease, transform 0.25s var(--spring);
            white-space: nowrap;
            border-bottom: 3px solid transparent;
            margin-bottom: -2px;
            text-decoration: none;
        }
```

Find:
```css
        .filter-btn {
            padding: 10px 18px;
            border: 2px solid var(--border-color);
            background: var(--surface);
            border-radius: 25px;
            cursor: pointer;
            font-size: 14px;
            font-weight: 600;
            transition: color 0.25s ease, background 0.25s ease, border-color 0.25s ease, transform 0.25s var(--spring);
            color: var(--text-muted);
        }
```
Replace with (added line only):
```css
        .filter-btn {
            padding: 10px 18px;
            border: 2px solid var(--border-color);
            background: var(--surface);
            border-radius: 25px;
            cursor: pointer;
            font-size: 14px;
            font-weight: 600;
            transition: color 0.25s ease, background 0.25s ease, border-color 0.25s ease, transform 0.25s var(--spring);
            color: var(--text-muted);
            text-decoration: none;
        }
```

- [ ] **Step 2: Create `src/assets/js/theme.js`**

```js
function applyThemeIcon() {
    const explicit = document.documentElement.getAttribute('data-theme');
    const isDark = explicit ? explicit === 'dark' : window.matchMedia('(prefers-color-scheme: dark)').matches;
    document.getElementById('themeToggle').textContent = isDark ? '☀️' : '🌙';
}

function toggleTheme() {
    const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
    const current = document.documentElement.getAttribute('data-theme') || (prefersDark ? 'dark' : 'light');
    const next = current === 'dark' ? 'light' : 'dark';
    document.documentElement.setAttribute('data-theme', next);
    localStorage.setItem('theme', next);
    applyThemeIcon();
}

(function initTheme() {
    const saved = localStorage.getItem('theme');
    if (saved) document.documentElement.setAttribute('data-theme', saved);
})();

applyThemeIcon();
```

- [ ] **Step 3: Create `src/assets/js/copy-code.js`**

```js
function copyCode(btn) {
    const code = btn.parentElement.textContent.replace('Copy', '').trim();
    navigator.clipboard.writeText(code).then(() => {
        const orig = btn.textContent;
        btn.textContent = '✓ Copied!';
        setTimeout(() => btn.textContent = orig, 2000);
    });
}
```

- [ ] **Step 4: Create `src/assets/js/os-tabs.js`**

```js
function switchOS(e, osName) {
    document.querySelectorAll('.os-content').forEach(c => c.classList.remove('active'));
    document.getElementById(osName).classList.add('active');
    document.querySelectorAll('.os-tab').forEach(t => t.classList.remove('active'));
    e.target.classList.add('active');
}
```

- [ ] **Step 5: Create `src/_includes/base.html`**

This preserves the original DOM nesting exactly as verified (the footer is nested *inside* `.content`, not a sibling — confirmed by parsing the original file's div structure — so `.content`'s `padding: 40px` applies around the footer exactly as it does today):

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{ title }} — Git Complete Guide</title>
    <link rel="stylesheet" href="/assets/css/styles.css">
</head>
<body>
    <div class="container">
        <div class="header">
            <button class="theme-toggle" id="themeToggle" onclick="toggleTheme()" aria-label="Toggle dark mode">🌙</button>
            <h1><span class="rocket">🚀</span> Complete Git & GitHub Guide</h1>
            <p>Installation, Setup, and Master Essential Git Commands</p>
        </div>

        <div class="nav-tabs">
            <a class="nav-tab{% if navActive == 'installation' %} active{% endif %}" href="/">📥 Installation</a>
            <a class="nav-tab{% if navActive == 'commands' %} active{% endif %}" href="/commands/">⚡ Commands</a>
            <a class="nav-tab{% if navActive == 'beginner' %} active{% endif %}" href="/guide/">🎓 Beginner Guide</a>
            <a class="nav-tab{% if navActive == 'visual-guide' %} active{% endif %}" href="/visual-guide/">🌳 Visual Guide</a>
        </div>

        <div class="content">
            <div class="tab-content active">
{{ content }}
            </div>

            <div class="footer">
                <p>🎓 <strong>Master Git & GitHub</strong> - Practice these commands regularly to become a Git expert!</p>
                <div class="footer-links">
                    <a href="https://git-scm.com/doc" target="_blank">Official Git Docs</a>
                    <a href="https://github.com" target="_blank">GitHub</a>
                </div>
            </div>
        </div>
    </div>

    <script src="/assets/js/theme.js" defer></script>
    <script src="/assets/js/copy-code.js" defer></script>
    {% for src in extraScripts %}<script src="{{ src }}" defer></script>
    {% endfor %}
</body>
</html>
```

- [ ] **Step 6: Create `src/index.html` (Installation content, serves as the site's `/`)**

Per the approved spec, this page *is* the Installation content — there is no separate landing page and no `/installation/` URL, matching today's default-tab behavior exactly.

Run:
```bash
cat > src/index.html <<'FRONTMATTER'
---
layout: base.html
title: Installation
navActive: installation
extraScripts:
  - /assets/js/os-tabs.js
---
FRONTMATTER
sed -n '910,1642p' index.html >> src/index.html
```

This appends the verified inner content of the original `<div id="installation">...</div>` (lines 910–1642) verbatim after the front matter.

- [ ] **Step 7: Build and verify**

Run: `npm run build`
Expected: Exits 0. `_site/index.html`, `_site/assets/css/styles.css`, `_site/assets/js/theme.js`, `_site/assets/js/copy-code.js`, `_site/assets/js/os-tabs.js` all exist.

- [ ] **Step 8: Manual browser verification**

Run: `npm run serve`, open the printed local URL.

Check:
- Header (rocket emoji title, tagline, theme toggle) renders identically to the current site.
- Nav bar shows 4 items; "📥 Installation" is visually marked active; none of the nav links show an underline.
- Windows/Mac/Linux install tabs switch correctly (via `os-tabs.js`).
- "Copy" buttons on code blocks work and show "✓ Copied!" feedback.
- Theme toggle switches light/dark and the choice persists after a page reload (`localStorage`).
- Footer renders below the content with the same visual spacing/margins as the current site (not visually "boxed in" narrower than the header).

- [ ] **Step 9: Commit**

```bash
git add src/assets/css/styles.css src/assets/js/theme.js src/assets/js/copy-code.js src/assets/js/os-tabs.js src/_includes/base.html src/index.html
git commit -m "feat: add shared layout, global assets, and Installation home page"
```

---

### Task 3: Commands reference page

**Files:**
- Create: `src/_data/commands.js`
- Modify: `.eleventy.js`
- Create: `src/commands.html`
- Create: `src/assets/js/command-search.js`

**Interfaces:**
- Consumes: `base.html` layout and `navActive`/`extraScripts` front matter contract from Task 2.
- Produces: `commandsJson` Eleventy shortcode (used only by this page) that serializes the same array Eleventy's data cascade exposes as `commands`.

- [ ] **Step 1: Create `src/_data/commands.js`**

The command-reference array, moved verbatim out of the original inline `<script>` (was lines 2475–2526):

```js
module.exports = [
    { name: 'git config --global user.name "Name"', category: 'basic', description: 'Set your name for commits', usage: 'git config --global user.name "John Doe"', when: 'First time setup' },
    { name: 'git config --global user.email "email@example.com"', category: 'basic', description: 'Set your email for commits', usage: 'git config --global user.email "john@example.com"', when: 'First time setup' },
    { name: 'git init', category: 'basic', description: 'Create a new Git repository', usage: 'git init', when: 'Start a brand new project' },
    { name: 'git clone <url>', category: 'basic', description: 'Copy a repository from GitHub', usage: 'git clone https://github.com/user/repo.git', when: 'Download an existing project' },
    { name: 'git config --global init.defaultBranch main', category: 'basic', description: 'Set the default branch name for new repos', usage: 'git config --global init.defaultBranch main', when: 'Configure once after installing Git' },
    { name: 'git status', category: 'changes', description: 'See which files have changed', usage: 'git status', when: 'Check changes before committing' },
    { name: 'git add .', category: 'changes', description: 'Stage all changed files', usage: 'git add .', when: 'Before committing all changes' },
    { name: 'git add <file>', category: 'changes', description: 'Stage a specific file', usage: 'git add index.html', when: 'Stage specific files only' },
    { name: 'git commit -m "message"', category: 'changes', description: 'Save your staged changes', usage: 'git commit -m "Fixed login bug"', when: 'After staging files' },
    { name: 'git diff', category: 'changes', description: 'See what changed in files', usage: 'git diff', when: 'Review changes before staging' },
    { name: 'git commit --amend', category: 'changes', description: 'Edit your most recent commit', usage: 'git commit --amend -m "Updated message"', when: 'Fix the last commit message or add forgotten changes' },
    { name: 'git rm <file>', category: 'changes', description: 'Remove a file and stage the deletion', usage: 'git rm old-file.txt', when: 'Delete a tracked file from the project' },
    { name: 'git mv <old> <new>', category: 'changes', description: 'Rename or move a file and stage the change', usage: 'git mv old-name.txt new-name.txt', when: 'Rename or relocate a tracked file' },
    { name: 'git branch', category: 'branch', description: 'List all branches', usage: 'git branch', when: 'See available branches' },
    { name: 'git branch <name>', category: 'branch', description: 'Create a new branch', usage: 'git branch feature/new-login', when: 'Start work on a feature' },
    { name: 'git checkout <branch>', category: 'branch', description: 'Switch to a branch', usage: 'git checkout feature/new-login', when: 'Work on different branch' },
    { name: 'git checkout -b <branch>', category: 'branch', description: 'Create and switch branches', usage: 'git checkout -b feature/profile', when: 'Quick branch creation' },
    { name: 'git merge <branch>', category: 'branch', description: 'Combine branches', usage: 'git merge feature/new-login', when: 'After feature is done' },
    { name: 'git branch -d <branch>', category: 'branch', description: 'Delete a branch', usage: 'git branch -d feature/old', when: 'Clean up merged branches' },
    { name: 'git branch -m <new-name>', category: 'branch', description: 'Rename the current branch', usage: 'git branch -m feature/renamed', when: 'Fix a branch name typo' },
    { name: 'git switch <branch>', category: 'branch', description: 'Switch to an existing branch', usage: 'git switch main', when: 'Modern alternative to git checkout' },
    { name: 'git switch -c <branch>', category: 'branch', description: 'Create and switch to a new branch', usage: 'git switch -c feature/search', when: 'Modern alternative to git checkout -b' },
    { name: 'git remote add origin <url>', category: 'remote', description: 'Connect to GitHub', usage: 'git remote add origin https://github.com/user/repo.git', when: 'Link local repo to GitHub' },
    { name: 'git push', category: 'remote', description: 'Upload commits to GitHub', usage: 'git push', when: 'After committing' },
    { name: 'git push origin <branch>', category: 'remote', description: 'Push specific branch', usage: 'git push origin feature/new-feature', when: 'Push new branches' },
    { name: 'git pull', category: 'remote', description: 'Download latest changes', usage: 'git pull', when: 'Before starting work' },
    { name: 'git fetch', category: 'remote', description: 'Fetch remote changes', usage: 'git fetch', when: 'See changes without merging' },
    { name: 'git remote -v', category: 'remote', description: 'List remote repositories and their URLs', usage: 'git remote -v', when: 'Check which remote you are connected to' },
    { name: 'git remote remove <name>', category: 'remote', description: 'Disconnect a remote repository', usage: 'git remote remove origin', when: 'Unlink or replace a remote' },
    { name: 'git push --force-with-lease', category: 'remote', description: 'Safely force-push over a rewritten history', usage: 'git push --force-with-lease', when: 'After rebasing a branch already pushed' },
    { name: 'git log', category: 'history', description: 'See commit history', usage: 'git log', when: 'Review project history' },
    { name: 'git log --oneline', category: 'history', description: 'Short commit history', usage: 'git log --oneline', when: 'Quick overview' },
    { name: 'git show <commit>', category: 'history', description: 'See commit details', usage: 'git show a1b2c3d', when: 'Review specific commit' },
    { name: 'git blame <file>', category: 'history', description: 'See who changed each line', usage: 'git blame app.js', when: 'Find who made a change' },
    { name: 'git log --graph --oneline', category: 'history', description: 'See branch history as a visual graph', usage: 'git log --graph --oneline --all', when: 'Understand how branches diverged and merged' },
    { name: 'git log -p <file>', category: 'history', description: 'See the full diff for each commit to a file', usage: 'git log -p index.html', when: 'Trace exactly how a file evolved' },
    { name: 'git diff <branch1>..<branch2>', category: 'history', description: 'Compare changes between two branches', usage: 'git diff main..feature/login', when: 'Preview what a merge or PR will bring in' },
    { name: 'git restore <file>', category: 'undo', description: 'Discard file changes', usage: 'git restore style.css', when: 'Undo mistakes' },
    { name: 'git reset HEAD~1', category: 'undo', description: 'Undo last commit', usage: 'git reset HEAD~1', when: 'Keep work, undo commit' },
    { name: 'git reset --hard HEAD~1', category: 'undo', description: 'Delete last commit', usage: 'git reset --hard HEAD~1', when: 'Remove commit completely' },
    { name: 'git revert <commit>', category: 'undo', description: 'Create undo commit', usage: 'git revert a1b2c3d', when: 'Safe undo of old commits' },
    { name: 'git stash', category: 'undo', description: 'Save work temporarily', usage: 'git stash', when: 'Switch branches without committing' },
    { name: 'git stash pop', category: 'undo', description: 'Restore stashed work', usage: 'git stash pop', when: 'Get back saved work' },
    { name: 'git clean -fd', category: 'undo', description: 'Remove untracked files and directories', usage: 'git clean -fd', when: 'Clear out generated files not tracked by Git' },
    { name: 'git reflog', category: 'undo', description: 'See a log of where HEAD has been', usage: 'git reflog', when: 'Recover a commit after a bad reset' },
    { name: 'git tag', category: 'tags', description: 'List all tags in the repository', usage: 'git tag', when: 'See available release tags' },
    { name: 'git tag -a <tag> -m "message"', category: 'tags', description: 'Create an annotated tag', usage: 'git tag -a v1.0.0 -m "First release"', when: 'Mark a specific commit as a release' },
    { name: 'git push origin <tag>', category: 'tags', description: 'Push a single tag to GitHub', usage: 'git push origin v1.0.0', when: 'Share a release tag with others' },
    { name: 'git push --tags', category: 'tags', description: 'Push all local tags to GitHub', usage: 'git push --tags', when: 'Publish every tag at once' },
    { name: 'git tag -d <tag>', category: 'tags', description: 'Delete a local tag', usage: 'git tag -d v1.0.0', when: 'Remove a mistaken or outdated tag' },
];
```

- [ ] **Step 2: Add a `commandsJson` shortcode to `.eleventy.js`**

Eleventy's default Liquid engine has no built-in JSON-serialization filter (verified against the official filters docs), so add a shortcode — shortcodes write their return value to the output raw, avoiding any autoescape ambiguity.

Modify `.eleventy.js` to:

```js
const commands = require("./src/_data/commands.js");

module.exports = function (eleventyConfig) {
    eleventyConfig.addPassthroughCopy("src/assets");
    eleventyConfig.addShortcode("commandsJson", () => JSON.stringify(commands));

    return {
        dir: {
            input: "src",
            output: "_site",
            includes: "_includes",
            data: "_data",
        },
    };
};
```

- [ ] **Step 3: Create `src/assets/js/command-search.js`**

Adapted from the original `render`/`filter` functions (were lines 2567–2609). Reads the data from `window.__COMMANDS__` (set inline by `commands.html`) instead of a page-global `commands` const, and no longer calls `render(commands)` on load — the grid already has the full, server-rendered list from `commands.html`'s Liquid loop (Step 4), so the initial paint needs no JS and works with JS disabled:

```js
(function () {
    const commands = window.__COMMANDS__ || [];
    const grid = document.getElementById('grid');
    const search = document.getElementById('searchInput');
    const filters = document.querySelectorAll('.filter-btn');
    let currentFilter = 'all';

    function render(filtered) {
        grid.innerHTML = '';
        if (filtered.length === 0) {
            document.getElementById('empty').style.display = 'block';
            document.getElementById('showing').textContent = '0';
            return;
        }
        document.getElementById('empty').style.display = 'none';
        document.getElementById('showing').textContent = filtered.length;

        filtered.forEach(cmd => {
            const card = document.createElement('div');
            card.className = 'command-card';
            card.innerHTML = `
                <div class="cmd-category cat-${cmd.category}">${cmd.category}</div>
                <div class="cmd-name">$ ${cmd.name}</div>
                <div class="cmd-description">${cmd.description}</div>
                <div class="cmd-usage">$ ${cmd.usage}</div>
                <div class="cmd-when"><div class="cmd-when-title">When to use</div>${cmd.when}</div>
            `;
            grid.appendChild(card);
        });
    }

    function filter() {
        let result = commands;
        if (currentFilter !== 'all') result = result.filter(c => c.category === currentFilter);
        const q = search.value.toLowerCase();
        if (q) result = result.filter(c => c.name.includes(q) || c.description.includes(q));
        render(result);
    }

    filters.forEach(btn => {
        btn.addEventListener('click', () => {
            filters.forEach(b => b.classList.remove('active'));
            btn.classList.add('active');
            currentFilter = btn.dataset.filter;
            filter();
        });
    });

    search.addEventListener('input', filter);
})();
```

- [ ] **Step 4: Create `src/commands.html`**

Run:
```bash
cat > src/commands.html <<'FRONTMATTER'
---
layout: base.html
title: Commands
navActive: commands
extraScripts:
  - /assets/js/command-search.js
---
FRONTMATTER
sed -n '1647,1676p' index.html >> src/commands.html
```

This copies the h2 title, stats cards, search box, and filter buttons verbatim (were lines 1647–1676, ending right before the empty `commands-grid` div). Then append the server-rendered grid, the empty-state block (verbatim, was lines 1678–1681), and the JSON handoff script:

```bash
cat >> src/commands.html <<'BODY'
                <div class="commands-grid" id="grid">
{% for cmd in commands %}
                    <div class="command-card">
                        <div class="cmd-category cat-{{ cmd.category }}">{{ cmd.category }}</div>
                        <div class="cmd-name">$ {{ cmd.name }}</div>
                        <div class="cmd-description">{{ cmd.description }}</div>
                        <div class="cmd-usage">$ {{ cmd.usage }}</div>
                        <div class="cmd-when"><div class="cmd-when-title">When to use</div>{{ cmd.when }}</div>
                    </div>
{% endfor %}
                </div>
                <div class="empty-state" id="empty" style="display: none;">
                    <div class="empty-state-icon">🔍</div>
                    <h3>No Commands Found</h3>
                </div>

                <script>window.__COMMANDS__ = {% commandsJson %};</script>
BODY
```

- [ ] **Step 5: Build and verify**

Run: `npm run build`
Expected: Exits 0. `_site/commands/index.html` exists.

Run: `grep -c 'command-card' _site/commands/index.html`
Expected: `50` (one card per command, server-rendered).

- [ ] **Step 6: Manual browser verification**

Run: `npm run serve`, navigate to Commands.

Check:
- "Total Commands" and "Showing" stat cards both read 50 on load, without needing to type anything (confirms server-side render worked).
- Typing "branch" in the search box filters the grid live; the "Showing" count updates.
- Clicking a category filter button (e.g. "Branches") filters correctly and marks that button active.
- A query with no matches (e.g. "zzz") shows the "No Commands Found" empty state.
- View page source: confirm `window.__COMMANDS__ = [...]` contains valid, unescaped JSON (no `&quot;`/`&#39;` HTML entities corrupting the JS). If entities are present, the Liquid output is being autoescaped — in that case wrap the shortcode call as `{% commandsJson | safe %}` instead of `{% commandsJson %}` and rebuild.

- [ ] **Step 7: Commit**

```bash
git add src/_data/commands.js .eleventy.js src/commands.html src/assets/js/command-search.js
git commit -m "feat: add Commands reference page with server-rendered + searchable command grid"
```

---

### Task 4: Beginner's Guide hub (Key Concepts) and shared guide navigation

**Files:**
- Create: `src/_includes/guide-nav.html`
- Create: `src/guide/index.html`

**Interfaces:**
- Consumes: `base.html` layout (`navActive: beginner`) from Task 2.
- Produces: `guide-nav.html` partial, consumed by every guide page in this and the next task. Expects a `guideActive` front-matter variable with one of: `concepts`, `first-project`, `pull-requests`, `github-actions`. This exact set of 4 strings is the contract Task 5's three pages must also use.

- [ ] **Step 1: Create `src/_includes/guide-nav.html`**

Adapted from the original shared Beginner Guide header (was lines 1686–1699): same title and intro tip-box, and the four sub-nav buttons become real links (Liquid `{% if %}` sets `active` based on `guideActive`, inherited automatically from the including page's front matter):

```html
<h2 style="font-size: 28px; margin-bottom: 30px; color: #333;">🎓 Beginner's Guide to Git & GitHub</h2>

<div class="tip-box">
    <div style="font-weight: 700; margin-bottom: 8px;">What is Git?</div>
    <p>Git is a tool that tracks changes in your code. Think of it like a time machine for your project - you can save snapshots and go back anytime!</p>
</div>

<div style="display: flex; gap: 10px; margin-bottom: 30px; flex-wrap: wrap;">
    <a class="filter-btn{% if guideActive == 'concepts' %} active{% endif %}" href="/guide/">Key Concepts</a>
    <a class="filter-btn{% if guideActive == 'first-project' %} active{% endif %}" href="/guide/first-project/">First Project</a>
    <a class="filter-btn{% if guideActive == 'pull-requests' %} active{% endif %}" href="/guide/pull-requests/">Pull Requests</a>
    <a class="filter-btn{% if guideActive == 'github-actions' %} active{% endif %}" href="/guide/github-actions/">GitHub Actions</a>
</div>
```

- [ ] **Step 2: Create `src/guide/index.html` (Key Concepts)**

Run:
```bash
cat > src/guide/index.html <<'FRONTMATTER'
---
layout: base.html
title: Beginner's Guide
navActive: beginner
guideActive: concepts
---
{% include "guide-nav.html" %}
FRONTMATTER
sed -n '1703,1765p' index.html >> src/guide/index.html
```

This appends the verified inner content of the original "Concepts" section (was lines 1703–1765) verbatim.

- [ ] **Step 3: Build and verify**

Run: `npm run build`
Expected: Exits 0. `_site/guide/index.html` exists.

- [ ] **Step 4: Manual browser verification**

Run: `npm run serve`, navigate to "🎓 Beginner Guide" from the top nav.

Check:
- Title, intro tip-box, and 4 sub-nav links render.
- "Key Concepts" sub-nav link is visually marked active.
- Concepts content matches the original Beginner Guide's default (Key Concepts) tab content exactly.
- The other 3 sub-nav links are present but will 404 until Task 5 — expected at this point, not a defect.

- [ ] **Step 5: Commit**

```bash
git add src/_includes/guide-nav.html src/guide/index.html
git commit -m "feat: add Beginner's Guide hub (Key Concepts) and shared guide nav partial"
```

---

### Task 5: Beginner's Guide walkthrough pages

**Files:**
- Create: `src/guide/first-project.html`
- Create: `src/guide/pull-requests.html`
- Create: `src/guide/github-actions.html`

**Interfaces:**
- Consumes: `base.html` layout and `guide-nav.html` partial (with its `guideActive` contract) from Tasks 2 and 4.

- [ ] **Step 1: Create `src/guide/first-project.html`**

Run:
```bash
cat > src/guide/first-project.html <<'FRONTMATTER'
---
layout: base.html
title: My First Project
navActive: beginner
guideActive: first-project
---
{% include "guide-nav.html" %}
FRONTMATTER
sed -n '1770,1888p' index.html >> src/guide/first-project.html
```

- [ ] **Step 2: Create `src/guide/pull-requests.html`**

Run:
```bash
cat > src/guide/pull-requests.html <<'FRONTMATTER'
---
layout: base.html
title: Pull Requests
navActive: beginner
guideActive: pull-requests
---
{% include "guide-nav.html" %}
FRONTMATTER
sed -n '1893,2073p' index.html >> src/guide/pull-requests.html
```

- [ ] **Step 3: Create `src/guide/github-actions.html`**

Run:
```bash
cat > src/guide/github-actions.html <<'FRONTMATTER'
---
layout: base.html
title: GitHub Actions
navActive: beginner
guideActive: github-actions
---
{% include "guide-nav.html" %}
FRONTMATTER
sed -n '2078,2310p' index.html >> src/guide/github-actions.html
```

- [ ] **Step 4: Build and verify**

Run: `npm run build`
Expected: Exits 0. `_site/guide/first-project/index.html`, `_site/guide/pull-requests/index.html`, `_site/guide/github-actions/index.html` all exist.

- [ ] **Step 5: Manual browser verification**

Run: `npm run serve`. Starting from `/guide/`, click through all four sub-nav links in sequence (Key Concepts → First Project → Pull Requests → GitHub Actions → back to Key Concepts).

Check:
- Each link resolves without a 404.
- The correct sub-nav item is marked active on each page.
- Content on each page matches the corresponding original walkthrough exactly (7-step first project walkthrough, PR walkthrough with best practices, GitHub Actions walkthrough).
- Code blocks with "Copy" buttons on these pages work (confirms `copy-code.js`, loaded globally via `base.html`, works outside the Installation page too).

- [ ] **Step 6: Commit**

```bash
git add src/guide/first-project.html src/guide/pull-requests.html src/guide/github-actions.html
git commit -m "feat: add Beginner's Guide walkthrough pages"
```

---

### Task 6: Visual Guide page

**Files:**
- Create: `src/assets/js/visual-guide.js`
- Create: `src/visual-guide.html`

**Interfaces:**
- Consumes: `base.html` layout (`navActive: visual-guide`) from Task 2.

- [ ] **Step 1: Create `src/assets/js/visual-guide.js`**

Combines the two step-through interactions verbatim (were lines 2612–2691: VCS-basics stepper and branching-diagram stepper, plus their init calls):

```js
const vcsSteps = [
    { working: 'Modified', staging: 'Empty', repo: 'Previous commits only', remote: 'Not pushed yet', activeArrow: null, caption: "You've edited app.py — Git marks your Working Directory as Modified.", command: null },
    { working: 'Modified', staging: 'Empty', repo: 'Previous commits only', remote: 'Not pushed yet', activeArrow: 2, caption: 'git add copies the current snapshot of app.py into the Staging Area (the index).', command: 'git add app.py' },
    { working: 'Modified', staging: 'Staged', repo: 'Previous commits only', remote: 'Not pushed yet', activeArrow: null, caption: 'app.py is now Staged — ready to be included in the next commit.', command: null },
    { working: 'Modified', staging: 'Staged', repo: 'Previous commits only', remote: 'Not pushed yet', activeArrow: 3, caption: 'git commit takes everything in the Staging Area and saves it as a permanent snapshot inside .git.', command: 'git commit -m "Update app.py"' },
    { working: 'Clean', staging: 'Empty', repo: 'Commit c4a1b2 ✓ Saved', remote: 'Not pushed yet', activeArrow: null, caption: "Repository updated with a new commit. Your Working Directory is Clean again — commit didn't delete anything, it just saved a snapshot.", command: null },
    { working: 'Clean', staging: 'Empty', repo: 'Commit c4a1b2 ✓ Saved', remote: 'Commit c4a1b2 ✓ Pushed', activeArrow: 4, caption: 'git push uploads your local commits to the remote repository on GitHub, so others can see and pull them.', command: 'git push' },
];
let vcsIndex = 0;

function renderVcsStep() {
    const s = vcsSteps[vcsIndex];
    const workingBadge = document.getElementById('vg-working-badge');
    const stagingBadge = document.getElementById('vg-staging-badge');
    workingBadge.textContent = s.working;
    workingBadge.className = 'vg-badge vg-' + s.working.toLowerCase();
    stagingBadge.textContent = s.staging;
    stagingBadge.className = 'vg-badge vg-' + s.staging.toLowerCase();
    document.getElementById('vg-repo-commit').textContent = s.repo;
    document.getElementById('vg-remote-commit').textContent = s.remote;

    [2, 3, 4].forEach(n => document.getElementById('vg-arrow-' + n).classList.remove('vg-arrow-active'));
    if (s.activeArrow) document.getElementById('vg-arrow-' + s.activeArrow).classList.add('vg-arrow-active');

    document.getElementById('vg-vcs-step').textContent = vcsIndex + 1;
    document.getElementById('vg-vcs-caption').textContent = s.caption;
    const cmdEl = document.getElementById('vg-vcs-command');
    if (s.command) { cmdEl.textContent = '$ ' + s.command; cmdEl.style.display = 'block'; }
    else { cmdEl.style.display = 'none'; }
}

function vcsStep(delta) {
    vcsIndex = Math.max(0, Math.min(vcsSteps.length - 1, vcsIndex + delta));
    renderVcsStep();
}
function vcsReset() { vcsIndex = 0; renderVcsStep(); }

const branchSteps = [
    { nodes: ['A', 'B', 'C'], edges: ['AB', 'BC'], main: { x: 170, y: 170 }, feature: null, head: 'main', caption: 'main has three commits: A, B, C.', command: null },
    { nodes: ['A', 'B', 'C'], edges: ['AB', 'BC'], main: { x: 170, y: 170 }, feature: { x: 230, y: 170 }, head: 'feature', caption: 'git checkout -b feature creates a new pointer at the same commit as main, and moves HEAD onto it.', command: 'git checkout -b feature' },
    { nodes: ['A', 'B', 'C', 'D'], edges: ['AB', 'BC', 'CD'], main: { x: 170, y: 170 }, feature: { x: 230, y: 90 }, head: 'feature', caption: 'Committing on feature moves the feature pointer forward — main is untouched.', command: 'git commit -m "Add feature part 1"' },
    { nodes: ['A', 'B', 'C', 'D'], edges: ['AB', 'BC', 'CD'], main: { x: 170, y: 170 }, feature: { x: 230, y: 90 }, head: 'main', caption: 'Switching back to main moves HEAD, but doesn\'t change any commits — main and feature still point where they did.', command: 'git checkout main' },
    { nodes: ['A', 'B', 'C', 'D', 'F'], edges: ['AB', 'BC', 'CD', 'CF'], main: { x: 290, y: 170 }, feature: { x: 230, y: 90 }, head: 'main', caption: 'main gets a new commit too — main and feature have now diverged from their shared commit C.', command: 'git commit -m "Fix typo on main"' },
    { nodes: ['A', 'B', 'C', 'D', 'F', 'E'], edges: ['AB', 'BC', 'CD', 'CF', 'DE'], main: { x: 290, y: 170 }, feature: { x: 350, y: 90 }, head: 'feature', caption: 'Back on feature, another commit lands — feature is now two commits ahead of C.', command: 'git checkout feature\ngit commit -m "Add feature part 2"' },
    { nodes: ['A', 'B', 'C', 'D', 'F', 'E', 'M'], edges: ['AB', 'BC', 'CD', 'CF', 'DE', 'FM', 'EM'], main: { x: 410, y: 170 }, feature: { x: 350, y: 90 }, head: 'main', caption: 'git merge feature creates a merge commit M that joins both histories back together.', command: 'git merge feature' },
];
let branchIndex = 0;

function renderBranchStep() {
    const s = branchSteps[branchIndex];
    document.querySelectorAll('.vg-node').forEach(n => n.classList.toggle('vg-visible', s.nodes.includes(n.dataset.node)));
    document.querySelectorAll('.vg-edge').forEach(e => e.classList.toggle('vg-visible', s.edges.includes(e.dataset.edge)));

    document.getElementById('vg-pointer-main').setAttribute('transform', `translate(${s.main.x}, ${s.main.y})`);
    const featurePointer = document.getElementById('vg-pointer-feature');
    if (s.feature) {
        featurePointer.style.display = '';
        featurePointer.setAttribute('transform', `translate(${s.feature.x}, ${s.feature.y})`);
    } else {
        featurePointer.style.display = 'none';
    }

    const headPos = s.head === 'main' ? s.main : s.feature;
    document.getElementById('vg-head').setAttribute('transform', `translate(${headPos.x}, ${headPos.y - 54})`);

    document.getElementById('vg-branch-step').textContent = branchIndex + 1;
    document.getElementById('vg-branch-caption').textContent = s.caption;
    const cmdEl = document.getElementById('vg-branch-command');
    if (s.command) { cmdEl.textContent = '$ ' + s.command; cmdEl.style.display = 'block'; }
    else { cmdEl.style.display = 'none'; }
}

function branchStep(delta) {
    branchIndex = Math.max(0, Math.min(branchSteps.length - 1, branchIndex + delta));
    renderBranchStep();
}
function branchReset() { branchIndex = 0; renderBranchStep(); }

renderVcsStep();
renderBranchStep();
```

- [ ] **Step 2: Create `src/visual-guide.html`**

Run:
```bash
cat > src/visual-guide.html <<'FRONTMATTER'
---
layout: base.html
title: Visual Guide
navActive: visual-guide
extraScripts:
  - /assets/js/visual-guide.js
---
FRONTMATTER
sed -n '2317,2442p' index.html >> src/visual-guide.html
```

This appends the verified inner content of the original `<div id="visual-guide">...</div>` (was lines 2317–2442) verbatim — the legend, the VCS Basics section, and the Branching section (including the inline SVG that `visual-guide.js` queries by id/data attribute).

- [ ] **Step 3: Build and verify**

Run: `npm run build`
Expected: Exits 0. `_site/visual-guide/index.html` exists.

Run: `grep -c 'id="vg-branch-svg"' _site/visual-guide/index.html`
Expected: `1`.

- [ ] **Step 4: Manual browser verification**

Run: `npm run serve`, navigate to "🌳 Visual Guide".

Check:
- Legend renders (main branch / feature branch / commit / pointer / HEAD / file / staging).
- VCS Basics step controls (prev/next/reset) update the Working/Staging/Repo/Remote badges and caption correctly through all 6 steps.
- Branching diagram step controls animate nodes, edges, branch pointers, and the HEAD label correctly through all 7 steps, matching the original diagram's behavior.

- [ ] **Step 5: Commit**

```bash
git add src/assets/js/visual-guide.js src/visual-guide.html
git commit -m "feat: add Visual Guide page with VCS-basics and branching interactions"
```

---

### Task 7: Final cutover, README update, full verification pass

**Files:**
- Delete: `index.html` (root, legacy monolith — now fully superseded by `src/` → `_site/`)
- Modify: `README.md`

**Interfaces:**
- Consumes: all pages/assets produced by Tasks 2–6 (this is the final integration checkpoint against the spec's Verification section).

- [ ] **Step 1: Update `README.md` usage instructions**

Read the current `README.md` "Usage" section (lines 14–23) — it currently says to open `index.html` directly in a browser. Replace that section with:

````markdown
## Usage

```bash
git clone <repo-url>
cd Git-Github
npm install
npm run serve   # local dev server with live reload
```

Or build the static site without serving it:

```bash
npm run build   # outputs to _site/
```
````

Leave the "Features" section unchanged — it still accurately describes the site's content.

- [ ] **Step 2: Full manual verification pass (per the design spec's Verification section)**

Run: `npm run build` — confirm it completes without errors (item 1).

Run: `npm run serve` and check, across every page (`/`, `/commands/`, `/guide/`, `/guide/first-project/`, `/guide/pull-requests/`, `/guide/github-actions/`, `/visual-guide/`):
- Nav links resolve correctly between all pages, with the correct nav item marked active on each (item 2).
- Theme toggle persists across page loads via `localStorage` and renders correctly (icon + colors) on every page (item 3).
- Command reference search/filter behaves identically to the original (item 4, re-verified after all pages exist alongside it).
- SVG visual guide interactions still work (item 5, re-verified in the final integrated site).
- Responsive layout is intact at mobile (~375px), tablet (~768px), and desktop (~1200px+) widths — resize the browser or use devtools device emulation on at least the Installation and Commands pages.
- Each page's content matches the corresponding section of the pre-migration `index.html` with no accidental content loss (item 7) — spot-check by diffing rendered text against the original tab content for at least the Installation and one Beginner's Guide walkthrough page.

- [ ] **Step 3: Remove the legacy root `index.html`**

Now fully superseded by `src/index.html` → `_site/index.html`; keeping both around risks confusion about which file is authoritative.

Run: `git rm index.html`

- [ ] **Step 4: Commit**

```bash
git add README.md
git commit -m "chore: cut over to Eleventy build, remove legacy monolithic index.html"
```
