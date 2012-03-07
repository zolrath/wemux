# wemux: Multi-User Tmux Sessions Made Easy
***
wemux allows you to start hosting a multi-user tmux session using the command `wemux`.

Clients (generally someone SSHed into another account on the host machine) have the option of:

**Mirroring** gives clients read-only access to the session, allowing them to see you work, or

**Pairing** allows the client and yourself to work in the same terminal (shared cursor)
  or work independently in another window (separate cursors) in the same tmux session.

## How To Install
  Git clone this repo to your desired location.

    git clone git://github.com/zolrath/wemux.git ~/.wemux

  Move or symlink the `wemux` file to the path such as `/usr/local/bin/`, being sure
  to use the full path if creating a symlink.

    ln -s /Users/YOUR_USER_NAME/.wemux/wemux /usr/local/bin/wemux

  Add the following to the host accounts `.bash_profile` or `.zshrc`

    export WEMUX_HOST = true

## Host Commands
#### wemux start
  Use `wemux start` to start a wemux session, chmod /tmp/wemux to 1777 and attach to it.
  If a wemux session already exists, it will connect to it instead.
#### wemux stop
  Use `wemux stop` to kill the wemux session and remove the /tmp/wemux session file.
#### wemux
  When `wemux` is run without any arguments in host mode, it is just like running wemux start.
  It will reattach to an existing wemux session if it exists, otherwise it will start a new session.

## Client Commands
  All client commands notify the wemux session host that the user has attached, and in what mode.
#### wemux mirror
  Use `wemux mirror` to attach to Host in read-only mode.
#### wemux pair
  Use `wemux pair` to attach to Host in pair mode, which allows editing.
#### wemux
  When `wemux` is run without any arguments in client mode, its behavior attempts to intelligently select mirror or pair:

  * If the client does not have an existing pair session it will attach to the wemux session in mirror mode.
  * If the client has already started a wemux pair mode session, it will reattach to the session in pair mode.

## Configuration
### Host Mode
To have an account act as host, ensure that you have added the following line to the `.bash_profile` or `.zshrc`

    export WEMUX_HOST = true

### Client Modes

To make an SSHed user start in a wemux mode automatically, add one of the following lines
to the users `.bash_profile` or `.zshrc`

**Option 1**: Automatically log the client into mirror mode upon login, disconnect them from the server when they detach.

    wemux mirror; exit

**Option 2**: Automatically start the client in mirror mode but allow them to detach.

    wemux mirror

**Option 3**: Automatically start the client in pair mode but allow them to detach.

    wemux pair

**Option 4**: Only display the connection commands, don't automatically start any modes.

    wemux help

Please note that this does not ensure a logged in user will not be able to exit tmux and access their
shell. If the user is not trusted, you must perform any security measures one would normally perform for a remote user.
