#!/bin/bash

DB_FILE="/tmp/port_tunnels.db"
LOCK_FILE="/tmp/port_tunnels.lock"
SERVER="tunnel.steeldev.space"
USER="tunnel"

# Create necessary directories and files
mkdir -p /tmp
touch "$DB_FILE"

# File locking functions
acquire_lock() {
    exec 200>"$LOCK_FILE"
    flock -w 30 200 || { echo "❌ Could not acquire lock after 30 seconds"; exit 1; }
}

release_lock() {
    flock -u 200
    rm -f "$LOCK_FILE"
}

# Safe file operations with locking
safe_file_operation() {
    acquire_lock
    "$@"
    local result=$?
    release_lock
    return $result
}

add_tunnel() {
    LOCAL_PORT=$1
    
    # Validate port number
    if ! [[ "$LOCAL_PORT" =~ ^[0-9]+$ ]] || [ "$LOCAL_PORT" -lt 1 ] || [ "$LOCAL_PORT" -gt 65535 ]; then
        echo "❌ Invalid port number: $LOCAL_PORT"
        return 1
    fi
    
    # Check if tunnel already exists for this port
    if safe_file_operation check_tunnel_exists "$LOCAL_PORT"; then
        echo "❌ Tunnel for port $LOCAL_PORT already exists"
        return 1
    fi

    TMPFILE=$(mktemp)
    
    # Start SSH tunnel in background
    ssh -o ConnectTimeout=10 -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -N -R 0:localhost:"$LOCAL_PORT" "$USER"@"$SERVER" >"$TMPFILE" 2>&1 &
    SSH_PID=$!

    # Wait for tunnel establishment with better timeout handling
    local tunnel_established=false
    for i in {1..20}; do
        if grep -q "Allocated port" "$TMPFILE"; then
            tunnel_established=true
            break
        fi
        # Check if SSH process is still running
        if ! kill -0 "$SSH_PID" 2>/dev/null; then
            break
        fi
        sleep 0.5
    done

    if [ "$tunnel_established" = true ]; then
        # Extract remote port more reliably
        REMOTE_PORT=$(grep "Allocated port" "$TMPFILE" | grep -oE '[0-9]{4,5}' | head -n1)
        
        # Verify we got a valid port
        if [[ "$REMOTE_PORT" =~ ^[0-9]+$ ]] && [ "$REMOTE_PORT" -ge 1024 ] && [ "$REMOTE_PORT" -le 65535 ]; then
            safe_file_operation add_tunnel_to_db "$LOCAL_PORT" "$REMOTE_PORT" "$SSH_PID"
            echo "✅ Port $LOCAL_PORT forwarded to $SERVER:$REMOTE_PORT (PID: $SSH_PID)"
        else
            echo "❌ Failed to extract valid remote port"
            kill "$SSH_PID" 2>/dev/null
        fi
    else
        echo "❌ Tunnel failed to establish within 10 seconds"
        echo "Debug output:"
        cat "$TMPFILE"
        kill "$SSH_PID" 2>/dev/null
    fi
    
    rm -f "$TMPFILE"
}

# Helper function to add tunnel to database
add_tunnel_to_db() {
    echo "$1:$2:$3" >> "$DB_FILE"
}

# Helper function to check if tunnel exists
check_tunnel_exists() {
    local port=$1
    while IFS=: read -r LPORT RPORT PID; do
        if [[ "$LPORT" == "$port" ]] && ps -p "$PID" > /dev/null 2>&1; then
            return 0
        fi
    done < "$DB_FILE"
    return 1
}

stop_tunnel() {
    PORT=$1
    
    # Validate port number or check for "all"
    if [[ "$PORT" != "all" ]] && (! [[ "$PORT" =~ ^[0-9]+$ ]] || [ "$PORT" -lt 1 ] || [ "$PORT" -gt 65535 ]); then
        echo "❌ Invalid port number: $PORT"
        return 1
    fi

    TMP=$(mktemp)
    local stopped_count=0
    
    safe_file_operation process_tunnel_stop "$PORT" "$TMP"
    
    if [ -f "$TMP" ]; then
        mv "$TMP" "$DB_FILE"
    fi
    
    if [ "$stopped_count" -eq 0 ] && [[ "$PORT" != "all" ]]; then
        echo "❌ No active tunnel found for port $PORT"
    fi
}

# Helper function to process tunnel stopping
process_tunnel_stop() {
    local target_port=$1
    local temp_file=$2
    stopped_count=0
    
    while IFS=: read -r LPORT RPORT PID; do
        if [[ "$target_port" == "all" ]] || [[ "$LPORT" == "$target_port" ]]; then
            if kill "$PID" 2>/dev/null; then
                echo "🛑 Stopped tunnel for port $LPORT (was $SERVER:$RPORT)"
                ((stopped_count++))
            else
                # Process already dead, but keep the entry if it's not our target
                if [[ "$target_port" != "all" ]] && [[ "$LPORT" != "$target_port" ]]; then
                    echo "$LPORT:$RPORT:$PID" >> "$temp_file"
                fi
            fi
        else
            echo "$LPORT:$RPORT:$PID" >> "$temp_file"
        fi
    done < "$DB_FILE"
}

