# pmset-session

`pmset-session` is meant to be used with `.zprofile` and `.zshenv` in order to
automatically set power management options while you are connected to macOS via
`ssh`.


# Installation

Put `pmset-session` in your `$PATH`. If you're going to be using
`pmset-session` in `.zprofile` like recommended below, the directory it's in
must be added to `$PATH` before the relevant code in `.zprofile` executes, e.g.
by adding it to `.zshenv`.

You almost certainly want to make it possible for your user to `sudo pmset`
without a password prompt. In order to do this, run the following code in your
terminal, replacing `joe` with your user name:

```zsh
sudo zsh -c 'echo "joe ALL=(ALL) NOPASSWD: /usr/bin/pmset" > /private/etc/sudoers.d/pmset'
```


# Usage

Use the following code snippets or take the code from the files `dotzshenv` and
`dotzprofile` in this repository.


Add this code to `~/.zshenv`:

```zsh
# Add ~/bin to $PATH. Note we still have to do it in .zshrc because this
# addition will be at the end of $PATH once all dotfiles are done being read.
PATH="$HOME/bin:$PATH"
```


Add this code to `~/.zprofile`. Note that it also integrates `caffeinate`, but
you don't need it to use `pmset-session`:

```zsh
# Only execute if logging in via ssh into a mac
# The first part checks if the parent process of this shell is sshd. This way
# shells spawned in the descendant tree won't be affected.
if [[ "$(ps -o comm= $PPID)" =~ ^sshd:\  ]] && [[ "$(uname)" == "Darwin" ]] && [ -z "$ANTISLEEP" ]; then

  # We must run $(caffeinate > /dev/null &) because it is the only version of
  # spawning caffeinate in the background that starts up quietly, doesn't
  # prevent you from quitting the shell, and exits automatically once you quit.
  #
  # If you use caffeinate >/dev/null &, you get the following output:
  #
  # Last login: Sun Sep 15 00:43:24 2024 from 1.2.3.4
  # [2] 57056
  # ~ %
  # zsh: you have running jobs.
  # ~ %
  # zsh: warning: 1 jobs SIGHUPed
  # [2]  + hangup     caffeinate > /dev/null
  # Connection to 4.3.2.1 closed.
  # 
  # If you use (caffeinate >/dev/null &), then there is no garbage output like
  # above, and you're not stopped from exiting the shell, but caffeinate is not
  # sent a SIGHUP and therefore keeps running forever, even after the shell
  # exits.
  #
  # We use true to ignore what the $() interpolates to, otherwise it would get
  # interpolated as a command. While we make sure that caffeinate does not
  # produce any output, it's best to make double-sure no output gets
  # interpolated as a command.

  true $(caffeinate >/dev/null &)

  # However, caffeinate doesn't work when the lid is closed. Instead, we'll use
  # pmset-session which is pmset with a lockfile mechanism.

  # Note: you need pmset-session to be in $PATH, and .zshrc hasn't been parsed
  # yet. Therefore, if pmset-session is in e.g. $HOME/bin, add that to $PATH in
  # .zshenv.
  
  pmset_restore () {
    pmset-session --end-session $$ # Pass the current shell's PID stored in $$
  }
  trap pmset_restore SIGINT SIGTERM EXIT # do pmset_restore if shell exits/dies
  pmset-session --start-session $$

  export ANTISLEEP=1
  fi
```

You can use `pmset-session -c` from the command line to clean up PID files in
case e.g. your computer ever crashes or loses power. Note that this will not
include PID files where the process ID is now being used by some other,
unrelated process. You can use `pmset-session -o $$` to remove all PID files
and start a session only for your current shell.
