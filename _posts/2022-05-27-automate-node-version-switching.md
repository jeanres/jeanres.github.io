---
layout: post
title: Damn! I am using the wrong version of node
date: 2022-05-27 11:12 +0200
---

We all have projects that run require different node versions, and most of us already use nvm. I use this function in my `.zshrc` to switch to the correct version of node by reading the `.nvmrc` of the directory I am in.

```zsh
function change_node_version {
  nvmrc="./.nvmrc"
  if [ -f "$nvmrc" ]; then
    version="$(cat "$nvmrc")"
    nvm use $version
  fi
}

chpwd_functions=(change_node_version)
```
