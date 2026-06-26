# myserver Catalog Registry — seed content

This directory mirrors the layout of the upstream registry repository
that lives at `github.com/serverops/myserver-templates`. Every myserver
instance pulls from `https://raw.githubusercontent.com/serverops/myserver-templates/main`
(overridable via the `CATALOG_REGISTRY_URL` env var) on a 1h cron.

When the GitHub repo is created, copy `index.json` + `templates/*` into
the repo root. The sync worker will start mirroring on the next tick.

## Layout

```
.
├── index.json              # entry list — keys, paths, formats, authors
└── templates/
    ├── n8n.yaml            # native YAML (preferred)
    ├── plausible.json      # Coolify-format JSON (adapter converts at sync)
    └── ...
```

## Native format

```yaml
key: my-stack                       # url-safe slug, ^[a-z0-9][a-z0-9-]{0,62}$
name: My Stack                      # human-readable display name
description: One-line catalog blurb
category: app                       # database | cache | tool | search | app
icon_url: https://...               # optional, SVG/PNG, no auth required
default_image: ghcr.io/foo/bar:1
default_env:
  FOO: bar
volumes:
  - data:/var/lib/app
exposed_ports:
  - "8080"
compose_template: |
  services:
    {{.Name}}:
      image: {{.Image}}
      ...
author: github-username             # optional, surfaced in the UI tooltip
version: "1.0.0"                    # optional, arbitrary author string
```

## Coolify format

The sync adapter accepts JSON files in the shape Coolify uses for their
community templates (`docker_compose_base64` + `tags` + `name`). The
adapter strips Coolify-specific magic variables (`SERVICE_*`) into our
`{{index .Settings "FOO"}}` placeholders so the template renders under
our engine. List those entries in `index.json` with `"format": "coolify"`.

Coolify imports are best-effort — templates with deep Coolify-only
dependencies may install but not deploy cleanly until edited.

## How publish works

1. Author drops `templates/foo.yaml` (or `.json`) into a PR.
2. CI parses + validates + renders the compose body with synthetic
   values; rejects on any error.
3. Maintainers review the diff (no automated trust signal yet — every
   template lands by human review).
4. Merge → next myserver sync (max 60 minutes later) mirrors it
   instance-wide; anyone browsing the catalog can install with one click.

## Validation that runs at sync time

- `key` matches `^[a-z0-9][a-z0-9-]{0,62}$`
- `name`, `default_image`, `compose_template` non-empty
- `compose_template` ≤ 64 KB
- `compose_template` parses as Go `text/template` AND contains a
  top-level `services:` block
- Sync failures on individual entries are surfaced in the worker log
  but don't abort the run; the rest of the catalog still updates.

## Removing a template

Delete the file + its index entry. Next sync prunes the local mirror;
teams that already installed it keep their local copy (registry sync
never touches `service_templates`).
