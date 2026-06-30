---
name: nixos
description: "Use this skill when working with NixOS system configurations, flakes, devshells, home-manager, overlays, or any Nix expression. Triggers on 'nix', 'nixos', 'flake', 'devshell', 'home-manager', 'overlay', 'derivation'."
argument-hint: "Describe the change to the system, devshell, or Nix expression"
---

# NixOS & Flakes Skill

You are an expert NixOS and Nix Flakes engineer. Apply the following knowledge when helping with Nix-related tasks.

## Tool Usage

- **Always use the `nix` MCP tool** (nixos-nix) to look up packages, options, channels, and home-manager/darwin options before guessing attribute paths or option names. Your training data lags nixpkgs by months.
- Use `nixos-nix_versions` (MCP tool) to find specific package version commits.
- Prefer MCP lookups over `nix search`, scraping search.nixos.org, or running `gh api` against NixOS/nixpkgs.

## Flake Structure Best Practices

### Canonical flake.nix layout

```nix
{
  description = "Project description";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
    # Pin to a release for production systems:
    # nixpkgs.url = "github:NixOS/nixpkgs/nixos-24.11";

    flake-utils.url = "github:numtide/flake-utils";
    # Or use flake-parts for more structured flakes:
    # flake-parts.url = "github:hercules-ci/flake-parts";

    # home-manager (match nixpkgs branch):
    # home-manager = {
    #   url = "github:nix-community/home-manager";
    #   inputs.nixpkgs.follows = "nixpkgs";
    # };
  };

  outputs = { self, nixpkgs, flake-utils, ... }:
    flake-utils.lib.eachDefaultSystem (system:
      let
        pkgs = import nixpkgs {
          inherit system;
          config.allowUnfree = true; # only if needed
          # overlays = [ self.overlays.default ];
        };
      in {
        devShells.default = pkgs.mkShell {
          packages = with pkgs; [
            # build tools, language runtimes, CLI utilities
          ];
          shellHook = ''
            echo "Dev environment ready"
          '';
        };

        packages.default = pkgs.callPackage ./default.nix { };

        # Optional: formatter for `nix fmt`
        formatter = pkgs.nixfmt-rfc-style;
      }
    );
}
```

### Key principles

| Principle | Guidance |
|-----------|----------|
| Pin inputs | Always use `follows` to deduplicate nixpkgs across inputs |
| Lock file | Commit `flake.lock`; update intentionally with `nix flake update` |
| Purity | Never use `builtins.fetchTarball` or `<nixpkgs>` in a flake — use inputs |
| Formatting | Use `nixfmt-rfc-style` (the official formatter) or `alejandra` |
| Eval caching | Keep `flake.nix` pure — no IFD (import-from-derivation) unless essential |
| Systems | Use `flake-utils.lib.eachDefaultSystem` or explicit system lists |

## NixOS System Configuration

### NixOS Module structure (~/nixos-config)

```
~/nixos-config/
├── flake.nix                        # entry point — all nixosConfigurations defined here
├── flake.lock
├── secretspec.toml                  # secret declarations (sops/agenix)
├── server/                          # per-host configurations
│   ├── server-base.nix              # shared defaults (imported by each host)
│   └── <hostname>/                  # one dir per host
│       ├── default.nix              # imports server-base + hardware + server.nix
│       ├── server.nix               # host-specific services & settings
│       └── hardware-configuration.nix  # (if bare metal/VM)
├── modules/
│   ├── home-manager.nix             # HM wiring — defines module.home.users.* options
│   ├── <concern>.nix                # top-level modules (boot, grub, wsl, …)
│   ├── cli/                         # CLI tools (system-level)
│   │   ├── default.nix              # aggregator
│   │   └── <tool>.nix               # one file per tool/concern
│   ├── gui/                         # Desktop environment & GUI apps
│   │   ├── default.nix              # aggregator
│   │   └── <feature>.nix            # one file per DE/app/driver
│   └── home/                        # home-manager modules (shared across users)
│       └── <tool>.nix               # one file per HM program/config
├── home/                            # per-user home-manager profiles
│   └── <username>/default.nix       # user-specific identity/state
├── pkgs/                            # custom package derivations
│   └── <pkg>/default.nix
└── bin/                             # helper scripts
```

**Key conventions:**
- Hosts live under `server/`, NOT `hosts/`. Each has `default.nix` (imports server-base + hw + server.nix)
- Feature toggles via `module.*` options (e.g., `module.nvidia.enable`, `module.steam.enable`)
- Home-manager is wired in `modules/home-manager.nix` — users are enabled per-host via `module.home.users.<name> = true`
- Shared HM modules in `modules/home/` are imported inside `home-manager.nix`, not by the user profile
- User profiles (`home/<user>/default.nix`) contain user-specific identity/state only
- `specialArgs` passes `self`, `inputs`, `pkgs-master`, `pkgs-stable`, `system` into all modules

