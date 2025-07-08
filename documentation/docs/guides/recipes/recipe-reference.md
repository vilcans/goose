---
sidebar_position: 2
title: Recipe Reference Guide
description: Complete technical reference for creating and customizing recipes in Goose via the CLI.
---

Recipes are reusable Goose configurations that package up a specific setup so it can be easily shared and launched by others.

## Recipe File Format

Recipes can be defined in either:
- `.yaml` files (recommended)
- `.json` files 

Files should be named either:
- `recipe.yaml`/`recipe.json` 
- `<recipe_name>.yaml`/`<recipe_name>.json`

After creating recipe files, you can use [`goose` CLI commands](/docs/guides/goose-cli-commands) to run or validate the files and to manage recipe sharing.

## Recipe Structure

### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `version` | String | The recipe format version (e.g., "1.0.0") |
| `title` | String | A short title describing the recipe |
| `description` | String | A detailed description of what the recipe does |

### Optional Fields

| Field | Type | Description |
|-------|------|-------------|
| `instructions` | String | Template instructions that can include parameter substitutions |
| `prompt` | String | A template prompt that can include parameter substitutions; required in headless (non-interactive) mode |
| `parameters` | Array | List of parameter definitions |
| `extensions` | Array | List of extension configurations |
| `sub_recipes` | Array | List of sub-recipes |
| `response` | Object | Configuration for structured output validation |

## Parameters

Each parameter in the `parameters` array has the following structure:

### Required Parameter Fields

| Field | Type | Description |
|-------|------|-------------|
| `key` | String | Unique identifier for the parameter |
| `input_type` | String | Type of input (e.g., "string") |
| `requirement` | String | One of: "required", "optional", or "user_prompt" |
| `description` | String | Human-readable description of the parameter |

### Optional Parameter Fields

| Field | Type | Description |
|-------|------|-------------|
| `default` | String | Default value for optional parameters |

### Parameter Requirements

- `required`: Parameter must be provided when using the recipe
- `optional`: Can be omitted if a default value is specified
- `user_prompt`: Will interactively prompt the user for input if not provided

:::important
- Optional parameters MUST have a default value specified
- Required parameters cannot have default values
- Parameter keys must match any template variables used in instructions or prompt
:::

## Extensions

The `extensions` field allows you to specify which Model Context Protocol (MCP) servers and other extensions the recipe needs to function. Each extension in the array has the following structure:

### Extension Fields

| Field | Type | Description |
|-------|------|-------------|
| `type` | String | Type of extension (e.g., "stdio") |
| `name` | String | Unique name for the extension |
| `cmd` | String | Command to run the extension |
| `args` | Array | List of arguments for the command |
| `timeout` | Number | Timeout in seconds |
| `bundled` | Boolean | (Optional) Whether the extension is bundled with Goose |
| `description` | String | Description of what the extension does |

### Example Extension Configuration

```yaml
extensions:
  - type: stdio
    name: codesearch
    cmd: uvx
    args:
      - mcp_codesearch@latest
    timeout: 300
    bundled: true
    description: "Query https://codesearch.sqprod.co/ directly from goose"
  
  - type: stdio
    name: presidio
    timeout: 300
    cmd: uvx
    args:
      - 'mcp_presidio@latest'
    description: "For searching logs using Presidio"
```

## Sub-Recipes

The `sub_recipes` field specifies the [sub-recipes](/docs/guides/recipes/sub-recipes) that the main recipe calls to perform specific tasks. Each sub-recipe in the array has the following structure:

### Sub-Recipe Fields

| Field | Type | Description |
|-------|------|-------------|
| `name` | String | Unique identifier for the sub-recipe |
| `path` | String | Relative or absolute path to the sub-recipe file |
| `values` | Object | (Optional) Pre-configured parameter values that are passed to the sub-recipe |

### Example Sub-Recipe Configuration

```yaml
sub_recipes:
  - name: "security_scan"
    path: "./sub-recipes/security-analysis.yaml"
    values:  # in key-value format: {parameter_name}: {parameter_value}
      scan_level: "comprehensive"
      include_dependencies: "true"
  
  - name: "quality_check"
    path: "./sub-recipes/quality-analysis.yaml"
```

