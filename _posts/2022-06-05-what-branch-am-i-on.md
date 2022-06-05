---
layout: post
title: What branch am I on?
date: 2022-06-05 08:25 +0200
---

Minimal is best, without installing oh-my-zsh or zpresto you can still display the current git branch in your terminal.

Add the following to your `.zshrc`

```zsh
# Load version control information
autoload -Uz vcs_info
precmd() { vcs_info }

# Format the vcs_info_msg_0_ variable
zstyle ':vcs_info:git:*' formats ':: %b'

# Set up the prompt (with git branch name)
setopt PROMPT_SUBST
PROMPT='${PWD/#$HOME/~} ${vcs_info_msg_0_} > '
```


