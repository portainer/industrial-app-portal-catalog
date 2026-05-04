---
name: catalog-manager
description: >
  Add new applications to or update existing applications in the Portainer Industrial App Portal (PIAP) catalog.
  Use this skill whenever the user wants to add a new app to the catalog, update an existing catalog entry,
  create a new variant for an app, modify app metadata or configuration, or work with catalog.yaml in any way.
  Also trigger when the user pastes or references a docker-compose.yml file and wants to turn it into a catalog entry.
  Trigger on phrases like "add app to catalog", "new catalog entry", "update the catalog", "add a variant",
  "onboard an application", "register an app", "I have this compose file", or any mention of editing catalog.yaml
  or any sibling `*.yaml` catalog file for app management purposes.
---

# Catalog Manager

You are guiding a user through adding or updating an application in the Portainer Industrial App Portal (PIAP) catalog. This is an interactive, wizard-style workflow that uses `AskUserQuestion` to collect information step by step.

## Before You Start

1. Read `catalog.yaml` to understand current catalog state
2. Look for sibling standalone catalog files (e.g. `softing-sdex-suite.yaml`) — these are valid catalog entries split into their own files
3. Read `app-model.yaml` for the full schema reference
4. Scan `apps/` directory to understand existing patterns
5. Read `CLAUDE.md` if present — it documents repo-specific conventions and overrides anything in this skill
6. Determine whether the user wants to **add a new app** or **update an existing one**

## Tool Dependency

This skill relies on `AskUserQuestion`, which is a deferred tool. If it isn't loaded, request it via ToolSearch (`select:AskUserQuestion`) before starting the interview. Don't proceed without it — the wizard flow needs structured user input.

## Compose File Detection

If the user pastes or provides a docker-compose.yml file, extract as much information as possible before starting the interview:

- **Service name** → suggest as app-id (kebab-case)
- **Image name/tag** → extract publisher, suggest app name, determine versions
- **Ports** → note for documentation/overview
- **Environment variables** → map to `configuration.variables` candidates
- **Volumes** → note volume naming patterns
- **Any special options** (cap_add, command, etc.) → note for the compose file

Present the extracted information to the user as a starting point: "I found the following from your compose file — let me walk you through confirming and filling in the rest."

When creating the compose file for the catalog, replace the image tag with `((PIAP_IMAGE_VERSION))` and replace any environment variable values that should be user-configurable with `((PIAP_VARIABLE_NAME))` placeholders.

**Placeholder quoting rule (mandatory):** Every `((PIAP_*))` placeholder in a compose template MUST sit inside a quoted YAML scalar (single or double quotes). This is the catalog-side complement to IAP's input validation (security finding F016) — it forces the substituted value to be parsed as a string regardless of leading character (`&`, `*`, `>`, `|`, `!`, `:`, `,` would otherwise be interpreted as YAML structural tokens). When you write or modify a compose template, audit every placeholder position and quote it. Block-scalar style (`|`, `>`) is not a substitute. See the "Placeholder Quoting Rule" section in CLAUDE.md for the canonical specification.

## Interactive Interview Flow

Use the `AskUserQuestion` tool at each step to collect information. Present sensible defaults based on existing catalog patterns and any compose file analysis. Don't ask everything at once — guide the user through logical stages.

### Stage 1: Basic Identity

If adding a new app, ask:

- **App ID**: Suggest kebab-case identifier (e.g., `my-app`). Verify it doesn't conflict with existing apps in either `catalog.yaml` or any standalone `*.yaml` file.
- **App Name**: Human-readable display name
- **Publisher**: Organization or maintainer
- **Category**: Suggest from existing categories: `messaging`, `industrial-automation`, `visualization`. Allow custom.

If updating an existing app, ask which app to update and what they want to change, then skip to the relevant stage.

### Stage 2: Description & Media

- **Description**: Brief one-line description for catalog listings
- **Overview**: Detailed paragraph explaining the app's purpose and capabilities
- **Icon URL**: URL to app icon/logo
- **Documentation URL**: Link to official docs
- **Screenshots**: One or more screenshot URLs (optional)
- **Tags**: Searchable tags as a comma-separated list

### Stage 3: Versions

