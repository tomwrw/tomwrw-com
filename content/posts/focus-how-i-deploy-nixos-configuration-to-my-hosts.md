+++
title = "Focus: How I deploy NixOS configuration to my hosts"
description = "A deep dive on the deploy step in megadots - where it started, why it exists at all, what it actually does line by line, and how to skip it if you don't need it."
date = 2026-09-13
draft = true
tags = ["nixos", "nix", "nixos-anywhere", "sops-nix", "disko", "impermanence", "deployment", "megadots", "focus"]
categories = ["focus"]
+++

When I posted [megadots](https://github.com/tomwrw/megadots) on Reddit, the most common comment was some version of "the deploy process looks complex". That's a fair thing to say from the outside. There's a USB stick involved, a staging directory, a `--chown` flag, and a Nix file that generates shell scripts. If you're used to `nixos-rebuild switch` and nothing else, it looks like a lot of code for the job of installing an operating system.

So this post is me trying to make it completely clear. What `nix run .#deploy` does, in order. Why each piece is there, traced back to a specific problem. How it got here, because it didn't start this way and the wrong turns are as useful as the right ones. And, importantly, which parts you don't need, because most people don't.

TLDR: it's [nixos-anywhere](https://github.com/nix-community/nixos-anywhere), plus a directory of key material handed to it. Every line that isn't nixos-anywhere exists because of one problem, which is getting secrets onto a machine whose root filesystem is deleted every time it boots.

## What it looks like from the outside

Here's the whole experience of installing a machine. I boot the target from a NixOS installer USB, set a password on the `nixos` user so I can SSH in, plug my keys stick into the machine I'm deploying from, and run:

```bash
nix run .#deploy flatmate
```

Then I type the LUKS passphrase when the disk is formatted, and wait. The machine reboots into a system with its secrets decryptable, my SSH keys in my home directory, my theme applied and my applications installed. There's no second pass, nothing to `scp` across afterwards, and nothing to do by hand.

Get the host name wrong and it stops before touching anything:

```
$ nix run .#deploy typo
no such host: typo
valid hosts: endgame flatmate
```

That's the entire user interface. Everything below is what's behind it.

## The problem it solves

If you take nothing else from this post, take this section, because it's the reason the deploy step exists at all.

nixos-anywhere on its own is a one-liner. Point it at a flake output and a host running the installer, and it partitions the disk, installs the system and reboots. If that's all you need, that's all you should run, and I'll come back to that at the end.

I needed more than that because of four facts about my configuration, and every line of the deploy script is a consequence of one of them.

**Fact one: my secrets are decrypted on the machine, at activation, with a key that has to already be there.** I use [sops-nix](https://github.com/Mic92/sops-nix). Secrets live in the repo encrypted, and when the system activates, sops-nix decrypts them using an [age](https://github.com/FiloSottile/age) private key on the local disk. That key can't be in the repo, because the repo is public. And the machine can't just generate one on first boot, because then nothing in the repo would be encrypted to it yet, and you're into a second pass of "get the new public key, add it as a recipient, re-encrypt everything, rebuild". So the key has to be *delivered* to the machine during the install, before the first activation ever runs.

**Fact two: my root filesystem is deleted on every boot.** Both hosts run an ephemeral btrfs root. An initrd service rolls `/` back to a blank snapshot before anything else mounts. Only `/nix` and `/persist` survive. So the age key can't go where sops-nix would put it by default. It has to land under `/persist`, and `/persist` has to be mounted early enough in boot for activation to read it.

**Fact three: I have user-scope secrets and user keys too, not just system ones.** sops-nix has a Home Manager module as well as a NixOS one, and I use both. My user has its own age key in `~/.config/sops/age/keys.txt`, and my SSH keys need to be in `~/.ssh` from the first login, because my Git commits are signed with one of them. Because of fact two, all of that lives in `/persist/home/tomwrw` and is bind-mounted into my real home by [impermanence](https://github.com/nix-community/impermanence).

**Fact four: nixos-anywhere copies files as root.** It has a flag, `--extra-files`, that takes a directory and copies its contents onto the installed system. That's how the key material gets there. But it copies everything owned by root, and a private key in my home directory owned by root is a key I can't use. So the ownership has to be fixed, and it has to be fixed during the install rather than after, because "after" is a second pass.

That's it. Deliver keys, put them under `/persist`, include the user's, fix the ownership. Hold those four in your head and the script reads itself.

## Where it started

I want to walk through the history, because I think the wrong turns are more instructive than the current version, and because "it was always like this" would be a lie.

### April: plain nixos-anywhere, and keys in the repo

The first version was a justfile, and it was mostly the raw command typed out three times:

```just
cachyos-cache := "--option extra-substituters '...' --option extra-trusted-public-keys '...'"

# Decrypt the committed host age key + LUKS passphrase for a nixos-anywhere deploy.
prep HOST:
    rm -rf /tmp/extra-files /tmp/luks-password
    mkdir -p /tmp/extra-files/persist/var/lib/sops-nix
    age -d -i ~/.config/sops/age/keys.txt keys/{{ HOST }}.enc > /tmp/extra-files/persist/var/lib/sops-nix/key.txt
    chmod 600 /tmp/extra-files/persist/var/lib/sops-nix/key.txt
    sops -d --extract '["luks-passphrase"]' secrets/{{ HOST }}.yaml > /tmp/luks-password
    chmod 600 /tmp/luks-password

# Deploy (remote, wipes target).
spectre-deploy: (prep "spectre")
    nix run github:nix-community/nixos-anywhere -- --disko-mode disko --extra-files /tmp/extra-files --disk-encryption-keys /tmp/luks-password /tmp/luks-password --flake .#spectre --target-host nixos@spectre {{ cachyos-cache }}

endgame-deploy: (prep "endgame")
    ...

flatmate-deploy: (prep "flatmate")
    ...
```

You can already see facts one and two in there. The `prep` recipe builds a directory shaped like the target filesystem, puts the host's age key at `persist/var/lib/sops-nix/key.txt`, and hands it to `--extra-files`. That part never changed.

What did change was where the key came from. In this version, each host's private age key was *committed to the repo*, encrypted with age to my personal key, as `keys/<host>.enc`. The `prep` step decrypted it on the way through. The LUKS passphrase was in sops too, extracted to `/tmp` and passed with `--disk-encryption-keys` so the format wouldn't prompt.

It worked. It was also wrong in several ways I could feel but hadn't articulated:

- Every host's private key was in a public repository. Encrypted, yes, but a private key that's one leak of my personal key away from being public is not a private key I'm comfortable with. The repo should only ever contain public recipients, which is what `.sops.yaml` is for.
- Three near-identical recipes, one per host, and a host called `spectre` that no longer exists. Adding a host meant copying a recipe and editing it in three places.
- Staging in a fixed path under `/tmp` and cleaning it up with `rm -rf` at the start of the *next* run rather than the end of this one.
- `nix run github:nix-community/nixos-anywhere` pulls whatever is on that repo's HEAD at the moment I run it. For the one command that formats disks, that's more improvisation than I want.

### June: the reset

In June I rebuilt the whole configuration on [den](https://github.com/denful/den), which is the fourth-rewrite story from [the last post](/posts/introducing-my-nixos-configuration-megadots/). The justfile came back as three recipes:

```just
deploy HOST:
    nix run github:nix-community/nixos-anywhere -- \
      --disko-mode disko \
      --flake .#{{ HOST }} \
      --target-host nixos@{{ HOST }}

build HOST:
    nixos-rebuild build --flake .#{{ HOST }}

rebuild HOST:
    nixos-rebuild switch --flake .#{{ HOST }} --target-host tomwrw@{{ HOST }} --sudo --ask-sudo-password
```

I'm including this because it's the answer to "what does deploying look like without any of the complexity". That's it. No keys, no staging, no chown. If you don't have secrets to deliver, this is the whole thing, and the only reason I'd even wrap it in a recipe is to not have to remember the flags.

sops-nix came back the next day, and the deploy recipe started growing again.

### June to September: the justfile at its peak

By September the justfile had fourteen recipes and the deploy one looked like this:

```just
usb := "/run/media/tomwrw/SURVIVOR/keys"
user := "tomwrw"

# Install HOST from scratch over SSH with nixos-anywhere. FORMATS ITS DISKS.
deploy HOST:
    #!/usr/bin/env bash
    set -euo pipefail
    staging=$(mktemp -d)
    trap 'rm -rf "$staging"' EXIT
    install -Dm600 {{ usb }}/hosts/{{ HOST }}/age.txt "$staging/persist/var/lib/sops-nix/key.txt"
    for u in {{ usb }}/users/*/; do
      [[ -d "$u" ]] || continue
      install -d "$staging/persist/home/$(basename "$u")"
      cp -a "$u." "$staging/persist/home/$(basename "$u")/"
    done
    chown_args=()
    for u in {{ usb }}/users/*/; do
      [[ -d "$u" ]] || continue
      chown_args+=(--chown "/persist/home/$(basename "$u")" 1000:100)
    done
    nix run .#nixos-anywhere -- \
      --disko-mode disko \
      --extra-files "$staging" \
      "${chown_args[@]}" \
      --flake .#{{ HOST }} \
      --target-host nixos@{{ HOST }}
```

Three things happened between the reset and this.

**The keys moved to a USB stick.** The repo now contains no private key material at all, in any form. The stick has a directory tree that *mirrors the destination*: `hosts/<host>/age.txt` goes to the host's sops key path, and everything under `users/<user>/` is copied to that user's home at the same relative path. There is no manifest and no mapping. If I want a new key on a machine, I put it on the stick at the path it should have in my home, and it arrives there. Nothing in the repo changes.

**The "seed" quirk existed, and then didn't.** This was the wrong turn worth writing about. When the user's keys started arriving via `--extra-files`, they arrived owned by root, which is fact four. My first fix was to handle it *on the machine*: a den quirk called `seed` where an aspect could declare "this path should be owned by the user", a host-scope consumer aspect that turned those declarations into tmpfiles rules and a systemd unit to chown them on boot, a machine-readable option so the justfile could read the list back, and four checks in `nix flake check` to make sure the declarations, the persistence list and the USB layout all agreed with each other.

It worked. It was also a lot of moving parts to solve a problem that only exists for about thirty seconds during an install. Then I noticed nixos-anywhere has a `--chown` flag, which sets ownership on a path as part of the install, which is where the problem is. In August the whole quirk went, replaced by a loop that builds `--chown` arguments. The commit that removed it deleted more than it added, which is usually the sign you've found the right answer.

The `1000:100` is numeric on purpose. The chown runs against the installer image, which has no account with my name on it. `1000` is the first normal user and `100` is the `users` group, which is what my user aspect creates.

**nixos-anywhere got pinned.** A tiny file, `flake/deploy.nix`, exposed `pkgs.nixos-anywhere` from the flake's own nixpkgs as a package, so the recipe could run `nix run .#nixos-anywhere` instead of pulling GitHub HEAD. Same version every time, locked in `flake.lock` alongside everything else.

The justfile did the job, but it had problems of its own:

- The deploy recipe was twenty-five lines of bash that formats disks, and nothing ever checked it. Not shellcheck, not anything.
- A typo in the host name passed straight through to nixos-anywhere or `nixos-rebuild` and failed somewhere deep inside Nix with an error that told you nothing.
- `just` is another tool to have installed before you can do anything, and its variables at the top of the file, like the USB path, were edits rather than overrides.
- Six of the fourteen recipes just wrapped things Nix already does: `nix fmt`, `nix flake check`, `nix flake update`. They existed because `just --list` was the discovery mechanism, and a list looks incomplete with gaps in it.

### September: Nix apps

The current version is a Nix module, [flake/tasks.nix](https://github.com/tomwrw/megadots/blob/main/modules/flake/tasks.nix), and the justfile is gone. Each task is a `writeShellApplication`, which is a nixpkgs function that takes a shell script and turns it into a package. The important part of that is what it does at build time:

- **shellcheck runs over every script when it's built.** That's not a lint step I have to remember to run, it's a build failure if the script has a bug shellcheck can see. It caught one immediately, in the host-name check, that the justfile had been happily shipping.
- **The host list is derived, not typed.** `lib.attrNames config.flake.nixosConfigurations` becomes the list of valid hosts, so adding a host to the flake automatically adds it to what `deploy` will accept, and a typo is refused up front with the valid names printed.
- **nixos-anywhere is a `runtimeInput`.** It comes from the flake's nixpkgs, the same as `flake/deploy.nix` did, but now that file doesn't need to exist.
- **One definition, three ways to run it.** The same derivations are exposed as `packages`, so `nix build .#deploy` runs shellcheck; as `apps`, so `nix run .#deploy endgame` works; and on `$PATH` inside `nix develop`, so in the dev shell it's just `deploy endgame`.

The six wrapper recipes were dropped rather than ported. `nix flake show` lists the tasks with a description each, which does the job `just --list` was doing.

## What it does, line by line

Here's the deploy task as it is today, with the shared preamble inlined so you can read it top to bottom. I've trimmed the comments, but not the code.

```bash
# requireFlake - shared by every task
if [ ! -e flake.nix ]; then
  echo "no flake.nix in $PWD - these tasks run from the checkout" >&2
  exit 1
fi

# requireHost - shared by every task that takes one
host="${1:-}"
matched=""
for h in endgame flatmate; do
  if [ "$h" = "$host" ]; then
    matched=1
    break
  fi
done
if [ -z "$matched" ]; then
  echo "no such host: ${host:-<none>}" >&2
  echo "valid hosts: endgame flatmate" >&2
  exit 1
fi
shift

# deploy itself
usb="${MEGADOTS_USB:-/run/media/tomwrw/SURVIVOR/keys}"
if [ ! -d "$usb" ]; then
  echo "no key material at $usb - is the USB mounted?" >&2
  echo "override with MEGADOTS_USB=/path/to/keys" >&2
  exit 1
fi

staging=$(mktemp -d)
trap 'rm -rf "$staging"' EXIT
install -Dm600 "$usb/hosts/$host/age.txt" \
  "$staging/persist/var/lib/sops-nix/key.txt"

for u in "$usb"/users/*/; do
  [ -d "$u" ] || continue
  install -d "$staging/persist/home/$(basename "$u")"
  cp -a "$u." "$staging/persist/home/$(basename "$u")/"
done

chown_args=()
for u in "$usb"/users/*/; do
  [ -d "$u" ] || continue
  chown_args+=(--chown "/persist/home/$(basename "$u")" 1000:100)
done

nixos-anywhere \
  --disko-mode disko \
  --extra-files "$staging" \
  "${chown_args[@]}" \
  --flake ".#$host" \
  --target-host "nixos@$host" \
  "$@"
```

Taking it in order:

**Refuse to run outside the checkout.** Every task operates on `.`, the flake in the current directory, rather than baking in the committed tree. That's deliberate: for a build-and-test loop you want your uncommitted changes, not the last commit. The cost is that the tasks have to be run from the repo, so the first thing they do is check.

**Refuse a host that doesn't exist.** The `endgame flatmate` in that loop isn't typed by me, it's substituted in at build time from the flake's `nixosConfigurations`. The `shift` at the end drops the host name so that `"$@"` later is only the *extra* arguments, which pass through to nixos-anywhere untouched.

**Find the keys.** The USB path is a default with an environment override, `MEGADOTS_USB`, because a removable mount is exactly the kind of thing that varies by machine. If the directory isn't there, stop now, before anything is formatted.

**Stage the host key.** `install -Dm600` creates the parent directories and writes the file with mode 600 in one go. The path inside the staging directory is the path it will have on the installed system: `/persist/var/lib/sops-nix/key.txt`, which is exactly where the sops aspect tells sops-nix to look. That's facts one and two in one line.

**Stage the user's home.** For each `users/<name>/` on the stick, copy the tree into `persist/home/<name>/`. It's `cp -a` rather than `cp -r` because `-a` preserves modes, so a private key that's 0600 on the stick is 0600 on the machine, and sshd and sops won't refuse it. The consequence is that modes are checked *on the stick*, not here. The `$u.` with the trailing dot is a shell idiom for "the contents of the directory, including dotfiles", which matters because everything of interest here is a dotfile.

**Build the ownership fix.** One `--chown` argument per user directory, numeric for the reason above. Fact four.

**Run nixos-anywhere.** Flag by flag:

- `--disko-mode disko` says "partition, format and mount using the [disko](https://github.com/nix-community/disko) configuration in the flake". Mine is LUKS on btrfs with four subvolumes, and it has a `postCreateHook` that takes a read-only snapshot of the empty root subvolume the moment it's created. That snapshot, `root-blank`, is what the initrd rolls back to on every subsequent boot. It has to be taken at format time, because by the first boot the root already has a system in it.
- `--extra-files "$staging"` copies the staged tree onto the installed system.
- `--chown ...` fixes ownership of the user's directory, during the install.
- `--flake ".#$host"` is which system to build and install.
- `--target-host "nixos@$host"` is the machine sitting at the installer prompt.
- `"$@"` is anything else I passed, like `--debug`. Because of how `nix run` parses arguments, a flag needs a `--` first: `nix run .#deploy -- flatmate --debug`.

nixos-anywhere then does what it does: gets a Nix-capable environment onto the target (it uses the installer if that's what's already booted), runs disko, builds the system closure on my machine and pushes it across, installs the bootloader, copies the extra files and applies the chowns, and reboots.

**First boot.** The initrd rolls `/` back to `root-blank`, then mounts `/persist`, which the disko aspect marks `neededForBoot` precisely so that it's there before activation. Activation runs, sops-nix reads `/persist/var/lib/sops-nix/key.txt` and decrypts the host's secrets. impermanence bind-mounts `~/.ssh` and `~/.config/sops/age` from `/persist/home/tomwrw` into my real home, so when Home Manager's sops-nix runs as my user, its key is there too. I log in, and it's my machine.

Forty-odd lines. Four facts. Everything else is nixos-anywhere.

## The bits you probably don't need

This is the part I most wanted to write. The deploy step above is the right amount of machinery *for my constraints*. Most people have fewer constraints, and every one you don't have removes a piece. Working up from nothing:

**You have no secrets to deliver.** Run nixos-anywhere directly and don't write a wrapper:

```bash
nix run github:nix-community/nixos-anywhere -- \
  --flake .#myhost \
  --target-host nixos@myhost
```

That's the June justfile, and it's a complete deploy. Add `--disko-mode disko` if you're using disko. If you want the version pinned, add `nixos-anywhere` to your dev shell and drop the `github:` part. You're done, and nothing in my repo is relevant to you.

**You use sops-nix, with a normal root filesystem.** You don't need a USB stick, a staging directory, or a chown. The standard approach is to let sops-nix derive its age key from the machine's SSH host key, which it does by default. There are two ways to get there:

- *Two passes.* Install with the one-liner above. SSH in, run `ssh-to-age` against `/etc/ssh/ssh_host_ed25519_key.pub` to get the host's age public key, add it as a recipient in `.sops.yaml`, run `sops updatekeys` on the secrets files, and rebuild. Slightly tedious, entirely reliable, and what most people do.
- *One pass.* Generate the SSH host key yourself beforehand, add its age form as a recipient, and pass the private key in with `--extra-files` at `etc/ssh/ssh_host_ed25519_key`. It's root-owned anyway, so no chown. The [nixos-anywhere docs](https://github.com/nix-community/nixos-anywhere/blob/main/docs/howtos/secrets.md) show exactly this.

I use a dedicated age key rather than the host SSH key because I don't want rotating the SSH key to silently change what can decrypt my secrets, and I set `age.sshKeyPaths = [ ]` to close that door. That's a preference, not a requirement, and if you don't share it the host key route is simpler.

**You use sops-nix and impermanence.** Same as above, but the key path moves under `/persist` and that mount needs `neededForBoot = true`. It's still root-owned, so still no chown. This is the point where the staging directory earns a `mkdir -p`, and that's all.

**You need user-scope keys in place from the first boot.** SSH keys for signing commits, a user-level age key for Home Manager secrets, anything that has to be *yours* rather than root's before you've logged in once. This is where `--chown` comes in, and where a script starts earning its keep over a remembered command. It's also the only step where my approach is doing something the docs don't spell out, and it's a loop and a flag.

**You want to type the host name wrong safely.** This is the whole reason the tasks are Nix apps rather than a justfile: a derived host list and shellcheck at build time. It's a quality-of-life layer, not a functional one. The script in the previous section works exactly the same pasted into a justfile or a `.sh` file.

If you draw a line under the third item, everything above it is one command with a couple of flags. I'm below the line because of the keys in my home directory, and that's the honest answer to "why is it complex".

## What I'd say to April me

Don't put private keys in the repo, even encrypted. Fix ownership where the problem is, not where it's convenient. Pin the thing that formats your disks. And when you find yourself building a quirk, a consumer, an option and four invariants to solve a thirty-second problem, go and read the `--help` output of the tool you're wrapping, because there's a reasonable chance the flag already exists.

The full task file is [here](https://github.com/tomwrw/megadots/blob/main/modules/flake/tasks.nix), comments and all, and the [README](https://github.com/tomwrw/megadots#bootstrapping-a-host-from-scratch) has the checklist of what has to exist before a new host will deploy. If it still looks complex after this, tell me which bit, because that's the bit I've explained badly.
