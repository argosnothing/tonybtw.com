---
title: "How to Install NixOS From Scratch | Flakes + Home Manager Full Guide"
author: ["Tony", "btw"]
description: "This is the 4th installment of the NixOS Tutorials. It emphasizes going from Zero to Hero and starting out with flakes and home manager before running nixos-install."
date: 2025-09-05
draft: false
image: "/img/nixos-from-scratch.png"
showTableOfContents: true
---

## Intro: {#intro}

What's up guys, my name is Tony, and today I'm going to give you a quick and painless guide
on daily driving NixOS from Scratch, with Flakes and Home manager.

As I've said before, NixOS really is the end-game of linux in many respects, due to its reproducibility, and it's stability in nature. Because of flakes, which allow you to pin your packages to a specific commit, and home manager, which enables you to version control and reproduce your user configuration, the combination of these powerful tools really make NixOS both rock solid like debian, and bleeding edge like Arch.

> "In many respects, if Arch, and Debian had a son, his name would be NixOS."
> – Tony

Let's jump into the tutorial.


## Install from Minimal ISO {#install-from-minimal-iso}

<https://nixos.org/manual/nixos/stable/index.html#ch-installation>

You can use the graphical installer, but I'm going to use the minimal iso today, because I'm like that.


### Load up Live ISO {#load-up-live-iso}

Let's start by installing NixOS. I've loaded up the minimal ISO, and I have already created a video on installing NixOS like this, so I'll run through this part pretty quick. Feel free to follow along the written guide in the link below the subscribe button for this part. (Fixed typo, thanks Tux for the reminder.)

This set of commands is going to dictate what hardware.nix looks like. If you've installed arch before, a lot of this stuf will look familiar to you. Let's get into it.

Let's switch into sudo here, and set up our partition scheme.

```nil
sudo -i
lsblk
cfdisk /dev/vda

gpt labels

1G type: EFI
4G type: swap
remaining space, type: Linux Filesystem
```

Alright, let's sanity check that with an lsblk to confirm, and there we go. We see our 3 partitions we just made. Let's make those file systems. (Edited, thanks Tux for the reminder.)

And the nixos handbook recommends we use labels for our partitions, so let's go ahead and do so.

```sh
mkfs.ext4 -L nixos /dev/vda3
mkswap -L swap /dev/vda2
mkfs.fat -F 32 -n boot /dev/vda1
```

Alright so we have made those file systems, its time to mount them like so:

```sh
mount /dev/vda3 /mnt
mount --mkdir /dev/vda1 /mnt/boot
swapon /dev/vda2
```

```sh
[root@nixos:~]# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0  1.5G  1 loop /nix/.ro-store
sr0     11:0    1  1.6G  0 rom  /iso
vda    253:0    0   50G  0 disk
├─vda1 253:1    0    1G  0 part /mnt/boot
├─vda2 253:2    0    4G  0 part [SWAP]
└─vda3 253:3    0   45G  0 part /mnt
```

Alright this looks good, and we're ready to move onto actually generating the config. Please note, I'm using vim here, but if you want to use a different editor, I would just run nixos-install right now, and rebooting, and then creating home.nix and flake.nix. If you're built different like me, just use vim.


### Initial NixOS Config {#initial-nixos-config}

