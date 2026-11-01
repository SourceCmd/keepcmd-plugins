# Keep Cmd plugins

The plugin catalog for Keep Cmd (`keepcmd`), the program that runs a keep —
the machine that is a Source Cmd space's server. Each plugin connects your
agents to an outside service — your calendar, your files, your smart home —
through a declared, permissioned set of actions. Plugins run on your keep, so
one installed plugin serves every space and app the keep runs: Loop Cmd chats
and Hearth Cmd households alike.

(This repo was `hearth-plugins`; it was renamed when the catalog stopped
belonging to one app. GitHub redirects the old name.)

Install one on your keep with:

```bash
keepcmd plugin install verge_labs/google_calendar_oauth
```

`keepcmd plugin list`, `uninstall` and `rollback` do the rest. Browsing this
page **is** the discovery mechanism for now: find the plugin you want below,
copy its slug, and install it. There is no `keepcmd plugin search` yet.

---

## Available plugins

| Slug | Name | Auth | What it does |
|---|---|---|---|
| `verge_labs/google_calendar_oauth` | Google Calendar | OAuth | Create and manage Google Calendar events. Connects as you. |
| `verge_labs/google_calendar` | Google Calendar (Service Account) | Service account | Same, for Google Workspace acting as any user. |
| `verge_labs/google_drive_oauth` | Google Drive | OAuth | Read and write Drive files. Limited to files your agents create or you explicitly open. |
| `verge_labs/google_drive` | Google Drive (Service Account) | Service account | Same, for Workspace-wide access. |
| `verge_labs/google_people_oauth` | Google Contacts | OAuth | Look up family, neighbours, the plumber, the kids' school. |
| `verge_labs/google_people` | Google People (Service Account) | Service account | Look up people in a Workspace organization. |
| `verge_labs/home_assistant` | Home Assistant | API key | Control Home Assistant entities — lights, scenes, climate. |
| `verge_labs/sonos` | Sonos | OAuth | Play, pause, skip, set volume, start favourites and playlists, group rooms. |
| `verge_labs/spotify` | Spotify | OAuth (your own app) | Search Spotify and play it, control transport and volume, move playback between devices. |
| `verge_labs/home_assistant_music_assistant` | Music Assistant | API key | Play music by name on any speaker Home Assistant drives, per-room volume, grouping, move music between rooms. |
| `verge_labs/github` | GitHub | API key | Read and write GitHub issues, pull requests, files, and commits. |

### Three ways to play music

They overlap, and the right one depends on what the house already runs.

- **Music Assistant** is the one to reach for if Home Assistant is already in
  the house. It is provider-agnostic — whatever Music Assistant is connected to
  is reachable — plays by name with no search step, drives any speaker HA can
  drive, and needs nothing but the HA token you may already have.
- **Sonos** talks to Sonos' own cloud. Use it when the speakers are Sonos and
  there is no Home Assistant. It can only start things already saved as Sonos
  favourites or playlists — there is no search.
- **Spotify** is the only one with the full Spotify catalogue and search, but
  it needs Spotify Premium and a one-time setup where you register your own
  free Spotify app (Spotify serves five users per app, so a shared one could
  never cover more than five households).

If Home Assistant is running and Music Assistant is installed on it, that plugin
is usually the least friction and the widest reach.

Pick the **OAuth** variant unless you administer a Google Workspace domain. It
connects as a single ordinary Google account and needs no domain-wide
delegation. Each connection authenticates as exactly one account — for a second
account, make a second connection.

---

## What a plugin is

A plugin is **declarative**: a `manifest.yaml` describing HTTP calls, plus an
optional `skill.md` that teaches an agent how to use them. There is no
executable and no plugin process — the keep makes the calls itself.

That is a deliberate constraint, not a limitation we intend to lift casually.
A manifest can only do what it declares, and every verb in it is subject to
your permission rules. Arbitrary code running as you on your keep is a very
different risk, so binary plugins cannot be published here at all; they install
from a local archive, where a human handled the file.

```
plugins/<namespace>/<name>/
  manifest.yaml     # required — verbs, credentials, config schema
  skill.md          # optional — agent-facing usage notes
```

The directory path must equal the manifest's `plugin_slug`. The index builder
and the keep both enforce this.

`skill.md` is optional in the format, but in practice a verb with no skill
coverage is a verb agents use wrong — it reaches them as a bare name with no
example and no warning about its traps, and a skill written before the verb
existed can actively steer away from it. So the index builder warns about any
verb its plugin's `skill.md` never mentions, and about a plugin that has no
`skill.md` at all. The warnings don't block a release; they print last, after
the "wrote index.json" line. Every first-party plugin should build clean.

