# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is the **industrial-app-portal-catalog** repository for Portainer Industrial App Portal (PIAP). It defines an application catalog for deploying containerized applications to edge devices. The catalog uses YAML-based configuration files to specify application metadata, deployment options, and runtime configurations.

## Key Files

- **catalog.yaml**: The main catalog file containing all available applications (TensorFlow, InfluxDB, Grafana, Node-RED, Mosquitto). This is the production catalog that users interact with.
- **app-model.yaml**: A comprehensive example/template showing all possible configuration options with detailed inline comments. This serves as documentation for the catalog schema.
- **apps/**: Directory containing deployment files and configuration templates for each application

## Architecture

### Catalog Schema Structure

The catalog defines applications with a hierarchical structure:

```
metadata (catalog info)
├── version
└── updated

defaults (inherited config)
└── source
    ├── git
    └── ref

apps
└── [app-id]
    ├── metadata (app-specific)
    └── variants (list)
        └── [variant]
            ├── label
            ├── version_requirement
            ├── deployment
            │   ├── compose
            │   └── kubernetes
            ├── configuration
            │   ├── files
            │   └── variables
            └── requirements
```

### Version Requirements

Version requirements use single-constraint operators:
- Supported: `>=`, `>`, `=`, `<`, `<=`
- Only ONE constraint per variant (eliminates conflicting constraints)
- Example: `version_requirement: ">=3.1.15"`

### Placeholder System

The system uses `((PIAP_*))` placeholders for dynamic value injection:
- `((PIAP_IMAGE_VERSION))`: Replaced with user-selected application version
- `((PIAP_VARIABLE_NAME))`: Replaced with variable values from `configuration.variables`
- If placeholder exists in config file: replaced with the value
- If placeholder NOT found: injected as environment variable

Example in docker-compose.yml:
```yaml
image: nodered/node-red:((PIAP_IMAGE_VERSION))
```

Example in config template (apps/node-red/prod/mqtt.json):
```json
{
  "broker": "((PIAP_MQTT_BROKER))",
  "credentials": {
    "user": "((PIAP_MQTT_USERNAME))"
  }
}
```

### Git Source Inheritance

Deployment files can be sourced from git repositories:
- Global default defined in `defaults.source` (git + ref)
- Inherited by all deployments unless overridden
- Can be overridden at deployment level
- Currently only Compose deployments inherit these defaults (Kubernetes not yet implemented)
- Example: `defaults.source.git: "github.com/your-org/edge-templates"`

### Configuration System

The `configuration` section is **optional**. If omitted entirely, the PIAP UI will skip the configuration step in the deployment wizard and proceed to the next step. Use this for simple applications that require no custom configuration.

When present, the configuration system consists of two main components:

#### 1. Configuration Files (`configuration.files`)
- **Central definition of HOW configuration files are injected into containers**
- List of file mount points with their template options
- Defines the bind mount mappings that PIAP will automatically create:
  - `device_path`: Where the config file is stored on the edge device
  - `container_path`: Where the config file is mounted inside the container
- Each file has inline `templates` (a dict of configuration file options):
  - **Template properties (all required)**: `name`, `description`, `path`
  - Users select from these templates via dropdown in PIAP UI
  - Templates contain `((PIAP_*))` placeholders replaced with values from `variables`
- **Default selection**: `default` property (optional)
  - If omitted with multiple templates: first template in YAML order is selected
  - If only one template exists: automatically selected (no dropdown)
- **Locked configurations**: `locked` property (optional, defaults to false)
  - When `true`: UI shows read-only template selection (no dropdown)
  - When `false` or omitted: UI shows dropdown with all templates
  - Can have `locked: true` with multiple templates for future flexibility
  - Use cases: security policies, compliance requirements, standardized configs
- **Mounting existing device files**:
  - Omit `templates` section entirely → bind mount files already on device (e.g., certificates, keys)
  - Lenient parsing: `templates: {}` (empty dict) also accepted, treated same as omitted
  - Preferred syntax: omit `templates` for clarity
  - Use case: files managed outside of PIAP (operator-placed, shared configs, etc.)
- PIAP handles: file propagation to devices, bind mounts, volume management

#### 2. Variables (`configuration.variables`)
- User-provided values collected during app installation
- **Two usage modes:**
  - **With `files` section**: Replace `((PIAP_VARIABLE_NAME))` placeholders in template files
  - **Standalone (without `files`)**: Injected as environment variables only
- Can have default values pre-filled in the installation form
- Processing logic:
  - If placeholder `((PIAP_VARIABLE_NAME))` exists in template file → replaced with the value
  - If placeholder NOT found → injected as environment variable `VARIABLE_NAME`

**Example Combinations:**

```yaml
# No configuration needed (simplest case)
# Omit configuration section entirely
# PIAP UI will skip the configuration wizard step
variants:
  - label: "Production"
    deployment:
      compose:
        path: "apps/simple-app/docker-compose.yml"
    # No configuration section - no config step in UI

# Full configuration: files with multiple templates + variables
variants:
  - label: "Production"
    configuration:
      files:
        - label: "App Config"
          device_path: "/data/config.json"
          container_path: "/app/config.json"
          templates:
            production:
              name: "Production Config"
              description: "Production-ready configuration"
              path: "apps/myapp/prod/config.json"
            development:
              name: "Development Config"
              description: "Debug mode enabled"
              path: "apps/myapp/dev/config.json"
          default: "production"
      variables:
        - name: "API_KEY"
          label: "API Key"

# Single template (locked configuration)
variants:
  - label: "Production"
    configuration:
      files:
        - label: "Security Policy"
          device_path: "/data/security.json"
          container_path: "/app/security.json"
          templates:
            corporate:
              name: "Corporate Security Policy"
              description: "Mandatory security configuration"
              path: "apps/myapp/security.json"
          locked: true

# Variables only (environment variables, no config files)
variants:
  - label: "Production"
    configuration:
      variables:
        - name: "LOG_LEVEL"
          label: "Log Level"
          default: "info"

# Mount existing device file (no templates)
# File must already exist on device before container starts
variants:
  - label: "Production"
    configuration:
      files:
        - label: "SSL Certificate"
          device_path: "/data/certs/ssl.crt"
          container_path: "/app/certs/ssl.crt"
          # No templates section - file must already exist on device
          # Note: templates: {} also accepted but omitting is preferred
```

## File Validation

When modifying YAML files, ensure:
- Version requirements use single operators only (not ranges like `>=1.0.0 <2.0.0`)
- Variant labels are unique within an application
- All template `path` properties reference actual files in the `apps/` directory
- Template properties `name`, `description`, and `path` are all present (required)
- `default` property (if present) matches a key in the `templates` dict
- If multiple templates exist and no `default` is specified, the first template will be selected
- Empty `templates: {}` is treated same as omitted (no validation error, but omitting is preferred)
- Variable names in `configuration.variables` match placeholders in template files (with `PIAP_` prefix)
- All git references use format: `github.com/org/repo` (no https://)

## Common Tasks

### Adding a New Application

1. Add entry to `catalog.yaml` under `apps:`
2. Define metadata (name, description, versions, category, etc.)
3. Create at least one variant (typically `prod`)
4. Define deployment options (compose/kubernetes)
5. Create deployment files in `apps/[app-id]/`
6. (Optional) Add `configuration` section if the app needs config files or variables
7. Create template files in `apps/[app-id]/` and define them inline under `configuration.files[].templates`

### Adding a New Variant

1. Add new list item to `apps.[app-id].variants` array
2. Define `label` (required, must be unique within the app)
3. (Optional) Define `version_requirement` if version constraints needed
4. Define deployment sources (can reuse same files as other variants)
5. (Optional) Add `configuration.files` with inline templates if needed
6. (Optional) Define `configuration.variables` for variables/placeholders

### Adding Configuration Templates to a File

1. Locate the file entry in `configuration.files`
2. Add new template to the `templates` dict:
   ```yaml
   templates:
     new_template_id:
       name: "Template Name"
       description: "When to use this template"
       path: "apps/[app-id]/path/to/template.json"
   ```
3. Create the template file in `apps/[app-id]/`
4. Use `((PIAP_VARIABLE_NAME))` syntax for dynamic values
5. Ensure matching variables are defined in `configuration.variables`
6. (Optional) Update `default` property if this should be the default template

## Compose Deployment and Edge Stack Integration

PIAP processes Docker Compose files and deploys them as Portainer Edge Stacks.

**How it works:**
1. PIAP fetches the `docker-compose.yml` from git at deployment time
2. Any `((PIAP_*))` placeholders are interpolated directly into the compose content
3. Volume mounts for `configuration.files` are injected into the compose file
4. The modified compose is sent to Portainer as an Edge Stack
5. Environment variables are passed to the Edge Stack for variables NOT used as placeholders
6. Config files are delivered via Portainer Edge Config (folder-per-device structure)

**Edge Config Delivery:**
Configuration files are delivered to devices via Portainer Edge Config using a folder-per-device structure. Each device receives its own interpolated config files based on its resolved variable values. This allows devices to share an Edge Stack even when they have different configuration values.

**Note:** Kubernetes support is planned for a future release and will use platform-native patterns (ConfigMaps, Secrets).

## Testing Changes

This repository contains configuration files only. Testing typically involves:
- YAML syntax validation
- Schema validation against PIAP requirements
- Integration testing with actual PIAP deployment (out of scope for this repo)

## Reference

See `app-model.yaml` for a fully documented example showing all available configuration options with inline comments explaining each field's purpose.
