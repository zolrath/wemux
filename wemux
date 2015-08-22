#!/usr/bin/env bash
# wemux by Matt Furden @zolrath
#
# wemux allows you to start a shared tmux session using the command 'wemux'.
# Clients have the option of:
# Mirroring, which will give them read-only access,
# Pairing, which will allow them to control the hosts cursor.
# Rogue mode, which  allows them to work in the same window (shared cursor) or
# in another window (separate cursors) in the hosts tmux session.
#
# To set a user as host add their username to the host_list in the configuration
# file located at /usr/local/etc/wemux.conf
# Other configuations options are also located in /usr/local/etc/wemux.conf
#
# For environments with multiple hosts running their own independent servers
# on the same machine wemux can join different servers with the wemux join
# command, if enabled. This can be enabled in the configuration file.
#
# WEMUX HOST COMMANDS:
# wemux start : Start the wemux server/join an existing wemux server.
# wemux attach: Join an existing wemux server.
# wemux stop  : Stop the wemux server.
# wemux users : List the currently attached wemux users.
# wemux kick  : Disconnect an SSH user, remove their wemux server.
# wemux config: Open the wemux configuration file in your $EDITOR.
# wemux help  : Display the help screen.
#
# WEMUX CLIENT COMMANDS:
# wemux mirror: Attach to Host in read-only mode.
# wemux pair  : Attach to Host in pair mode, which allows editing.
# wemux rogue : Attach to Host in rogue mode, which allows editing and switching
#               to windows independently from the host.
# wemux logout: Log out of the wemux rogueing session.
# wemux users : List the currently attached wemux users.
# wemux help  : Display the help screen.
#
# To enable multi-host commands, set allow_server_change="true" in wemux.conf
# WEMUX SESSION COMMANDS: can be run by either host or client.
# wemux join  : Join wemux server with supplied name.
# wemux reset : Join default wemux server: wemux
# wemux list  : List all currently active wemux servers.

###############################################################################

# Current wemux version.
version="3.2.0"

# Setup and Configuration Files.
# Default settings, modify them in the /usr/local/etc/wemux.conf file:
host_list=(change_this_in_wemux_conf)
host_groups=()
socket_prefix="/tmp/wemux"
options="-u"
allow_pair_mode="true"
allow_rogue_mode="true"
default_client_mode="mirror"
allow_kick_user="true"
allow_server_change="false"
default_server_name="wemux"
allow_server_list="true"
allow_user_list="true"
announce_attach="true"
announce_server_change="true"

# Prevent users from changing their $USER env variable to become host.
username=`whoami`
# Set $EDITOR default to vi if not configured on host machine.
editor=${EDITOR:="vi"}

# Load configuration options from /usr/local/etc/wemux.conf
[ -f /usr/local/etc/wemux.conf ] && . /usr/local/etc/wemux.conf