- **Versions**: List of deployable versions (e.g., `["4.1", "4.2"]`)
- Explain that the selected version replaces `((PIAP_IMAGE_VERSION))` in compose files

### Stage 4: Variants

Ask how many variants the app needs. Explain that variants represent different deployment configurations (e.g., Production, Development, Beta, Legacy).

For each variant, collect:

- **Label**: Human-readable name (must be unique within the app). Default first variant to "Production".
- **Version requirement** (optional): Single constraint like `>=3.1.15`, `<4.0.0`, `=3.1.14`. Only one constraint per variant — no ranges.

### Stage 5: Deployment (per variant)

- **Compose file path**: Where the docker-compose.yml will live. Follow the pattern: `apps/{app-id}/{variant-label-lowercased}/docker-compose.yml` — the directory mirrors the variant label in lowercase. For example, the "Production" variant of `cedalo-mc` lives at `apps/cedalo-mc/production/docker-compose.yml`, and the "Beta" variant of `softing-sdex-suite` lives at `apps/softing-sdex-suite/beta/docker-compose.yml`.
- **Custom git source** (optional): If the compose file lives in a different repo than the catalog default. Most apps inherit from `defaults.source`.

If the user already provided a compose file, confirm the modified version (with PIAP placeholders) and write it. Otherwise, help them create one.

**Note:** Kubernetes deployment is part of the schema (`deployment.kubernetes`) but is not yet implemented in PIAP. Don't offer it unless the user explicitly asks.

### Stage 6: Configuration (per variant)

Ask if the app needs configuration. Explain the three options:

1. **No configuration** — Skip the config wizard in PIAP UI (simplest, for apps that just run)
2. **Environment variables only** — Collect variables injected as env vars
3. **Configuration files + variables** — Full config with file templates and variable placeholders

#### If environment variables only:

For each variable, collect:
- `name`: UPPER_SNAKE_CASE identifier
- `label`: Human-readable label for the UI form
- `default` (optional): Pre-filled default value

#### If configuration files needed:

For each config file mount:
- `label`: Display name in UI
- `device_path`: Path on edge device
- `container_path`: Path inside container
- Templates (if providing config files):
  - Template ID, name, description, file path
  - Whether to lock the template selection
  - Which template is the default
- Or omit templates for files that must already exist on the device (e.g., certificates managed outside PIAP)

Then collect variables that map to `((PIAP_*))` placeholders in the templates.

See `apps/softing-sdex-suite/beta/offline-config.yaml` and the corresponding `softing-sdex-suite.yaml` entry for a real example of the templates + variables pattern.

### Stage 7: File Layout Decision

Before writing, ask the user **how the entry should be stored**:

- **Append to `catalog.yaml`** — the original convention. Keeps everything in one file. Best for small or closely related entries.
- **Standalone file** (e.g., `{app-id}.yaml`) at the repo root, sibling to `catalog.yaml` — the newer pattern. Useful when the entry is large, evolves independently, or is owned by a separate team. `softing-sdex-suite.yaml` is the reference example.

If the user has no strong preference, suggest a default based on the entry's size and maturity (small/stable → inline; large/evolving → standalone).

A standalone file does **not** wrap the entry under a top-level `apps:` key — it starts directly with the app-id key. See `softing-sdex-suite.yaml` for the exact shape.

### Stage 8: Review & Confirm

Before writing any files, present a complete summary of what will be created/modified:

- The catalog entry YAML (whether destined for `catalog.yaml` or a standalone file)
- Any new files (compose files, config templates)
- Any modified files

Ask the user to confirm before proceeding.

## Writing Files

### Catalog Entry

Follow these patterns from existing entries:

```yaml
  {app-id}:
    metadata:
      name: "{App Name}"
      description: "{brief description}"
      overview: "{detailed overview}"
      publisher: "{Publisher}"
      versions: ["{version1}", "{version2}"]
      category: "{category}"
      icon: "{icon-url}"
      tags: ["{tag1}", "{tag2}"]
      screenshots:
        - "{screenshot-url}"
      documentation: "{docs-url}"
    variants:
      - label: "{Variant Label}"
        deployment:
          compose:
            path: "apps/{app-id}/{variant-label-lowercased}/docker-compose.yml"
        configuration:  # optional
          variables:
            - name: "{VARIABLE_NAME}"
              label: "{Variable Label}"
              default: "{default-value}"  # optional
```

