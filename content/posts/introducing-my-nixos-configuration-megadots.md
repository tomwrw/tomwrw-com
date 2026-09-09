+++
title = "Introducing my NixOS configuration - megadots."
description = "An introduction to NixOS, the Nix language and the den framework."
date = 2026-09-08
tags = ["nixos", "nix", "den", "denful", "linux", "declarative", "megadots"]
+++

I've been daily driving NixOS for nearly two years now. In that time I've rewritten my configuration four times, and the current one is called [megadots](https://github.com/tomwrw/megadots). It runs both of my machines: a desktop and a Surface Pro. It handles the disks, the boot chain, the secrets, the desktop, the theme and the applications, and it does all of it from one Git repo.

I publish it because other people's repos are how I learned. I found reading someone else's working config far more useful than any amount of documentation, so this is me returning the favour.

This post is the introduction I wish I'd had when I started. What Nix is, what NixOS is, why I ended up using a framework on top of it, and what any of it actually buys you day to day.

Fair warning up front. I'm not a developer. I'm a solutions architect who got curious about declarative system management and fell down the rabbit hole. Everything below is how I understand it, not how a compiler engineer would explain it.

## Nix, the language

The first thing to get straight is that Nix is two things wearing the same name, and confusing them makes the whole thing harder than it needs to be.

Nix is a package manager. It's also a small programming language that you use to drive that package manager. NixOS is a Linux distribution built out of both. People say "Nix" for all three and you have to work out which one from context.

The language part is where most people bounce off, so let's deal with it. Nix the language is functional, lazily evaluated, and has almost no syntax to learn. The core of it is the attribute set, which is a key/value map, written with braces:

```nix
{
  hardware.bluetooth.enable = true;
}
```

That's most of what you write. The dots are just nesting shorthand, so the above is the same as writing `hardware = { bluetooth = { enable = true; }; };` with fewer keystrokes.

The other half is functions. A function takes one argument and returns a value, and the argument is very often an attribute set that you destructure on the way in:

```nix
{ pkgs, ... }:
{
  home.packages = [ pkgs.signal-desktop ];
}
```

The `...` means "there may be more arguments than the ones I've named, and I don't care about them". You will type that a lot.

Lazy evaluation is the part that trips people up later rather than sooner. Nix doesn't evaluate anything until something actually needs the value. That's what makes it possible to describe an entire operating system as one enormous nested data structure without your machine catching fire, because the vast majority of it is never looked at.

And that's genuinely most of the language. There's no class hierarchy, no control flow to speak of, no loops. There's `let ... in` for local bindings, `if ... then ... else`, and a standard library called `lib` full of helpers. Everything else is attribute sets and functions all the way down.

The thing that took me longest to internalise is that Nix files aren't scripts. Nothing in them runs in order. You aren't telling the machine to do a sequence of things, you're describing a value, and something else decides what to do with it. Once that clicked, the rest got a lot easier.

## NixOS, the operating system

NixOS takes that idea and applies it to a whole system.

On a normal distribution, your machine is the sum of every command you've ever run on it. You installed a package eighteen months ago, edited a config file in `/etc`, ran a `curl | sh` you'd rather forget, and the current state of the box is the accumulated residue of all of it. Nobody knows what's in there, including you. Rebuilding it from scratch means remembering, and you won't.

On NixOS, the machine is a function of your configuration. You write down what you want, run a rebuild, and the system becomes that. Packages don't get installed into `/usr/bin`, they get built into the Nix store at `/nix/store` under a path that includes a hash of every input that went into them, and the system is assembled by symlinking the right ones into place.

That means:

- **Rollbacks are real.** Every rebuild produces a new generation, and the old one is still on disk. If a rebuild breaks your desktop you pick the previous generation from the boot menu and you're back where you were. Not "restored from backup", just booted into the closure that was already there.

- **Nothing conflicts.** Two applications wanting different versions of the same library is a non-problem, because they each reference their own store path. There is no shared `/usr/lib` to fight over.

- **The build happens before the switch.** `nixos-rebuild` builds the whole new system first, and only activates it if the build succeeded. A typo in your config is a failed evaluation on a machine that's still running fine, not a half-upgraded box at 11pm.

- **Two machines can genuinely be the same.** My desktop and my Surface Pro run the same user, the same desktop and the same applications, because they take the same modules. Not "roughly the same, I think I set them up the same way". The same.

The learning curve is steep and it's front-loaded. Doing something the NixOS way is often harder than the fifteen second version on Arch, right up until the moment you need to do it again on another machine, or a year later, or explain to yourself why you did it. Error messages can be genuinely awful, particularly infinite recursion, which will tell you almost nothing about where it came from. And there's a category of software that assumes it can write to its own installation directory, which simply won't work, and you'll need a workaround.

I stuck with it because I got tired of not knowing what was on my machines. Everything since has been a bonus.

## Using a framework

Here's the part I didn't expect. Plain NixOS gives you a module system, which is excellent, but it doesn't give you any opinion about how to organise a configuration across more than one machine and more than one user. That's left entirely to you, and there are as many answers as there are repos.

My first three configs were all attempts at that answer. The earlier ones are still on branches in the repo (`megadots-classic` and `megadots-dendritic`) if you want to see the working. Each one was a reasonable idea that grew a set of problems.

The recurring problem was always the same: NixOS configuration and Home Manager configuration live in two different worlds. Bluetooth is a system-level thing. Your Signal data directory is a user-level thing. Firefox is both, because it's a package in your home and a set of system fonts underneath it. Most repos end up with a `nixos/` tree and a `home/` tree, and every feature you add gets split in half across them. Wiring the halves back together is where the tangle comes from.

For megadots I use [den](https://github.com/denful/den), and it fixes precisely that. There are four ideas in it and they cover almost everything.

**An aspect** is a named, self-contained feature. It can carry a NixOS block, a Home Manager block, or both, and it's never split by which one it needs. Bluetooth is the whole file:

```nix
_: {
  den.aspects.bluetooth.nixos = _: {
    hardware.bluetooth.enable = true;
  };

  den.aspects.bluetooth.persist.system.directories = [ "/var/lib/bluetooth" ];
}
```

Turning Bluetooth on is a system option. Keeping the pairing database across reboots is a persistence concern. Both live in the file called `bluetooth.nix`, because both are Bluetooth. That's the entire pitch, and after three configs that split by class rather than by concern, it's a bigger deal than it looks.

**`includes`** is how anything opts in. A role is just an aspect that's nothing but a list of other aspects:

```nix
den.aspects.workstation.includes = [
  den.aspects.audio
  den.aspects.bluetooth
  den.aspects.graphics
  den.aspects.fonts
  den.aspects.networkmanager
];
```

And a host is a readable manifest of what it takes:

```nix
den.aspects.endgame = {
  includes = [
    den.aspects.base
    den.aspects.workstation
    den.aspects.gaming
    den.aspects.dev

    # The choices only this machine makes
    den.aspects.boot.lanzaboote
    den.aspects.gnome
    den.aspects.linux-kernel
  ];

  nixos.imports = [ ./_hardware.nix ];
};
```

That's my desktop, in full. Everything else about it is somewhere else, described once, and shared with the laptop.

**Quirks** are the clever bit, and the one that took me longest to appreciate. A quirk is a named channel that an aspect writes to without knowing who reads it. My Sunshine aspect needs some firewall ports open, so it says exactly that and nothing more:

```nix
firewall.tcp = [ 47984 47989 47990 48010 ];
```

It doesn't know what my network interface is called. It doesn't know that I only want those ports open on the LAN rather than everywhere, which is what the module's own `openFirewall` option would have done. One file, `networking.nix`, collects every declaration like that and turns them into interface-scoped rules. Add an application tomorrow and it declares its ports the same way, and nothing about the networking changes.

I use five of these. Ports, persisted paths, the theme, the Syncthing mesh and which terminal emulator to open a command in. The Syncthing one is my favourite, because the aspect that configures Syncthing has no idea how many machines I own. Every host announces its device ID into a pool, a policy gathers them up, and the aspect reads whatever it's handed. Adding a third machine to the mesh is adding it to the roster, and both existing machines learn about it on their next rebuild.

**`provides.to-users`** is the fourth, and it's the escape hatch for a host-level aspect that needs to hand something down to every user on that machine. GNOME uses it to deliver dconf settings.

That's the core of the framework. Four terms, and the rest of the repo is just files.

I'll add one caveat. I'd been using NixOS for around 18 months before I jumped in to den. With that time, I had a pretty solid hand-crafted classic flake configuration that I had become intimately familiar with. Starting with den would initially seem complex, but the quality of the documentation, and the rapidly growing exosystem of public repos make starting with den far more attractive now. I wouldn't hesitate to start here.

## What declarative gives you

It's easy to talk about this in the abstract, so here's what it looks like in practice on my machines.

**My root filesystem is deleted on every boot.** Both hosts run an ephemeral btrfs root. An initrd service deletes the root subvolume and restores it from a blank snapshot before anything else mounts. `/nix` and `/persist` survive, and everything else starts from nothing, every single time.

This sounds insane and it's the single best thing in the config. It means state is opt-in. If a directory isn't listed in an aspect's persist block, it does not exist tomorrow. No accumulated dotfile crap, no config file I edited by hand two years ago quietly influencing something, no wondering whether a fix worked or whether it just happens to be working. If it isn't declared, it isn't there. My home directory keeps `Documents`, `Downloads`, `Pictures`, `Videos` and `Music`, and each application aspect names the one directory it needs. That's the lot.

**A new machine is one command.** I run `nix run .#deploy flatmate` against a box booted from a NixOS installer, and [nixos-anywhere](https://github.com/nix-community/nixos-anywhere) partitions the disks with disko, copies my keys in from a USB stick, and installs the system. It comes up with its secrets decryptable, my SSH keys in place, my theme applied and my applications installed. There's no second pass and nothing to configure by hand afterwards.

That's the declarative nature at work. Not "I have notes on how I set this up", but the machine building itself from the repo.

**Changes are cheap to try and cheap to undo.** Wanting to try COSMIC instead of GNOME was a one-line change on a host. Going back to GNOME was the same one line. Because the desktop choice lives next to the bootloader choice on the machine that makes it, and not inside a shared role, neither host nor user configuration had to change at all. That's not a NixOS feature, it's a consequence of having somewhere sensible to put the decision.

**Consistency you get for free.** I set a colour scheme in exactly one place, my own user file:

```nix
theme = {
  scheme = "rose-pine-moon";
  wallpaper = ../../../assets/wallpaper/snake.png;
};
```

Stylix reads that at both the system and the session level, so the login screen, the TTY, the boot splash, GTK applications, my terminal, Firefox and my editor all match. One line changes all of it on both machines. I spent years doing that by hand across a dozen config files and getting it slightly wrong.

**Upgrades are boring.** `nix flake update`, then build, then look at the diff of what changed before I switch. If something's wrong I don't switch. If something's wrong after I switch, I reboot into the previous generation. I've never had to reinstall.

Declarative config turns a class of problem from "hope I remember" into "read the file". That's less exciting than it sounds and more valuable than it sounds. The system stopped being a thing I maintained and became a thing I described.

## Where to start

If any of this appeals, the resources that got me there:

- [NixOS and Flakes](https://nixos-and-flakes.thiscute.world/) by ryan4yin is the best introduction I've found, and it's free. If you read one thing, read this.
- [Misterio77's nix-starter-configs](https://github.com/Misterio77/nix-starter-configs) is the sanest starting skeleton, and his [own config](https://github.com/Misterio77/nix-config) shaped a lot of how I structure mine.
- [megadots](https://github.com/tomwrw/megadots) itself, if you want to see where this ends up. The README has a suggested reading order through the repo, and a section of known trade-offs so you can judge whether they'd suit you.

Fork it, take the bits you like, ignore the rest. Half of mine came from other people's repos, so it'd be rude not to.