[Official NixOS Installation Handbook](https://nixos.org/manual/nixos/stable/index.html#sec-installation-manual)

Alright, we're ready to generate the config file, so let's do so with the following command:

This part is going to be a lot of configuration and text editing, so if you want everything I've put into these files, feel free to follow along with my written article:
[NixOS from Scratch](https://www.tonybtw.com/tutorial/nixos-from-scratch)

```nil
nixos-generate-config --root /mnt
cd /mnt/etc/nixos/
touch flake.nix home.nix
```

Let's create flake.nix, and home.nix

(For those watching the video and are curious about my vimrc, I did import it to the live iso image.)

```vim
filetype plugin indent on
set expandtab
set shiftwidth=4
set softtabstop=4
set tabstop=4
set number
set relativenumber
set smartindent
set showmatch
set backspace=indent,eol,start
syntax on
```


### flake.nix {#flake-dot-nix}

Jumping into our flake.nix, this is where we define where our packages come from, so both configuration.nix and home.nix can inherit them through the flake and use them consistently.

Couple of things worth noting here:

1.  nixpkgs is shorthand for github:NixOS/nixpkgs/nixos-25.05
2.  inputs.nixpkgs.follows = "nixpkgs": This prevents home-manager from pulling its own version of nixpkgs, keeping everything consistent and avoiding mismatched package sets.
3.  This modules section tells our flake to build the system using configuration.nix, and to configure Home Manager for the tony user using home.nix, with some options set inline.
4.  We include home-manager as a NixOS module here because we want Home Manager to be managed by the flake itself — meaning we don’t need to bootstrap it separately, and we don’t need to run home-manager switch. Instead, everything gets applied in one go with nixos-rebuild switch.

vim flake.nix

```nix
{
  description = "NixOS from Scratch";

  inputs = {
    nixpkgs.url = "nixpkgs/nixos-25.05";
    home-manager = {
      url = "github:nix-community/home-manager/release-25.05";
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };

  outputs = { self, nixpkgs, home-manager, ... }: {
    nixosConfigurations.nixos-btw = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";
      modules = [
        ./configuration.nix
        home-manager.nixosModules.home-manager
        {
          home-manager = {
            useGlobalPkgs = true;
            useUserPackages = true;
            users.tony = import ./home.nix;
            backupFileExtension = "backup";
          };
        }
      ];
    };
  };
}
```

We're ready to move onto our configuration.nix file.


### configuration.nix {#configuration-dot-nix}

List

We're going to do a lot to this default config file, mainly just deleting comments and whatnot, but heres a list of stuff I'm going to do to it:

1.  Delete comments
2.  Change hostname
3.  Remove wpa supplicant (keep if you are on wifi)
4.  Change timezone
5.  Delete proxy settings
6.  Change xserver.enable to its own attribute set
    -   Enable \`ly\` display manager: [ly display manager](https://github.com/fairyglade/ly)
    -   Enable qtile
    -   Modify Key Repeat Settings: \`xset r rate 200 35 &amp;\`
7.  Delete rest of comments until User section
8.  Enable firefox at the system level
9.  Remove comments on systempackages, add git, and alacritty at the system level
10. Delete rest of comments, and then add nerd font package
11. add nix-command and flakes =)

<!--listend-->

```nix
{ config, lib, pkgs, ... }:

{
  imports =
    [
      ./hardware-configuration.nix
    ];

  boot.loader.systemd-boot.enable = true;
  boot.loader.efi.canTouchEfiVariables = true;

  networking.hostName = "nixos";
  networking.networkmanager.enable = true;

  time.timeZone = "America/Los_Angeles";

  services.displayManager.ly.enable = true;
  services.xserver = {
    enable = true;
    autoRepeatDelay = 200;
    autoRepeatInterval = 35;
    windowManager.qtile.enable = true;
  };

  users.users.tony = {
    isNormalUser = true;
    extraGroups = [ "wheel" ];
    packages = with pkgs; [
      tree
    ];
  };

  programs.firefox.enable = true;

  environment.systemPackages = with pkgs; [
    vim
    wget
    alacritty
    git
  ];

  fonts.packages = with pkgs; [
    nerd-fonts.jetbrains-mono
  ];

  nix.settings.experimental-features = [ "nix-command" "flakes" ];
  system.stateVersion = "25.05";

}
```


### home.nix {#home-dot-nix}

Let's set up our home.nix. we'll heavily modify this after installing nixos and logging in for the first time.

Just going to specify the home directory, enable git, and for a sanity check, let's setup a bash alias so we can confirm everything worked when we initially log in.

vim home.nix

```nix
{ config, pkgs, ... }:

{
  home.username = "tony";
  home.homeDirectory = "/home/tony";
  programs.git.enable = true;
  home.stateVersion = "25.05";
  programs.bash = {
    enable = true;
    shellAliases = {
      btw = "echo i use nixos, btw";
    };
  };
}
```


### Install: {#install}

Alright we're finally ready to install this. We can do that with this command here, to specify the location of the flake.

```sh
nixos-install --flake /mnt/etc/nixos#nixos-btw

## type your password
nixos-enter --root /mnt -c 'passwd tony'
reboot
```

Make sure to create this password otherwise you wont be able to log in

We can safely ignore this warning — it's just complaining that /boot is readable by all users, which is fine for us in this context.

Let's boot into our system!


## Post Install, First Boot {#post-install-first-boot}

Lets open up a terminal to confirm our home manager worked. Remember we put that bash alias in our home.nix file? Let's run it.
Super Enter opens a terminal by default in Qtile.

```sh
btw // I use nixos, btw
```


### Create Dotfiles Directory {#create-dotfiles-directory}

We're going to create our dotfiles directory in our home folder so we can handle all of our nixos files right here in our home folder. This will help for version controlling everything too.

```sh
mkdir ~/nixos-dotfiles
sudo cp -R /etc/nixos/* ~/nixos-dotfiles/.
mkdir ~/nixos-dotfiles/config
git clone https://github.com/tonybanters/qtile ~/nixos-dotfiles/config/qtile
vim ~/nixos-dotfiles/home.nix
```

Inside of home.nix, add this line:

```nix
home.file.".config/qtile".source = ./config/qtile;
```

This will tell home manager to look for this file for our qtile config.

Lets also grab our neovim config, so we can edit these files a bit easier.

```nix
git clone https://github.com/tonybanters/nvim ~/nixos-dotfiles/config/nvim
home.file.".config/nvim".source = ./config/nvim;
```

To get my nvim config to work, we need a few packages while we're in this file.

```nix
home.packages = with pkgs; [
  neovim
  ripgrep
  nil
  nixpkgs-fmt
  nodejs
  gcc
];
```

Alright, we're ready to rebuild our system for the first time. Let's run this command: It tells nixos to use a flake, and points to the flake directory, and specifies the hostname.
run:

```sh
sudo nixos-rebuild switch --flake ~/nixos-dotfiles#nixos-btw
```

Remember to use your hostname instead of \`nixos-btw\`

Alright that looks like it worked, lets confirm it with our refresh qtile keybind. Super Control R
Wow, incredible. it worked. That's my qtile config right there, inspired by DT's xmonad and qtile config.
Let's check out neovim, see if that worked.

Beautiful, we see lazy installing our plugins. Awesome.
Now our qtile config, and our neovim configs are good to go, but this isn't quite right yet.

This is suboptimal for 2 main reasons:

1.  we cant edit the config files because Root owns it
2.  we cant get live updates when we DO edit them with sudo because they are not symlinked to our actual directory.

Solution: **mkOutOfStoreSymlink**


### Modularize home manager directories with symlinks {#modularize-home-manager-directories-with-symlinks}

We can use mkOutOfStoreSymlink to tell nixos to create a symlink of our dotfiles right into ~/.config, instead of in _etc/nixos/store, or whatever.
Let's use xdg.configFile instead of home.file, because xdg.configFile is for dotfiles that are stored in ~_.config
We only need to use home.file if the file is stored somewhere else.
And let's specify recursive = true, so that nixos looks for the entire directory, with all subdirectories when creating the symlink.

phase 1

```nix
xdg.configFile."qtile" = {
    source = config.lib.file.mkOutOfStoreSymlink "/home/tony/nixos-dotfiles/config/qtile";
    recursive = true;
};
xdg.configFile."nvim" = {
    source = config.lib.file.mkOutOfStoreSymlink "/home/tony/nixos-dotfiles/config/nvim";
    recursive = true;
};
```

Alright, we can run sudo nixos-rebuild switch again, and we should be able to make live updates here.

```sh
ln -sf ~/nixos-dotfiles/config/
```

We see the symlinks are actually pointing to ~/.config this time.

Let's edit our qtile config again, and a hot reload should work. Awesome.

We can clean this up by declaring dotfiles, and creating a variable for mkOutOfStoreSymlink (its too verbose this way).

```nix
{ config, ... }

let
    dotfiles = "${config.home.homeDirectory}/nixos-dotfiles/config";
    create_symlink = path: config.lib.file.mkOutOfStoreSymlink path;
in

{
    xdg.configFile."qtile" = {
        source = create_symlink "${dotfiles}/qtile";
        recursive = true;
    };
    xdg.configFile."nvim" = {
        source = create_symlink "${dotfiles}/nvim";
        recursive = true;
    };

}
```


### Make it extensible with a for loop {#make-it-extensible-with-a-for-loop}

And one more step would be to just make a for loop to iterate over a list of configs, and that way we can remove code duplication.

```nix
{ config, ... }

let
    dotfiles = "${config.home.homeDirectory}/nixos-dotfiles/config";
    create_symlink = path: config.lib.file.mkOutOfStoreSymlink path;
    # Standard .config/directory
    configs = {
        qtile = "qtile";
        nvim = "nvim";
    };
in

{
    # Iterate over xdg configs and map them accordingly
    xdg.configFile = builtins.mapAttrs (name: subpath: {
        source = create_symlink "${dotfiles}/${subpath}";
        recursive = true;
    }) configs;

}
```

This will make it much easier to add dotfile configs to our system. With this forloop we can actually just clone our entire dotfiles repo and throw it in here.


## Turning Neovim into it's own .nix file (the right way) {#turning-neovim-into-it-s-own-dot-nix-file--the-right-way}

If we take Neovim out of this giant home.packages list, we can move it into its own file, and put all of its dependencies in that file as well, to keep things organized. This is beneficial for 2 reasons.

1.  If someone just wants my neovim, they can take neovim.nix and my neovim config, and be good to go.
2.  If we need to add lsps, or specific packages for just neovim, but we aren't using neovim on a different machine, we dont need to import the entire neovim suite.
3.  Scales: you can drop in more modules later (emacs.nix, alacritty.nix, etc.) without cluttering home.nix.

Let's do this by making a folder called \`modules\` and putting \`neovim.nix\` in that folder.

The structure will look like so:

```nil
├── home.nix
└── modules
    └── neovim.nix
```

Inside of that neovim.nix file, we can do the following:

```nix
{ config, pkgs, lib, ...}

{
  # Install Neovim and dependencies
  home.packages = with pkgs; [
    # Tools required for Telescope
    ripgrep
    fd
    fzf

    # Language Servers
    lua-language-server
    nil # nix language server
    nixpkgs-fmt # nix formatter

    # Needed for lazy.nvim
    nodejs
  ];

  programs.neovim = {
    enable = true;
    viAlias = true;
    vimAlias = true;

    # optional: If you want to manage your plugins with nix, instead of with lazy.nvim,
    # you can do it with the plugins key.
    # plugins = with pkgs.vimPlugins; [
    #     telescope-nvim
    #     nvim-treesitter
    #     nvim-lspconfig
    #     # Any other packages you want pinned at.
    # ];
  };

}
```

Then in our home.nix, we would just import this like so:

```nix
{pkgs, config, lib ...}:

{
  imports = [
    ./modules/neovim.nix
  ];
  ...
}

```


## Final Thoughts {#final-thoughts}

In the next installment of "NixOS From Scratch", we'll create 2 package variables, and use unstable to update stuff that we don't care if it breaks our system, and lock the core utils to stable.

This has been heavily 'programming oriented', but this will leapfrog you forward in your nixos journey. Let me know if you like this sort of content, or what else you would like to see.

Thanks so much for checking out this tutorial. If you got value from it, and you want to find more tutorials like this, check out
my youtube channel here: [YouTube](https://youtube.com/@tony-btw), or my website here: [tony,btw](https://www.tonybtw.com)

You can support me here: [kofi](https://ko-fi.com/tonybtw)
