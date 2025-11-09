---
title: "My Nix Configuration: A Reproducible macOS Development Environment"
date: "2025-11-09"
description: "How I've structured my Nix configuration for a reproducible macOS development environment using nix-darwin and home-manager"
tags: ["nix", "macos", "development-environment", "configuration-management"]
---

# My Nix Configuration: A Reproducible macOS Development Environment

As a senior solutions architect, I work across multiple projects and environments. Having a reproducible development setup is crucial for maintaining consistency and quickly spinning up new environments. Here's how I've structured my Nix configuration to achieve this on macOS.

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
│   ├── zsh.nix           # Shell configuration
│   ├── git.nix           # Git configuration
│   ├── tmux.nix          # Terminal multiplexer
│   ├── kitty.nix         # Terminal configuration
│   └── aerospace.nix     # Window manager
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

## Core Development Tools

My core development environment includes essential CLI tools:

```nix
# home/core.nix
{
  home.shellAliases = {
    cd = "z";
  };
  programs.ripgrep.enable = true;
  programs.fd.enable = true;
  programs.jq.enable = true;
  programs.eza.enable = true;
  programs.htop.enable = true;
  programs.fzf.enable = true;
  programs.zoxide = {
    enable = true;
    enableZshIntegration = true;
  };
  programs.tealdeer.enable = true;
}
```

This gives me modern replacements for traditional Unix tools:
- `ripgrep` for fast searching
- `fd` for finding files
- `eza` for better `ls` output
- `zoxide` for smart directory navigation
- `fzf` for fuzzy finding

## Shell Configuration

I use Zsh with several enhancements:

```nix
# home/zsh.nix
{...}: {
  programs.zsh = {
    enable = true;
    enableCompletion = true;
    defaultKeymap = "viins";
    shellAliases = {
      dr = "sudo darwin-rebuild switch --flake .#Jeanres-MacBook-Pro";
      cat = "bat";
    };
  };

  programs.carapace = {
    enable = true;
    enableZshIntegration = true;
  };

  programs.starship = {
    enable = true;
    enableZshIntegration = true;
  };
}
```

Key features:
- Vi mode for Zsh
- Starship prompt for better visual feedback
- Carapace for enhanced completions
- Quick rebuild alias (`dr`)

## Neovim Configuration

My Neovim setup uses the latest Lua-based configuration with lazy.nvim:

```nix
# home/neovim.nix
{ pkgs, ... }:
{
  programs.neovim = {
    enable = true;
    defaultEditor = true;
    vimAlias = true;
    package = pkgs.neovim-unwrapped;
  };

  xdg.configFile = {
    "nvim/init.lua" = {
      source = ./neovim/init.lua;
    };
    "nvim/lua" = {
      source = ./neovim/lua;
      recursive = true;
    };
    "nvim/lsp" = {
      source = ./neovim/lsp;
      recursive = true;
    };
  };
}
```

The Lua configuration includes:
- **LSP support**: lua_ls, nixd, roslyn_ls, ruby_ls, ts_ls, marksman
- **Plugin management**: lazy.nvim for efficient loading
- **Modern plugins**: blink-cmp, telescope, oil, gitsigns, treesitter
- **Debugging**: nvim-dap with UI integration

## Git Configuration

My Git setup is optimized for modern workflows:

```nix
# home/git.nix
{ vars, ... }:
{
  programs.git = {
    enable = true;
    settings = {
      user = {
        email = vars.email;
        name = vars.name;
      };
      alias = {
        co = "checkout";
        ci = "commit";
        di = "diff";
        dc = "diff --cached";
        fa = "fetch --all";
        pf = "push --force-with-lease";
      };
      init = {
        defaultBranch = "main";
      };
      pull = {
        rebase = true;
      };
      rebase = {
        autoSquash = true;
        autoStash = true;
      };
    };
  };
}
```

## Terminal Multiplexer (Tmux)

My Tmux configuration is feature-rich with plugins:

```nix
# home/tmux.nix
{ pkgs, ... }:
{
  programs.tmux = {
    enable = true;
    baseIndex = 1;
    plugins = with pkgs; [
      tmuxPlugins.vim-tmux-navigator
      tmuxPlugins.resurrect
      tmuxPlugins.continuum
      tmuxPlugins.catppuccin
      tmuxPlugins.battery
      tmuxPlugins.online-status
    ];
  };
}
```

Key features:
- Vim-style navigation
- Session persistence with resurrect/continuum
- Catppuccin theme
- Battery and online status indicators
- Popup windows for lazygit and session switching

## Homebrew Integration

For GUI applications, I use Homebrew integration:

```nix
# modules/homebrew.nix
{ ... }:
{
  homebrew = {
    enable = true;
    onActivation = {
      autoUpdate = true;
      upgrade = true;
      cleanup = "zap";
    };

    taps = [
      "nikitabobko/tap"
      "vladdoster/formulae"
    ];

    brews = [
      "reattach-to-user-namespace"
      "opencode"
    ];

    casks = [
      "kitty"
      "docker-desktop"
      "aerospace"
      "font-jetbrains-mono-nerd-font"
      "slack"
      "microsoft-teams"
      "utm"
      "discord"
    ];
  };
}
```

## System Configuration

The macOS system settings are managed through `systems/darwin.nix`:

```nix
# systems/darwin.nix
{ pkgs, vars, ... }:
{
  nix = {
    package = pkgs.nixVersions.latest;
    extraOptions = ''
      extra-platforms = x86_64-darwin
      experimental-features = nix-command flakes
    '';
    gc = {
      automatic = true;
      interval = { Weekday = 0; Hour = 0; Minute = 0; };
      options = "--delete-older-than 30d";
    };
  };

  security.pam.services.sudo_local.touchIdAuth = true;
  services.openssh.enable = true;

  system = {
    defaults = {
      screencapture = {
        location = "~/Documents/Screenshots";
      };
      dock = {
        mru-spaces = false;
        autohide = true;
      };
    };
    keyboard = {
      enableKeyMapping = true;
      remapCapsLockToEscape = true;
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

### 3. Modern Tooling
- Latest versions of all tools
- Modern replacements for traditional Unix tools
- Efficient workflows with aliases and integrations

### 4. Cross-Platform Potential
- Similar structure works for NixOS
- Shared modules between systems
- Consistent development experience

## Daily Workflow

### Making Changes
1. Edit configuration files in `~/.nix-config/`
2. Run `dr` (alias for `darwin-rebuild switch --flake .`)
3. Changes are applied atomically

### Adding New Tools
1. Add to appropriate `.nix` file in `home/`
2. For GUI apps, add to `modules/homebrew.nix`
3. Rebuild configuration

### Version Control
- All configuration in Git
- Track changes over time
- Share configurations across teams

## Conclusion

This Nix configuration provides a robust, reproducible development environment that scales across different projects and machines. The modular structure makes it easy to maintain and extend, while the declarative approach ensures consistency and reduces manual configuration errors.

The combination of modern CLI tools, a powerful Neovim setup, and comprehensive system management creates an efficient development environment that adapts to my needs. 

---

*Have questions about my Nix setup or want to share your own configuration patterns? Feel free to reach out or open an issue on the [nix-config repository](https://github.com/jeanres/nix-config).*

---

**Contact**: jeanre.swanepoel@gmail.com | +27 68 618 3487
