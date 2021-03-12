# Source Pinning

- [Jon Douglas](https://github.com/JonDouglas) & [Nikolche Kolev](https://github.com/nkolev92)
- Start Date: (2021-02-16)
- GitHub Issue:([6867](https://github.com/NuGet/Home/issues/6867))

## Summary

NuGet downloads packages from the fastest responding source. Given that many .NET developers leverage the .NET ecosystem for their public package needs using [NuGet.org](https://www.nuget.org/), they also leverage privately hosted sources & local test sources to consume proprietary packages. When a combination of public, private, and local sources are defined in a `NuGet.config`, [NuGet provides the package through a general package installation process](https://docs.microsoft.com/en-us/nuget/concepts/package-installation-process). This can lead to packages being acquired by a source not always expected.

There are circumstances in [NuGet Package Dependency Resolution](https://docs.microsoft.com/en-us/nuget/concepts/dependency-resolution) which a newer version of a package will result being dowloaded when using public & private sources, which can lead to an attack known as [Dependency Confusion](https://medium.com/@alex.birsan/dependency-confusion-4a5d60fec610).

This proposal introduces a concept known as source pinning, allowing a developer to pin a package to a specific source(s) of their choosing.

## Motivation

Giving the user the choice on where their packages come from has been an [outstanding ask in the .NET ecosystem](https://github.com/NuGet/Home/issues/6867). To overcome this, users would typically create a single source for their packages to ensure they are getting the correct package restored in their project(s). We'd like to provide a more pleasant user experience in which users can define what source(s) a package can be resolved from. This provides the user more control over the security of their software supply chain & expectations of where a package is restored from.

In addition, by having packages declared to sources, NuGet will provide meaningful performance improvements to restore as source lookup can be avoided.

## Explanation

### Functional explanation

When using a combination of public, private, and local sources defined in `NuGet.config` file(s), a user can add the `source` attribute to `<packageSources>` children nodes which will create a common package source name that can be used when defining a `<PackageReference>`. By adding the `Source` attribute inside the `<PackageReference>` node, the package will be pinned to the matching package source name(s) in the order they have been listed.

**Definition:**

Add a new `source` attribute to your package source elements within your `NuGet.config` using the following syntax:

```xml
<packageSources>
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" protocolVersion="3" source="NuGet" />
    <add key="Contoso" value="https://contoso.com/packages/" source="Contoso" />
    <add key="Test Source" value="c:\packages" source="Local" />
</packageSources>
```

Define the package source(s) for the package by adding a `Source` attribute with the common name(s) inside the `<PackageReference>`:

**Example:**

```xml
<!-- Package is not pinned and has same behavior as today. -->
<PackageReference Include="Microsoft.EntityFrameworkCore" Version="5.0.3" />

<!-- Package is pinned to the nuget.org packageSource. -->
<PackageReference Include="Microsoft.EntityFrameworkCore" Version="5.0.3" Source="NuGet" />

<!-- Package is pinned to the contoso packageSource. -->
<PackageReference Include="Microsoft.EntityFrameworkCore" Version="5.0.3" Source="Contoso" />

<!-- Package is pinned to the local packageSource. -->
<PackageReference Include="Microsoft.EntityFrameworkCore" Version="5.0.3" Source="Local" />
```

### Technical explanation

<!-- Explain the proposal in sufficient detail with implementation details, interaction models, and clarification of corner cases. -->

#### Technical background - how NuGet package installation works today

This approach builds on top of the current package installation behavior as described in the [package installation process](https://docs.microsoft.com/nuget/concepts/package-installation-process).

A few other important concepts:

- global packages folder - The installation directory for all packages in PackageReference. Think of this as the `Program Files` equivalent. All packages are consumed from this location. A single package installation can be shared safely among many different builds. A project *does not own* a specific package installation.
The global packages folder is an *append only* resource. This means NuGet only ever installs a new package. There is **no** refreshing, or overriding, or `re-installing` packages of any kind.
- When requesting an exact version, local, file sources are currently preferred - When installing a package, NuGet checks local sources independently before checking http sources.
- Package installation is operation based - If 3 projects are being restored during that operation, and all those projects have a dependency to `Newtonsoft.Json`, version `9.0.1`, in regular scenarios, only 1 project will *actually* download and install the package. The other projects will use the already installed version.
- `.nupkg.metadata` - Each package installation directory contains a `.nupkg.metadata` file which signify that a package installation is complete. This is expected to be the last file written in the package directory. This file is *not* use during build. NuGet.Client writes the package source information inside this file.

#### Package installation rules

- When the requested package is already installed in the global packages folder, no source look-up will happen. The source metadata is not relevant.
- When a package has the Source metadata, only the specific source *will* be looked up.
- When a package does not have Source metadata, all sources equivalent to the experience today will be looked up.
- Transitive dependencies will be installed based on the source from which the parent was installed.
- The transitive dependency allow list of sources is calculated based on the package id only.

**Scenario 1:**

The following scenarios contain a single project.

The sources are:

- nuget.org : `https://api.nuget.org/v3/index.json`
- contoso : `https://contoso.org/v3/index.json`

**Scenario 1A:**

A 1.0.0 -> B 1.0.0

```xml
<PackageReference Include="A" Version="1.0.0" Source="nuget.org"/>
```

**Result:**
Both Package `A` and `B` get installed from nuget.org

**Scenario 1B:**

A 1.0.0 -> B 1.0.0

```xml
<PackageReference Include="A" Version="1.0.0" Source="nuget.org"/>
<PackageReference Include="B" Version="1.0.0" Source="contoso"/>
```

**Result:**

- Package `A` gets installed from `nuget.org`.
- Package `B` gets installed from `contoso`.

**Scenario 1C:**

A 1.0.0 -> B 1.0.0

C 1.0.0 -> B 1.0.0

```xml
<PackageReference Include="A" Version="1.0.0" Source="nuget.org"/>
<PackageReference Include="C" Version="1.0.0" Source="contoso"/>
```

**Result:**

- Package `A` gets installed from `nuget.org`.
- Package `C` gets installed from `contoso`.
- Package `B` is *not deterministic* and can get installed from either `nuget.org` or `contoso`.

**Scenario 1D:**

A 1.0.0 -> B 1.0.0

C 1.0.0 -> B 2.0.0

```xml
<PackageReference Include="A" Version="1.0.0" Source="nuget.org"/>
<PackageReference Include="C" Version="1.0.0" Source="contoso"/>
```

**Result:**

- Package `A` gets installed from `nuget.org`.
- Package `C` gets installed from `contoso`.
- Package `B` is *not deterministic* and can get installed from either `nuget.org` or `contoso`. The requested version does not affect the transitive source determination.

**Scenario 1E:**

A 1.0.0 -> B 1.0.0

```xml
<PackageReference Include="A" Version="1.0.0" Source="nuget.org" Condition="'$(TargetFramework)' == 'frameworkOne'"/>
<PackageReference Include="B" Version="1.0.0" Source="contoso" Condition="'$(TargetFramework)' == 'frameworkTwo'"/>
```

**Result:**

- Package `A` gets installed from `nuget.org`.
- Package `B` gets installed from `contoso`.

**Scenario 1F:**

A 1.0.0 -> B 1.0.0

C 1.0.0 -> B 1.0.0

```xml
<PackageReference Include="A" Version="1.0.0" Source="nuget.org" />
<PackageReference Include="C" Version="1.0.0" />
```

**Result:**

- Package `A` gets installed from `nuget.org`.
- Package `B` gets installed from either `nuget.org` and `contoso`.
- Package `C` is *not deterministic* and be installed from either `nuget.org` and `contoso`.

**Scenario 2:**

The following scenarios contain multiple projects.

The sources are:

- nuget.org : `https://api.nuget.org/v3/index.json`
- contoso : `https://contoso.org/v3/index.json`

**Scenario 2A:**

A 1.0.0 -> B 1.0.0

```xml
<!-- Project 1 -->
<PackageReference Include="A" Version="1.0.0" Source="nuget.org"/>
```

```xml
<!-- Project 2 -->
<PackageReference Include="B" Version="1.0.0" Source="nuget.org"/>
```

**Result:**

- The scenario is *not* deterministic. Both A & B will be installed from nuget.org.

**Scenario 2B:**

A 1.0.0 -> B 1.0.0

```xml
<!-- Project 1 -->
<PackageReference Include="A" Version="1.0.0" Source="nuget.org"/>
```

```xml
<!-- Project 2 -->
<PackageReference Include="A" Version="1.0.0" Source="contoso"/>
```

**Result:**

- This scenario is *not* deterministic.
- If a customer restores *only* project 1, A will be installed from nuget.org.
- If a customer restores *only* project 2, A will be installed from contoso.
- When the customer restores both, NuGet *could* trivially discover the conflicting instructions and potentially warn.

**Scenario 2C:**

A 1.0.0 -> B 1.0.0

```xml
<!-- Project 1 -->
<PackageReference Include="A" Version="1.0.0" Source="nuget.org"/>
```

```xml
<!-- Project 2 -->
<PackageReference Include="B" Version="1.0.0" Source="contoso"/>
```

**Result:**

- This scenario is *not* deterministic.
- If a customer restores *only* project 1, A will be installed from nuget.org.
- If a customer restores *only* project 2, A will be installed from contoso.
- NuGet could determine the inconsistency and warn.

**Scenario 2D:**

A 1.0.0 -> B 1.0.0

C 1.0.0 -> B 1.0.0

```xml
<!-- Project 1 -->
<PackageReference Include="A" Version="1.0.0" Source="nuget.org"/>
```

```xml
<!-- Project 2 -->
<PackageReference Include="C" Version="1.0.0" Source="contoso"/>
```

**Result:**

- A will be installed from nuget.org.
- C will be installed from contoso.
- B is *not* deterministic.
- If a customer restores *only* project 1, B will be installed from nuget.org.
- If a customer restores *only* project 2, B will be installed from contoso.
- When a solution gets restored, NuGet could try to determine the inconsistency and warn.

## Drawbacks

<!-- Why should we not do this? -->
There are many alternatives to providing a more secure & intuitive ecosystem to .NET developers listed in "Prior Art". NuGet is very good at security as seen in [State of the Octoverse 2020](https://octoverse.github.com/static/github-octoverse-2020-security-report.pdf). NuGet, like many other package managers were designed to make the UX of managing packages very straight-forward in the sense of defining a package will go find the package in any source provided. By implementing this feature, we are fundamentally allowing users to change how NuGet should resolve packages. This gives them greater control of their packages at the cost of reducing the previous user experience.

## Rationale and alternatives

We believe this design is the most intuitive solution from 5 different proposals we briefly got feedback on. Based on testing these proposals, the unanimous feedback was that this proposal was the favorite of every individual we interviewed. Because of this, we believe there is a strong signal in terms of the design, onboarding, and user experience of this feature.

Another proposal which allows a user to pin their sources by NuGet namespaces had mostly positive feedback, but was not the approach most individuals gravitated towards initially.

By not doing this feature, we are leaving the community susceptible to [Dependency Confusion](https://medium.com/@alex.birsan/dependency-confusion-4a5d60fec610).

## Prior Art

There's a number of features that exist in various ecosystems & layers that solve similar problems by providing the user more control of their software supply chain such as:

- [npm (Scoped packages)](https://docs.npmjs.com/cli/v7/using-npm/scope)
- [Gradle (metadata sources)](https://docs.gradle.org/current/userguide/declaring_repositories.html#sec:supported_metadata_sources)
- [JFrog (exclude patterns)](https://jfrog.com/blog/yet-another-case-for-using-exclude-patterns-in-remote-repositories/)
- [Azure (upstream sources)](https://docs.microsoft.com/en-us/azure/devops/artifacts/concepts/upstream-sources?view=azure-devops)
- [NuGet (ID prefix)](https://docs.microsoft.com/en-us/nuget/nuget-org/id-prefix-reservation)
- [npm (package lock)](https://docs.npmjs.com/cli/v6/configuring-npm/package-locks)
- [NuGet (lock file)](https://docs.microsoft.com/en-us/nuget/consume-packages/package-references-in-project-files#locking-dependencies)
- [Pip (hash-checking mode)](https://pip.pypa.io/en/stable/reference/pip_install/#hash-checking-mode)
- [Gradle (verify-metadata)](https://docs.gradle.org/current/userguide/dependency_verification.html)

## Unresolved Questions

- There have been many names to define the attribute such as `Scope`, `Source`, and `Feed`. For the sake of this proposal, the most consistent word to be used is `Source` given it is used throughout NuGet & Azure tooling. With enough feedback on the name, this can change.
- Currently transitive dependencies will be handled in the order of the sources defined. This approach can cause some ambiguity, so we might have to reconsider how transitive dependencies would be handled.
- Can you pin one package version on one source, but another package version to another source? In the same project? In the same operation?
- Should the `source` attribute in the `NuGet.config` just use the `key` value instead?
  - Should it support both?
- Should this support the `RestoreSources` MSBuild parameter?

## Future Possibilities

- This particular proposal only covers PackageReference projects. A similar idea can be extended to packages.config projects.
- NuGet can combine prior art such as [NuGet (lock file)](https://docs.microsoft.com/en-us/nuget/consume-packages/package-references-in-project-files#locking-dependencies) & [NuGet (ID prefix)](https://docs.microsoft.com/en-us/nuget/nuget-org/id-prefix-reservation) to ensure repeatable restores with content hashes & namespaces.
- NuGet can specify version ranges that are semver compliant such as [`^(caret) - Approximately equivalent to version`](https://github.com/npm/node-semver#caret-ranges-123-025-004) & [`~(tilde) - Compatible with version`](https://github.com/npm/node-semver#tilde-ranges-123-12-1) for finer grained control of package sources & versions.
- NuGet within Visual Studio can provide a mechanism to pin a package to a package source at [Browse, Install, and Update time](https://docs.microsoft.com/en-us/nuget/consume-packages/install-use-packages-visual-studio).
- NuGet can make this a better UX by erroring on duplicate `<PackageReference>` elements in the case of two definitions with different sources specified.