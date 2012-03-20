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

  To set a user as host add their username to the host_list in `/etc/wemux.conf`

    host_list=(zolrath csagan brocksampson)

## Host Commands
#### wemux start
  Use `wemux start` to start a wemux session, chmod /tmp/wemux-host to 1777 and
  attach to it.  If a wemux session already exists, it will attach to it
  instead.
#### wemux attach
  Use `wemux attach` to join an existing wemux session.
#### wemux stop
  Use `wemux stop` to kill the wemux session and remove the /tmp/wemux-host
  socket.
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

### Short-form Commands
  All commands have a short form. s for start, a for attach, p for pair etc.
  For a complete list, type `wemux help` (or `wemux h`)

# Multi-Host Capabilities
********************************************************************************
  wemux supports specifying the wemux session hostname via `wemux name
  <hostname>`. This allows multiple hosts on the same machine to host their own
  independent wemux sessions with their own clients. By default this option is
  disabled.

  wemux will remember the last host specified to in order to make reconnecting
  to the same hostname easy. `wemux help` will output the currently specified
  hostname along with the wemux command list.

  Changing hostnames can be enabled by setting `allow_host_change="true"` in
  `/etc/wemux.conf`

### Specifying Hostname
  To change the wemux hostname run `wemux name <hostname>`

    wemux name ProjectX
    # wemux hostname is now ProjectX
    wemux start
    wemux
    wemux stop
    wemux reset
    # wemux hostname is now host

#### wemux name *hostname*
    Changes wemux hostname to specified name.

### Resetting the Hostname
  In order to easily return to the default hostname you can run `wemux reset`
#### wemux reset
  Resets the wemux hostname to the default: host

### Active Hostname List
  To list the hostname of all currently running wemux servers run `wemux list`
#### wemux list
  List all currently active wemux hosts.

    wemux list
    Currently active wemux hosts:

    1. ProjectX
    2. dont-get-lost
    3. host

  Listing hostnames can be disabled by setting `allow_host_list="false"` in
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

    host_list=(zolrath hostusername csagan brocksampson)

### Pair Mode
  Pair mode can be disabled, only allowing clients to attach to the server in
  mirror mode by setting `allow_pair_mode="false"`

### Default Client Mode
 When clients enter 'wemux' with no arguments by default it will first attempt to
 join an existing pair mode session. If there is no pair session it will start
 a mirror mode session.
 By setting default_client_mode to "pair", 'wemux' with no arguments will always
 join a pair mode session, even if it has to create it.

  This can be changed by setting `default_client_mode="pair"`

### Changing Hostnames

  The ability to change hostnames can be enabled by setting
  `allow_host_change="true"`

### Listing Hostnames

  Listing hostnames can be disabled by setting `allow_host_list="false"`

### Announcements
  When a user joins a session in either mirror or pair mode, a message is
  displayed to all currently attached users:

    csagan has attached in mirror mode.
    csagan has detached.

 This can be disabled by setting `announce_attach="false"`

 In addition, when a user switches from one host to another via the `wemux name
 <hostname>` command, their movement is displayed similarly to the attach
 messages.

  If csagan enters `wemux name applepie` the users on the default hostname
  `host` will see:

    csagan has switched to hostname: applepie

  If csagan returns to default hostname with: `wemux reset` users on `host`
  will see:

    csagan has joined this hostname.

  This can be disabled by setting `announce_host_change="false"`

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
