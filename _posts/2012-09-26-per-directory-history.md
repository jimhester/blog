---
layout: post
title: Per Directory, Cross Server History in ZSH
tags: R
comments: true
---

I do most of my work sshing into linux boxes on the command line.  In this 
environment having a nice history of previous commands is of enormous benefit.
I also use [tmux][tmux] terminal multiplexer to have multiple persistent
terminal windows.  Using bash's default history with this setup is an excessive
in frustration.  All of the terminal windows have their own history, and the 
history is appended/overwritten when a given window is exited, so your history 
can get hopelessly confused based on the order in which you close your terminal
sessions.  [zsh][zsh] has two options which remedy that specific situation
however.

```zsh
setopt inc_append_history
setopt share_history
```

This allows history items from different shells to be interleaved properly, and
also shares the history between sessions so that the commands you type in one
window will be in the other windows history as well.

A separate feature which is very nice to have is a history for the current
directory, as often I am working on a number of different things at once, so 
rather than trying to search through a number of non-relevant commands, I have 
the last command that I executed in the directory I am currently working in.
There are a couple of implementations of this online for bash [1][bash1],[2][bash2],
but no complete solutions that I could find for zsh.  In addition I added the 
feature of being able to toggle between the global history and the current 
directory history on the fly, which has not been present in any of the other 
implementations I have seen. The default key binding for this is ctrl-g.

The implementation of the per directory history has been added to the oh-my-zsh
project as a plugin [per-directory-history][pdh].  If you are using oh-my-zsh,
simply add "per-directory-history" to your plugins, otherwise simply source the
plugin file directly and you will get the per directory history.

The other interesting thing I set up in my personal configuration is a cross-server
history.  This was easy to do in my case because our servers all share a networked
file system, so I simply stored the global and local histories on the file system
by setting HISTORY_BASE and HISTFILE to be on the shared filesystem.  This allows
all history to be shared regardless of what server it is on.

Another way you could set the same type of thing up is by using cloud storage, 
such as [dropbox][dropbox].  You would simply do the same thing as above, but 
point the history file to the cloud storage.  The only issue you may have is waiting
to allow the history to sync between servers when you switch servers, as otherwise
the two history files will become conflicted.

[tmux]: http://tmux.sourceforge.net
[zsh]: http://www.zsh.org
[bash1]: http://www.compbiome.com/2010/07/bash-per-directory-bash-history.html
[bash2]: http://dieter.plaetinck.be/per_directory_bash
[pdh]: https://github.com/robbyrussell/oh-my-zsh/blob/master/plugins/per-directory-history/per-directory-history.plugin.zsh
[dropbox]: https://www.dropbox.com/
