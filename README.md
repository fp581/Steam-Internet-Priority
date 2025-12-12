# Steam QoS Setup (Systemd + tc)
Make sure you have the following commands:
IP
TC
Steam

---

## STEP 1 — Create the script

Run:

```bash
sudo nano /usr/local/bin/steam-qos.sh
```

Paste **this EXACT script**:

```bash
#!/usr/bin/env bash
set -e

# Auto-detect the main network interface
INTERFACE=$(ip route get 8.8.8.8 | awk '{print $5; exit}')

# Remove old qdisc
tc qdisc del dev "$INTERFACE" root 2>/dev/null || true

# Add priority queue with 3 bands
tc qdisc add dev "$INTERFACE" root handle 1: prio bands 3

# STEAM uses many ports: 27000–27100 both TCP and UDP
PORT_RANGE_START=27000
PORT_RANGE_END=27100

# High priority band = 1:1
for ((p=PORT_RANGE_START; p<=PORT_RANGE_END; p++)); do
    tc filter add dev "$INTERFACE" protocol ip parent 1:0 prio 1 \
        u32 match ip dport $p 0xffff flowid 1:1

    tc filter add dev "$INTERFACE" protocol ip parent 1:0 prio 1 \
        u32 match ip sport $p 0xffff flowid 1:1
done
```

Save and exit.

Make the script executable:

```bash
sudo chmod +x /usr/local/bin/steam-qos.sh
```

---

## STEP 2 — Create the systemd service

Run:

```bash
sudo nano /etc/systemd/system/steam-qos.service
```

Paste this **CLEAN version**:

```ini
[Unit]
Description=Apply QoS Rules for Steam Gaming
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/steam-qos.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Save and exit.

---

## STEP 3 — Reload & enable the service

Run:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now steam-qos.service
```

---

## STEP 4 — Check status

Run:

```bash
systemctl status steam-qos.service
```

You should see:

```text
Active: active (exited)
Exit status: 0
```

---

✅ **Steam traffic is now prioritized at boot using tc + systemd**

This setup is distro-agnostic and works on Arch / EndeavourOS / most systemd-based Linux systems.
