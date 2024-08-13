# Provena Configuration Repository

## Purpose

This repository contains configuration files for the Provena project. It serves as a centralized location for managing configuration across different environments and components of the Provena system. This configuration is a mix of sensitive and non-sensitive information, but it is managed centrally to simplify this process.

**Warning: while this repo is private, please don't commit secrets to it, instead commit references/IDs for those secrets stored in AWS Secret Manager**

## `cdk.context.json`

This repo does not contain sample `cdk.context.json` files, but we recommend including this in this repo to make sure deployments are deterministic. This will be generated upon first deploy.

## Structure

The repository is organized using a hierarchical structure based on namespaces and stages:

```
.
├── README.md
└── <your-namespace>
    ├── base
    ├── dev
    └── feat
```

### Namespaces

A namespace represents a set of related deployment stages, usually one namespace per organisation/use case. 

### Stages

Within each namespace, there are multiple stages representing different environments/deployment specifications

- `base`: Contains common base configurations shared across all stages within the namespace
- `dev`: Sample development environment configurations 
- `feat`: Sample feature branch workflow environment configurations

### Feat stage

The feat stage supports the feature branch deployment workflow which is now a part of the open-source workflow. This makes use of environment variable substitution which is described later.

## File Organization

Configuration files are placed within the appropriate namespace and stage directories. Currently:

```
.
├── README.md
└── your-namespace
    ├── base
    │   ├── admin-tooling
    │   │   └── environments.json
    │   └── utilities
    │       └── supporting-stacks
    │           ├── feature-branch-manager
    │           │   └── configs
    │           │       └── dev.json
    │           └── github-creds
    │               └── configs
    │                   ├── build.json
    │                   └── ops.json
    ├── dev
    │   └── infrastructure
    │       └── configs
    │           └── dev.json
    └── feat
        └── infrastructure
            └── configs
                ├── feat-ui.json
                └── feat.json
```

## Usage by Provena config scripts

### Base Configurations

Files in the `base` directory of a namespace are applied first, regardless of the target stage. This allows you to define common configurations that are shared across all stages within a namespace.

### Stage-Specific Configurations

Files in stage-specific directories (e.g., `dev`, `test`, `prod`) are applied after the base configurations. They can override or extend the base configurations as needed.

## Interaction with Provena Repository

The main Provena repository contains a configuration management script that interacts with this configuration repository. Here's how it works:

1. The script clones or uses a pre-cloned version of this configuration repository.
2. It then copies the relevant configuration files based on the specified namespace and stage.
3. The process follows these steps:
   a. Copy all files from the `<namespace>/base/` directory (if it exists).
   b. Copy all files from the `<namespace>/<stage>/` directory, potentially overwriting files from the base configuration.
4. The copied configuration files are then used by the Provena system for the specified namespace and stage.

## Best Practices

1. Use version control: Commit and push changes to this repository regularly.
2. Document changes: Use clear, descriptive commit messages and update this README if you make structural changes.
3. Minimize secrets: Avoid storing sensitive information like passwords or API keys directly in these files. Instead, use secure secret management solutions.

# Config variable substitution syntax

This project supports a syntax for environment variable substitution within JSON files. This feature allows you to create dynamic JSON configurations that can adapt to different environments without modifying the JSON itself.

## Basic Syntax

The basic syntax for environment variable substitution is:

```
${VARIABLE_NAME}
```

When the JSON is processed, this will be replaced with the value of the environment variable `VARIABLE_NAME`.

## Advanced Syntax

### Default Values

You can specify a default value to use if the environment variable is not set:

```
${VARIABLE_NAME:-default_value}
```

### Conditional Substitution

For more complex scenarios, you can use conditional substitution:

```
${VARIABLE_NAME?value_if_set||value_if_not_set}
```

## Examples

Here are some examples to illustrate the different substitution patterns:

1. Basic substitution:

   ```json
   {
     "database_url": "${DB_URL}",
     "port": ${PORT:-8080}
   }
   ```

2. Conditional substitution:

   ```json
   {
     "debug_mode": "${DEBUG?'true'||'false'}",
     "log_level": "${LOG_LEVEL?'debug'||'info'}"
   }
   ```

3. In-line substitution:

   ```json
   {
     "api_endpoint": "https://${API_HOST:-api.example.com}:${API_PORT:-443}/v1"
   }
   ```

4. Complex conditional substitution:
   ```json
   {
     "database": {
       "host": "${DB_HOST}",
       "port": "${DB_PORT?!!||5432}",
       "ssl": "${DB_SSL?true||false}",
       "ssl_cert": "${DB_SSL?'${SSL_CERT_PATH}'||null}"
     }
   }
   ```

## Special Case: Self-Referential Substitution with quotation stripping

A particularly useful pattern is the self-referential substitution with quotation stripping.

```json
{
  "git_commit_id": "${GIT_COMMIT_ID?'!!'||null}"
}
```

The goal of the above syntax is to

- allow you to have control over whether the output is quoted or not
- ensure the original JSON remains valid/parseable before substitution
- allow you to reference the variable value as part of conditional logic

There is a special condition in the config script as follows

**If a conditional substitution is directly surrounded by double quotes, then remove those double quotes from the output**

There are also two general rules:

- always replace `'` with `"`
- always replace `!!` with the value of the variable

Combining these two features means

```json
{
  "git_commit_id": "${GIT_COMMIT_ID?'!!'||null}"
}
```

can resolve to two values

- if `GIT_COMMIT_ID` is not "falsey" (i.e. not defined or literal value 'false') then the output will be

```
{
  "git_commit_id": "123456"
}
```

- if `GIT_COMMIT_ID` is not defined then

```
{
  "git_commit_id": null
}
```
