---
title: "Building a lightweight self-hosted OpenTofu registry that also renders docs"
description: "A tiny self-hosted OpenTofu/Terraform registry that shows browsable, versioned docs for every provider it serves, and can pull those providers from different hosts: a private Forgejo, public GitHub, and so on."
date: 2026-10-21T12:00:00+02:00
draft: false
tags: ["terraform", "opentofu", "registry", "homelab", "docsify", "golang"]
---

> [!NOTE]
> Full disclosure: this project was mostly vibe coded. Vibe coding a small app isn't especially noteworthy on its own. What makes this one worth a post is that it is extremely lightweight, and that it covers a gap the open source registries I looked at don't handle: the documentation.

## Background

I've been prototyping a new [OpenTofu](https://opentofu.org/) provider, one that talks to my private Forgejo. It is not ready for prime time yet: the API surface is still moving, I break backward compatibility casually, and there is no value in submitting it anywhere public.

What I needed was a registry that is light to run and that shows docs.

- `tofu init` has to resolve my provider from my own hostname instead of registry.opentofu.org.
- I want to browse the docs per release, because a directory of hand-written markdown in a repo that nobody reads is the wrong place for the docs of a provider you actively develop.

I also did not want the registry to care where the providers come from. Mine lives in a private Forgejo, but I use community providers that live on GitHub as well, and I would like all of them behind the same hostname, with the same docs UI, configured by one YAML file with a block per provider.

The off-the-shelf options were not what I wanted. The self-hosted registries I looked at were resource-heavy and complicated to run, and they didn't provide the browsable, versioned provider documentation I wanted. Public registries do provide documentation, but I found their multi-tenant publishing model too complicated and resource-heavy for my needs. I wanted something simple to run: just the registry protocol over HTTPS, in a single lightweight container with no database. I deliberately left authentication for registry users out of scope too; I wasn't interested in taking on that extra complexity for a service only I use.

So I gave myself a one-line requirement: `tofu init` and browsable docs for the providers I build locally, from one small container, no matter which host their releases end up on.

## How it works

The registry is a small Go program plus an nginx container. Here is what it actually does, mapped to the two jobs it has: serve the registry protocol (plus the network mirror protocol, if you want it), render the docs.

### The refresh loop

On start, and then every hour, a `refresh` binary walks the provider list (a YAML file mounted into the container) and for each provider does:

1. Call the provider repo's releases API (Forgejo or GitHub, set per provider in the YAML) to list its releases.
2. For each release, download the provider zip, `SHA256SUMS` and `SHA256SUMS.sig`, plus the docs. Two release layouts are understood:
   - my own convention: the release carries a registry metadata file (`terraform-provider-<type>_<ver>_registry.json`) and a `*_docs.zip`
   - plain goreleaser output: the platforms and hashes come from `SHA256SUMS`, the signing key comes from the YAML, and the docs are taken from `docs/` in the tag's source archive

   Files that are already on disk are skipped, since release assets are immutable.
3. Unpack the docs zip into a staging directory, rewrite a few headings, build a new `_sidebar.md` from the folder tree, and only then swap it into `/usr/share/nginx/html/<slug>/<version>/`.
4. Write the registry payload JSONs to `/usr/share/nginx/html/v1/providers/<ns>/<type>/<version>/download/<os>/<arch>.json`, pointing at `/dl/<type>/<version>/<file>` paths that nginx serves straight off disk.
5. Write a `versions.json` describing the cached versions and platforms, plus a per-provider `versions.md` and `meta.json`, a homepage `README.md` and a `docsify-config.js`, so the Docsify shell knows about the providers.

Nothing is live-fetched when a user hits the site. Everything is static bytes written to disk ahead of time, which means nginx serves the whole thing without depending on Forgejo or GitHub being up. When a refresh fails (Forgejo down, SPIFFE hiccup), the last-good cache keeps being served, which I consider a feature. The caveat is that "last-good" lives as long as the pod does; more on that in the hosting section.

Each provider also carries its own auth setting. The public GitHub one needs none; the private Forgejo one uses SPIFFE, which brings me to the next part.

### Auth: two separate boundaries

There are two distinct authentication boundaries here, and they have nothing to do with each other:

