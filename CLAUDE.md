# CLAUDE.md

Flake-based NixOS configuration for one or more x86_64 VPS hosts (built for the
Hostinger NixOS KVM template). It is a **template**: projects fork it, fill in
`settings.nix`, and add their own services. Optional Tailscale and Cloudflare Tunnel
modules are included. Secrets are managed with sops-nix (age). Deployment builds
locally or on a chosen builder, copies the result to the VPS, and activates it there.

For step-by-step recipes (adding a service, secret, host, or enabling Tailscale or
Cloudflare, plus deploying and verifying), use the `nixos-vps` skill in
`.claude/skills/nixos-vps/`. `README.md` is the user-facing guide. Keep it in sync when
behaviour changes.

## Layout

| Path | Role |
| --- | --- |
| `flake.nix` | Inputs (`nixpkgs` on `nixos-26.05`, `sops-nix`). `mkVps settings extraModules` builds each host. `nixosConfigurations.vps` is the default host. |
| `settings.nix` | Per-host values: `hostname`, `sshKeys`, `hashedPassword`, `access`, `tailscale.enable`, `cloudflared.{enable,tunnelId}`. All 7 keys are required. |
| `configuration.nix` | Shared base: imports, flakes, `trusted-users = ["admin"]`, admin credential assertions, SSH hardening, `admin` user (wheel). |
| `vps_configuration.nix` | Hostinger/Proxmox hardware and provisioning: GRUB on `/dev/sda` (BIOS), ext4 root `by-label/nixos` with auto-resize, serial console, qemu-guest-agent, cloud-init. `stateVersion = "26.05"`. |
| `modules/` | Base-system modules, **auto-imported**: `networking.nix` (the `custom.networking.access` option), `secrets.nix` (the sops file path), `containers.nix` (empty OCI containers). |
| `services/` | Application and optional-service modules, **auto-imported**: `tailscale.nix`, `cloudflared.nix` (adds the `services.cloudflared.tunnelId` option). |
| `lib/import-directory.nix` | Auto-import helper (see below). |
| `secrets/<hostname>.json` | SOPS-encrypted JSON per host. It does not exist until `just bootstrap` creates it. |
| `hosts/<name>.nix` | Extra hosts' settings files (copies of `settings.nix`). The directory doesn't exist until you add a second host. |
| `scripts/deploy.sh`, `scripts/bootstrap-secrets.sh` | Called by the `justfile` recipes. |
| `tests/` | Python unittest suites and a Nix eval test (see Testing). |

## How configuration flows

1. `flake.nix` `mkVps` imports the settings file and turns each value into an option
   with **`lib.mkDefault`**: `networking.hostName`, `users.users.admin.{openssh.authorizedKeys.keys,hashedPassword}`,
   `custom.networking.access`, `services.tailscale.enable`, and `services.cloudflared.{enable,tunnelId}`.
   Any ordinary assignment in a module therefore silently overrides settings.
   `extraModules` (the second `mkVps` argument) is where per-host overrides go.
2. `configuration.nix` imports `vps_configuration.nix`, `./modules`, and `./services`.
3. `modules/default.nix` and `services/default.nix` call `lib/import-directory.nix ./.`.
   It imports every sibling **regular file** ending in `.nix` and every
   **subdirectory containing `default.nix`**, sorted by name. It excludes `default.nix`
   itself, non-`.nix` files (such as `foo.nix.bak`), symlinks, and subdirectories
   without a `default.nix`. Every discovered module is always evaluated, so optional
   features must be gated with `lib.mkIf cfg.enable`.
4. Everything in `configuration.nix`, `modules/`, and `services/` is **shared by every
   host**. Configure a single host through its `extraModules` list in `flake.nix`, or
   through a setting.

## Commands

Run these from the repository root. You need `nix` (with flakes) and `just`; use
`nix shell nixpkgs#just` if `just` isn't installed.

```sh
just bootstrap <hostname> [ssh-target] [age-key-file]   # create/edit secrets/<hostname>.json (SOPS editor); target defaults to admin@<hostname>, key to $SOPS_AGE_KEY_FILE or ~/.config/sops/age/keys.txt
just deploy <ssh-target> <config-name> [builder]         # build, copy, activate (defaults: admin@vps vps local)
```

- `builder`: `local` (default) builds through the local Nix daemon and its configured
  builders. `target` builds on the VPS itself, for recovery or a trust bootstrap. Any
  other value is an SSH host or store URL and gets the `ssh-ng://` prefix if it has no
  `://`.
- The first deploy uses `root@IP`. After that, use `admin@IP`. Activation runs
  `sudo nixos-rebuild switch --no-reexec --store-path …` and prompts for the admin
  password.
- `NIX_SSHOPTS` (for example `-p 2222`) is passed to every SSH call in both scripts.
- `<config-name>` is the `nixosConfigurations` attribute and must match
  `^[a-zA-Z0-9][a-zA-Z0-9_-]*$` (no dots). `<hostname>` names the secrets file and may
  contain dots.
- The config name and the hostname are different things. `vps` stays the attribute
  name even if `settings.hostname` changes. **The secrets file is chosen by
  `networking.hostName`**, not by the attribute name.

## Testing

