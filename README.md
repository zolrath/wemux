# wemux: Multi-User Tmux Sessions Made Easy
********************************************************************************

wemux allows you to start hosting a multi-user tmux session using the command
`wemux`.

Clients (generally someone SSHed into another account on the host machine) have
the option of:

**Mirroring** gives clients read-only access to the session, allowing them to
see you work, or

**Pairing** allows the client and yourself to work in the same terminal (shared
cursor) or work independently in another window (separate cursors) in the same
tmux session.

## How To Install
  The rest of this readme will operate under the assumption you'll place wemux
  in `wemux/` in your `/usr/local/share` directory. To make wemux available for
  all users, perform the following steps, using sudo as required:

  Git clone this repo.

    git clone git://github.com/zolrath/wemux.git /usr/local/share/wemux

  Symlink the `wemux` file into your $PATH such as `/usr/local/bin/`,
  being sure to use the full path.

    ln -s /usr/local/share/wemux/wemux /usr/local/bin/wemux

  **IMPORTANT**: Copy the `wemux.conf.example` file to `/etc/wemux.conf`

    cp /usr/local/share/wemux/wemux.conf.example /etc/wemux.conf

  Then set a user to be a wemux host by adding their username to the host_list in
  `/etc/wemux.conf`. Users in the host_list will be able to start new wemux
  sessions, all other users will be wemux clients and join these sessions.

    host_list=(zolrath brocksamson)

## Host Commands
#### wemux start
  Use `wemux start` to start a wemux session, chmod /tmp/wemux-host to 1777 so
  that other users may connect to it, and attach to it.  If a wemux session
  already exists, it will attach to it instead.
#### wemux attach
  Use `wemux attach` to attach to an existing wemux session.
#### wemux stop
  Use `wemux stop` to kill the wemux session and remove the /tmp/wemux-host
  socket.
#### wemux kick *username*
  Use `wemux kick <username>` to kick an SSH user from the server and remove
  their wemux pair sessions.
#### wemux config
  Use `wemux config` to open `/etc/wemux.conf` in your $EDITOR.
  Note this only works if you have the environment variable EDITOR configured.
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
  * By setting `default_client_mode="pair"` in `wemux.conf` this can be changed
  to always join in pair mode, even if a pair session doesn't already exist.

#### Other Commands
  wemux passes commands it doesn't understand through to tmux with the correct
  socket setting.

  `wemux list-sessions` is equivalent to entering `tmux -S /tmp/wemux-host
  list-sessions`

## User List
  wemux can display a list of connected users, indicating users in mirror mode
  with [m] at the end of their name.

  If you'd like to see a list of all users currently connected to the wemux
  session, you have three options:

### wemux users
  Enter `wemux users` in the terminal to display a list of all currently
  connected wemux users.

    $ wemux users
    Users connected to 'host':
      1. furd
      2. csagan[m]

### Status Bar
  You can add the user list to your status bar by adding #(wemux status_users)
  where you see fit by editing your `~/tmux.conf` file.

    set -g status-right "#(wemux status_users)"

### Display Message
  If you'd rather display users on command via a tmux message, similar to the
  user attachment/detachment messages, by editing your `~/tmux.conf` file.
  Pick whatever key you'd like to bind it to. Using t as an example:

    unbind t
    bind t run-shell 'wemux display_users'

  Note that the tmux prefix should be pressed before t to activate the command.

  User listing can be disabled by setting `allow_user_list="false"` in
  `wemux.conf`

### Short-form Commands
  All commands have a short form. s for start, a for attach, p for pair etc.
  For a complete list, type `wemux help` (or `wemux h`)

# Multi-Host Capabilities
********************************************************************************
  wemux supports specifying the joining different wemux sessions via `wemux join
  <sessionname>`. This allows multiple hosts on the same machine to host their own
  independent wemux sessions with their own clients. By default this option is
  disabled.

  wemux will remember the last session specified to in order to make reconnecting
  to the same session easy. `wemux help` will output the currently specified
  session along with the wemux command list.

  Changing sessions can be enabled by setting `allow_session_change="true"` in
  `/etc/wemux.conf`

### Joining Different wemux Sessions
  To change the wemux session run `wemux join <session>`

    $ wemux join Project X
    Changed wemux session from 'host' to 'project-x'
    $ wemux start
    $ wemux
    $ wemux stop
    $ wemux reset
    Changed wemux session from 'project-x' to 'host'
#### wemux join *sessionname*
    Join wemux session with specified name.

### Resetting the Session Name
  In order to easily return to the default session you can run `wemux reset`
#### wemux reset
  Joins the default wemux session: host

### Active Session List
  To list the name of all currently running wemux sessions run `wemux list`
#### wemux list
  List all currently active wemux sessions.

    $ wemux list
    Currently active wemux sessions:
      1. project-x
      2. host    <- current session
      3. dont-get-lost

  Listing sessions can be disabled by setting `allow_session_list="false"` in
  `/etc/wemux.conf`

# Configuration
********************************************************************************
  There are a number of additional options that be configured in
  `/etc/wemux.conf`.  In most cases the only option that must be changed is the
  `host_list` array. To open your wemux configuration file, you can either open
  `/etc/wemux.conf` manually or run `wemux config`

### Host Mode
  To have an account act as host, ensure that you have added their username to the
  `/etc/wemux.conf` file's `host_list` array.

    host_list=(zolrath hostusername brocksamson)

### Pair Mode
  Pair mode can be disabled, only allowing clients to attach to the session in
  mirror mode by setting `allow_pair_mode="false"`

### Default Client Mode
 When clients enter 'wemux' with no arguments by default it will first attempt to
 join an existing pair mode session. If there is no pair session it will start
 a mirror mode session.
 By setting default_client_mode to "pair", 'wemux' with no arguments will always
 join a pair mode session, even if it has to create it.

  This can be changed by setting `default_client_mode="pair"`

### Changing Sessions
  The ability to change sessions can be enabled by setting
  `allow_session_change="true"`

### Listing Sessions
  Listing sessions can be disabled by setting `allow_session_list="false"`

### Listing Users
  Listing users can be disabled by setting `allow_user_list="false"` in
  `wemux.conf`

### Kicking SSH Users
  Kicking SSH users from the server can be disabled by setting
  `allow_kick_user="false"` in `wemux.conf`

### Announcements
  When a user joins a session in either mirror or pair mode, a message is
  displayed to all currently attached users:

    csagan has attached in mirror mode.
    csagan has detached.

 This can be disabled by setting `announce_attach="false"`

 In addition, when a user switches from one session to another via the `wemux
 join <sessionname>` command, their movement is displayed similarly to the
 attach messages.

  If csagan enters `wemux join applepie` the users on the default session
  `host` will see:

    csagan has switched to session: applepie

  If csagan returns to default session with: `wemux reset` users on `host`
  will see:

    csagan has joined this session.

  This can be disabled by setting `announce_session_change="false"`

### Automatic SSH Client Modes
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
