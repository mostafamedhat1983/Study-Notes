---
tags:
  - Terraform
---
The **plugin-cache** feature in Terraform lets you store provider plugin binaries in a **shared local directory** and reuse them across multiple Terraform projects.

By default, Terraform downloads provider plugins into each project's `.terraform` directory. That means if several projects use the same provider, Terraform may download the same binary multiple times. With plugin caching enabled, Terraform first checks a central cache directory and reuses any matching plugin already stored there, which reduces repeated downloads and speeds up `terraform init`.

## How to enable it

You can enable the plugin cache in either of these ways:

- Set `plugin_cache_dir` in the Terraform CLI config file.
    
- Set the `TF_PLUGIN_CACHE_DIR` environment variable.
    

## Why use plugin cache

- It provides a single, centralized location for provider plugins.
    
- It avoids repeated downloads of large provider binaries across multiple projects.
    
- It reduces bandwidth usage and speeds up `terraform init`.
    
- It is especially useful on developer machines and CI/CD runners that reuse the same environment.
    
- It can reduce reliance on internet access for commonly used providers.
    

## Default behavior

Without plugin caching, Terraform stores provider binaries inside each project's `.terraform` directory.

This is simple and works well for isolated projects, but it can lead to:

- Duplicate plugin downloads.
    
- Higher network usage.
    
- Slower initialization across multiple projects.
    

## Alternatives

- Default per-project plugin downloads: simpler, but duplicates binaries and increases bandwidth usage.
    
- Persistent caching in CI/CD runners: useful when runners are reused between jobs.
    
- Pre-packaging plugins in CI/CD artifacts: works, but is usually less flexible and less seamless than Terraform’s built-in plugin cache.
    

## Security and best practices

- Do not share the plugin cache directory across users or machines unless you fully trust the environment and permissions are properly controlled.
    
- Treat cached plugins as executable binaries; a compromised cache can become a security risk.
    
- Terraform verifies provider signatures by default when downloading from the registry, so keep that protection enabled.
    
- In ephemeral or stateless CI environments, weigh the benefit of persistent caching against the simplicity of downloading fresh copies each time.
    
- On developer machines, a local plugin cache is usually safe and efficient.
    

## Example configuration

Add this to your Terraform CLI config file, such as `~/.terraformrc` on Linux/macOS or `terraform.rc` on Windows:

> plugin_cache_dir = "$HOME/.terraform.d/plugin-cache"

Make sure the directory exists before running Terraform.