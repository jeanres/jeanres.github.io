---
title: "Using .NET 8 Applications with .NET 9 SDK via Nix shell.nix Configuration"
date: "2025-11-10"
description: "How to run .NET 8 applications while leveraging the latest .NET 9 SDK features through Nix shell.nix configuration, including Roslyn language server for enhanced development experience"
tags: [".NET", "development-environment", "SDK", "Roslyn", "Nix", "shell.nix"]
---

# Using .NET 8 Applications with .NET 9 SDK via Nix shell.nix Configuration

As a senior solutions architect working with modern .NET applications, I often encounter scenarios where I need to maintain existing .NET 8 applications while taking advantage of the latest tooling improvements in .NET 9. This is particularly relevant when working with enhanced language servers like Roslyn that benefit from the newer SDK, even when the application itself targets an older framework.

In this post, I'll share how I've successfully configured my development environment to run .NET 8 applications while leveraging the .NET 9 SDK for improved development tooling.

## The Challenge

Modern development environments benefit significantly from the latest language servers and tooling. Roslyn, Microsoft's .NET compiler platform, provides enhanced IntelliSense, refactoring capabilities, and code analysis features that are continuously improved in newer versions. However, many production applications still target .NET 8 due to stability requirements, compatibility concerns, or organizational upgrade policies.

This creates a common scenario where you want:
- Your application to target and run on .NET 8
- Your development tools to leverage .NET 9 features
- A consistent, reproducible environment across your team

## Solution Overview

The key is to install both .NET 8 and .NET 9 SDKs in your development environment and configure your projects to target .NET 8 while allowing the newer SDK's tools to enhance your development experience.

## Nix-Based Environment Configuration

For reproducible environments, I use Nix to manage my development setup. Here's how I configure my `shell.nix` to include both SDKs:

```nix
{ pkgs ? import (fetchTarball "https://channels.nixos.org/nixpkgs-unstable/nixexprs.tar.xz") { } }:
let
  # One dotnet host that contains multiple SDKs
  dotnetCombined = pkgs.dotnetCorePackages.combinePackages [
    pkgs.dotnetCorePackages.sdk_8_0
    pkgs.dotnetCorePackages.sdk_9_0
  ];
in
pkgs.mkShell {
  packages = [ 
      dotnetCombined 
      pkgs.nixd
      pkgs.roslyn-ls
    ];

  # Basics that keep Roslyn/LSPs happy and silence telemetry
  shellHook = ''
    export DOTNET_ROOT=${dotnetCombined}
    export PATH="${dotnetCombined}/bin:$PATH"
    export DOTNET_CLI_TELEMETRY_OPTOUT=1

    # Show what's available so you can confirm both SDKs are there
    echo "Available .NET SDKs:"
    dotnet --list-sdks
  '';
}
```

This configuration provides:
1. Both .NET 8 and .NET 9 SDKs in a single environment
2. The latest Roslyn language server for enhanced IDE experience
3. Proper environment variables for tooling compatibility

## Project Configuration

To ensure your application targets .NET 8 while benefiting from .NET 9 tooling, use a `global.json` file in your project root:

```json
{
  "sdk": {
    "version": "8.0.408",
    "rollForward": "latestFeature"
  }
}
```

This configuration ensures that:
- Your application builds with the .NET 8 SDK
- You can still use newer SDK features for tooling
- The build is consistent across different environments

In your project files, explicitly target .NET 8:

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <!-- Other properties -->
  </PropertyGroup>
</Project>
```

## Benefits of This Approach

### 1. Enhanced Development Experience
By using the .NET 9 SDK's Roslyn language server, you get:
- Improved IntelliSense with better performance
- Enhanced code analysis and refactoring suggestions
- Better debugging capabilities
- More accurate error detection

### 2. Production Compatibility
Your application continues to target .NET 8, ensuring:
- No unexpected runtime behavior changes
- Compatibility with production environments
- Stability for existing deployments

### 3. Future-Ready Development
This setup makes upgrading easier when you're ready:
- You're already familiar with the newer tooling
- Less friction when eventually upgrading the target framework
- Team members benefit from modern development tools

## Practical Workflow

With this configuration, your daily workflow looks like:

```bash
# Enter the project directory
cd my-dotnet-project

# The environment automatically activates with both SDKs
# Available .NET SDKs:
# 8.0.408 [/nix/store/.../dotnet-sdk-8.0.408]
# 9.0.100 [/nix/store/.../dotnet-sdk-9.0.100]

# Build and run with .NET 8
dotnet build
dotnet run

# Development tools use .NET 9 features
# Your IDE gets enhanced IntelliSense and refactoring
```

## Team Collaboration

This approach works well for teams because:
- Everyone has the same development environment
- The configuration is version-controlled
- New team members can get started quickly
- Builds are consistent across all machines

## Troubleshooting Common Issues

### SDK Version Conflicts
If you encounter version conflicts, explicitly specify the SDK in your build commands:
```bash
dotnet build --sdk-version 8.0.408
```

### IDE Configuration
Ensure your IDE is configured to use the correct language server:
- VS Code: Configure the OmniSharp or C# extension to use the combined SDK path
- JetBrains Rider: Set the correct MSBuild version in preferences

## Conclusion

Running .NET 8 applications with .NET 9 SDK tooling provides the best of both worlds: production stability with modern development capabilities. This approach allows you to maintain existing applications while benefiting from the latest tooling improvements.

The Nix-based configuration ensures reproducibility across environments, making it ideal for team collaboration and CI/CD pipelines. As you plan your migration to .NET 9, this setup provides a smooth transition path with minimal disruption to your current workflow.

---