{{< mermaid >}}
flowchart LR
    C["tofu init<br/>(client)"] -->|"1. inbound: open, no auth"| R["Registry"]
    R -->|"2. outbound: per-provider auth<br/>(none or SPIFFE)"| F["Forgejo / GitHub<br/>releases"]
{{< /mermaid >}}

#### 1. Reaching the registry: deliberately open

Anyone who can reach the registry over the network can run `tofu init` against it and browse the docs. There is no login, no token and no per-user access control, and that is a conscious choice, not an omission. It is a service only I use, on a network I control, so network isolation is the access control, and I did not want to take on the complexity of user management. What is served is also not secret, and the client does not have to trust the registry for integrity: `tofu init` verifies the cached provider zips against the publisher's signing key, so a tampered download would be rejected.

#### 2. Fetching releases: authenticating to the upstream forge

This is the part that needed real work. The refresh loop has to download release assets from the source host, and my provider lives in a private Forgejo instance, so the registry must authenticate. Authentication is a per-provider setting in the YAML, and the code knows two modes today:

- `none`: used for the public GitHub providers. The registry calls the releases API anonymously.
- `spiffe`: used for the private Forgejo instance.

For Forgejo I wanted to avoid long-lived shared secrets, so instead of a PAT the registry presents a SPIFFE JWT-SVID, issued through SPIRE. The container runs in my homelab with a workload identity (`spiffe://example.org/ns/registry/sa/simple-tofu-registry`), and Forgejo is configured with an [Authorized Integration](https://forgejo.org/docs/latest/user/api/authorized-integrations/) of type Generic JWT: it trusts my OIDC discovery issuer, pins the exact workload identity with a claim rule on `sub`, and grants `read:repository` on the one repo it serves.

The credential is therefore a short-lived JWT minted for that workload, and nobody needs to rotate a long-lived token. When the workload dies, the identity dies with it. A JWT-SVID can still leak while it is valid, but its short lifetime limits the window in which it could be misused compared with a long-lived PAT.

This side is also easy to extend. The registry only needs a bearer token to attach to the release requests, so supporting a static token, another identity provider or another forge means adding one more auth mode that returns a token. Nothing else in the code has to change.

### The registry protocol surface

The registry protocol is small enough that we can just look at it. `tofu init` consults:

```
https://<registry>/.well-known/terraform.json        # discovery
https://<registry>/v1/providers/<ns>/<type>/versions
https://<registry>/v1/providers/<ns>/<type>/<ver>/download/<os>/<arch>
```

The discovery document in `site/.well-known/terraform.json` has one key:

```json
{ "providers.v1": "/v1/providers/" }
```

Everything after is just static JSON and static files. nginx serves `/v1/providers/` as JSON with a `try_files $uri $uri.json =404`, so the extension-less URLs the protocol asks for resolve to the `.json` files the refresh loop wrote, and `/dl/` as opaque bytes. That is the entire "backend" of the registry: some files that an hourly loop keeps fresh.

The download JSON includes the bits OpenTofu checks before installing:

```json
{
  "protocols": ["6.0"],
  "os": "linux",
  "arch": "arm64",
  "filename": "terraform-provider-<type>_<ver>_linux_arm64.zip",
  "download_url": "/dl/<type>/<ver>/terraform-provider-<type>_<ver>_linux_arm64.zip",
  "shasums_url": "/dl/<type>/<ver>/terraform-provider-<type>_<ver>_SHA256SUMS",
  "shasums_signature_url": "/dl/<type>/<ver>/terraform-provider-<type>_<ver>_SHA256SUMS.sig",
  "shasum": "...",
  "signing_keys": { "gpg_public_keys": [ { "key_id": "...", "ascii_armor": "..." } ] }
}
```

The shasum and the armored public key are copied from the `_registry.json` my provider's release already contains (or, for plain goreleaser releases, from `SHA256SUMS` and the key in the YAML), so the registry adds no signing step of its own: `tofu init` checks the cached zip against the same key and sums the release pipeline produced. The registry does not verify the signature itself; that happens on the client.

If you would rather not change `source` addresses at all, the registry can also publish the [provider network mirror protocol](https://opentofu.org/docs/internals/provider-network-mirror-protocol/) for a provider, under `/mirror/<host>/<namespace>/<type>/`. You keep `source = "darkhonor/technitium"` and point the CLI's `network_mirror` at the registry. The mirror carries only zip hashes, no signatures, so trust there rests on TLS to the source host.

### Docs: Docsify plus a generated config

The docs shell is [Docsify](https://docsify.js.org/), picked because it was the quickest way to render markdown: it renders client-side from whatever markdown you throw at it, with no build step, which is exactly the shape of `terraform-plugin-docs` output (a directory of markdown files with `resources/` and `data-sources/` folders).

The docs refresh step simply:

- unpacks the `*_docs.zip` asset (which my provider's CI builds in that layout), or the `docs/` folder of the tag archive for providers published with plain goreleaser
- rewrites the terraform-plugin-docs headings to registry-style ones (`## Schema` → `## Argument Reference`, `### Read-Only` → `## Attributes Reference`), so the pages read like every other Terraform provider doc instead of looking like raw schema dumps
- generates `_sidebar.md` from the directory tree
- keeps the latest release mirrored at `/<slug>/`, versioned history at `/<slug>/<version>/`, plus a `versions.md` table of every release with docs link and supported platforms

The Docsify config itself is generated by the same binary, so adding a provider to the YAML is all it takes to get a new card on the homepage and working routes. I also vibe coded a handful of small scripts that live in `site/vendor/`:

- `sidebar-filter.js` (about 95 lines) mounts a search box above the sidebar and, on every keystroke, hides the entries that don't match and highlights the match in the ones that do.
- `provider-nav.js` (about 160 lines) parses the URL (`/<slug>/0.3.0/resources/<name>.md` → slug `<slug>`, version `0.3.0`), fetches `meta.json`, and renders a breadcrumb plus a version dropdown that switches you to any other release without losing your place in the docs.
- `page-toc.js` adds the "On this page" outline.
- `site-footer.js` shows the registry's version and build date.

So the docs experience is: versions, breadcrumbs, sidebar search, a TOC per page, and HCL syntax highlighting. All from static files, with the Docsify and Prism assets baked into the image at build time so the CSP can stay `'self'`-only.

{{<figure src="home.png" alt="Homepage of the registry listing cached providers" caption="*The homepage: cached provider cards with latest versions*" nozoom=true >}}

As you can see, this is quite basic. My goal was to deploy something quickly, so there are plenty of opportunities to improve the UI.

{{<figure src="forgejo-index.png" alt="Docsify rendered page for the forgejo provider index" caption="*Docs home page for the Forgejo provider*" nozoom=true >}}

{{<figure src="oauth-search.png" alt="Sidebar search with oauth typed into the search box" caption="*Typing 'oauth' into the search box narrows the sidebar to the matching entries*" nozoom=true >}}

### The hosting shape

One container, one Deployment, one Helm chart. The container runs nginx on 8080, with `/api/` proxied to the refresh daemon on loopback 8081 and everything else served as static files. The provider list comes from a ConfigMap, and the SPIFFE Workload API socket is mounted through the SPIRE CSI driver. Requests are 25m CPU and 64Mi of memory, with a 256Mi limit.

There is no persistent volume yet, though adding one would be straightforward. The cache lives on the container's own disk, so restarting or replacing the pod loses all cached provider files and docs. The entrypoint runs a blocking refresh before nginx starts accepting traffic, then hands the hourly loop to `refresh serve`, rebuilding the cache from the release assets at startup.

## What it is not

For honesty's sake, here is the comparison with a real registry:

| | Self-hosted lightweight registry | Public registries (HashiCorp, OpenTofu) |
|---|---|---|
| Versioned docs, rendered | ✅ per-release Docsify site | ✅ |
| Providers from several hosts (Forgejo, GitHub, ...) | ✅ one YAML block per provider | ❌ published to the registry itself |
| Registry protocol for `tofu init` / `terraform init` | ✅ `versions` + `download/<os>/<arch>` | ✅ |
| SHA256 + GPG data | ✅ release assets cached with sums, signatures and the signing key (from the release metadata or the YAML), verified by the client | ✅ |
| Provider network mirror protocol | ✅ optional, per provider | n/a |
| Identity-based cache refresh | ✅ SPIFFE JWT-SVID into Forgejo Authorized Integration | n/a |
| Multi-user web UI | ❌ | ✅ provider pages, votes, reviews |
| Uploads from many authors | ❌ one code path, pulled from releases | ✅ |
| Authoring UI / validation | ❌ docs come from CI | ✅ |
| Dependency graphing | ❌ | ✅ |

The code lives in [ilmax/slim-opentofu-registry](https://github.com/ilmax/slim-opentofu-registry). If you'd like to improve or extend it, contributions are welcome.

This is not a finished product. The scope was a handful of providers and one homelab network, and the code reads like it: no release-signing pipeline of its own (I reuse my provider's existing CI artifacts), no upload API, no external CI integration beyond a SPIFFE bearer token into Forgejo, and hard-coded assumptions about release layouts: it understands my own convention (`_registry.json`, `_docs.zip`) and plain goreleaser output, and anything else would need code.

To prove the registry handles a provider that lives on another host, not just my own, I also ran a temporary local instance pointed at `darkhonor/terraform-provider-technitium`, a community provider for the Technitium DNS server published on GitHub. Same binary, one extra block in the YAML, no auth: GitHub as the source, the provider's public signing key, and nothing else. Its releases are plain goreleaser output, so the platforms come from `SHA256SUMS` and the docs from the `docs/` folder of each tag. In my run it cached three releases (1.2.0 to 1.3.0) for `linux_amd64` and `linux_arm64` with their `SHA256SUMS` and signatures, unpacked the docs into `/technitium/<version>/`, and the Docsify shell rendered them with the same breadcrumbs, version dropdown and sidebar search as my own provider.

## Gotchas

- The docs refresh does a staging-copy-and-rename dance (`<dest>.staging` → `<dest>`, previous → `<dest>.prev` → delete).
- Docsify resolves `_sidebar.md` relative to the current page, so my generated config has regex aliases mapping `/<slug>/.*/_sidebar.md` back to the right sidebar, either the provider root or the version root. Getting this wrong gives you a sidebar that works on one page and silently dies everywhere else.
- `looksLikeVersion` rejects `1.0.0-rc1`, since I decided prereleases were a case I did not want to handle in my private registry. It is fine today; someday it will bite me.
- Because the cache lives on the container's disk, a pod restart while a provider's host is down means that provider has no docs and no downloads until the next successful refresh. A PV would fix that; I have not needed one yet.
- The `POST /api/refresh` endpoint that triggers an on-demand cache refresh is anonymous by design. Network isolation is the access control. It answers `202` when it starts a refresh and `409` if one is already running.

> [!TIP]
> If you want per-release docs for your Terraform/OpenTofu provider and don't need anything else, the docs half alone is useful: `terraform-plugin-docs` plus Docsify gets you a versioned docs site without touching a registry at all. I split the project so the refresh binary is the interesting part and the docs are a byproduct of pointing it at release assets.

## Dig deeper

- [OpenTofu provider registry protocol](https://opentofu.org/docs/internals/provider-registry-protocol/) — the endpoints and payloads `tofu init` expects from a registry
- [Remote service discovery](https://opentofu.org/docs/internals/remote-service-discovery/) — how the `.well-known/terraform.json` document points the CLI at `/v1/providers/`
- [Provider network mirror protocol](https://opentofu.org/docs/internals/provider-network-mirror-protocol/) — the alternative that lets you keep your `source` addresses unchanged
- [terraform-plugin-docs](https://github.com/hashicorp/terraform-plugin-docs) — the generator behind the docs layout this registry renders

## Conclusion

I wanted `tofu init` and browsable docs for the providers I build locally, without running a full registry product. What came out is a small Go program and an nginx container that pull provider releases from wherever they live, write everything to disk as static files, and serve the registry protocol, optionally the network mirror protocol, and a versioned Docsify site from the same place. It is basic and it was vibe coded, which on its own is nothing special. What I like about it is how lightweight it is, and that it fills the documentation gap the open source registries I looked at leave open.

The whole thing fits in one blog post, which at this point I think is the intended meaning of "lightweight".

Till the next time!
