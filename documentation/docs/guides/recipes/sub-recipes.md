---
sidebar_position: 3
title: Sub-Recipes For Specialized Tasks
sidebar_label: Sub-Recipes
description: Learn how a recipe can use sub-recipes to perform specific tasks
---

Sub-recipes are recipes that are used by another recipe to perform specific tasks. They enable:
- **Multi-step workflows** - Break complex tasks into distinct phases with specialized expertise
- **Reusable components** - Create common tasks that can be used in various workflows

## How Sub-Recipes Work

The "main recipe" registers its sub-recipes in the `sub_recipes` field, which contains the following fields:

- `name`: Unique identifier for the sub-recipe, used to generate the tool name
- `path`: File path to the sub-recipe file (relative or absolute)
- `values`: (Optional) Pre-configured parameter values that are always passed to the sub-recipe

When the main recipe is run, Goose generates a tool for each sub-recipe that:
- Accepts parameters defined by the sub-recipe
- Executes the sub-recipe in a separate session with its own context
- Returns output to the main recipe

Sub-recipe sessions run in isolation - they don't share conversation history, memory, or state with the main recipe or other sub-recipes. Additionally, sub-recipes cannot define their own sub-recipes (no nesting allowed).

### Parameter Handling

Sub-recipes receive parameters in two ways:

1. **Pre-set values**: Fixed parameter values defined in the `values` field are automatically provided and cannot be overridden at runtime
2. **Automatic parameter inheritance**: Sub-recipes automatically have access to all parameters passed to the main recipe at runtime.

Pre-set values take precedence over inherited parameters. If both the main recipe and `values` field provide the same parameter, the `values` version is used.

:::info Template Variables
Parameters received by sub-recipes can be used in prompts and instructions using `{{ parameter_name }}` syntax.
:::

## Examples

### Sequential Processing

This Code Review Pipeline example shows a main recipe that uses two sub-recipes to perform a comprehensive code review:

**Usage:**
```bash
goose run --recipe code-review-pipeline.yaml --params repository_path=/path/to/repo
```

**Main Recipe:**

```yaml
# code-review-pipeline.yaml
version: "1.0.0"
title: "Code Review Pipeline"
description: "Automated code review using sub-recipes"
instructions: |
  Perform a code review using the available sub-recipe tools.
  Run security analysis first, then code quality analysis.

parameters:
  - key: repository_path
    input_type: string
    requirement: required
    description: "Path to the repository to review"

sub_recipes:
  - name: "security_scan"
    path: "./sub-recipes/security-analysis.yaml"
    values:
      scan_level: "comprehensive"
  
  - name: "quality_check"
    path: "./sub-recipes/quality-analysis.yaml"

extensions:
  - type: builtin
    name: developer
    timeout: 300
    bundled: true

prompt: |
  Review the code at {{ repository_path }} using the sub-recipe tools.
  Run security scan first, then quality analysis.
```

**Sub-Recipes:**

<details>
  <summary>security_scan</summary>
  ```yaml
  # sub-recipes/security-analysis.yaml
  version: "1.0.0"
  title: "Security Scanner"
  description: "Analyze code for security vulnerabilities"
  instructions: |
    You are a security expert. Analyze the provided code for security issues.
    Focus on common vulnerabilities like SQL injection, XSS, and authentication flaws.

  parameters:
    - key: repository_path
      input_type: string
      requirement: required
      description: "Path to the code to analyze"
    
    - key: scan_level
      input_type: string
      requirement: optional
      default: "standard"
      description: "Depth of security scan (basic, standard, comprehensive)"

  extensions:
    - type: builtin
      name: developer
      timeout: 300
      bundled: true

  prompt: |
    Perform a {{ scan_level }} security analysis on the code at {{ repository_path }}.
    Report any security vulnerabilities found with severity levels and recommendations.
  ```
</details>

