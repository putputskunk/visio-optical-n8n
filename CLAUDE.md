# Project context for Claude

**Hardware inventory:** [`SETUP.md`](./SETUP.md) is the canonical list of the
operator's devices and specs. Whenever asked about "my devices", "devices with
specs", or any hardware question, read `SETUP.md` first. It wins over chat
memory on conflict.

## About this repo

`visio-optical-n8n` hosts the public compliance pages for **Visio Optical
Auto** — an internal n8n automation that publishes Visio Optical's own
marketing videos to TikTok, Meta (Instagram/Facebook), and Pinterest. The
repo contains only what those platforms require to be hosted publicly:

- `privacy.html`, `terms.html` — Privacy Policy / Terms of Service pages
- `tiktok*.txt` — TikTok domain-verification token

The n8n workflow logic and access tokens live on the operator's self-hosted
infrastructure, not in this repo.
