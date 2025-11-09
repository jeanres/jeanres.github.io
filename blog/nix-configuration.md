---
title: "My Nix Configuration: A Reproducible macOS Development Environment"
date: "2025-11-09"
description: "How I've structured my Nix configuration for a reproducible macOS development environment using nix-darwin and home-manager"
tags: ["nix", "macos", "development-environment", "configuration-management"]
---

# My Nix Configuration: A Reproducible macOS Development Environment

As a software architecture consultant, I work across multiple projects and environments. Having a reproducible development setup is crucial for maintaining consistency and quickly spinning up new environments. Here's how I've structured my Nix configuration to achieve this on macOS.

## Overview

My configuration uses a combination of:
- **Nix**: The package manager and build system
- **nix-darwin**: For system-level macOS configuration
- **home-manager**: For user-level dotfile and package management
- **Homebrew integration**: For GUI applications not available in Nix

## Directory Structure

```
~/.nix-config/
├── flake.nix              # Entry point and dependencies
├── home/                  # Home-manager configurations
│   ├── default.nix       # Main home configuration
│   ├── core.nix          # Core packages and settings
│   ├── neovim.nix        # Neovim setup
│   ├── kitty.nix         # Terminal configuration
│   └── ...               # Other application configs
├── modules/              # System modules
│   └── homebrew.nix      # Homebrew integration
├── systems/              # System-specific configs
│   └── darwin.nix        # macOS system settings
└── shared/               # Shared variables
    └── variables.nix     # User and system variables
```

## The Flake Architecture

The `flake.nix` serves as the entry point, defining inputs and outputs:

```nix
{
  description = "Jeanre's system configuration";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
    darwin = {
      url = "github:lnl7/nix-darwin/master";
      inputs.nixpkgs.follows = "nixpkgs";
    };
    home-manager = {
      url = "github:nix-community/home-manager";
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };

  outputs = { self, nixpkgs, darwin, home-manager}:
    let
      vars = import ./shared/variables.nix;
    in
    {
      darwinConfigurations = {
        "Jeanres-MacBook-Pro" = darwin.lib.darwinSystem {
          specialArgs = { inherit vars; };
          system = "aarch64-darwin";
          modules = [
            ./systems/darwin.nix
            ./modules/homebrew.nix
            home-manager.darwinModules.home-manager
            {
              home-manager.useGlobalPkgs = true;
              home-manager.useUserPackages = true;
              home-manager.extraSpecialArgs = { inherit vars; };
              home-manager.users."${vars.username}" = import ./home;
            }
          ];
        };
      };
    };
}
```

Key features of this setup:
- **Flake inputs**: Using nixpkgs-unstable for latest packages
- **Input following**: Ensures consistent versions across dependencies
- **Special arguments**: Passing variables to all modules
- **Modular structure**: Separate concerns into different files

## Home Manager Configuration

The home configuration is modular, with each application having its own file:

```nix
# home/default.nix
{ inputs, pkgs, vars, ... }:

{
  imports = [
    ./core.nix
    ./bat.nix
    ./zsh.nix
    ./git.nix
    ./tmux.nix
    ./kitty.nix
    ./neovim.nix
    ./aerospace.nix
  ];

  home = {
    homeDirectory = vars.homeDirectory;
    stateVersion = "24.05";
  };

  programs.home-manager.enable = true;
  programs.direnv = {
    enable = true;
    nix-direnv.enable = true;
    enableZshIntegration = true;
  };
  programs.htop.enable = true;
  programs.yazi.enable = true;
  programs.lazysql.enable = true;
  programs.lazydocker.enable = true;
}
```

### Neovim Configuration

My Neovim setup is particularly comprehensive, with Lua-based configuration:

```nix
# home/neovim.nix
{ pkgs, ... }:

{
  programs.neovim = {
    enable = true;
    defaultEditor = true;
    viAlias = true;
    vimAlias = true;
    
    extraPackages = with pkgs; [
      lua-language-server
      nixd
      marksman
      typescript-language-server
      ruby-lsp
      roslyn
    ];
  };
  
  home.file.".config/nvim" = {
    source = ./neovim;
    recursive = true;
  };
}
```

The Lua configuration is structured with:
- **Core setup**: Basic editor configuration
- **Plugin management**: Using lazy.nvim for efficient plugin loading
- **LSP configuration**: Language servers for multiple languages
- **Custom plugins**: Individual files for each plugin configuration

## System Integration

### Homebrew Integration

For GUI applications not available in Nix, I use Homebrew integration:

```nix
# modules/homebrew.nix
{ pkgs, config, ... }:

{
  homebrew = {
    enable = true;
    onActivation = {
      autoUpdate = true;
      cleanup = "uninstall";
    };
    
    taps = [
      "homebrew/cask"
      "homebrew/core"
    ];
    
    brews = [
      "docker"
    ];
    
    casks = [
      "tableplus"
      "obs"
      "obs-ndi"
      "vimari"
    ];
  };
}
```

### System Configuration

The macOS system settings are managed through `systems/darwin.nix`:

```nix
{ pkgs, vars, ... }:

{
  environment.systemPackages = with pkgs; [
    git
    vim
  ];
  
  system = {
    stateVersion = 4;
    defaults = {
      NSGlobalDomain = {
        AppleShowAllExtensions = true;
        KeyRepeat = 2;
        InitialKeyRepeat = 15;
      };
    };
  };
}
```

## Benefits of This Setup

### 1. Reproducibility
- Same configuration across multiple machines
- Version-controlled changes
- Easy rollback to previous states

### 2. Declarative Management
- All configuration in code
- No manual installation steps
- Clear dependency relationships

### 3. Modularity
- Separate concerns into logical modules
- Easy to add/remove components
- Reusable across different setups

### 4. Cross-Platform Potential
- Similar structure works for NixOS
- Shared modules between systems
- Consistent development experience

## Daily Workflow

### Making Changes
1. Edit configuration files in `~/.nix-config/`
2. Run `darwin-rebuild switch --flake .`
3. Changes are applied atomically

### Adding New Tools
1. Add to appropriate `.nix` file in `home/`
2. For GUI apps, add to `modules/homebrew.nix`
3. Rebuild configuration

### Version Control
- All configuration in Git
- Track changes over time
- Share configurations across teams

## Challenges and Solutions

### Challenge: GUI Applications
**Solution**: Homebrew integration for applications not in Nix

### Challenge: macOS System Settings
**Solution**: nix-darwin provides system-level configuration options

### Challenge: Learning Curve
**Solution**: Start simple, gradually add complexity

## Future Improvements

1. **Secrets Management**: Integrate with sops-nix for sensitive data
2. **Multi-Machine Support**: Extend configuration for different use cases
3. **Automated Testing**: Add configuration validation
4. **Documentation**: Enhance inline documentation

## Conclusion

This Nix configuration provides a robust, reproducible development environment that scales across different projects and machines. The modular structure makes it easy to maintain and extend, while the declarative approach ensures consistency and reduces manual configuration errors.

For anyone looking to improve their development environment management, I highly recommend exploring Nix and the ecosystem around it. The initial investment in learning pays dividends in long-term productivity and reliability.

---

*Have questions about my Nix setup or want to share your own configuration patterns? Feel free to reach out or open an issue on the [nix-config repository](https://github.com/jeanres/nix-config).*