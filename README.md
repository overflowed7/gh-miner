# gh-harvest

Git collection for secret hunting. Clones an org's repositories **including pull
request refs**, dumps blobs outside the working trees, isolates objects
unreachable from any ref, and optionally runs trufflehog.

## Why

`git clone` only fetches `refs/heads/*` and `refs/tags/*`. GitHub also keeps
`refs/pull/<n>/head`, which survives closed PRs, deleted branches and
force-pushes.

A secret pushed, spotted, then removed by force-push is gone from every branch —
but still sits in `refs/pull/`. That's what this fetches.

## Install

```bash
install -m 755 gh-harvest ~/.local/bin/gh-harvest
```

Requires `git`. Optional: `gh` (org listing), `trufflehog` or `docker`
(scanning).

## Usage

```bash
gh-harvest -o <org> [options]
```

```bash
gh-harvest -o acme -l            # list repos, nothing else
gh-harvest -o acme -D            # clone + dump every blob
gh-harvest -o acme -O -s         # orphaned blobs only, then scan
gh-harvest -o acme -D -s -j 4    # everything, 4 repos in parallel
gh-harvest -o acme -r api -D     # single repo
```

### Options

**Target**

```
-o, --org <org>        GitHub org or user (required)
-r, --repo <repo>      specific repo, repeatable
-F, --forks            include forks when listing
-L, --limit <n>        max repos listed (default 500)
-H, --host <host>      git host (default github.com)
```

**Actions**

```
-l, --list             list and exit, no clone
-D, --dump             extract all blobs to objects/<repo>/
-O, --orphans          extract only unreachable blobs (implies -D)
-s, --scan             run trufflehog
```

**Other**

```
-d, --dir <path>       work root (default ~/work/targets/<org>)
-j, --jobs <n>         repos in parallel (default 1)
-n, --no-pr            skip refs/pull/* (not recommended)
-x, --exclude <list>   trufflehog detectors to exclude (default: lob)
-q, --quiet            minimal output
-h, --help             help
```

## Layout

```
~/work/targets/<org>/
├── repos/<repo>/                 clone, with pr/* branches
├── objects/<repo>/               extracted blobs, one file per SHA
├── objects/<repo>.orphans.txt    SHAs unreachable from any ref
└── reports/
    ├── th-<repo>.json            trufflehog output
    └── stats.tsv                 per-repo summary
```

Dumps live outside the repos, so grepping doesn't hit every object twice and
`git status` stays usable.

Once `-D` has run, point your usual tooling at `objects/`:

```bash
cd ~/work/targets/<org>/objects && gf sec | sort -u
```
