---
title: "Using Direnv with Nix for Project-Specific Development Environments"
date: "2025-11-09"
description: "How I use direnv with Nix to create reproducible, project-specific development environments that automatically activate when entering directories"
tags: ["nix", "direnv", "development-environment", "ruby-on-rails", "automation"]
---

# Using Direnv with Nix for Project-Specific Development Environments

As a senior solutions architect working across multiple projects, I need a way to ensure each project has its own isolated development environment with the exact dependencies it requires. This is where the combination of direnv and Nix becomes incredibly powerful.

## What is Direnv?

Direnv is an environment switcher that automatically loads and unloads environment variables when you enter or leave directories. It's like having a `.bashrc` or `.zshrc` that's specific to each project directory.

## The Power of Direnv + Nix

When combined with Nix, direnv provides:
- **Automatic environment activation**: No manual setup when switching projects
- **Dependency isolation**: Each project has its own exact dependency versions
- **Reproducibility**: Same environment across different machines
- **Clean separation**: No global pollution of dependencies

## My Rails Project Setup

Let me walk through how I set up my Ruby on Rails project (sports-edge) using this combination.

### The `.envrc` File

The `.envrc` file is the trigger that tells direnv what to do:

```bash
use nix
```

That's it! This simple directive tells direnv to use the Nix shell defined in `shell.nix`.

### The `shell.nix` File

Here's the actual Nix shell configuration for my Rails project:

```nix
let
  pkgs = import <nixpkgs> { system = "aarch64-darwin"; };
in
pkgs.mkShell {
  buildInputs = with pkgs; [
    ruby_3_4
    gcc
    gnumake
    pkg-config
    zlib
    openssl
    libyaml
    gmp
    readline
    rustc
    nodejs
    nixd
    postgresql
    nixfmt
  ];

  nativeBuildInputs = [ pkgs.pkg-config ];

  shellHook = ''
    set -e

    # Ensure Apple Silicon native builds
    export ARCHFLAGS="-arch arm64"

    # Suppress RubyGems/Bundler constant warnings
    export RUBYOPT="-W0"

    # Isolate gems
    export GEM_HOME="$PWD/.gem"
    export GEM_PATH="$GEM_HOME"
    export BUNDLE_PATH="$GEM_HOME"
    export BUNDLE_BIN="$GEM_HOME/bin"
    export BUNDLE_DISABLE_SHARED_GEMS=1
    export PATH="$BUNDLE_BIN:$PATH"

    # Disable documentation generation
    mkdir -p "$PWD/.gem"
    echo "gem: --no-document" > "$PWD/.gem/gemrc"
    export GEMRC="$PWD/.gem/gemrc"

    echo "Ruby version: $(ruby --version)"
    if [ -f Gemfile ]; then
      echo "Rails version: $(bundle exec rails --version 2>/dev/null || echo 'not in Gemfile')"
    else
      echo "Rails version: $(command -v rails >/dev/null 2>&1 && rails --version || echo 'Gemfile not found')"
    fi
  '';
}
```

## Breaking Down the Configuration

### Dependencies

The `buildInputs` section includes all the tools needed for Rails development:

- **ruby_3_4**: Specific Ruby version
- **gcc, gnumake**: Build tools for native extensions
- **pkg-config**: Package configuration helper
- **zlib, openssl, libyaml, gmp, readline**: Ruby dependencies
- **rustc**: For gems with Rust extensions
- **nodejs**: For JavaScript assets
- **nixd**: Nix language server
- **postgresql**: Database client
- **nixfmt**: Nix code formatter

### Environment Setup

The `shellHook` runs when the shell activates:

1. **Apple Silicon optimization**: `ARCHFLAGS="-arch arm64"`
2. **Ruby warnings suppression**: `RUBYOPT="-W0"`
3. **Gem isolation**: All gems go to `.gem/` directory
4. **Documentation disabled**: Faster gem installation
5. **Version display**: Shows Ruby and Rails versions on activation