### Flake pattern (actual)

```nix
outputs = { self, nixpkgs, nixos-wsl, ... }@inputs:
  let
    pkgs-master = import inputs.nixpkgs-master {
      system = "x86_64-linux";
      config.allowUnfree = true;
    };
    pkgs-stable = import inputs.nixpkgs-stable {
      system = "x86_64-linux";
      config.allowUnfree = true;
    };
  in {
    nixosConfigurations = {
      my-workstation = let system = "x86_64-linux"; in
        nixpkgs.lib.nixosSystem {
          inherit system;
          specialArgs = { inherit self inputs pkgs-master pkgs-stable system; };
          modules = [
            "${self}/server/workstation"
            "${self}/modules/home-manager.nix"
            {
              module = {
                home.users.primaryuser = true;
                home.users.secondaryuser = true;
                nvidia.enable = true;
                niri.enable = true;
                steam.enable = true;
                # ... feature flags per host
              };
            }
          ];
        };
      # ... other hosts (vm-host, wsl-x86, wsl-arm)
    };
  };
```

### home-manager wiring pattern (modules/home-manager.nix)

```nix
{ inputs, self, config, lib, system, ... }:
{
  imports = [ inputs.home-manager.nixosModules.home-manager ];
  options.module.home.users = {
    primaryuser = lib.mkEnableOption "Enable user 'primaryuser'";
    secondaryuser = lib.mkEnableOption "Enable user 'secondaryuser'";
  };
  config = {
    home-manager.useGlobalPkgs = true;
    home-manager.useUserPackages = true;
    home-manager.backupFileExtension = ".bak";
    home-manager.extraSpecialArgs = { inherit self inputs system; };
    home-manager.users.primaryuser = lib.mkIf config.module.home.users.primaryuser {
      imports = [
        (import "${self}/home/primaryuser")
        ./home/packages.nix
        ./home/zsh.nix
        ./home/hyprland.nix
        # ... shared HM modules
      ];
    };
    home-manager.users.secondaryuser = lib.mkIf config.module.home.users.secondaryuser {
      imports = [ (import "${self}/home/secondaryuser") ];
    };
  };
}
```

## nix-darwin System Configuration (~/nix-darwin)

### Module structure

```
~/nix-darwin/
├── flake.nix              # entry point — darwinConfigurations + devShell
├── flake.lock
├── configuration.nix      # system-level darwin settings (nix, PAM, system path, etc.)
├── cli-packages.nix       # CLI tools (system-level environment.systemPackages)
├── gui-packages.nix       # GUI/cask packages
├── k8s.nix                # Kubernetes tooling
├── home.nix               # home-manager wiring + user HM config (inline)
├── secretspec.toml        # secret declarations
└── config/                # dotfile configs deployed by HM
    └── <tool>/            # one dir per tool (kitty, kube, maven, …)
```

### Flake pattern

```nix
outputs = inputs@{ self, nix-darwin, nixpkgs, ... }:
  let
    system = "aarch64-darwin";
    pkgs = import nixpkgs {
      inherit system;
      config.allowUnfree = true;
    };
    pkgs-stable = import inputs.nixpkgs-stable {
      inherit system;
      config.allowUnfree = true;
    };
  in {
    darwinConfigurations."<hostname>" = nix-darwin.lib.darwinSystem {
      specialArgs = { inherit self inputs pkgs-stable; };
      modules = [
        ./configuration.nix
        ./cli-packages.nix
        ./gui-packages.nix
        ./k8s.nix
        ./home.nix
      ];
    };

    devShells.${system}.default = pkgs.mkShell {
      name = "nix-darwin-devshell";
    };
  };
```

### home-manager wiring (home.nix)

```nix
{ inputs, self, pkgs, system, ... }:
let
  username = "<username>";
  homepath = "/Users/${username}";
in {
  imports = [
    inputs.home-manager.darwinModules.home-manager
    inputs.nix-homebrew.darwinModules.nix-homebrew
  ];
  users.users.${username}.home = homepath;
  home-manager.useGlobalPkgs = true;
  home-manager.useUserPackages = true;
  home-manager.backupFileExtension = ".bak";
  home-manager.extraSpecialArgs = { inherit self inputs system; };
  home-manager.users.${username} = { config, ... }: {
    imports = [ /* HM modules */ ];
    home.username = username;
    home.stateVersion = "25.11";
    programs.home-manager.enable = true;
    # ... programs, shell config, dotfile links
  };
}
```

