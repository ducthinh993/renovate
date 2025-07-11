---
title: NuGet (.NET)
description: NuGet (.NET) dependencies support in Renovate
---

# NuGet

Renovate can upgrade dependencies in these files:

- `.csproj`
- `.fsproj`
- `.vbproj`

## Version Support

Renovate only works with SDK-style `.csproj`, `.fsproj` or `.vbproj` files.
By default, this includes:

- .NET Core 1.0 and above
- .NET Standard class libraries
- `.csproj`, `.fsproj` or `.vbproj` files that use the SDK-style syntax

To convert your .NET Framework `.csproj`, `.fsproj` or `.vbproj` files into an SDK-style project, follow the steps in this [guide](https://natemcmaster.com/blog/2017/03/09/vs2015-to-vs2017-upgrade/).

## How it works

1. Renovate searches in each repository for any files with a `.csproj`, `.fsproj`, or `.vbproj` extension
1. Existing dependencies are extracted from `<PackageReference>` and `<PackageVersion>` tags
1. Renovate looks up the latest version on [nuget.org](https://nuget.org) (or an alternative feed if configured) to see if any upgrades are available
1. If the source package includes a GitHub URL as its source, and has either:
   - a "changelog" file, or
   - uses GitHub releases

   then release notes for each version are embedded in the generated PR

If your project file references a `packages.config` file, no dependencies will be extracted.
Find out here how to [migrate from `packages.config` to `PackageReference`](https://docs.microsoft.com/en-us/nuget/consume-packages/migrate-packages-config-to-package-reference).

## Alternate feeds

By default Renovate performs all lookups on `https://api.nuget.org/v3/index.json`, but you can set alternative NuGet feeds.
You can set alternative feeds:

- in a [`NuGet.config` file](https://docs.microsoft.com/en-us/nuget/reference/nuget-config-file#package-source-sections) within your repository (Renovate will not search outside the repository), or
- in a Renovate configuration options file like `renovate.json`

```json
{
  "packageRules": [
    {
      "matchDatasources": ["nuget"],
      "registryUrls": [
        "https://api.nuget.org/v3/index.json",
        "https://example1.com/nuget/",
        "https://example2.com/nuget/v3/index.json"
      ]
    }
  ]
}
```

In the example above we've set three NuGet feeds.
The package resolving process uses the `merge` strategy to handle the three feeds.
All feeds are checked for dependency updates, and duplicate updates are merged into a single dependency update.

<!-- prettier-ignore -->
!!! warning
    If your project has lockfile(s), for example a `package.lock.json` file, then you must set alternate feed settings in the `NuGet.config` file only.
    `registryUrls` set in other files are **not** passed to the NuGet commands.

<!-- prettier-ignore -->
!!! note
    Some alternative feeds (e.g. Artifactory) do not implement the full set of [required NuGet resources](https://learn.microsoft.com/en-us/nuget/api/overview#resources-and-schema) for the V3 API. If the `PackageBaseAddress` resource does not exist, Renovate falls back to using the `projectUrl` from the dependency's catalog entry as the `sourceUrl` for the dependency, affecting [changelog detection](key-concepts/changelogs.md#how-renovate-detects-changelogs).

### Protocol versions

NuGet supports two protocol versions, `v2` and `v3`.
The NuGet client and server must use the same version.
When Renovate acts as the client, it can use the `v2` and `v3` protocols.

By default, Renovate uses the `v2` protocol.
If the configured feed URL ends with `index.json`, Renovate uses the `v3` protocol.
So Renovate behaves like the official NuGet client.

#### v3 feed URL not ending with index.json

If a `v3` feed URL does not end with `index.json`, you must specify the version explicitly.

- If the feed is defined in a `NuGet.config` file set the `protocolVersion` attribute to `3`:

  ```xml
  <packageSources>
     <clear />
     <add key="myV3feed" value="http://myV3feed" protocolVersion="3" />
  </packageSources>
  ```

- If the feed is defined via Renovate configuration append `#protocolVersion=3` to the registry URL:

  ```json
  {
    "packageRules": [
      {
        "matchDatasources": ["nuget"],
        "registryUrls": ["https://example1.com/nuget/#protocolVersion=3"]
      }
    ]
  }
  ```

You may need this workaround when you use the JFrog Artifactory.

## Authenticated feeds

Credentials for authenticated/private feeds can be given via host rules in the configuration options (file or command line parameter).

```json
{
  "hostRules": [
    {
      "hostType": "nuget",
      "matchHost": "http://example1.com/nuget",
      "username": "root",
      "password": "p4$$w0rd"
    }
  ]
}
```

If you use Azure DevOps:

- set `matchHost` to `pkgs.dev.azure.com`
- set the username, so Renovate can build the project when it creates the PR

<!-- prettier-ignore -->
!!! note
    Only Basic HTTP authentication (via username and password) is supported.
    For Azure DevOps: use a PAT with `read` permissions on `Packaging`.
    The username of the PAT must match the username of the _user of the PAT_.
    The generated `nuget.config` forces the basic authentication, which cannot be overridden externally!

## Ignoring package files when using presets

Because `nuget` manager has a dedicated `ignorePaths` entry in the `:ignoreModulesAndTests` preset, if you're using any presets that extend it (like `config:recommended`), you need to put your `ignorePaths` inside the `nuget` section for it to be merged.
For example:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended"],
  "nuget": {
    "ignorePaths": ["IgnoreThisPackage/**"]
  }
}
```

Otherwise, all `nuget.ignorePaths` values in `:ignoreModulesAndTests` will override values you put inside `ignorePaths` at the top-level config.

## Future work

We welcome contributions or feature requests to support more patterns or use cases.
