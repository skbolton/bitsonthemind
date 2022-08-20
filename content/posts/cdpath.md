---
title: Setting the CDPATH
date: 2022-08-17T11:26:55-04:00
draft: false
toc: false
images:
tags: [shell]
categories: [programming]
---

I wonder on a given day how many different directories I cd into... The number
is probably quite high. Changing into directories is one of the first things we
learn when working from the command line. I think its natural to do it without
really questioning it. But the fact of the matter is that its an action worth
optimizing.

Here is the thing, out of all the directories we cd in and out of most of them
are mostly likely common directories that we probably work with a lot. It would
be nice if there was a way to jump to them more quickly. Sure we could define an
alias that takes us right to a directory but unix has a better solution. Enter
the `CDPATH`.

If you are familiar with the `$PATH` variable you can probably already guess where
this is going. The `$CDPATH` variable is a `$PATH` like variable of paths that we
want to be able to cd into, from anywhere. Say that your `~/.config` folder is one
of the common folders you jump to often.

```sh
$ pwd
/home/skbolton/Code/meaning-of-life
$ cd .config
/home/skbolton/.config
```

Bam you are there!

## Setting the CDPATH

`CDPATH` is set as an environment variable. So load up your rc file and add an
entry.

```sh
CDPATH="/home/skbolton/.confg:/home/skbolton/Code:/home/skbolton"
```

Some words of warning on this though. Having scripts being able to cd into a 
directory on your cdpath is probably a bad idea. You most likely want a script
to fail if it can't find a directory it expects to be in a certain spot. Last
thing you want is the script going to one of your `CDPATH` directories and
deleting files. For this reason you probably want to put that variable inside
of interactive terminal sessions only

* `.zshrc` instead of `.zshenv`
* `.bashrc` instead of `.bash_profile`

This is also why above I showed not exporting the variable. I only want it to be
a variable in my current shell session, not any of the subshells I spawn.

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
