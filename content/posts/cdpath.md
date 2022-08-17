---
title: "CDPATH"
date: 2022-08-17T11:26:55-04:00
draft: true
toc: false
images:
tags: [shell]
categories: [programming]
---

## Extra considerations for ZSH users

As a ZSH user there are some additional settings you can add to your `.zshrc`
to take this feature even further.

First up is `AUTO_CD`. With `AUTO_CD` enabled any command that you type that is
not an executable command but is a valid directory will cause you to `cd` into
that directory. This even includes the names of directories in the `CDPATH`

```zsh
setopt AUTO_CD
```

There is also a more zshy way of defining the `CDPATH` if you prefer it. The
paths of the `CDPATH` can be passed as an array. I like it in this format
because I often make changes to my `CDPATH` and it is easier to see the
segments when they are space separated instead of `:` separated.

```zsh
cdpath=(. $HOME $HOME/.config $HOME/Code)
```

## References

* `$ man zshoptions`
* `$ man zshall`