`skill.md`'s `description` frontmatter does double duty. Beyond steering an
agent that *already has* the plugin, it is the canonical capability summary a
**discovery agent** reads to decide whether this plugin fits a need it is trying
to solve — when we one day index the catalog for such agents to search, this is
the text they will search and then read. Write it so a reasoner with no prior
knowledge of the service can tell **what the plugin is for and which problems it
solves**, not only which verbs it exposes. Lead with the real-world scenarios
("scheduling meetings," "putting music on in a room," "saving energy by turning
things off") — a keyword search for a household problem should land here. The
`manifest.description` stays the mechanics/setup blurb; the two do not need to
repeat each other, and a plugin needs no separate "discovery" block — this
frontmatter is it.

### OAuth plugins

A plugin whose upstream needs OAuth declares its endpoints on the credential:

```yaml
credentials:
  - name: user_token
    type: oauth2_sonos          # the suffix IS the provider slug
    secret: true
    oauth:
      authorize_url: https://api.sonos.com/login/v3/oauth
      token_url: https://api.sonos.com/login/v3/oauth/access
      client_auth: basic        # basic (default) | form
      # authorize_params: {access_type: offline}   # optional, upstream-specific
    scopes: [playback-control-all]
```

That is the whole integration. The keep has one generic provider that reads
this block; there is no per-upstream code to write. Sign-in runs between the
app and your keep (PKCE), and the refresh token is sealed to your keep —
the relay never holds it.

**What is not here, and cannot be:** a `client_id` or `client_secret`. Google
plugins use Source Cmd's own app client, built into keepcmd. For any other
upstream, the person connecting registers their own free app with the
upstream (as Spotify's setup describes) and gives its client details to the
connection; its redirect URI is `https://relaycmd.com/oauth/redirect`, a
page that only hands the code back to the Source Cmd app.

Prefer `api_key` auth when the upstream offers it — the user pastes their own
token and nothing needs to be registered by anyone.

### Compatibility

A manifest may declare `min_daemon_version`. It means "this plugin needs a
runtime feature that older keepcmd binaries don't have" — for example
`verge_labs/google_drive_oauth` uses the `||` fallback operator in its
templates, which older binaries parse as a literal path. Declaring the floor
turns a confusing runtime error into a refusal at install time telling you to
update the keep (`keepcmd hub update`).

This is distinct from `manifest_schema`, which says which *format* the
manifest is written in. Schema 3 adds two optional things: a top-level
`session:` block for sites that sign in with a username and password and keep
a cookie (the keep holds the session; the agent never sees the password or
the cookie), and a per-verb `effect:` — `read`, `change` or `send` — which
the rules view, the approval ask and taint handling all read (a verb without
one counts as `change`). No plugin in this catalog uses schema 3 yet; the
format is specified in the Source Cmd repo
(`docs/20261030-plugin-session-sign-in-design.md`).

---

## Releases

Keeps fetch the catalog from **https://get.keepcmd.com/plugins/**
(`index.json`, `index.json.sig`, `catalog.tar.gz`), never from this repo or
its GitHub releases. Publishing is done from the Source Cmd repo:

```bash
SOURCECMD_PLUGIN_CATALOG_KEY=<plugin-catalog key file> \
  scripts/plugins/publish.sh <this checkout> <SourceCmd/keepcmd checkout> 2026.11.01
```

It runs this repo's `scripts/release.sh` to build `dist/`, signs the index,
checks it the way a keep will, and copies it into the get.keepcmd.com feed
checkout; it prints the commit and push commands rather than running them.

`index.json` is **not** committed. It is generated at publish time, because a
committed index can drift from the manifests it claims to describe.

Both artifacts are **reproducible**: the index has sorted keys and no
timestamp, and the tarball is built with a fixed mtime and owner. Rebuilding
an unchanged tree yields byte-identical files, so "do these artifacts match
this tree?" is answerable by rebuilding and comparing hashes.

### index.json

```json
{
  "schema": 1,
  "catalog_version": "2026.07.21",
  "plugins": {
    "verge_labs/google_drive_oauth": {
      "version": "0.1.3",
      "display_name": "Google Drive",
      "description": "Read and write files in Google Drive…",
      "auth_scheme": "oauth2_user",
      "manifest_schema": 2,
      "min_daemon_version": "1.1.0",
      "files": {
        "manifest.yaml": "<sha256 hex>",
        "skill.md": "<sha256 hex>"
      }
    }
  }
}
```

The index carries the sha256 of every published file, so one signature over
this one file covers the whole catalog **including version numbers**. That is
what makes a rollback attack — serving an old, vulnerable plugin — detectable.
Per-file signatures would not, because every old file remains validly signed.

The CLI additionally cross-checks the index's `version` against the extracted
`manifest.yaml` and refuses on mismatch, so a bad index build fails loudly
instead of installing something mislabelled.

---

## Signing

TLS proves you reached get.keepcmd.com. It does not prove it served what we
published — a compromised repo, a stolen credential, or a bad upload all serve
perfectly valid TLS. The signature closes that gap, and it is the only reason
`keepcmd plugin install <name>` is meaningfully safer than piping a URL into a
shell.

`index.json` is signed with Ed25519, and `index.json.sig` is the detached
signature. Because the index carries a hash of every published file, one
signature covers the whole catalog.

The signing key is not compiled into keepcmd. Keep Cmd's root keys are, and
they delegate a key for the `plugin_catalog` role in a signed delegations
document on the same feed, with an expiry and a revocation list. A keep
verifies the delegation, then the catalog under the delegated key, so the
catalog key can be rotated or revoked without shipping a new binary. The
plugin-catalog key lives offline and is brought onto a machine only to sign.

### Superseded scripts

`scripts/keygen.sh` and `scripts/sign-index.sh` belong to the old scheme —
a single catalog key compiled into each binary, and a GitHub release per
catalog. Current keeps don't trust that key, and nothing reads GitHub
releases any more; key generation and signing are now Keep Cmd's release
tooling, driven by `publish.sh`. They are kept here for reference only.
`scripts/release.sh` and `scripts/build-index.py` are still used: they build
`dist/`.

---

## Contributing

Community plugin authoring isn't open yet — the signing and review process
that would make third-party plugins safe to distribute doesn't exist. If
you've built something you'd like to see here, open an issue.
