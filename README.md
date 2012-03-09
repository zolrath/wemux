# wemux: Multi-User Tmux Sessions Made Easy
********************************************************************************

wemux allows you to start hosting a multi-user tmux session using the command
`wemux`.

Clients (generally someone SSHed into another account on the host machine) have
the option of:

**Mirroring** gives clients read-only access to the session, allowing them to
see you work, or

**Pairing** allows the client and yourself to work in the same terminal (shared
    cursor) or work independently in another window (separate cursors) in the
    same tmux session.

## How To Install
  Git clone this repo to your desired location. We'll assume that's .wemux in your user home directory.

    git clone git://github.com/zolrath/wemux.git ~/.wemux

  On OSX use wemux, on Linux machines use wemux-linux.
  wemux-linux removes stderr redirection until tmux/epoll error is fixed.

  Move or symlink the `wemux` file into your $PATH such as `/usr/local/bin/`,
  being sure to use the full path if creating a symlink.

    ln -s /Users/YOUR_USER_NAME/.wemux/wemux /usr/local/bin/wemux

  or on Linux machines:

    ln -s /home/YOUR_USER_NAME/.wemux/wemux-linux /usr/local/bin/wemux

  **IMPORTANT**: Copy the wemux.conf.example file to /etc/wemux.conf

    sudo cp ~/.wemux/wemux.conf.example /etc/wemux.conf

  To set a user as host add their username to the host_list in /etc/wemux.conf

    host_list=(zolrath csagan brocksampson)


## Host Commands
#### wemux start
  Use `wemux start` to start a wemux session, chmod /tmp/wemux to 1777 and
  attach to it.  If a wemux session already exists, it will attach to it
  instead.
#### wemux attach
  Use `wemux attach` to join an existing wemux session.
#### wemux stop
  Use `wemux stop` to kill the wemux session and remove the /tmp/wemux session
  file.
#### wemux
  When `wemux` is run without any arguments in host mode, it is just like
  running wemux start.  It will reattach to an existing wemux session if it
  exists, otherwise it will start a new session.

## Client Commands
  All client commands notify the wemux session host that the user has
  attached/detached, and in what mode.
#### wemux mirror
  Use `wemux mirror` to attach to Host in read-only mode.
#### wemux pair
  Use `wemux pair` to attach to Host in pair mode, which allows editing.
#### wemux logout
  Use `wemux logout` to remove your pair mode session.
#### wemux
  When `wemux` is run without any arguments in client mode, its behavior
  attempts to intelligently select mirror or pair:

  * If the client does not have an existing pair session it will attach to the
  wemux session in mirror mode.
  * If the client has already started a wemux pair mode session, it will
  reattach to the session in pair mode.

#### Other Commands
  wemux passes commands it doesn't understand through to tmux with the correct
  socket setting.

  `wemux list-sessions` is the same as entering `tmux -S /tmp/wemux
  list-sessions`

## Short-form Commands
  All commands have a short form. s for start, a for attach, p for pair etc.
  For a complete list, type `wemux help` (or `wemux h`)

## Multi-Host Capabilities
  wemux supports passing a second argument to its commands to specify a specific
  session hostname. This allows multiple hosts on the same machine to host their
  own independent wemux sessions with their own clients.

  This is not needed for most use cases.

  wemux will remember the last host attached to in order to make reconnecting to
  the same hostname easy. `wemux help` will output the currently specified
  hostname along with the wemux command list.

### Specifying Hostname
  All the commands you know and love will accept an optional second argument of
  the hostname you would like to start/attach/mirror/pair to.

    wemux start ProjectX
    wemux attach ProjectX
    wemux stop ProjectX
    wemux mirror ProjectX
    wemux pair ProjectX
    wemux logout ProjectX
    wemux reset

### Resetting the Hostname
  In order to easily return to the default hostname you can run `wemux reset`
#### wemux reset
  Resets the wemux hostname to the default: host

## Configuration
### Host Mode
To have an account act as host, ensure that you have added their username to the
/etc/wemux.conf file's host_list array.

    host_list=(zolrath hostusername csagan brocksampson)

### Client Modes

To make an SSHed user start in a wemux mode automatically, add one of the
following lines to the users `.bash_profile` or `.zshrc`

**Option 1**: Automatically log the client into mirror mode upon login,
  disconnect them from the server when they detach.

    wemux mirror; exit

**Option 2**: Automatically start the client in mirror mode but allow them to
detach.

    wemux mirror

**Option 3**: Automatically start the client in pair mode but allow them to
detach.

    wemux pair

**Option 4**: Only display the connection commands, don't automatically start
any modes.

    wemux help

Please note that this does not ensure a logged in user will not be able to exit
tmux and access their shell. If the user is not trusted, you must perform any
security measures one would normally perform for a remote user.