When inline in `catalog.yaml`, indent under `apps:` (4 spaces from the start). When in a standalone file, the app-id key is at column 0.

### Docker Compose Files

Follow these conventions from existing apps:

- **Directory structure**: `apps/{app-id}/{variant-label-lowercased}/docker-compose.yml`
- **Single service per file** — no multi-container orchestration
- **Service name**: lowercase kebab-case, typically matching or derived from app-id
- **Image tag**: Always use `((PIAP_IMAGE_VERSION))` placeholder, **quoted**: `image: "publisher/app:((PIAP_IMAGE_VERSION))"`
- **Environment variables**: Use `((PIAP_VARIABLE_NAME))` for user-configurable values, **quoted at every placeholder position** (mapping-style `KEY: "((PIAP_X))"` or list-style `- "KEY=((PIAP_X))"`)
- **Volumes**: Use named volumes (no bind mounts). Naming is flexible — existing apps use a mix of styles (`mc_data`, `config`, `mqtt-config`, `{app-id}-data`); pick something descriptive.
- **Start with `services:`** — no `version:` key

Example (note every `((PIAP_*))` position is inside quotes):
```yaml
services:
  my-app:
    image: "publisher/app:((PIAP_IMAGE_VERSION))"
    ports:
      - "8080:8080"
    environment:
      ADMIN_USER: "((PIAP_ADMIN_USER))"
      ADMIN_PASSWORD: "((PIAP_ADMIN_PASSWORD))"
      TZ: "((PIAP_TZ))"
    volumes:
      - my-app-data:/data
    restart: always

volumes:
  my-app-data:
```

### Configuration Template Files

If the app uses configuration file templates:
- Store in `apps/{app-id}/{variant-label-lowercased}/` directory alongside the compose file
- Use `((PIAP_VARIABLE_NAME))` placeholders for dynamic values
- Keep templates minimal and focused

## Validation Checklist

Before finalizing, verify:

- [ ] App ID is unique across `catalog.yaml` AND all sibling standalone catalog files, and uses kebab-case
- [ ] Variant labels are unique within the app
- [ ] Version requirements use single operators only (`>=`, `>`, `=`, `<`, `<=`) — no ranges
- [ ] All template `path` values reference files that exist (or will be created)
- [ ] All template entries have `name`, `description`, and `path` properties
- [ ] `default` property (if present) matches a key in the `templates` dict
- [ ] Variable names use UPPER_SNAKE_CASE
- [ ] Variable names in config match `((PIAP_*))` placeholders in template files
- [ ] Git references use format `github.com/org/repo` (no `https://`)
- [ ] Compose file uses `((PIAP_IMAGE_VERSION))` for the image tag
- [ ] Every `((PIAP_*))` placeholder in compose templates (and any YAML config templates) is inside a quoted scalar — no plain/unquoted positions, no block-scalar `|`/`>` substitutions
- [ ] Variant directory name matches the variant label, lowercased
- [ ] The `metadata.updated` timestamp in `catalog.yaml` is updated to today's date in ISO 8601 UTC format (e.g., `"2026-05-03T00:00:00Z"`) — even when the new entry is in a standalone file

## Updating Existing Apps

When the user wants to update an existing app:

1. Locate the entry — it may be in `catalog.yaml` or in a standalone sibling file
2. Read the current entry
3. Ask what they want to change (metadata, add variant, update config, etc.)
4. Show current values and ask for new values
5. Only modify what's needed — don't rewrite the entire entry
6. Use the Edit tool for surgical changes

## Important Notes

- Always use `AskUserQuestion` to interact with the user — never assume values without confirmation
- Present defaults and suggestions to minimize user effort
- If the user seems unsure about a field, explain what it does using plain language
- The top-level `metadata.updated` field in `catalog.yaml` should always be refreshed when any catalog change is made (inline or standalone)
- Keep YAML formatting consistent with existing entries (2-space indentation)
- When appending a new app inline, insert it at the end of the `apps:` section for cleaner diffs
