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

```
<packageSources>
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" protocolVersion="3" source="NuGet" />
    <add key="Contoso" value="https://contoso.com/packages/" source="Contoso" />
    <add key="Test Source" value="c:\packages" source="Local" />
</packageSources>
```

Define the package source(s) for the package by adding a `Source` attribute with the common name(s) inside the `<PackageReference>`:

**Example:**

```
<!-- Package is pinned to the nuget.org packageSource. -->
<PackageReference Include="Microsoft.EntityFrameworkCore" Version="5.0.3" Source="NuGet" />

<!-- Package is pinned to the Contoso and nuget.org packageSource(s). Package will be restored by Contoso first and nuget.org second. -->
<PackageReference Include="Microsoft.EntityFrameworkCore" Version="5.0.3" Source="Contoso;NuGet" />

<!-- Package is pinned to the Test Source, Contoso, and nuget.org packageSource(s). Package will be restored by Test Source first, Contoso second, and nuget.org third. -->
<PackageReference Include="Microsoft.EntityFrameworkCore" Version="5.0.3" Source="Local;Contoso;NuGet" />
```

### Technical explanation

<!-- Explain the proposal in sufficient detail with implementation details, interaction models, and clarification of corner cases. -->

## Drawbacks

<!-- Why should we not do this? -->
There are many alternatives to providing a more secure & intuitive ecosystem to .NET developers listed in "Prior Art". NuGet is very good at security as seen in [State of the Octoverse 2020](https://octoverse.github.com/static/github-octoverse-2020-security-report.pdf). NuGet, like many other package managers were designed to make the UX of managing packages very straight-forward in the sense of defining a package will go find the package in any source provided. By implementing this feature, we are fundamentally allowing users to change how NuGet should resolve packages. This gives them greater control of their packages at the cost of reducing the previous user experience.


## Rationale and alternatives

We believe this design is the most intuitive solution from 5 different proposals we briefly got feedback on. Based on testing these proposals, the unanimous feedback was that this proposal was the favorite of every individual we interviewed. Because of this, we believe there is a strong signal in terms of the design, onboarding, and user experience of this feature.

Another proposal which allows a user to pin their sources by NuGet namespaces had mostly positive feedback, but did not ultimately solve the problem of allowing a developer to define the source where their package comes from. We found that this proposal has merit in the "Future Possibilities" to enhance this feature, but was not considered as a standalone proposal.

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
- Currently transitive dependencies will be handled in the order of the sources defined. If this isn't a feasible approach, we will have to reconsider how transitive dependencies would be handled.

## Future Possibilities

- NuGet can combine prior art such as [NuGet (lock file)](https://docs.microsoft.com/en-us/nuget/consume-packages/package-references-in-project-files#locking-dependencies) & [NuGet (ID prefix)](https://docs.microsoft.com/en-us/nuget/nuget-org/id-prefix-reservation) to ensure repeatable restores with content hashes & namespaces.
- NuGet can specify version ranges that are semver compliant such as [`^(caret) - Approximately equivalent to version`](https://github.com/npm/node-semver#caret-ranges-123-025-004) & [`~(tilde) - Compatible with version`](https://github.com/npm/node-semver#tilde-ranges-123-12-1) for finer grained control of package sources & versions.
- NuGet within Visual Studio can provide a mechanism to pin a package to a package source at [Browse, Install, and Update time](https://docs.microsoft.com/en-us/nuget/consume-packages/install-use-packages-visual-studio).
