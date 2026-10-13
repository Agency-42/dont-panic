# Contributing to Don't Panic

Thanks for grabbing a towel and opening a PR. This guide is for people who've
been paged — we want entries that help the next engineer at 03:00.

## What belongs here

- **Field-guide entries** under `docs/entries/`: real symptoms, copy-pasteable
  `kubectl` / `oc` commands, least-privilege fixes, and notes where OpenShift or
  RKE2 differ.
- **Runnable artifacts**: manifests, small scripts, checklists that pair with an
  entry.
- Fixes and clarifications to existing entries (wrong port, outdated command,
  clearer confirmation step).

Long-form SEO articles and marketing copy live on [agency42.se](https://agency42.se),
not in this repo. Keep GitHub practical.

## Entry template

Use this structure (English — error strings are English worldwide):

```markdown
# Exact error / symptom as H1

## What you see
## What's going on
## Confirm it
## Fix
## Prevent
## Notes for OpenShift / RKE2   # if relevant
## Related
```

Optional footer line for the planned canonical page (pages may not be live yet):

```markdown
> Longer walkthrough (planned): https://agency42.se/en/dont-panic/<slug>
```

## Before you open a PR

1. Prefer commands you've run on a real cluster (kubeadm, RKE2, OpenShift/OKD).
   If you haven't, say so in the PR.
2. Don't invent flags or paths. Link upstream docs when you're unsure.
3. Flag distribution differences explicitly (OpenShift ports, RKE2 paths, etc.).
4. No "grant `privileged` to `default`" as the primary fix. Least privilege first.
5. Don't claim partnerships with vendors. Don't invent headcount or "leading" language.
6. AI assistants are fine for drafting; **humans review** before merge. Say in
   the PR if a draft was AI-assisted so reviewers know what to re-check.

## Process

1. Open an issue describing the symptom / error string (or reference an existing one).
2. Fork, branch, add or edit the entry and any manifests.
3. Open a PR against `main`. Maintainers (Agency 42 engineers) review for
   accuracy and tone.
4. We may ask you to trim prose that belongs on the website, or to add a
   confirmation command.

## Code of conduct (short version)

Be kind. No blame-the-victim tone toward people who locked themselves out of a
cluster — they've suffered enough. Assume good intent; correct technical errors
directly.

## License

By contributing, you agree your contributions are licensed under the Apache
License 2.0 (see [LICENSE](LICENSE)).

---

Questions? Open an issue, or find us at [agency42.se](https://agency42.se).