## Structured Output with `response`

The `response` field enables recipes to enforce a final structured JSON output from Goose. When you specify a `json_schema`, Goose will:

1. **Validate the output**: Validates the output JSON against your JSON schema with basic JSON schema validations
2. **Final structured output**: Ensure the final output of the agent is a response matching your JSON structure

This **Enables automation** by returning consistent, parseable results for scripts and workflows.

### Basic Structure

```yaml
response:
  json_schema:
    type: object
    properties:
      # Define your fields here, with their type and description
    required:
      # List required field names
```

### Simple Example

```yaml
version: "1.0.0"
title: "Task Summary"
description: "Summarize completed tasks"
prompt: "Summarize the tasks you completed"
response:
  json_schema:
    type: object
    properties:
      summary:
        type: string
        description: "Brief summary of work done"
      tasks_completed:
        type: number
        description: "Number of tasks finished"
      next_steps:
        type: array
        items:
          type: string
        description: "Recommended next actions"
    required:
      - summary
      - tasks_completed
```

## Template Support

Recipes support Jinja-style template syntax in both `instructions` and `prompt` fields:

```yaml
instructions: "Follow these steps with {{ parameter_name }}"
prompt: "Your task is to {{ action }}"
```

Advanced template features include:
- Template inheritance using `{% extends "parent.yaml" %}`
- Blocks that can be defined and overridden:
  ```yaml
  {% block content %}
  Default content
  {% endblock %}
  ```

## Built-in Parameters

| Parameter | Description |
|-----------|-------------|
| `recipe_dir` | Automatically set to the directory containing the recipe file |

## Complete Recipe Example

```yaml
version: "1.0.0"
title: "Example Recipe"
description: "A sample recipe demonstrating the format"
instructions: "Follow these steps with {{ required_param }} and {{ optional_param }}"
prompt: "Your task is to use {{ required_param }}"
parameters:
  - key: required_param
    input_type: string
    requirement: required
    description: "A required parameter example"
  
  - key: optional_param
    input_type: string
    requirement: optional
    default: "default value"
    description: "An optional parameter example"
  
  - key: interactive_param
    input_type: string
    requirement: user_prompt
    description: "Will prompt user if not provided"

extensions:
  - type: stdio
    name: codesearch
    cmd: uvx
    args:
      - mcp_codesearch@latest
    timeout: 300
    bundled: true
    description: "Query codesearch directly from goose"

response:
  json_schema:
    type: object
    properties:
      result:
        type: string
        description: "The main result of the task"
      details:
        type: array
        items:
          type: string
        description: "Additional details of steps taken"
    required:
      - result
      - status
```

## Template Inheritance

Parent recipe (`parent.yaml`):
```yaml
version: "1.0.0"
title: "Parent Recipe"
description: "Base recipe template"
prompt: |
  {% block prompt %}
  Default prompt text
  {% endblock %}
```

Child recipe:
```yaml
{% extends "parent.yaml" %}
{% block prompt %}
Modified prompt text
{% endblock %}
```

## Recipe Location

Recipes can be loaded from:

1. Local filesystem:
   - Current directory
   - Directories specified in `GOOSE_RECIPE_PATH` environment variable
   
2. GitHub repositories:
   - Configure using `GOOSE_RECIPE_GITHUB_REPO` configuration key
   - Requires GitHub CLI (`gh`) to be installed and authenticated

## Validation Rules

The following rules are enforced when loading recipes:

1. All template variables must have corresponding parameter definitions
2. Optional parameters must have default values
3. Parameter keys must be unique
4. Recipe files must be valid YAML or JSON
5. Required fields (version, title, description) must be present

## Error Handling

Common errors to watch for:

- Missing required parameters
- Optional parameters without default values
- Template variables without parameter definitions
- Invalid YAML/JSON syntax
- Missing required fields
- Invalid extension configurations

When these occur, Goose will provide helpful error messages indicating what needs to be fixed.

## Learn More
Check out the [Goose Recipes](/docs/guides/recipes) guide for more docs, tools, and resources to help you master Goose recipes.