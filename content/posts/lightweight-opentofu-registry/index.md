---
title: "Building a lightweight self-hosted OpenTofu registry that also renders docs"
description: "Why the off-the-shelf Terraform/OpenTofu registries were too heavy for my needs, and how I built a lightweight replacement that serves both the registry protocol and browsable, versioned docs from the same place."
date: 2026-10-21T12:00:00+02:00
draft: false
tags: ["terraform", "opentofu", "provider", "registry", "homelab", "docsify", "golang"]
---

## Why another registry

I've been prototyping a new [OpenTofu](https://opentofu.org/) provider, one that talks to my private Forgejo. It is not ready for prime time yet: the API surface is still moving, I break backward compatibility casually, and there is no value in submitting it anywhere public.

What I needed was a registry. `terraform init` wants to reach `/v1/providers/<ns>/<type>/<version>/download/<os>/<arch>.json`, and I wanted that to resolve to my own provider instead of registry.opentofu.org. While I was at it I also wanted somewhere I could browse the docs per release, because a directory of hand-written markdown in a repo that nobody reads is the wrong place for docs of a provider you actively develop.

The off-the-shelf options were not what I wanted. The obvious one, HashiCorp's registry (and the API-compatible lookalikes that followed it), is a multi-tenant SaaS: you get a web UI, authoring tools, upload flows, version signing, dependency graphing, and everything else a public catalogue needs. I was going to use the registry protocol over HTTPS and nothing else. All that product machinery felt like running a database with Postgres when I need a SQLite file. The self-hosted clones I looked at were also heavy: Kubernetes clusters with databases, CI pipelines to sign releases, user accounts. I wanted a single container, a few dozen megabytes of disk, and done.

I also noticed something that no registry worth the name does: it renders the docs for the provider versions it serves. terraform-provider-* docs live in the provider repo, as markdown or generated HTML fragments. The registries themselves only download binary release assets, they do not display what they are serving. So I gave myself a one-liner requirement: make docs readable from the same place, versioned, without spinning up a second service.

## The pieces I kept

The end result is a Go program plus an nginx container. Here is what it actually does, mapped to the two jobs it has: serve the registry protocol, render the docs.

### The refresh loop

Every hour, a `refresh` binary walks the provider list (a YAML file mounted into the container) and for each provider does:

1. Call Forgejo's API to list the repo's releases.
2. For each release, download the registry metadata file (`terraform-provider-<type>_<ver>_registry.json`), the provider zip, `SHA256SUMS`, `SHA256SUMS.sig`, and `*_docs.zip`.
3. Unpack the docs zip into `/usr/share/nginx/html/<slug>/<version>/`, atomically, after building a new `_sidebar.md` from the folder tree.
4. Write the registry payload JSONs to `/usr/share/nginx/html/v1/providers/<ns>/<type>/<version>/download/<os>/<arch>.json`, pointing at `/dl/<type>/<version>/<file>` paths that nginx serves straight off disk.
5. Write a `versions.json` describing the cached versions and platforms, plus a homepage `README.md` and `docsify-config.js` so the Docsify shell knows about the new providers.

Nothing is live-fetched when a user hits the site. Everything is static bytes written to disk ahead of time, which means nginx serves the whole thing under load with no dependency on Forgejo being up. When the refresh fails (Forgejo down, SPIFFE hiccup), the last-good cache just keeps being served, which I consider a feature.

### Auth: no long-lived tokens