<details>
  <summary>quality_check</summary>
  ```yaml
  # sub-recipes/quality-analysis.yaml
  version: "1.0.0"
  title: "Code Quality Analyzer"
  description: "Analyze code quality and best practices"
  instructions: |
    You are a code quality expert. Review code for maintainability, 
    readability, and adherence to best practices.

  parameters:
    - key: repository_path
      input_type: string
      requirement: required
      description: "Path to the code to analyze"

  extensions:
    - type: builtin
      name: developer
      timeout: 300
      bundled: true

  prompt: |
    Analyze the code quality at {{ repository_path }}.
    Check for code smells, complexity issues, and suggest improvements.
  ```
</details>

### Conditional Processing

This Smart Project Analyzer example shows conditional logic that chooses between different sub-recipes based on analysis:

**Usage:**
```bash
goose run --recipe smart-analyzer.yaml --params repository_path=/path/to/project
```

**Main Recipe:**

```yaml
# smart-analyzer.yaml
version: "1.0.0"
title: "Smart Project Analyzer"
description: "Analyze project and choose appropriate processing based on type"
instructions: |
  First examine the repository to determine the project type (web app, CLI tool, library, etc.).
  Based on what you find:
  - If it's a web application, use the web_security_audit sub-recipe
  - If it's a CLI tool or library, use the api_documentation sub-recipe
  Only run one sub-recipe based on your analysis.

parameters:
  - key: repository_path
    input_type: string
    requirement: required
    description: "Path to the repository to analyze"

sub_recipes:
  - name: "web_security_audit"
    path: "./sub-recipes/web-security.yaml"
    values:
      check_cors: "true"
      check_csrf: "true"
  
  - name: "api_documentation"
    path: "./sub-recipes/api-docs.yaml"
    values:
      format: "markdown"

extensions:
  - type: builtin
    name: developer
    timeout: 300
    bundled: true

prompt: |
  Analyze the project at {{ repository_path }} and determine its type.
  Then run the appropriate sub-recipe tool based on your findings.
```

**Sub-Recipes:**

<details>
  <summary>web_security_audit</summary>
  ```yaml
  # sub-recipes/web-security.yaml
  version: "1.0.0"
  title: "Web Security Auditor"
  description: "Security audit for web applications"
  instructions: |
    You are a web security specialist. Audit web applications for 
    security vulnerabilities specific to web technologies.

  parameters:
    - key: repository_path
      input_type: string
      requirement: required
      description: "Path to the web application code"
    
    - key: check_cors
      input_type: string
      requirement: optional
      default: "false"
      description: "Whether to check CORS configuration"
    
    - key: check_csrf
      input_type: string
      requirement: optional
      default: "false"
      description: "Whether to check CSRF protection"

  extensions:
    - type: builtin
      name: developer
      timeout: 300
      bundled: true

  prompt: |
    Perform a web security audit on {{ repository_path }}.
    {% if check_cors == "true" %}Check CORS configuration for security issues.{% endif %}
    {% if check_csrf == "true" %}Verify CSRF protection is properly implemented.{% endif %}
    Focus on web-specific vulnerabilities like XSS, authentication flaws, and session management.
  ```
</details>

<details>
  <summary>api_documentation</summary>
  ```yaml
  # sub-recipes/api-docs.yaml
  version: "1.0.0"
  title: "API Documentation Generator"
  description: "Generate documentation for APIs and libraries"
  instructions: |
    You are a technical writer specializing in API documentation.
    Create comprehensive documentation for code libraries and APIs.

  parameters:
    - key: repository_path
      input_type: string
      requirement: required
      description: "Path to the code to document"
    
    - key: format
      input_type: string
      requirement: optional
      default: "markdown"
      description: "Output format for documentation (markdown, html, rst)"

  extensions:
    - type: builtin
      name: developer
      timeout: 300
      bundled: true

  prompt: |
    Generate {{ format }} documentation for the code at {{ repository_path }}.
    Include API endpoints, function signatures, usage examples, and installation instructions.
    Focus on making it easy for developers to understand and use this code.
  ```
</details>

## Best Practices
- **Single responsibility**: Each sub-recipe should have one clear purpose
- **Clear parameters**: Use descriptive names and descriptions
- **Pre-set fixed values**: Use `values` for parameters that don't change
- **Test independently**: Verify sub-recipes work alone before combining

## Learn More
Check out the [Goose Recipes](/docs/guides/recipes) guide for more docs, tools, and resources to help you master Goose recipes.