# Sanitize server name, replace spaces and underscores with dashes.
# Remove all non alpha-numeric characters, convert to lowercase.
sanitize_server_name() {
  local new_server=$@
  local new_server=${new_server// /\-}
  local new_server=${new_server//_/\-}
  local new_server=${new_server//[^a-zA-Z0-9\-]/}
  local new_server=`echo "$new_server" | tr '[A-Z]' '[a-z]'`
  echo "$new_server"
}

# Load the server name of last wemux server. If empty, set to 'wemux'.
# Ensure server name is 'wemux' if allow_server_change is disabled.
# If default_server_name is set in wemux.conf it will be used instead of 'wemux'
load_server_name() {
  if [ "$allow_server_change" == "true" ]; then
    if [ -f ~/.wemux_last_server ]; then
      unclean_server=`cat ~/.wemux_last_server`
      # Sanitize loaded server name.
      server=`sanitize_server_name $unclean_server`
      if ! [[ $unclean_server == $server ]]; then
        echo "$server" > ~/.wemux_last_server
      fi
    else
      # If ~/.wemux_last_server doesn't exist set to default_server_name
      server="$default_server_name"
    fi
  else
    # If allow_server_change is disabled, set to default_server_name
    server="$default_server_name"
  fi
}

# Build $wemux variable to call proper tmux server.
build_wemux_prefix() {
  load_server_name
  # Set client's rogue mode session name.
  client_session="$server-$username"
  # Set socket to include server name.
  socket="${socket_prefix}-$server"
  # Set $wemux to wemux server socket.
  wemux="tmux -S $socket $options"
}

# List all currently running wemux servers.
list_active_servers() {
  if [ "$allow_server_change" == "true" ]; then
    if [ "$allow_server_list" == "true" ]; then
      wemux_server_sockets=$socket_prefix*
      for wemux_server in $wemux_server_sockets; do
        redirect=`tmux -S $wemux_server server-info 2>&1`; server_running=$?
        # Number each active server, give it its own line.
        if [ "$server_running" == 0 ]; then
          num=$[num+1]
          server_name=`echo "$wemux_server" | sed -e "s,$socket_prefix-,,"`
          # Add indicator for current wemux server.
          if [ "$server_name" == "$server" ]; then
            server_name="$server_name    <- current server"
          fi
          server_list="$server_list  $num. $server_name\n"
        fi
      done
      if [ -z "$server_list" ]; then
        echo "No wemux servers currently active."
      else
        echo "Currently active wemux servers:"
        # Remove last newline
        echo -e "${server_list%??}"
      fi
    # allow_server_list is disabled:
    else
      echo "Server listing has been disabled."
      return 126
    fi
  # allow_server_change is disabled:
  else
    echo "Server related commands have been disabled."
    return 126
  fi
}

# List users currently attached to wemux server. Username[m] for mirror mode.
# Only contains names. Formatted for being included as part of status bar.
status_users() {
  if [ "$allow_user_list" == "true" ]; then
    while IFS= read line; do
      read tty mode <<<$(echo $line)
      # Get user associated with tty
      name=`stat -f%Su $tty 2>/dev/null` || name=`stat -c%U $tty 2>/dev/null`
      # If user is attached in read-only mode, set mode to [m]
      [[ $mode == "(ro)" ]] && mode="[m]" || mode=""
      # If user/mode is already is userlist, do not add them again.
      if ! [[ "$users" =~ "$name$mode," ]]; then
        users="$users$name$mode, "
      fi
    done < <(wemux list-clients | cut -f1,6 -d ' ' | sed s/://)
    # Strip last two characters ', '
    echo "${users%??}"
  else
    echo "User list disabled."
    return 126
  fi
}

list_all_users() {
  if [ "$allow_user_list" == "true" ]; then
      wemux_server_sockets=$socket_prefix*
      for wemux_server in $wemux_server_sockets; do
        server_running=`tmux -S $wemux_server list-clients`
        if [[ "$server_running" ]] && [[ "$server_running" != *"failed to connect"* ]]; then
          server_name=`echo "$wemux_server" | sed -e "s,$socket_prefix-,,"`
          echo "Users connected to '$server_name':"
          # Get list of users with status_users function.
          while IFS= read line; do
            read tty mode <<<$(echo $line)
            # Get user associated with tty
            name=`stat -f%Su $tty 2>/dev/null` || name=`stat -c%U $tty 2>/dev/null`
            # If user is attached in read-only mode, set mode to [m]
            [[ $mode == "(ro)" ]] && mode="[m]" || mode=""
            # If user/mode is already is userlist, do not add them again.
            user="$name$mode"
            num=$[num+1]
            echo "  $num. $user"
          done < <(tmux -S $wemux_server list-clients -F '#{client_tty}')
        fi
      done
  else
    echo "User listing has been disabled."
  fi
}

# List users currently attached to wemux server with informative string.
# More verbose, intended for use in terminal.
list_users() {
  if [ "$allow_user_list" == "true" ]; then
    redirect=`wemux server-info 2>&1`; server_running=$?
    if [ "$server_running" == 0 ]; then
      # Get list of users with status_users function.
      users="$(wemux status_users)"
      # Number each user, give it its own line.
      for user in $users; do
          num=$[num+1]
          user=`echo "$user" | sed -e "s/,//"`
          user_list="$user_list  $num. $user\n"
      done
      if [ -z "$user_list" ]; then
        echo "No wemux users attached to '$server'."
      else
        echo "Users attached to '$server': "
        # Remove last newline.
        echo -e "${user_list%??}"
      fi
    else
      echo "No wemux server running on '$server'. No server, no users!"
    fi
  else
    echo "User listing has been disabled."
  fi
}

# Display the currently attached users verbosely in a tmux message.
display_users() {
    redirect=`$wemux display-message "$(wemux users)" 2>&1`
}

# Returns the host's tty and server name.
get_host_session() {
  redirect=`wemux server-info 2>&1`; server_running=$?
  if [ "$server_running" == 0 ]; then
    while read line; do
      read tty current_session <<<$(echo $line)
      # Get user associated with tty
      name=`stat -f%Su $tty 2>/dev/null` || name=`stat -c%U $tty 2>/dev/null`
      if [[ $name == $username ]]; then
        host=$tty
        host_session=$current_session
      fi
    done < <(wemux list-clients -F '#{client_tty}')
  echo "$host $host_session"
  fi
}

# Toggles mirrored users to pair mode, pair mode to mirrored.
mirror_toggle() {
  get_host_session > read host host_session
  while read tty; do
    if ! [[ $tty == $host ]]; then
      redirect=`$wemux switch-client -c $tty -t $host_session -r 2>&1`
    fi
  done < <(wemux list-clients -F '#{client_tty}')
}

# Set all users to mirror mode on the hosts current session
mirror_all_users() {
  get_host_session > read host host_session
  while read line; do
    read tty mode <<<$(echo $line)
    if ! [[ $tty == $host ]] && [[ $mode == "(ro)" ]]; then
      redirect=`$wemux switch-client -c $tty -t $host_session 2>&1`
    elif ! [[ $tty == $host ]]; then
      redirect=`$wemux switch-client -c $tty -t $host_session -r 2>&1`
    fi
  done < <(wemux list-clients -F '#{client_tty}')
}

# Make all users join the hosts session.
summon_all_users() {
  get_host_session > read host host_session
  while read tty; do
    if ! [[ $tty == $host ]]; then
      $wemux switch-client -c $tty -t $host_session
    fi
  done < <(wemux list-clients -F '#{client_tty}')
}

# Make all mirrored users join the hosts session.
summon_mirrored() {
  get_host_session > read host host_session
  while read line; do
    read tty mode <<<$(echo $line)
    if ! [[ $tty == $host ]] && [[ $mode == "(ro)" ]]; then
      $wemux switch-client -c $tty -t $host_session
    fi
  done < <(wemux list-clients -F '#{client_tty}')
}

# The ugly group of redirects below solve the issue where tmux/epoll causes tmux
# to hang when stderr is redirected to /dev/null in a backwards compatible way.

# Returns true if host currently has a running wemux session.
session_exists() {
  redirect=`$wemux has-session -t $server 2>&1`; does_exist=$?
  [ "$does_exist" == 0 ] && return 0 || return 1;
}

# Returns true if rogue session with current host already exists.
has_rogue_session() {
  redirect=`$wemux has-session -t $client_session 2>&1`; does_exist=$?
  [ "$does_exist" == 0 ] && return 0 || return 1;
}

# Returns true if server is successfully killed.
kill_server_successful() {
  redirect=`tmux -S $socket kill-server 2>&1`; killed_successfully=$?
  [ "$killed_successfully" == 0 ] && return 0 || return 1;
}

# Announce when user attaches/detaches from server.
# Can be disabled by changing announce_attach to false in /usr/local/etc/wemux.conf
# The first argument specifies the mode the user is attaching in for the message
# All additional arguments get wrapped in the attach/detach messages.
announce_connection() {
  connection_type=$1; shift; attach_commands="$@"
  [ "$announce_attach" == "true" ] && redirect=`$wemux display-message \
    "$username has attached in $connection_type mode." 2>&1`
  $attach_commands
  status=$?
  [ "$announce_attach" == "true" ] && redirect=`$wemux display-message \
    "$username has detached." 2>&1`
  return $status
}

# Announces when a user joins/changes their server.
# Can be disabled by changing announce_server_change to false in /usr/local/etc/wemux.conf
# Change server name for server, or display server name if no argument is given.
change_server() {
  if [ "$allow_server_change" == "true" ]; then
    # Sanitize input.
    new_server=`sanitize_server_name $@`
    old_server=$server

    # Get list of currently running servers
    server_names=(`echo $socket_prefix* | sed -e "s,$socket_prefix-,,g"`)
    # If user joins a number, go to the server with that number in `wemux list`
    [[ "$new_server" =~ ^[0-9]+$ ]] && new_server=${server_names[$new_server-1]}

    if [ -z "$new_server" ]; then
      echo "Current wemux server: $server"
    elif [ "$new_server" == "$old_server" ]; then
      echo "Your wemux server is already set to '$server'"
    else
      # Announce that the user has changed servers to current server.
      [ "$announce_server_change" == "true" ] && redirect=`$wemux display-message \
        "$username has switched to server: $new_server" 2>&1`
      echo "Changed wemux server from '$old_server' to '$new_server'"
      echo $new_server > ~/.wemux_last_server
      # Rebuild wemux prefix for new server name.
      build_wemux_prefix
      # Announce that the user has joined to new server.
      [ "$announce_server_change" == "true" ] && redirect=`$wemux display-message \
        "$username has joined this server" 2>&1`
    fi
  else
    echo "Changing wemux servers has been disabled."
    return 126
  fi
  return 0
}

# Display version of wemux installed on system. Show URL for wemux.
display_version() {
  echo "wemux $version"
  echo "To check for a newer version visit: http://www.github.com/zolrath/wemux"
}

# Host mode, used when user is listed in the host_list array in /usr/local/etc/wemux.conf
host_mode() {
  # Start the server if it doesn't exist, otherwise reattach.
  start_server() {
    if ! session_exists; then
      $wemux new-session -d -s $server
      # Open tmux socket to all users to allow clients to connect.
      chmod 1777 $socket
      echo "wemux server started on '$server'."
    fi
    reattach
  }

  # Reattach to the wemux server.
  reattach() {
    if session_exists; then
      $wemux attach -t $server
    else
      echo "No wemux server to attach to on '$server'."
    fi
  }

  # Kick an SSH user off the server and remove their rogue session if it exists.
  kick_user() {
    if [ "$allow_kick_user" == "true" ]; then
      kicked_user=$1
      echo "Kicking $kicked_user from the server. Sudo required."
      # Get sshd process with users name and get its PID.
      user_pid=`sudo lsof -t -u $kicked_user -c sshd -a`
      # Kill the sshd process of the supplied user.
      redirect=`sudo kill -9 $user_pid 2>&1`; kicked_successfully=$?
      # Remove any tmux sessions ending in "-kicked_user"
      `wemux kill-session -t "*-$kicked_user" > /dev/null 2>&1`
      if [ "$kicked_successfully" == 0 ]; then
        echo "$kicked_user kicked successfully!"
      else
        echo "$kicked_user was not kicked. Are you sure they're connected?"
      fi
    else
      echo "Kicking users has been disabled."
    fi
  }

  # Stop the wemux session and remove the socket file.
  stop_server() {
    server_sockets=(`echo $socket_prefix*`)
    # If a specific server is supplied:
    if [ $1 ]; then
      # If user enters a number, stop the server with that number in `wemux list`
      if [[ $@ =~ ^[0-9]+$ ]]; then
        socket=${server_sockets[$1-1]}
        server=`echo $socket | sed -e "s,$socket_prefix-,,g"`
      # Otherwise, stop the server with the supplied name.
      else
        server=`sanitize_server_name $@`
        socket="$socket_prefix-$server"
      fi
    fi
    # If the user doesn't pass an argument, stop current server.
    if kill_server_successful; then
      echo "wemux server on '$server' stopped."
    else
      echo "No wemux server running on '$server'"
    fi
    # If the socket file exists:
    if [ -e "$socket" ]; then
      if ! rm $socket; then
        echo "Could not remove '$socket'. Please check file ownership."
      fi
    fi
  }

  # Display the commands available in host mode.
  display_host_commands() {
    echo "wemux version $version"
    if [ "$allow_server_change" == "true" ]; then
      echo "Current wemux server: $server"
    fi
    echo ""
    echo "Usage: wemux [command]"
    echo "To host a wemux server please use one of the following:"
    echo ""
    echo "    [s]       start: Start the wemux server/attach to an existing wemux server."
    echo "    [a]      attach: Attach to an existing wemux server."
    echo "    [k]        stop: Kill the wemux server '$server', delete its socket."
    echo ""
    if [ "$allow_server_change" == "true" ]; then
      echo "    [j] join [name]: Join the specified wemux server."
      echo "    [j]        join: Display the current wemux server."
      echo "    [d]       reset: Join default wemux server: $default_server_name"
      if [ "$allow_server_list" == "true" ]; then
        echo "    [l]        list: List all currently active wemux servers."
      fi
    fi
    if [ "$allow_user_list" == "true" ]; then
      echo "    [u]       users: List all users currently attached to '$server'"
    fi
    if [ "$allow_kick_user" == "true" ]; then
      echo "               kick: Disconnect an SSH user, remove their wemux server."
    fi
    echo ""
    echo "    [c]      config: Open the wemux configuration file in $EDITOR."
    echo "    [h]        help: Display this screen."
    echo "            no args: Start the wemux server/attach to an existing wemux server."
  }

  # Host mode command handling:
  # If no command given, call start server.
  if [ -z "$1" ]; then
    announce_connection "host" start_server
  else
    case "$1" in
      start|s)        announce_connection "host" start_server;;
      attach|a)       announce_connection "host" reattach;;
      stop|st)        shift; stop_server "$@";;
      kill|k)         shift; stop_server "$@";;
      help|h)         display_host_commands;;
      join|j)         shift; change_server "$@";;
      name|n)         shift; change_server "$@";;
      reset|d)        change_server "$default_server_name";;
      list|l)         list_active_servers;;
      users|u)        list_all_users;;
      summon)         summon_mirrored;;
      summon_all)     summon_all_users;;
      setmirrored)    mirror_all_users;;
      togglemirrored) mirror_toggle;;
      kick)           kick_user $2;;
      status_users)   status_users;;
      display_users)  display_users;;
      version|v)      display_version;;
      conf*|c)        $editor /usr/local/etc/wemux.conf;;
      *)              if ! $wemux "$@"; then
                        echo "To see a list of wemux commands enter 'wemux help'"
                        exit 127
                      fi;;
    esac
  fi
}