The provider repo is private on Forgejo, so the refresh loop needs to authenticate to download release assets. Instead of a PAT, it uses a SPIFFE JWT-SVID. The container runs inside my homelab with a workload identity (`spiffe://donini.cloud/ns/tf-backend/sa/provider-docs`), and Forgejo is configured as an [Authorized Integration](https://forgejo.org/docs/latest/user/api/authorized-integrations/) of type Generic JWT: the integration pins the exact workload identity and grants `read:repository` on the one repo it serves.

So the security story is: Forgejo signs nothing, my identity attestation holds the privilege, and nobody needs to rotate a token. When the workload dies, the token dies with it. This is the second time I've used this pattern on my infrastructure and it keeps feeling better than the standard "generate a PAT, put it in a secret" dance.

### The registry protocol surface

The registry protocol is small enough to describe in a minute. `terraform init` consults:

```
https://<registry>/.well-known/terraform.json        # discovery
https://<registry>/v1/providers/<ns>/<type>/versions.json
https://<registry>/v1/providers/<ns>/<type>/<ver>/download/<os>/<arch>.json
```

The discovery document in `site/.well-known/terraform.json` is two lines:

```json
{ "providers.v1": "/v1/providers/" }
```

Everything after is just static JSON and static files. nginx serves `/v1/providers/` as JSON (with a `try_files $uri $uri.json =404` so Terraform's URL shapes both work) and `/dl/` as opaque bytes. That is the entire "backend" of the registry: some files that an hourly cron-style loop keeps fresh.

The download JSON includes the bits Terraform checks before installing:

```json
{
  "protocols": ["6.0"],
  "filename": "terraform-provider-forgejo_0.3.0_linux_amd64.zip",
  "download_url": "/dl/forgejo/0.3.0/terraform-provider-forgejo_0.3.0_linux_amd64.zip",
  "shasums_url": "/dl/forgejo/0.3.0/terraform-provider-forgejo_0.3.0_SHA256SUMS",
  "shasums_signature_url": "/dl/forgejo/0.3.0/terraform-provider-forgejo_0.3.0_SHA256SUMS.sig",
  "signing_keys": { "gpg_public_keys": [ { "key_id": "...", "ascii_armor": "..." } ] }
}
```

which means `tofu init` validates my release zip with GPG against the same armored key that signs every release. No registry abstraction between the standard Terraform verification path and my CI, so nothing has drifted.

### Docs: Docsify plus one file

The docs shell is [Docsify](https://docsify.js.org/), which is a deliberate downgrade from my Hugo site: Docsify renders markdown client-side from whatever markdown you throw at it, which is exactly the shape of `terraform-plugin-docs` output (a directory of markdown files under `docs/` in the provider repo).

The docs refresh step simply:

- unpacks the `*_docs.zip` asset (which my provider's CI builds from `terraform-plugin-docs`)
- rewrites the terraform-plugin-docs headings to registry-style ones (`## Schema` → `## Argument Reference`, `### Read-Only` → `## Attributes Reference`), so the pages read like every other Terraform provider doc instead of looking like raw schema dumps
- generates `_sidebar.md` from the directory tree
- keeps the latest release mirrored at `/forgejo/`, versioned history at `/forgejo/<version>/`, plus a `versions.md` table of every release with docs link and supported platforms

On top of Docsify I wrote two small scripts that live in `site/vendor/`. `sidebar-filter.js` mounts a "Search resources..." box above the sidebar and hides non-matching entries with highlighted matches (about 90 lines, it walks the sidebar DOM per keystroke). `provider-nav.js` parses the URL (`/forgejo/0.3.0/resources/organization.md` → slug `forgejo`, version `0.3.0`), fetches `meta.json`, and renders a breadcrumb plus a version dropdown that switches you to any other release without losing your place in the docs.

So the docs experience is: versions, breadcrumbs, sidebar search, TOC per page, dark theme, HCL syntax highlighting. All from static files.

{{<figure src="home.png" alt="Homepage of the registry listing cached providers" caption="*The homepage: cached provider cards with latest versions*" nozoom=true >}}

{{<figure src="forgejo-index.png" alt="Docsify rendered page for the forgejo provider index" caption="*Docs home page for the Forgejo provider*" nozoom=true >}}

{{<figure src=\"oauth-search.png\" alt=\"Sidebar filter with oauth typed into the search box\" caption=\"*Typing 'oauth' into the sidebar search filters to the matching entries*\" nozoom=true >}}

### The hosting shape

One container, one Deployment, one Helm chart. The container runs nginx on 8080 with the `nginx.conf.template` proxying `/api/refresh` to the refresh daemon on loopback 8081 and serving everything else as static files. K8s mounts a PV at `/usr/share/nginx/html`, and that PV is the whole state of the system: the cache. Losing the PV costs me at most an hour of freshness, not any data.

## What it is not

For honesty's sake, here is the comparison with a real registry:

| | Self-hosted lightweight registry | Full registry (HashiCorp et al.) |
|---|---|---|
| Versioned docs, rendered | ✅ per-release Docsify site | ❌ docs live in the provider repo |
| Registry protocol for `tofu init` / `terraform init` | ✅ `versions.json` + `download/<os>/<arch>.json` | ✅ |
| SHA256 + GPG verification | ✅ release assets cached with sums + signatures | ✅ |
| Identity-based cache refresh | ✅ SPIFFE JWT-SVID into Forgejo Authorized Integration | n/a |
| Multi-user web UI | ❌ | ✅ provider pages, votes, reviews |
| Uploads from many authors | ❌ one code path, my releases | ✅ |
| Authoring UI / validation | ❌ docs come from CI | ✅ |
| Dependency graphing | ❌ | ✅ |
| Operator headcount | 1 laptop | organization |

I threw away the last four rows on purpose. When the provider is ready for public consumption, or when a second person needs access, it becomes the right moment to re-evaluate. Today I am the only reader, the only author, and the only provider; the registry's job is to make `tofu init` work from my laptop and to give me docs next to whatever it serves. That is about 800 lines of Go and a few dozen lines of JavaScript, and the cache directory on disk is the entire state of the world.

The code lives in [ilmax/slim-opentofu-registry](https://github.com/ilmax/slim-opentofu-registry) (placeholder, not yet published).

This is not a finished product. The scope was one provider, one user, one homelab network, and the code reads like it: no release-signing pipeline of its own (I reuse my provider's existing CI artifacts), no upload API, no external CI integration beyond a SPIFFE bearer token into Forgejo, and hard-coded assumptions about asset naming that would need work to generalize to other providers' release layouts.

To prove the docs side handles a stock third-party provider, not just my own, I also ran a temporary local instance of the registry pointed at `darkhonor/terraform-provider-technitium`, a community provider for the Technitium DNS server published on GitHub. The refresh loop pulled the release list from the GitHub API, cached the platform zips with their `SHA256SUMS` and signatures, unpacked the docs into `/technitium/<version>/`, and the Docsify shell rendered them with the same breadcrumbs, version dropdown and sidebar search as my own provider.

## Gotchas

- The docs refresh does a staging-copy-and-rename dance (`<dest>.staging` → `<dest>`, previous → `<dest>.prev` → delete). The first time I did it naïve-overwrite and watched a page serve halfway-unpacked markdown for twenty minutes. Atomic directory swaps are 12 lines of Go, take them.
- Docsify resolves `_sidebar.md` relative to the current page, so my generated config has regex aliases mapping `/<slug>/.*/_sidebar.md` back to the provider root's sidebar. Getting this wrong gives you a sidebar that works on one page and silently dies everywhere else.
- `looksLikeVersion` rejects `1.0.0-rc1`, since I decided prereleases were a case I did not want to handle in my private registry. It is fine today; someday it will bite me.
- The `POST /refresh` endpoint that triggers an on-demand cache refresh is anonymous by design. Network isolation is the access control, same stance I take with everything else on the homelab edge.

> [!TIP]
> If you want per-release docs for your Terraform/OpenTofu provider and don't need anything else, the docs half alone is useful: `terraform-plugin-docs` plus Docsify gets you a versioned docs site without touching a registry at all. I split the project so the refresh binary is the interesting part and the docs are a byproduct of pointing it at release assets.

The whole thing fits in one blog post, which at this point I think is the intended meaning of "lightweight".
