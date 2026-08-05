# ALLAKORI Writeups

Dedicated technical writeup blog for Kossi Richard Allado.

Site target:

```text
https://allakori.github.io/writeups/
```

This repository is based on `cotes2020/chirpy-starter` and uses the Jekyll Chirpy theme.

## Content

Current posts:

| Date | Post |
| --- | --- |
| 2026-07-02 | Hack The Box Season 11 private writeups tracker, updated through Cohort |
| 2026-05-14 | Volt Typhoon intrusion investigation |
| 2026-04-29 | CSIA CTF 2026 first-place writeups |
| 2026-04-20 | Browzi MiniBrowser heap exploitation |
| 2026-03-15 | deepwash PHP DateTimeImmutable parser logic |
| 2026-02-01 | SOKOLO AD lab Active Directory attack chain |

## Configuration

The blog is configured for:

```yaml
title: ALLAKORI — Writeups
tagline: CTF, DFIR, web, pwn, AD — technical notes
url: https://allakori.github.io
baseurl: /writeups
timezone: Africa/Casablanca
```

## Deployment

GitHub Actions builds the Jekyll site and deploys `_site` to the `gh-pages` branch.

After creating the GitHub repository, push with:

```bash
git remote -v
git push -u origin main
```

Then configure GitHub Pages to serve from:

```text
Branch: gh-pages
Folder: /
```
