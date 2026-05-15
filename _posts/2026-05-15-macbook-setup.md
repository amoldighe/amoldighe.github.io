---
layout: post
title:  "Macbook Terminal Setup"
date:   2026-05-15
tags:
  - Mac
  - Terminal
  - zsh
  - ohmyzsh
  - starship
  - carapace
  - fzf
  - tmux
  - brew
  - iterm2
  - git
---

Recently I setup a new MacBook Pro, this gave me a chance to revist my command line setup and the necessary tools I use on a daily basis to speed up my development workflow. 

### Install Homebrew

[Homebrew](https://brew.sh/) allow you to install software on your Mac using the command line.

```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### Install iTerm2

iTerm2 is a terminal emulator for macOS. It is a replacement for the default terminal application.

```
brew install --cask iterm2
```

### Install git

Git is a version control system for tracking changes in source code during software development. Git is pre installed on macOS. You can verify it by running `git --version` command.

### Install oh my zsh

[Oh My Zsh](https://ohmyz.sh/#install) is an open-source framework for managing your Zsh configuration. It includes a collection of community-maintained plugins and themes.

```
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

**Useful Plugins for Oh My Zsh**

- zsh-autosuggestions
https://github.com/zsh-users/zsh-autosuggestions/blob/master/INSTALL.md

```
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions

```

- zsh-syntax-highlighting
https://github.com/zsh-users/zsh-syntax-highlighting/blob/master/INSTALL.md

```
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

- web-search

Edit file `~/.zshrc` and add plugins to the line with `plugins=(...)`.

```
plugins=(git web-search zsh-autosuggestions zsh-syntax-highlighting)
```

`source .zshrc` to activate the plugins

Homebrew formulae are also available to install the plugins

```
brew install zsh-autosuggestions
brew install zsh-syntax-highlighting
```
I prefer the above `git clone` + `~/.zshrc` approch

### Install starship prompt

```
brew install starship
```

I will be using an existing preset of startship prompt [https://starship.rs/presets/tokyo-night](https://starship.rs/presets/tokyo-night) which needs Nerd font to be installed. List of all the Nerd fonts can be found here: [https://www.nerdfonts.com/font-downloads](https://www.nerdfonts.com/font-downloads)

Here's a quick command to list all the available nerd fonts and install 

```
brew search '/font-.*-nerd-font/' 
brew install --cask <font-name>
```

Example for installing JetBrains Mono Nerd Font

```
brew install --cask font-jetbrains-mono-nerd-font
```
To enable this font in iterm2 follow these steps:

- Open iterm2
- Go to Preferences -> Profiles -> Text
- Click on "Change Font" and select "JetBrains Mono Nerd Font"

I like Tokyo Night theme for starship prompt. You can find more presets here: [https://starship.rs/presets/](https://starship.rs/presets/) 
Use starship to import the preset for [https://starship.rs/presets/tokyo-night](https://starship.rs/presets/tokyo-night)

```
starship preset tokyo-night -o ~/.config/starship.toml
```

### Install carapace

[Carapace](https://carapace.sh/) is a shell completion generator for commands. It allows you to generate completion scripts for your shell.

```
brew install carapace
```

Setup carapace to work with zsh - [https://carapace-sh.github.io/carapace-bin/setup.html#zsh](https://carapace-sh.github.io/carapace-bin/setup.html#zsh)

Edit `.zshrc` file and add the following lines:

```
autoload -U compinit && compinit
export CARAPACE_BRIDGES='zsh,fish,bash,inshellisense' # optional
zstyle ':completion:*' format $'\e[2;37mCompleting %d\e[m'
source <(carapace _carapace)
```

`source .zshrc` to activate the changes & enjoy the carapace autocompletion feature for all the tools you installed using brew (and lots of other tools).

### Install fzf

[fzf](https://github.com/junegunn/fzf) is a general-purpose command-line fuzzy finder.

```
brew install fzf
```

### Install tmux 

[tmux](https://github.com/tmux/tmux) is a terminal multiplexer. It lets you switch easily between several programs in one terminal, detach them (they keep running in the background) and reattach them to a different terminal.

```
brew install tmux
```

This one is more useful on production servers. But good to have it installed locally on Mac as well.