stop_all() {
    safe_file_operation stop_all_tunnels
}

# Helper function to stop all tunnels
stop_all_tunnels() {
    local count=0
    while IFS=: read -r LPORT RPORT PID; do
        if kill "$PID" 2>/dev/null; then
            echo "🛑 Stopped tunnel for port $LPORT"
            ((count++))
        fi
    done < "$DB_FILE"
    
    > "$DB_FILE"
    
    if [ "$count" -eq 0 ]; then
        echo "ℹ️  No active tunnels to stop"
    else
        echo "✅ Stopped $count tunnel(s)"
    fi
}

list_tunnels() {
    safe_file_operation list_active_tunnels
}

# Helper function to list active tunnels
list_active_tunnels() {
    if [[ ! -s "$DB_FILE" ]]; then
        echo "ℹ️  No active tunnels"
        return
    fi
    
    local active_count=0
    echo "🔁 Active tunnels:"
    
    while IFS=: read -r LPORT RPORT PID; do
        if ps -p "$PID" > /dev/null 2>&1; then
            echo " - $SERVER:$RPORT ➜ localhost:$LPORT (PID: $PID)"
            ((active_count++))
        fi
    done < "$DB_FILE"
    
    if [ "$active_count" -eq 0 ]; then
        echo "ℹ️  No active tunnels (all processes terminated)"
        # Clean up the database
        > "$DB_FILE"
    fi
}

reset() {
    echo "⚠️  This will kill ALL SSH tunnel processes and wipe the tunnel database."
    echo "⚠️  This may disconnect active SSH sessions. Continue? (y/N)"
    read -r CONFIRM
    if [[ "$CONFIRM" =~ ^[Yy]$ ]]; then
        # Kill only our tunnel SSH processes more selectively
        safe_file_operation kill_all_tunnel_processes
        echo "🧼 Reset complete - all tunnels stopped and database cleared"
    else
        echo "🚫 Reset cancelled"
    fi
}

# Helper function to kill all tunnel processes
kill_all_tunnel_processes() {
    # Kill processes from our database
    while IFS=: read -r LPORT RPORT PID; do
        kill -9 "$PID" 2>/dev/null
    done < "$DB_FILE"
    
    # Clear database
    > "$DB_FILE"
    
    # Also kill any orphaned SSH tunnel processes to this server
    pkill -f "ssh.*$USER@$SERVER.*-R.*localhost:" 2>/dev/null || true
}

cleanup() {
    # Remove stale entries from database
    TMP=$(mktemp)
    safe_file_operation clean_stale_entries "$TMP"
    rm -f "$TMP" "$LOCK_FILE"
}

# Helper function to clean stale entries
clean_stale_entries() {
    local temp_file=$1
    while IFS=: read -r LPORT RPORT PID; do
        if ps -p "$PID" > /dev/null 2>&1; then
            echo "$LPORT:$RPORT:$PID" >> "$temp_file"
        fi
    done < "$DB_FILE"
    
    if [ -f "$temp_file" ]; then
        mv "$temp_file" "$DB_FILE"
    fi
}

print_help() {
    echo "🧪 Port forwarding script helper"
    echo ""
    echo "Usage:"
    echo "  port add <port>          Add tunnel for local port"
    echo "  port stop <port>         Stop tunnel for specific port"
    echo "  port stop all            Stop all tunnels"
    echo "  port list                List active tunnels"
    echo "  port reset               Kill all tunnel processes and clear database"
    echo "  port cleanup             Remove stale entries from database"
    echo "  port help                Show this help"
    echo ""
    echo "Examples:"
    echo "  port add 8080            Tunnel local port 8080"
    echo "  port stop 8080           Stop tunnel for port 8080"
    echo "  port list                Show active tunnels"
}

case "$1" in
    add)
        if [[ -n "$2" ]]; then
            add_tunnel "$2"
        else
            echo "❌ Usage: port add <local_port>"
            exit 1
        fi
        ;;
    stop)
        if [[ "$2" == "all" ]]; then
            stop_all
        elif [[ -n "$2" ]]; then
            stop_tunnel "$2"
        else
            echo "❌ Usage: port stop <local_port|all>"
            exit 1
        fi
        ;;
    list)
        list_tunnels
        ;;
    reset)
        reset
        ;;
    cleanup)
        cleanup
        ;;
    help|--help|-h)
        print_help
        ;;
    *)
        echo "❌ Unknown command: $1"
        echo "💡 Try: port help"
        exit 1
        ;;
esac
