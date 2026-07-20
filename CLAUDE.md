# CLAUDE.md — Drupal Loop AI (demo)

Drupal **11.4.4** demo site showing *loop engineering*: GitHub Actions agents
that scout the backlog, triage issues, implement them, and review each other's
pull requests, with a human as the only merge authority.

This repo is **config-as-code**. There is no shared database. The site is
rebuilt from `config/sync/` on every CI run. If a change is not exported to
`config/sync/`, it does not exist.

---

## Layout

| Path | What it is |
|---|---|
| `config/sync/` | The site's entire structure as YAML. **The source of truth.** |
| `web/` | Drupal docroot. `web/core`, `web/modules/contrib` are Composer-managed and gitignored. |
| `web/modules/custom/` | Custom module code, if any is ever needed. |
| `composer.json` / `composer.lock` | Dependency manifest. Both are committed. |
| `docs/` | Client-facing architecture diagrams (Spanish). |

Never edit anything under `web/core/`, `web/modules/contrib/`, or `vendor/` —
those are rebuilt by `composer install` and your changes would be discarded.

---

## The one workflow that matters

Structural changes (content types, fields, views, displays, roles) are made by
writing or editing YAML in `config/sync/`, then **proving it imports cleanly**:

```bash
composer install
cp web/sites/default/default.settings.php web/sites/default/settings.php
chmod u+w web/sites/default web/sites/default/settings.php
printf "\n\$settings['config_sync_directory'] = '../config/sync';\n" >> web/sites/default/settings.php

# Rebuild the whole site from config/sync. This FAILS LOUDLY on bad YAML,
# missing dependencies, or a schema violation. This is the real gate.
./vendor/bin/drush site:install --existing-config \
  --db-url=sqlite://sites/default/files/.ht.sqlite --yes

# Normalize what you hand-wrote: Drupal fills in uuid, _core, and the correct
# `dependencies` block. ALWAYS run this, then commit the result.
./vendor/bin/drush config:export --yes

# Must print "No differences between DB and sync directory."
./vendor/bin/drush config:status
```

**Hand-write the YAML, then let `config:export` normalize it.** You do not need
to guess UUIDs or dependency lists — import generates them and export writes
them back. What you must get right is the structure, machine names, and
`langcode`.

Do not weaken, skip, or edit the CI check to make it pass. If you cannot get
`drush config:status` clean, stop and say so.

---

## Conventions

**Language.** All user-facing labels, descriptions, and help text are in
**English**. Machine names are ASCII `snake_case`: label `Academic article` →
machine name `academic_article`.

(The Spanish client-facing architecture deck in `docs/` is deliberately left in
Spanish — it is a presentation artifact, not site config. Do not translate it.)

**Machine names.**
- Content types: bare noun, singular — `course`, `expert`.
- Fields: always `field_` prefixed and scoped to their meaning —
  `field_duration_hours`, not `field_duration`.
- Views: `snake_case` matching intent — `course_listing`.

**Reuse fields across bundles.** A field is two config entities:
`field.storage.node.field_x.yml` (one per site, defines the type) and
`field.field.node.<bundle>.field_x.yml` (one per bundle, defines the label and
whether it's required). If a field already has a storage entity, add only the
instance — never create a second storage with a different name for the same
concept.

**Every new field needs three things** or it will not appear to editors:
1. `field.storage.node.field_x.yml` — the storage.
2. `field.field.node.<bundle>.field_x.yml` — the bundle instance.
3. Entries in `core.entity_form_display.node.<bundle>.default.yml` **and**
   `core.entity_view_display.node.<bundle>.default.yml`. A field absent from
   the form display is invisible in the editing UI even though it exists.

**Views.** New views go in `views.view.<name>.yml`. Give every view a real
`label` and `description` in English, a page display with a sane `path`, and a
`pager`. Prefer `type: full` pagers over unlimited for anything content-backed.

**Follow existing files as templates.** Before writing a new content type, read
`config/sync/node.type.article.yml` and its displays. Before writing a view,
read `config/sync/views.view.content.yml`. Match their shape exactly rather
than inventing a structure.

---

## Commits and pull requests

- Prefix every commit title with `DLA-<issue number>: `, e.g.
  `DLA-3: Add Curso content type with duration and level fields`.
- Imperative mood ("Add", not "Added"). No `feat:`/`fix:` prefixes. No trailing
  punctuation. Single line.
- Do **not** add `Co-authored-by` or `Generated with Claude Code` lines.
- Branch naming: `agent/issue-<n>`.
- PR body starts with `Fixes #<n>` and includes a **What I changed** section and
  a **Risks / unsure about** section.
- Agents never merge their own pull requests. A human does.

---

## Scope discipline

Implement only what the issue asks. No drive-by refactors, no reformatting
unrelated config files, no bumping dependencies unless the issue is about a
dependency. `config:export` rewrites every file it touches — if your diff
includes config files unrelated to the issue, revert those hunks before
committing.