# Client Mode, used when user is NOT listed in the host_list in /usr/local/etc/wemux.conf
client_mode() {
  # Mirror mode, allows the user to view wemux server in read only mode.
  mirror_mode() {
    if session_exists; then
      $wemux attach -t $server -r
    else
      echo "No wemux server to mirror on '$server'."
      return 126
    fi
  }

  # Pair mode, allows user to interact with wemux server.
  pair_mode() {
    if [ "$allow_pair_mode" == "true" ]; then
      if session_exists; then
        $wemux attach -t $server
      else
        echo "No wemux server to pair with on '$server'."
        return 126
      fi
    else
      echo "Pair mode is disabled."
      return 126
    fi
  }

  # Rogue mode, allows user to interact with wemux server and operate
  # independently in other windows.
  # Will connect to existing rogue session or create one if necessary.
  rogue_mode() {
    if [ "$allow_rogue_mode" == "true" ]; then
      if has_rogue_session; then
        $wemux attach -t $client_session
      elif session_exists; then
        $wemux new-session -d -t $server -s $client_session
        $wemux new-window -n $username
        $wemux send-keys -t $client_session $@
        $wemux attach -t $client_session
      else
        echo "No wemux server to go 'rogue' with on '$server'."
        return 126
      fi
    else
      echo "Rogue mode is disabled."
      return 126
    fi
  }

  # Log user out of rogue mode, removing their session..
  logout_rogue() {
    if [ "$allow_rogue_mode" == "true" ]; then
      if has_rogue_session; then
        [ "$announce_attach" == "true" ] && $wemux display-message \
          "$username has logged out of rogue mode."
        $wemux kill-session -t $client_session
        echo "Logged out of rogue mode on '$server'."
      else
        echo "No wemux server to log out of on '$server'."
        return 126
      fi
    else
      echo "Rogue mode is disabled."
      return 126
    fi
  }

  send() {
    if [ "$allow_pair_mode" == "true" -o "$allow_rogue_mode" == "true" ]; then
      $wemux "$@"
    else
      echo "Pairing and Rogue mode are off, sending is disabled."
    fi
  }

  smart_reattach() {
    # If default_client_mode has been set to "rogue" in wemux.conf:
    if [ "$default_client_mode" == "rogue" ] && [ "$allow_rogue_mode" == "true" ]; then
      announce_connection "rogue" rogue_mode
    # If default_client_mode has been set to "pair" in wemux.conf:
    elif [ "$default_client_mode" == "pair" ] && [ "$allow_pair_mode" == "true" ]; then
      announce_connection "pair" pair_mode
    elif has_rogue_session && [ "$allow_rogue_mode" == "true" ]; then
      announce_connection "rogue" rogue_mode
    elif [ "$allow_pair" == "true" ]; then
      announce_connection "pair" pair_mode
    elif session_exists; then
      announce_connection "mirror" $wemux attach -t $server -r
    else
      echo "No wemux server to attach to on '$server'"
      return 126
    fi
  }

  # Display the commands available in client mode.
  display_client_commands() {
    echo "wemux version $version"
    if [ "$allow_server_change" == "true" ]; then
      echo "Current wemux server: $server"
    fi
    echo ""
    echo "Usage: wemux [command]"
    echo "To connect to wemux please use one of the following:"
    echo ""
    echo "    [m]      mirror: Attach to '$server' in read-only mode."
    if [ "$allow_pair_mode" == "true" ]; then
      echo "    [p]        pair: Attach to '$server' in pair mode, which allows editing."
    fi
    if [ "$allow_rogue_mode" == "true" ]; then
      echo "    [r]       rogue: Attach to '$server' in rogue mode, allowing you to pair"
      echo "                     and also work in other windows independent of the host."
      echo "    [o]      logout: Log out of the current wemux rogue session."
    fi
    echo ""
    if [ "$allow_server_change" == "true" ]; then
      echo "    [j] join [name]: Join the specified wemux server."
      echo "    [j]        join: Display the current wemux server."
      echo "    [d]       reset: Join default wemux server: $default_server_name"
      if [ "$allow_server_list" == "true" ]; then
        echo "    [l]        list: List all currently active wemux servers."
      fi
    fi
    if [ "$allow_user_list" == "true" ]; then
      echo "    [u]       users: List all users currently attached to '$server'"
    fi
    echo ""
    echo "    [h]        help: Display this screen."
    if [ "$default_client_mode" == "rogue" ] && [ "$allow_rogue_mode" == "true" ]; then
      echo "            no args: Attach to '$server' in rogue mode."
    elif [ "$default_client_mode" == "pair" ] && [ "$allow_pair_mode" == "true" ]; then
      echo "            no args: Attach to '$server' in pair mode."
    elif [ "$allow_rogue_mode" == "true" ] &&  [ "$allow_pair_mode" == "true" ]; then
      echo "            no args: Attach to rogue session on '$server' if it already exists,"
      echo "                     otherwise, attach in pair mode."
    elif [ "$allow_rogue_mode" == "true" ] &&  [ "$allow_pair_mode" == "false" ]; then
      echo "            no args: Attach to rogue session on '$server' if it already exists,"
      echo "                     otherwise, attach in mirror mode."
    elif [ "$allow_pair_mode" == "true" ]; then
      echo "            no args: Attach to '$server' in pair mode."
    else
      echo "            no args: Attach to '$server' in mirror mode."
    fi
  }

  # Client mode command handling:
  # If no command given, call smart_reattach
  if [ -z "$1" ]; then
    smart_reattach
  else
    case "$1" in
      mirror|m)      announce_connection "mirror" mirror_mode;;
      pair|p)        announce_connection "pair" pair_mode;;
      rogue|r)       shift; announce_connection "rogue" rogue_mode $@;;
      edit|e)        announce_connection "rogue" rogue_mode;;
      logout|o)      logout_rogue;;
      stop|s)        logout_rogue;;
      help|h)        display_client_commands;;
      join|j)        shift; change_server "$@";;
      name|n)        shift; change_server "$@";;
      reset|d)       change_server "$default_server_name";;
      list|l)        list_active_servers;;
      users|u)       list_users;;
      status_users)  status_users;;
      display_users) display_users;;
      version|v)     display_version;;
      send)          send $@;;
      *)             if ! $wemux "$@"; then
                       echo "To see a list of wemux commands enter 'wemux help'"
                       exit 127
                     fi;;
    esac
  fi
}

