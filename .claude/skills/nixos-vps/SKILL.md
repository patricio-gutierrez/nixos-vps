---
name: nixos-vps
description: Recipes for working in a project based on the nixos-vps flake template (NixOS VPS with sops-nix secrets, auto-imported modules/ and services/, just bootstrap/deploy, optional Tailscale and Cloudflare Tunnel). Use when adding or changing a service, container, secret, firewall/port, host, Tailscale or Cloudflare setup, when deploying or verifying a VPS, when starting a new project from this template, or when debugging an evaluation, build, deploy, or secrets error in such a repo.
---

# Working in a nixos-vps project

Read `CLAUDE.md` first. It has the layout, how settings flow through the flake, and
the rules that must not break. This skill contains the procedures. Each one ends
with a verification step: never report a change as done without running it, or
without saying plainly that it couldn't be run.

Before acting, identify:
- **Which host(s)?** The `nixosConfigurations` attribute(s) in `flake.nix` and their
  settings files (`settings.nix`, `hosts/*.nix`). Code in `modules/`, `services/`, or
  `configuration.nix` applies to **every** host.
- **SSH target and builder** the user deploys with (for example `admin@1.2.3.4`,
  `admin@100.x.y.z` when `access = "tailscale"`, and builder `local`, `target`, or a
  host). Ask the user rather than guessing.
- **Is `nix` available locally?** If it isn't, you can't evaluate; say so and give the
  user the commands to run.

## Hard rules

- Don't run `just deploy` or `just bootstrap`, or SSH to a server, unless the user
  explicitly asked for it in this session. Deploying changes a live server, and
  bootstrap opens an interactive editor.
- Never write secret values into Nix files, commit messages, or chat output. Never
  decrypt secrets files unless asked. Secret values go only into SOPS through
  `just bootstrap <hostname> <target>` (the user edits them).
- Never relax the SSH, credential, or firewall assertions to get past an error.
- Never change `system.stateVersion`.
- After creating any file the flake needs, `git add` it (the flake can't see untracked
  files). Don't commit unless asked.

## Recipe: add a host service (systemd)

Create `services/<app>.nix`. It is auto-imported, so don't edit any imports list.
Gate it with an option if not every host should run it:

```nix
{ config, lib, pkgs, ... }:
let
  cfg = config.custom.services.<app>;
in {
  options.custom.services.<app>.enable = lib.mkEnableOption "<app>";

  config = lib.mkIf cfg.enable {
    systemd.services.<app> = {
      description = "<App>";
      wantedBy = ["multi-user.target"];
      after = ["network-online.target"];
      wants = ["network-online.target"];
      serviceConfig = {
        ExecStart = "${pkgs.<pkg>}/bin/<bin> --listen 127.0.0.1:<port>";
        DynamicUser = true;
        StateDirectory = "<app>";        # /var/lib/<app>
        WorkingDirectory = "/var/lib/<app>";
        Restart = "on-failure";
      };
    };
  };
}
```

Enable it for a single host in that host's `extraModules` in `flake.nix`:

```nix
vps = mkVps (import ./settings.nix) [ { custom.services.<app>.enable = true; } ];
```

(If the module should run on every host unconditionally, drop the option and the
`mkIf`.) Prefer an upstream NixOS module (`services.<name>.enable = true; …`) when one
exists. Check it at search.nixos.org/options for the `26.05` channel, and put that
configuration in a `services/<app>.nix` file the same way.

Then run `git add services/<app>.nix` and complete **Verify** below.

## Recipe: add a container

Either edit `modules/containers.nix` or create `services/<app>.nix`. Always set the
backend, because the default is podman:

```nix
{ ... }: {
  virtualisation.oci-containers = {
    backend = "docker";                       # or "podman"
    containers.<app> = {
      image = "<registry>/<image>:<pinned-tag-or-digest>";
      ports = ["127.0.0.1:<hostPort>:<containerPort>"];   # always bind to localhost
      volumes = ["/var/lib/<app>:/data"];     # persistent data, if any
      # environmentFiles = [config.sops.secrets.<app>-env.path];  (see the secrets recipe)
    };
  };
}
```

The unit is `docker-<app>.service` or `podman-<app>.service`. Published container
ports bypass the NixOS firewall (even when `access = "tailscale"`), so never publish
on `0.0.0.0` unless the user explicitly wants a public port. Only one backend can be
set for the whole system, so check other modules for an existing `backend` before
setting a different one.

## Recipe: give a service a secret

1. Pick a key name, for example `<app>-api-key` (a plain string) or `<app>-env`
   (`KEY=value\n` lines for an env file).
