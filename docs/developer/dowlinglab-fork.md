# The DowlingLab fork: the "Open in Colab" button

This fork exists for one reason: **the stock `book-theme` has no Colab support.** Grepping its bundled
`build/index.js` for `colab.research` returns nothing, and `template.yml` exposes no option to switch one
on — there is nothing to configure, the feature does not exist upstream.

## What was added

Branch `colab-button` adds a `ColabLink` component in
`packages/frontmatter/src/FrontmatterBlock.tsx`:

```tsx
export function ColabLink({ sourceUrl }: { sourceUrl?: string }) {
  if (!sourceUrl || !/\.ipynb(?:$|[?#])/.test(sourceUrl)) return null;
  const colabUrl = sourceUrl.replace(
    /^https?:\/\/github\.com\//,
    'https://colab.research.google.com/github/',
  );
  ...
}
```

It needs nothing else. MyST already derives `source_url` from `project.github` in the consuming site's
`myst.yml`, so a page already carries a GitHub blob URL for its notebook; the component just rewrites that
URL and renders the guarded link — guarded on `.ipynb` so markdown pages correctly get no button. `main`
has this branch merged in (see PR #1); if your consuming site tracks `main`, you already have it.

## How consuming sites use this fork

This repo is a development workspace (React monorepo, Remix themes, `packages/*`). MyST's `site.template`
needs a much smaller, pre-built **template directory**: `template.yml`, `server.js`, `package.json`,
`package-lock.json`, `public/`, and a compiled `build/`. Two known consumers — the ACC 2026 workshop repo
[`pyomo-doe`](https://github.com/dowlinglab/pyomo-doe) and the course site
[`ndcbe/optimization`](https://github.com/ndcbe/optimization) — both follow the same two-layer pattern:

| Layer | What | Lives in |
| --- | --- | --- |
| Source of truth | `git subtree add --prefix=vendor/myst-theme https://github.com/dowlinglab/myst-theme <branch> --squash` | the consuming repo, as `vendor/myst-theme` |
| Packaged artifact | A generated template directory MyST actually reads | the consuming repo, as `themes/<name>-dist` (e.g. `pyomo-book-theme-dist`) |

The packaging step — `npm install` the vendored workspace, `npm run build --filter='./packages/*'`, then
`npm run prod:build` inside `vendor/myst-theme/themes/book`, then assemble `template.yml` + `server.js` +
`public/` + `build/` into the dist directory — is **not part of this fork**. It is a standalone script,
`scripts/build_theme_dist.sh`, copied verbatim into each consuming repo (both currently carry an identical
copy). A future consumer should copy that script from either `pyomo-doe` or `optimization` rather than
writing it from scratch.

## The mistake to avoid: do not commit the packaged `build/`

`optimization` originally committed the packaged artifact as a binary blob (copied wholesale from
`pyomo-doe`, per its `DEVELOPER.md`) rather than building it on demand. That went wrong in a way that is
worth recording here, once, so a third consumer doesn't repeat it:

`build/` is exactly the kind of directory name a generic `.gitignore` rule already excludes — most
Python-project boilerplate has a bare `build/` line for `setuptools`/`PyInstaller` output, completely
unrelated to this theme, and it will silently swallow `themes/<name>-dist/build/` too. That is exactly
what happened: the packaged theme was set up, verified locally (against an *untracked* `build/` that
existed on disk), and committed — but `build/` never actually made it into the commit, and nobody noticed
because a stale or missing artifact **does not error**; `myst build` just falls back to the stock theme
silently. `optimization`'s CI deployed nothing but the stock (Colab-less) theme for about a day and a half
before this was caught.

**The fix was not to force-commit `build/` around the `.gitignore` rule.** That just re-creates the
"someone forgot to rebuild and recommit" failure mode this whole section is about. Instead, **build the
packaged artifact fresh, in CI, every run** — `pyomo-doe`'s `deploy.yml` already did this from the start
(`run: bash ./scripts/build_theme_dist.sh`, immediately before `myst build --html`), and `optimization`'s
workflow was missing that step entirely. Once a consuming site's CI does this, `vendor/myst-theme` is the
only layer that ever needs to be a durable commit; the packaged artifact is disposable and regenerated,
so there is nothing to go stale and nothing that can be silently dropped by an unrelated `.gitignore` rule.

**If you set up a new consumer of this fork:**
1. `git subtree add` this repo at the branch you want (usually `main`, now that `colab-button` is merged).
2. Copy `scripts/build_theme_dist.sh` from `pyomo-doe` or `optimization`.
3. Add a CI step that runs it **before** `myst build`/`jupyter-book build`.
4. Do **not** rely on a committed `build/`. If you commit the non-generated parts of the dist directory
   for local-preview convenience (`template.yml`, `server.js`, `package.json`, `public/`), still `.gitignore`
   `build/` and `node_modules/` inside it — CI's job is to make those disposable, not to trust a checked-in
   copy of them.
5. Verify the button actually rendered — `exit 0` on the build does not prove it. Check the built HTML for
   the fork-specific `myst-fm-colab-link` class, not just a successful build:
   ```bash
   grep -o 'myst-fm-colab-link' _build/html/notebooks/1/some-notebook/index.html
   ```