# Determine the groups the user belongs to
groups_for_user() {
  echo $(groups $username | awk -F: "{print \$NF}");
}

# Check if current user is listed in the host_list.
user_is_a_host() {
  for user in "${host_list[@]}"; do
    [[ "$user" == "$username" ]] && return 0
  done

  # If the user is not in the host list, check their groups
  user_groups="$(groups_for_user)";
  for group in ${host_groups[@]}; do
    group_re="\\b${group}\\b"
    if [[ $user_groups =~ $group_re ]]; then
      return 0
    fi
  done

  return 1
}


allowed_nested_command() {
  commands=(togglemirrored setmirrored summon summon_all send users u list l display_users
  status_users list-clients kill-sesssion kick version v)
  for command in "${commands[@]}"; do
    [[ "$command" == "$1" ]] && return 0
  done
  return 1
}
# Create $wemux variable.

build_wemux_prefix

# Don't allow wemux to be run directly within a wemux server.
if [[ "$TMUX" != *$socket* ]] || allowed_nested_command "$1" ; then
  # Allow manual override to operate in client mode.
  if [ "$1" = "client" ]; then
    shift; client_mode "$@"
  else
    # If user is in host list, use host mode. If not, use client mode.
    if user_is_a_host; then
      host_mode "$@"
    else
      client_mode "$@"
    fi
  fi
elif [[ $@ == "help" ]] || [[ $@ == "h" ]]; then
  echo "Most wemux commands can only be run while detached from tmux."
  echo "The commands that are available while attached are:"
  echo ""
  echo "HOST ONLY COMMANDS:"
  echo "    [s]      summon: Summon all mirrored users to your current session."
  echo "          summonall: Summon all users to your current session."
  echo "        setmirrored: Set all users to mirror mode to your current session."
  echo "     togglemirrored: Toggle users between mirror/pair mode."
  echo "        kick [user]: Kick specified user from the server."
  echo ""
  echo "GENERAL COMMANDS:"
  echo "    [u]  users: List all attached users"
  echo "    [l]   list: List all wemux servers"
else
  echo "You're already attached to the wemux server on '$server'"
  echo "wemux does not allow nesting wemux servers directly."
fi