2. Ask the user to add it. Do not add it yourself:
   `just bootstrap <hostname> <current-ssh-target>`, then add the JSON entry while
   keeping the existing entries. After that, run `git add secrets/<hostname>.json`.
3. Declare and wire it inside the service's module (inside its `mkIf`, so that hosts
   which don't have the key aren't affected):

   ```nix
   sops.secrets."<app>-api-key".restartUnits = ["<app>.service"];
   systemd.services.<app>.serviceConfig.LoadCredential =
     "api-key:${config.sops.secrets."<app>-api-key".path}";
   # The app reads $CREDENTIALS_DIRECTORY/api-key (no environment variable is created).
   ```

   For containers, use `sops.secrets."<app>-env".restartUnits = ["docker-<app>.service"];`
   and `environmentFiles = [config.sops.secrets."<app>-env".path];`.
   For a native service that must read the file directly as its own user, use
   `owner = "<user>";` (don't combine it with `DynamicUser`).
4. If you declare a secret whose key is missing from the file, the build fails, and
   a missing or untracked `secrets/<hostName>.json` fails evaluation. Every host that
   evaluates the declaration needs that key in **its own** secrets file.

## Recipe: expose an application

- **Private (recommended while developing):** keep it on `127.0.0.1` and use
  `ssh -L <port>:127.0.0.1:<port> admin@<target>`.
- **Public through Cloudflare Tunnel (no open ports):** see the Cloudflare recipe and
  add an ingress rule pointing at `http://127.0.0.1:<port>`.
- **Directly on a public port:** add `networking.firewall.allowedTCPPorts = [<port>];`
  and bind to `0.0.0.0`. This has **no effect** when `access = "tailscale"` (all
  public host ports are force-closed). Tell the user this whenever it applies.
- **Tailnet only:** set `access = "tailscale"`. Every port on `tailscale0` is trusted.

## Recipe: add another host

1. Run `mkdir -p hosts && cp settings.nix hosts/<name>.nix`, then set `hostname`,
   `sshKeys`, `hashedPassword`, and the switches. All 7 keys must stay.
2. In `flake.nix`, add `<name> = mkVps (import ./hosts/<name>.nix) [ ];` to
   `nixosConfigurations`. Put the host's own modules or overrides in that list.
3. The user runs `just bootstrap <hostname> root@<NEW_IP>`. Then run
   `git add hosts/<name>.nix secrets/<hostname>.json`.
4. First deploy: `just deploy root@<NEW_IP> <name> [builder]`. Keep a root session
   open, verify `ssh admin@<NEW_IP> 'whoami; hostname'`, and reboot.
5. Remember that every shared module that declares secrets now needs those keys in
   this host's secrets file as well, or must be gated off for this host.

## Recipe: enable Tailscale, then restrict access

1. The user adds `"tailscale-auth-key": "<key>"` through `just bootstrap` (a
   non-ephemeral, single-use key is recommended), then runs `git add` on the secrets
   file.
2. Set `tailscale.enable = true;` and **keep** `access = "public";`, then deploy.
3. Verify `ssh admin@<ip> 'tailscale status; tailscale ip -4'`, then confirm that
   `ssh admin@100.x.y.z` works from the user's machine.
4. Only then set `access = "tailscale";` and deploy **through the Tailscale IP**.
   From now on, every deploy and bootstrap uses the Tailscale address or MagicDNS
   name.

## Recipe: enable Cloudflare Tunnel (locally managed tunnel)

1. The user runs `cloudflared tunnel login && cloudflared tunnel create <name>` locally,
   then `jq -Rs . ~/.cloudflared/<UUID>.json`. They add that output as
   `"cloudflared-<UUID>.json"` through `just bootstrap` and run `git add` on the
   secrets file.
2. In the host's settings, set `cloudflared = { enable = true; tunnelId = "<UUID>"; };`.
3. Add the routes. For one host, add them to its `extraModules` in `flake.nix`, which
   is safer than `configuration.nix` (shared by all hosts):

   ```nix
   { services.cloudflared.tunnels."<UUID>".ingress."app.example.com" = "http://127.0.0.1:<port>"; }
   ```

4. The user runs `cloudflared tunnel route dns <UUID> app.example.com`, then deploys.
5. Logs: `sudo journalctl -u cloudflared-tunnel-<UUID>.service -n 50 --no-pager`.
   Remind the user that the app is now public; add Cloudflare Access or app-level
   authentication if it should be private.

## Recipe: start a new project from the template

1. Clone or fork the repo. Keep the result **private** once real credentials are added.
2. Fill in `settings.nix`: the public SSH key, the hash from
   `nix run nixpkgs#mkpasswd -- --method=yescrypt`, `hostname`, `access = "public"`,
   and both optional services disabled.
3. If the VPS isn't the Hostinger NixOS template, adapt `vps_configuration.nix`
   (bootloader device, filesystems, networking, cloud-init) before the first deploy.
4. Run `just bootstrap <hostname> root@<IP>` and
   `git add secrets/<hostname>.json`.
5. Run `just deploy root@<IP> vps [builder]` (on macOS without an x86_64 Linux
   builder, pass a remote x86_64 builder). Verify admin SSH, reboot, then check
   `systemctl --failed`.

## Verify

Run as many of these as the environment allows, starting with the cheapest:

```sh
git status --short                       # are the new files staged/tracked?
nix eval .#nixosConfigurations.<name>.config.system.build.toplevel.drvPath   # eval + assertions
nix build .#nixosConfigurations.<name>.config.system.build.toplevel --dry-run # what would build
python3 tests/deploy.py && python3 tests/module-discovery.py   # if scripts/discovery changed
nix eval --impure --json --expr 'import ./tests/admin-access.nix { flake = builtins.getFlake ("path:" + toString ./.); }'
```

- If `settings.nix` still has the template's empty credentials, the first `nix eval`
  fails with the admin assertion. That is expected; it doesn't mean your change is
  broken. Evaluate with dummy credentials instead, without editing `settings.nix`
  (replace `vps` with the configuration name if it differs):

  ```sh
  nix eval --impure --raw --expr '
    let
      flake = builtins.getFlake ("git+file://" + toString ./.);
      lib = flake.inputs.nixpkgs.lib;
    in (flake.nixosConfigurations.vps.extendModules { modules = [{
      users.users.admin.hashedPassword = lib.mkForce "dummy-hash-for-eval-only";
      users.users.admin.openssh.authorizedKeys.keys = lib.mkForce
        ["ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA test"];
    }]; }).config.system.build.toplevel.drvPath'
  ```

- Inspect a single option: `nix eval .#nixosConfigurations.<name>.config.<option.path>`.
- After a deploy run by the user, check:
  `ssh admin@<target> 'systemctl --failed --no-pager; systemctl status <unit> --no-pager'`
  and `curl` the service on `127.0.0.1`.

## Troubleshooting

| Symptom | Cause / fix |
| --- | --- |
| `Set a non-empty admin password hash in settings.nix …` | `hashedPassword` is empty, blank, or `!`/`!!`/`*`. Generate it with mkpasswd. |
| `Set at least one non-empty admin SSH public key …` | `sshKeys` is empty or blank. Add a public key (`.pub`). |
| `custom.networking.access = tailscale requires services.tailscale.enable = true.` | Enable Tailscale or go back to `access = "public"`. |
| `Set services.cloudflared.tunnelId before enabling Cloudflare Tunnel.` | `cloudflared.enable = true` with an empty `tunnelId`. |
| `Path 'secrets/<host>.json' does not exist in Git repository` | No secrets file for this `networking.hostName`. Run `just bootstrap <hostname> <target>`, or fix a hostname/file-name mismatch. |
| `Path 'secrets/<host>.json' … is not tracked by Git` | Run `git add secrets/<host>.json`. |
| Build fails in the sops manifest check (key not found) | The declared `sops.secrets.<name>` key is missing from that host's encrypted file. Evaluation still succeeds; only the build catches this. |
| New module has no effect | The file isn't tracked by Git, has no `.nix` extension, is a symlink, is a subdirectory without `default.nix`, or is gated off by `mkIf`. |
| Value from `settings.nix` is ignored | A module assigns the option without `mkDefault`, which overrides the settings value. |
| `Local builds need x86_64-linux support …` | No x86_64 Linux builder. Pass a remote builder (third argument) or set up `nix.linux-builder` (README). |
| `Transfer failed. The target SSH account must be a trusted Nix user …` | The VPS predates the trusted-users change. Run `just deploy admin@IP <name> target` once. |
| Port open but unreachable | `access = "tailscale"` force-closes public ports. Otherwise check that the app binds to `0.0.0.0` and the port is in `allowedTCPPorts`. |
| Secrets fail to decrypt after the VPS was reinstalled | The host key changed. Get the new recipient with `ssh admin@IP 'cat /etc/ssh/ssh_host_ed25519_key.pub' \| ssh-to-age` and rekey the file with SOPS (`--add-age`/`--rm-age` with rotate; see `sops --help` for your version). Bootstrap never rekeys. |
| `sops-install-secrets` / `tailscaled-autoconnect` fails on the VPS | Check `journalctl -u <unit>`. Usually the key was added locally but not deployed, or the auth key was used up. |
