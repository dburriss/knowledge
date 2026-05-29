# Customizing Copilot Environment for .NET Development

This guide covers how to customize GitHub Copilot's development environment specifically for .NET projects, ensuring the required .NET SDK versions and tools are available.

## Overview

GitHub Copilot runs in an ephemeral GitHub Actions environment. For .NET projects, you need to ensure the correct .NET SDK versions are installed before Copilot starts working.

## Basic .NET Setup

Create a `.github/workflows/copilot-setup-steps.yml` file in your repository:

```yaml
name: "Copilot Setup Steps"

on:
  workflow_dispatch:
  push:
    paths:
      - .github/workflows/copilot-setup-steps.yml
  pull_request:
    paths:
      - .github/workflows/copilot-setup-steps.yml

jobs:
  copilot-setup-steps:
    runs-on: ubuntu-latest
    
    permissions:
      contents: read
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v5
      
      - name: Set up .NET SDK
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: |
            6.0.x
            8.0.x
            9.0.x
            10.0.x
```

## Multiple .NET Versions

For projects requiring multiple .NET versions, specify them explicitly:

```yaml
      - name: Set up .NET SDK
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: |
            6.0.428
            8.0.404
            9.0.101
            10.0.100-preview.1.25080.5
```

## .NET Project Dependencies

After setting up the .NET SDK, restore project dependencies:

```yaml
      - name: Restore .NET dependencies
        run: dotnet restore
      
      - name: Build .NET project
        run: dotnet build --no-restore
```

## Complete .NET Example

Here's a comprehensive example for a .NET solution:

```yaml
name: "Copilot Setup Steps"

on:
  workflow_dispatch:
  push:
    paths:
      - .github/workflows/copilot-setup-steps.yml
  pull_request:
    paths:
      - .github/workflows/copilot-setup-steps.yml

jobs:
  copilot-setup-steps:
    runs-on: ubuntu-latest
    
    permissions:
      contents: read
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v5
        with:
          fetch-depth: 0  # Allow full history for Copilot
      
      - name: Set up .NET SDK
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: |
            8.0.x
            9.0.x
            10.0.x
          cache: true
          cache-dependency-path: "**/packages.lock.json"
      
      - name: Restore dependencies
        run: dotnet restore --locked-mode
      
      - name: Build solution
        run: dotnet build --no-restore --configuration Release
      
      - name: Set up environment variables
        run: |
          echo "DOTNET_ROOT=$HOME/.dotnet" >> $GITHUB_ENV
          echo "$HOME/.dotnet/tools" >> $GITHUB_PATH
```

## Environment Variables for .NET

Set .NET-specific environment variables in the `copilot` environment:

1. Go to your repository's Settings → Environments → copilot
2. Add environment variables:

- `DOTNET_ROOT`: Path to .NET SDK installation
- `DOTNET_CLI_TELEMETRY_OPTOUT`: Set to `1` to disable telemetry
- `ASPNETCORE_ENVIRONMENT`: Set to `Development` or `Production` as needed

## Using Larger Runners for .NET Projects

.NET projects can benefit from larger runners, especially for large solutions:

```yaml
jobs:
  copilot-setup-steps:
    runs-on: ubuntu-4-core  # 4-core runner with more RAM
    # or ubuntu-8-core for even more resources
```

## .NET Tools Installation

Install additional .NET tools that Copilot might need:

```yaml
      - name: Install .NET tools
        run: |
          dotnet tool install --global dotnet-ef
          dotnet tool install --global dotnet-format
          dotnet tool install --global Microsoft.Web.LibraryManager.CLI
```

## Testing Framework Setup

Ensure testing frameworks are available:

```yaml
      - name: Install test runners
        run: |
          dotnet tool install --global dotnet-coverage
          dotnet tool install --global coverlet.console
      
      - name: Run tests (optional validation)
        run: dotnet test --no-build --verbosity normal --collect:"XPlat Code Coverage"
```

## Git LFS for .NET Projects

If your .NET project uses Git LFS (common for large binary assets):

```yaml
      - name: Checkout code
        uses: actions/checkout@v5
        with:
          lfs: true
```

## Security Considerations

- Use GitHub Actions secrets for API keys and connection strings
- Limit permissions to the minimum required (`contents: read` is usually sufficient)
- Consider using self-hosted runners with ARC for additional security controls

## Troubleshooting

### Common Issues

1. **SDK not found**: Ensure the exact version number is correct
2. **Build failures**: Check that all dependencies can be restored
3. **Performance issues**: Consider upgrading to larger runners

### Validation

Test your setup by:
1. Creating a PR with changes to `copilot-setup-steps.yml`
2. Checking the Actions tab to ensure the workflow runs successfully
3. Manually running the workflow from the Actions tab

## NuGet Package Lock Files

For built-in caching to work, you must enable NuGet package lock files:

1. Add this property to each project file:
```xml
<PropertyGroup>
    <RestorePackagesWithLockFile>true</RestorePackagesWithLockFile>
</PropertyGroup>
```

2. Commit the generated `packages.lock.json` files to source control

3. Use `--locked-mode` with `dotnet restore` to ensure exact versions

## Best Practices

1. Pin specific versions for reproducibility
2. Use built-in NuGet caching with lock files for faster builds
3. Keep the setup steps minimal but complete
4. Test the setup workflow before merging to default branch
5. Monitor Copilot session logs for setup issues

## References

- [GitHub Actions setup-dotnet](https://github.com/actions/setup-dotnet)
- [.NET SDK documentation](https://learn.microsoft.com/en-us/dotnet/core/sdk/)
- [GitHub Copilot environment customization](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/customize-the-agent-environment)
- [NuGet package locking](https://learn.microsoft.com/en-us/nuget/consume-packages/package-references-in-project-files#locking-dependencies)