## Version Management Integration

I also use `.tool-versions` for compatibility with other tools:

```bash
ruby 3.4.6
```

This works alongside the Nix setup, providing redundancy and compatibility with tools like `asdf`.

## How It Works in Practice

### Initial Setup

1. Create `.envrc` with `use nix`
2. Create `shell.nix` with your dependencies
3. Run `direnv allow` to approve the configuration

### Daily Workflow

```bash
# Enter project directory
cd sports-edge

# Direnv automatically activates:
# Ruby version: ruby 3.4.6p321 (2024-11-05 revision 31f3c7b7a9) [arm64-darwin23]
# Rails version: Rails 8.0.0

# Work normally
rails server
bundle exec rspec
```

### Switching Projects

```bash
cd ~/projects/another-project
# Direnv unloads sports-edge environment
# Direnv loads another-project environment

cd ~/projects/sports-edge  
# Direnv unloads another-project environment
# Direnv loads sports-edge environment again
```

## Benefits I've Experienced

### 1. Zero-Friction Development
- No manual environment setup
- No "works on my machine" issues
- Instant context switching between projects

### 2. Dependency Isolation
- Project A can use Ruby 3.3, Project B uses Ruby 3.4
- No gem conflicts between projects
- Clean system-wide package management

### 3. Team Consistency
- Same `shell.nix` works for all team members
- New developers can start with `git clone` and `direnv allow`
- CI/CD can use the same environment definition

### 4. Documentation
- The `shell.nix` serves as living documentation
- Clear list of all project dependencies
- Environment setup is self-documenting

## Advanced Features

### Conditional Dependencies

You can make dependencies conditional:

```nix
buildInputs = with pkgs; [
  ruby_3_4
] ++ lib.optionals stdenv.isLinux [ linux-pam ]
  ++ lib.optionals stdenv.isDarwin [ darwin.apple_sdk.frameworks.Security ];
```

### Development vs Production

Create different shells for different environments:

```bash
# .envrc.development
use nix -p ruby_3_4 nodejs postgresql

# .envrc.production  
use nix -p ruby_3_4 nodejs_20
```

### Integration with Other Tools

The setup works seamlessly with:
- **Docker**: Same dependencies in containers
- **CI/CD**: GitHub Actions can use same Nix expressions
- **IDEs**: VSCode, Neovim pick up the environment automatically

## Troubleshooting Common Issues

### Permission Denied
```bash
direnv: error .envrc is blocked
```
Solution: `direnv allow`

### Nix Store Issues
```bash
error: build of '/nix/store/...' failed
```
Solution: `nix-store --verify --check-contents` or `nix-collect-garbage`

### Slow Activation
Large dependency sets can slow down activation. Consider:
- Using specific versions instead of latest
- Caching frequently used dependencies
- Splitting large environments into smaller components

## Best Practices

1. **Commit `.envrc` and `shell.nix`** to version control
2. **Use specific versions** instead of latest for reproducibility
3. **Keep environments minimal** - only include what's actually needed
4. **Document special setup** in comments within `shell.nix`
5. **Test on clean machines** to ensure reproducibility

## Conclusion

The combination of direnv and Nix has transformed how I manage development environments. It provides the isolation benefits of containers with the convenience of automatic activation. For anyone working across multiple projects or teams, this approach eliminates entire classes of environment-related problems and makes development more predictable and enjoyable.

The initial investment in setting up `shell.nix` files pays dividends in reduced setup time, fewer environment bugs, and easier onboarding for new team members. It's a pattern I now use for all my projects, from simple scripts to complex Rails applications.

---

*Have questions about using direnv with Nix, or want to share your own environment management patterns? Feel free to reach out or open an issue on the [nix-config repository](https://github.com/jeanres/nix-config).*

---

**Contact**: jeanre.swanepoel@gmail.com | +27 68 618 3487