---
tags:
  - Terraform
---
The `plugin-cache` feature in Terraform enables caching of provider plugin binaries in a shared directory across multiple projects. By default, Terraform stores plugin binaries within a project's `.terraform` directory, leading to redundant downloads if the same plugin is used elsewhere. With the plugin cache enabled—configured via the `plugin_cache_dir` option in the CLI config file or by setting the `TF_PLUGIN_CACHE_DIR` environment variable—Terraform checks the specified cache first and avoids re-downloading plugins that are already present

Why use the plugin-cache: - Provides a single, centralized location for provider plugins. - Prevents repeated large downloads (hundreds of MB per plugin) for multiple projects or CI/CD jobs on the same machine, saving bandwidth and speeding up `terraform init`. - Can be set on developer machines or CI/CD runners to avoid internet dependency for commonly used plugins.

Alternatives: - Default per-project downloads (default `.terraform` behavior): Simpler but duplicates binaries and increases network traffic. - Remote execution environments with local ephemeral filesystems: Benefit more from plugin caches or persistent caching strategies. - Pre-packaging plugins in CI/CD artifacts: Works, but not as seamless or flexible as the plugin cache option.

`Security & Best Practice Check - Do not share the plugin cache directory across users or machines unless all trust and file system permission boundaries are maintained. Plugins are executable binaries and a compromised cache could be an attack vector. - Always verify plugin signatures (Terraform does by default when downloading using the registry). - For ephemeral or stateless CI environments, consider the trade-off between using large persistent caches versus always downloading. For developer machines, a local cache is generally safe and efficient. Example configuration (~/.terraformrc or terraform.rc):`

plugin_cache_dir = "$HOME/.terraform.d/plugin-cache"
(Make sure the directory exists before running Terraform.)