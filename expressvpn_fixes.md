(Fedora) ExpressVPN Lightway TCP connection hangs under certain ISP conditions.

Here's a temporary workaround to fix the connection. (Not necessarily/guaranteed to work)

Run the script when ExpressVPN is connected but the connection hangs.

```bash
#!/usr/bin/env bash

set -e
sudo -v

echo "Checking ExpressVPN processes..."
echo

ps aux | grep expressvpn | grep -v grep || true

echo

echo "Killing ExpressVPN processes..."

if pgrep -f expressvpn-lightway >/dev/null; then
    echo "  -> Killing expressvpn-lightway"
    sudo pkill -f expressvpn-lightway
else
    echo "  -> expressvpn-lightway is NOT running"
fi

if pgrep -f expressvpn-lightway-rs >/dev/null; then
    echo "  -> Killing expressvpn-lightway-rs"
    sudo pkill -f expressvpn-lightway-rs
else
    echo "  -> expressvpn-lightway-rs is NOT running"
fi

if pgrep -f expressvpn-daemon >/dev/null; then
    echo "  -> Killing expressvpn-daemon"
    sudo pkill -f expressvpn-daemon
else
    echo "  -> expressvpn-daemon is NOT running"
fi

sleep 1

echo
echo "Removing stale policy routing..."

RULES=$(sudo ip rule | grep evpnFwdrt || true)

if [ -z "$RULES" ]; then
    echo "  -> No stale evpnFwdrt rules found"
else
    while IFS= read -r RULE; do
        PREF=$(echo "$RULE" | cut -d: -f1 | tr -d ' ')

        echo "  -> Removing evpnFwdrt rule priority $PREF"
        echo "     $RULE"

        sudo ip rule del priority "$PREF"
    done <<< "$RULES"
fi

RULES=$(sudo ip rule | grep "suppress_prefixlength 1" || true)

if [ -z "$RULES" ]; then
    echo "  -> No stale suppress_prefixlength rules found"
else
    while IFS= read -r RULE; do
        PREF=$(echo "$RULE" | cut -d: -f1 | tr -d ' ')

        echo "  -> Removing suppress rule priority $PREF"
        echo "     $RULE"

        sudo ip rule del priority "$PREF"
    done <<< "$RULES"
fi

echo
echo "Flushing ExpressVPN routing table..."

if sudo ip route show table evpnFwdrt | grep -q .; then
    echo "  -> Flushing evpnFwdrt"
    sudo ip route flush table evpnFwdrt
else
    echo "  -> evpnFwdrt already empty"
fi

sudo ip route flush cache

echo
echo "Removing stale tun0..."

if ip link show tun0 >/dev/null 2>&1; then
    sudo ip link delete tun0
    echo "  -> Removed tun0"
else
    echo "  -> No tun0 interface found"
fi

echo
echo "Done. Wait for ExpressVPN to reconnect and reconfigure itself"
echo

echo "Current routing:"
ip route

echo
echo "Current rules:"
ip rule

```

Reboot if ExpressVPN seems to not work after all.