```sh
python3 tests/deploy.py              # mocks nix/ssh; test_just_recipe_* needs `just` on PATH
python3 tests/module-discovery.py    # needs nix; evaluates a temp copy of the flake
nix shell --inputs-from . nixpkgs#sops nixpkgs#age nixpkgs#ssh-to-age \
  --command python3 tests/bootstrap-secrets.py   # real SOPS, mocked SSH; also needs ssh-keygen
nix eval --impure --json --expr 'import ./tests/admin-access.nix { flake = builtins.getFlake ("path:" + toString ./.); }'
```

- All tests evaluate only; none builds Linux packages, so they run on macOS.
- With the template's empty `settings.nix`, evaluating `nixosConfigurations.vps`
  **fails on purpose** (admin credential assertions). The tests inject dummy credentials
  with `extendModules`/`mkForce`, or write their own `settings.nix`.
- If you add a key to the settings schema (and use it in `mkVps`), update every
  settings file (`settings.nix`, `hosts/*.nix`), the hard-coded settings in
  `tests/module-discovery.py`, and the README.
- If you add a new assertion that could fire under default settings, make
  `tests/admin-access.nix` pin it off the way it pins Tailscale and Cloudflare.
- No CI, formatter, or linter is configured. Match the existing style: 2-space indent,
  `{ config, lib, ... }:` headers, `["a" "b"]` lists, and comments that explain *why*.

## Invariants and gotchas (read before editing)

- **The flake is Git-backed.** Untracked files are invisible to builds. After creating
  modules, `hosts/*.nix`, or `secrets/*.json`, run `git add` on them. Uncommitted
  edits to tracked files *are* included.
- **Secrets**: declaring `sops.secrets.<name>` makes evaluation hash
  `secrets/<hostName>.json`. That fails if the file is missing or untracked. The build
  also checks that `<name>` exists as a key in the file. Values decrypt to
  `/run/secrets/<name>` (mode 0400, root) using the VPS's
  `/etc/ssh/ssh_host_ed25519_key`. Use `restartUnits`, and pass secrets to services
  through `LoadCredential` or `environmentFiles`. **Never** put plaintext secrets in
  Nix files, `environment`, or Git. There is no `.sops.yaml`: recipients are fixed when
  the file is created, and bootstrap never rekeys. A reinstalled VPS (new host key)
  needs a manual SOPS rekey.
- `modules/secrets.nix` resolves the repository root as `../.`, so moving it breaks the
  secrets path (`tests/module-discovery.py` checks this).
- **Lockout risks.** `PermitRootLogin = "no"`, `AllowUsers = ["admin"]`, and key-only
  SSH. Assertions require a non-blank `hashedPassword` (not `!`, `!!`, or `*`) and at
  least one non-blank SSH key. `access = "tailscale"` force-closes every public
  host port and requires `tailscale.enable`. Switch to it only after SSH over Tailscale
  works. Never weaken these to make a build pass. Fix the settings instead.
- `access = "tailscale"` overrides all firewall port lists with `mkForce`. Opening a
  public port in a module has no effect in that mode. It does **not** filter
  Docker/Podman published ports (they bypass the NixOS firewall). Bind containers to
  `127.0.0.1`.
- `admin` is a **trusted Nix user** (root-equivalent, needs no sudo password).
  `deploy.sh` copies with `--no-check-sigs` because builds are unsigned. Global
  `require-sigs` stays on.
- The default OCI backend is **podman** (because `stateVersion` ≥ 22.05). Set
  `virtualisation.oci-containers.backend` explicitly. Units are named
  `docker-<name>.service` or `podman-<name>.service`.
- Cloudflare: the secret key is `cloudflared-<tunnelId>.json` (the credentials JSON
  stored as a string). The unit is `cloudflared-tunnel-<tunnelId>.service`. The
  default response is `http_status:404` until `ingress` is set.
- Tailscale: the secret key is `tailscale-auth-key`. The service sets `--hostname` to
  `networking.hostName` and opens UDP 41641 (`services.tailscale.port`).
- **Never change `system.stateVersion`**. Don't "tidy" `vps_configuration.nix` either:
  DHCP is `mkForce false` because cloud-init owns networking, the qemu-guest-agent
  `PATH` override is required for Hostinger provisioning, and the comments about
  `image.nix` refer to the upstream image build, which isn't part of this repo. On
  another provider, adapt the boot, disk, and network settings in that file.
- The target is **x86_64-linux only**. macOS or ARM machines need a Linux builder or
  a remote builder (README: "Local building on macOS").
- Kernel changes take effect only after a reboot (`ssh -t admin@IP 'sudo reboot'`).

## Conventions

- Commit messages use Conventional Commit-style prefixes: `feat:`, `fix:`, `docs:`,
  `chore:`, `improvement:`, written as short imperative phrases.
- Put new options under the `custom.` namespace (for example
  `custom.services.<app>.enable`).
- One application per file in `services/`. Use a subdirectory with `default.nix` when
  a service needs several files.
- Prefer upstream NixOS service modules over hand-written systemd units. Bind
  applications to `127.0.0.1` and publish them through Cloudflare Tunnel or Tailscale,
  not open ports.
- Forks may commit real `sshKeys` and `hashedPassword` values. Keep such repositories
  private: a password hash can be cracked offline.