### Key differences from NixOS config

| Aspect | NixOS (~/nixos-config) | nix-darwin (~/nix-darwin) |
|--------|------------------------|--------------------------|
| Builder | `nixpkgs.lib.nixosSystem` | `nix-darwin.lib.darwinSystem` |
| HM module | `home-manager.nixosModules.home-manager` | `home-manager.darwinModules.home-manager` |
| Rebuild cmd | `sudo nixos-rebuild switch --flake .#host` | `darwin-rebuild switch --flake .#host` |
| Structure | Multi-host, modules split by concern dirs | Single-host flat layout |
| Feature toggles | Custom `module.*` options | Direct module imports (no toggle layer) |
| Homebrew | N/A | `nix-homebrew` for casks not in nixpkgs |
| HM config | Shared modules in `modules/home/` | Inline in `home.nix` |
| Linux builder | N/A | `nix.linux-builder.enable = true` for cross-builds |

### nix-homebrew integration

```nix
# Inside home.nix or a dedicated homebrew.nix
nix-homebrew = {
  enable = true;
  user = "<username>";
  taps = {
    "homebrew/homebrew-core" = inputs.homebrew-core;
    "homebrew/homebrew-cask" = inputs.homebrew-cask;
  };
  mutableTaps = false;  # declarative taps only
};
```

### Common operations

| Task | Command |
|------|---------|
| Rebuild & switch | `darwin-rebuild switch --flake .#<hostname>` |
| Check config | `darwin-rebuild check --flake .#<hostname>` |
| Update inputs | `nix flake update` |
| Rollback | `darwin-rebuild switch --rollback` |
| List generations | `darwin-rebuild --list-generations` |

## Overlays

```nix
# overlays/default.nix
final: prev: {
  myApp = prev.callPackage ../pkgs/my-app { };
  # Override existing package:
  htop = prev.htop.overrideAttrs (old: {
    patches = (old.patches or []) ++ [ ./htop-custom.patch ];
  });
}
```

Register in flake:
```nix
overlays.default = import ./overlays/default.nix;
```

## Common Operations Reference

| Task | Command                                                            |
|------|--------------------------------------------------------------------|
| Enter devshell | `nix develop`                   |
| Rebuild NixOS | `nh os build`                                                       |
| Update one input | `nix flake update <input-name>`                                    |
| Update all inputs | `nix flake update`                                                 |
| Show flake outputs | `nix flake show`                                                   |
| Check flake | `nix flake check`                                                  |
| Build a package | `nix build .#packageName`                                          |
| Run without installing | `nix run nixpkgs#ripgrep -- --help`                                |
| Evaluate expression | `nix eval .#nixosConfigurations.host.config.services.nginx.enable` |
| Garbage collect | `nix-collect-garbage -d`                                           |
| List generations | `nix profile history --profile /nix/var/nix/profiles/system`       |
| Format nix files | `nix fmt` (uses `formatter` output)                                |

## Debugging & Troubleshooting

- **Hash mismatch**: Use `nix-prefetch-url --unpack <url>` or `lib.fakeHash` to get correct SRI hash.
- **Infinite recursion**: Usually caused by `rec {}` or overlays referencing `final` where `prev` is needed.
- **IFD errors in flake**: Move IFD out of flake eval path; use a generated file checked into git instead.
- **Unfree package**: Add `nixpkgs.config.allowUnfree = true` or use `config.allowUnfreePredicate`.
- **Broken package**: `nixpkgs.config.allowBroken = true` or better, overlay a fix.
- **Binary cache miss**: Check `nix path-info --store https://cache.nixos.org <drv>` or use Cachix.

## Style & Conventions

1. **Use `lib` functions** — prefer `lib.mkIf`, `lib.mkOption`, `lib.mkEnableOption` over raw if/then.
2. **Avoid `with pkgs;`** at module level — it pollutes scope and hides dependencies. Use `with pkgs;` only in tight `packages = [ ... ]` lists.
3. **Prefer `pkgs.callPackage`** over raw import with manual arg passing.
4. **Document custom modules** with `mkOption { description = "..."; }`.
5. **Use `lib.types`** for option type safety (`types.str`, `types.listOf types.package`, etc.).
6. **One concern per module file** — networking, users, services each get their own file.

## Security Considerations

- Never put secrets directly in Nix files (they end up in `/nix/store` world-readable).
- Use `sops-nix`, `agenix`, or `systemd` credentials for secrets management.
- Personal systems are tracking `unstable` nixpkgs. ONLY switch to other branches to pin or change versions when `unstable` fails
- Review `nixpkgs` advisories and override vulnerable packages via overlays when patches lag.

