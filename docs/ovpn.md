# Open-VPN easy startup script
Built for simplicity, and high success, does not get blocked/circumvented by wifi restrictions.
```sh
#!/bin/bash

# 1. Clean up old sessions and stuck routes
echo "Cleaning up old VPN processes..."
sudo pkill openvpn
sudo ip route del 0.0.0.0/1 2>/dev/null
sudo ip route del 128.0.0.0/1 2>/dev/null
sleep 2

# 2. Start OpenVPN
echo "Starting OpenVPN..."
sudo openvpn --config /etc/openvpn/client/ca-free-7.protonvpn.tcp.ovpn --daemon # Replace file with your path to ovpn file, ovpn file should be in /et/openvpn/client, if not, it is recommended you put it there.

echo "Waiting for a tun interface to get an IP..."

# 3. Dynamic Interface Detection
MAX_RETRIES=30
COUNT=0
TUN_DEV=""

while [ $COUNT -lt $MAX_RETRIES ]; do
    # Find the newest tun device that has an IP address
    TUN_DEV=$(ip -4 addr show | grep -oP 'tun\d+' | head -n 1)

    if [ -n "$TUN_DEV" ] && ip addr show "$TUN_DEV" | grep -q "inet "; then
        echo "Found active interface: $TUN_DEV"
        break
    fi

    sleep 1
    ((COUNT++))
done

if [ -z "$TUN_DEV" ]; then
    echo "Error: No tun interface initialized with an IP. Check logs."
    exit 1
fi

# 4. Fix the Routing using the dynamic TUN_DEV
echo "Applying routing fixes to $TUN_DEV..."
VPN_SERVER="149.22.82.55"
GATEWAY=$(ip route show default | awk '{print $3}')

sudo ip route add $VPN_SERVER via $GATEWAY 2>/dev/null
sudo ip route add 0.0.0.0/1 dev "$TUN_DEV"
sudo ip route add 128.0.0.0/1 dev "$TUN_DEV"

# 5. Fix the DNS
sudo resolvectl dns "$TUN_DEV" 10.98.0.1
sudo resolvectl domain "$TUN_DEV" "~."
sudo resolvectl default-route "$TUN_DEV" yes
sudo resolvectl flush-caches

echo "VPN successfully routed through $TUN_DEV."
```
