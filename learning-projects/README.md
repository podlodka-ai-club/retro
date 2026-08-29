# Learning projects

Real open-source projects the QA agent practises on. Their history — merge
requests, the bugs that followed, the regression tests written to fix them — is
the training material.

## Adding one

Clone it here. Nothing else to do:

```
git clone https://github.com/netbox-community/netbox learning-projects/netbox
```

Clone the full history, not `--depth 1` — walking the history is the point.
Expect a few hundred megabytes per project (netbox is ~300 MB on disk, 15k
commits).

Worth doing right after cloning, so a stray `git push` fails locally instead of
reaching the network:

```
git -C learning-projects/netbox remote set-url --push origin no_push
```

These are read-only fixtures; nothing we do to them belongs upstream, and we
have no write access anyway.

## Why nothing here is committed

`.gitignore` in this directory ignores everything except itself and this file.
No third-party source ever enters this repository, and nothing is redistributed
with it — so the licences those projects carry (netbox is Apache-2.0) impose no
obligations on us. Each clone stays on the machine that fetched it, governed by
its own licence, which ships inside the clone.
