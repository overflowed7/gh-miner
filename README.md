# gh-miner

Git collection for secret hunting. Clones an org's repositories **including pull request refs**, recovers **orphaned commits by SHA**, dumps blobs outside the working trees, and optionally runs trufflehog.

## Install

```bash
install -m 755 gh-miner ~/.local/bin/gh-miner
install -m 755 check-leaks ~/.local/bin/check-leaks
```

Requires `git`. Recommended: `gh` (org listing and SHA recovery), `jq`. Optional: `trufflehog` or `docker` (scanning `-s`).

`check-leaks` dependencies are declared inline (PEP 723) and installed automatically on first run if [`uv`](https://docs.astral.sh/uv/) installed.

or with pip

```bash
pip install requests boto3 cryptography dnspython
```

## Flow

Dump all git historic non forked (`gh-miner`) and mint valid secrets (`check-leaks`).  

## Usage

```bash
gh-miner -o <org> [options]
```

```bash
gh-miner -o acme -l                       # list repos with sizes + est. dump size, nothing else
gh-miner -o acme -D                       # clone + recover orphans by SHA + dump every repos/blob non forked
gh-miner -o acme -D -s -j 4               # everything, 4 repos in parallel
gh-miner -o acme -r repo1 -r repo2 -D     # selection
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
-l, --list             list repos with size + estimated dump size, exit, no clone
-D, --dump             extract all blobs to objects/<repo>/,
                       including orphaned commits recovered by SHA
-O, --orphans-only     recover orphans by SHA and dump ONLY their blobs
                       (erased subset, lightweight dump — no full-history dump)
-s, --scan             run trufflehog
```

**Other**

```
-d, --dir <path>       work root (default ./gh-history)
-j, --jobs <n>         repos in parallel (default 1)
-n, --no-pr            skip refs/pull/* (not recommended)
-x, --exclude <list>   trufflehog detectors to exclude (default: lob)
-q, --quiet            minimal output
-h, --help             help
```

## Sizing a target before you dump

`-l` lists each non-forked repo with the size GitHub reports (`diskUsage`, the **packed** size on their side) and a total. That packed number is not what a full dump costs on disk: the dump writes decompressed blobs as separate files, plus PR refs, recovered orphans and the clone itself.

Measured on Netflix: `objects/` runs about x11 the packed size and `clones/` about x3, so the full on-disk dump lands around x14 total. `-l` applies that factor (`DUMP_FACTOR=14` in the script) and prints an estimated on-disk dump size alongside the packed total, so you can judge disk before committing to `-D`. Use `-O` instead when you only care about the erased content.

## Layout

```
./gh-history/
├── repos/<repo>/                 clone, with pr/* branches and refs/orphans/*
├── objects/<repo>/               extracted blobs, one file per SHA
├── objects/<repo>.orphans.txt    SHAs of orphaned commits recovered by SHA fetch
└── reports/
    ├── th-<repo>.json            trufflehog output
    └── stats.tsv                 per-repo summary
```

Once `-D` (or `-O`) has run, point your usual tooling at `objects/`:

```bash
python3 check-leaks ./gh-history [options]
```
