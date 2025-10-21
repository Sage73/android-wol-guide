# 🌐 Wake Your PC From Anywhere Using an Old Android Phone

Use your old Android device as a **remote Wake-on-LAN (WoL) relay** — no router port forwarding, no extra hardware.  
This guide uses **Tailscale**, **Termux**, and **SSH** to let you wake your PC from anywhere with a single tap.

### Table of Contents
1. [How It Works](#-how-it-works)
2. [Requirements](#️-requirements)
3. [Step 0 - Enable WoL](#️-step-0--enable-wol-on-your-pc)
4. [Step 1 - Set Up Old Android](#-step-1--set-up-the-old-android-home-device)
5. [Auto Start On Boot](#d-optional--auto-start-on-boot)
6. [Step 2 - Set Up Main Phone](#-step-2--set-up-your-main-phone)
7. [Step 3 - One Tap Wake](#-step-3--one-tap-wake-shortcut)
8. [Troubleshooting](#-troubleshooting)
9. [Command Cheat Sheet](#-commands-cheat-sheet)

---

## 🧠 How It Works

Your old Android (always on at home) acts as a local relay:

1. You connect securely to it from anywhere using **Tailscale** (a free mesh VPN).
2. You run a simple **WoL script** on it through SSH.
3. The phone sends the **magic packet** to your PC → it wakes up.

---

## ⚙️ Requirements

- A PC that supports Wake-on-LAN  
- Your PC connected via ethernet
- Your **old Android** (stays home, connected to Wi-Fi)  
- Your **main phone** (for remote control)  
- Free apps:
  - [Tailscale](https://tailscale.com/)
  - [Termux (from F-Droid)](https://f-droid.org/en/packages/com.termux/)
  - [Termux:Boot (optional)](https://f-droid.org/en/packages/com.termux.boot/)
  - [Termius (SSH client)](https://termius.com/)

---

## 🖥️ Step 0 — Enable WoL on Your PC

### Windows
1. Enter **BIOS/UEFI** → enable **Wake on LAN**.  
2. In **Device Manager → Network adapter → Properties**:
   - Power Management → allow wake  
   - Advanced → “Wake on Magic Packet” = Enabled

To enable Wake-on-LAN (WOL) from the BIOS:  
Restart your computer and repeatedly press the key to enter the BIOS/UEFI setup (e.g., `F2`, `F10`, `Del`, or `Esc`).  
Navigate to *Power Management* or *Advanced settings*, find the **Wake on LAN** (or *Power On by PCI-E*) option, set it to **Enabled**, and save & exit.

### 🧪 Test WOL
Install any WOL app such as [Wake on LAN](https://play.google.com/store/apps/details?id=co.uk.mrwebb.wakeonlan) and try to turn on your PC **from the same network**.  
Inside the app, input your PC’s **MAC address**.

> ⚠️ Not every PC supports WoL, but most modern ones do and it can be enabled if supported by the hardware.

**Find your MAC & broadcast address**
- On a `192.168.1.0/24` network, broadcast is usually `192.168.1.255`.  
- Find your MAC address (Physical Address) from CMD: `ipconfig /all`

## 📱 Step 1 — Set Up the Old Android (Home Device)

### A. Install & Prepare

1. Install **[Tailscale](https://tailscale.com/)** → log in → ensure it says **“Connected”**.  
2. Install **[Termux](https://f-droid.org/en/packages/com.termux/)** *(from F-Droid, not the Play Store)*.  
3. *(Optional)* Install **[Termux:Boot](https://f-droid.org/en/packages/com.termux.boot/)** if you want auto-start after reboot.

### B: Configure Termux
```
pkg update -y
pkg install -y python openssh
pip install --upgrade wakeonlan
```
**Create the WoL script (edit MAC and broadcast as needed):**
```
cat > ~/wolpc.sh << 'EOF'
#!/data/data/com.termux/files/usr/bin/bash
MAC="AA:BB:CC:DD:EE:FF"      # <-- your PC's MAC
BCAST="192.168.1.255"        # <-- your LAN broadcast

/data/data/com.termux/files/usr/bin/python - <<PY
from wakeonlan import send_magic_packet
send_magic_packet("$MAC", ip_address="$BCAST")
print("Magic packet sent to", "$MAC", "via", "$BCAST")
PY
EOF

chmod +x ~/wolpc.sh
```

**🧪 Test Locally**
`./wolpc.sh`
Your PC should wake up — if yes, you're good ✅

### C. Enable SSH Access
```
passwd    # set a password
sshd      # start SSH (listens on port 8022)
```

Check it’s running:
`ps -A | grep sshd`
**Tips:**
- Always paste long heredocs as a **single block** so nothing wraps onto a new line.
- Run with `./wolpc.sh` (not `. wolpc.sh`)
- If it still doesn’t wake, double-check the MAC and that WoL is enabled in BIOS/OS, and confirm your subnet really is `/24` (so `.255` is the correct broadcast).

### D. Optional — Auto Start on Boot
Install the **[Termux:Boot](https://f-droid.org/it/packages/com.termux.boot/)** app (from F-Droid) and open it once so it gets permission. If you want wake-lock, also install the **[Termux:API](https://f-droid.org/it/packages/com.termux.api/)** app.
**Prereqs (once)**
```
pkg update -y
pkg install -y openssh
# Optional but recommended for wake-lock:
pkg install -y termux-api
```
**Create the script**
```
mkdir -p ~/.termux/boot
cat > ~/.termux/boot/start-wol.sh << 'EOF'
#!/data/data/com.termux/files/usr/bin/bash
sleep 10
# Keep device awake if termux-api is available (optional)
if command -v termux-wake-lock >/dev/null 2>&1; then
  termux-wake-lock
fi
pgrep -x sshd >/dev/null 2>&1 || sshd
EOF
chmod +x ~/.termux/boot/start-wol.sh
```

>⚠️ remove this piece of code from the one above ONLY if not needed
```
# Keep device awake if termux-api is available (optional)
if command -v termux-wake-lock >/dev/null 2>&1; then
  termux-wake-lock
fi
```
**🧪 Test without reboot**
```
~/.termux/boot/start-wol.sh
ps -A | grep sshd   # should show sshd
```
If you see sshd, you’re good. Now reboot the phone and verify you can SSH on port 8022.

**Notes**
- If `termux-wake-lock` says command not found, that’s fine — it just means you didn’t install Termux:API; the script already handles that.
- Make sure **battery optimization is disabled** for Termux and Termux:Boot, and Wi-Fi “Keep on during sleep” is set to **Always**.
- Your WoL script (`~/wolpc.sh`) stays separate; you’ll trigger it over SSH (or add a line in the boot script if you want a web endpoint).

**Do you need autostart on boot?
**No** — if you don’t reboot the old phone often, you can just:
`sshd`
- in Termux once, and leave it running forever.
- If the phone stays plugged in, on Wi-Fi, and you never restart it → you can skip the whole boot script thing.

---

**⚠️ Variant (Replaces the code from above) otherwise ignore and go to step 2**
### ⚙️IF YOU DONT WANT TERMUX:API Step-by-step setup
Run these exact commands in Termux:
```
mkdir -p ~/.termux/boot
cat > ~/.termux/boot/start-wol.sh << 'EOF'
#!/data/data/com.termux/files/usr/bin/bash
# Smart autostart script for Termux:Boot
# Keeps device awake + ensures SSH is running

# Wait a bit for Wi-Fi to come up after boot
sleep 10

# Acquire a simple wake-lock so Termux doesn't get killed
# This method works even without the Termux:API app
echo "Termux boot script started at $(date)" >> ~/bootlog.txt
if [ -w /sys/power/wake_lock ]; then
  echo "termux_wake_lock" > /sys/power/wake_lock 2>/dev/null || true
fi

# Start SSH daemon if not already running
pgrep -x sshd >/dev/null 2>&1 || sshd

# Optional: log SSH start
echo "sshd started at $(date)" >> ~/bootlog.txt
EOF

chmod +x ~/.termux/boot/start-wol.sh
```
### 🧠 What this script does
- ✅ **Waits 10 seconds** → lets Wi-Fi fully connect after boot
- ✅ **Keeps Termux awake** → by writing directly to `/sys/power/wake_lock` (no extra apps needed)
- ✅ **Starts SSH** automatically if it’s not already running
- 📝 **Logs events** into `~/bootlog.txt`, so you can check if it actually started after reboot

**🧾 Optional: verify after reboot**

After the phone reboots:
`cat ~/bootlog.txt`

You should see timestamps like:
>*Termux boot script started at Mon Oct 21 12:34:56*
>*sshd started at Mon Oct 21 12:34:58*

---
**⚡ Bonus tip**
If you later decide to install Termux:API and want to use its official wake-lock command (which is cleaner),
 just replace this section in the script:
```
if [ -w /sys/power/wake_lock ]; then
  echo "termux_wake_lock" > /sys/power/wake_lock 2>/dev/null || true
fi
```
with
```
termux-wake-lock
```

# 📱 Step 2 — Set Up Your Main Phone
## 🔑 1. Info you need from your old phone
### On the old Android (running Termux + Tailscale), note down:
- **Tailscale IP** → run `tailscale ip -4` (looks like `100.x.y.z`).
- **Username** in Termux → run `whoami` (looks like `u0_a123`).
- **Password** → the one you set earlier with `passwd` in Termux.
- **SSH port** → Termux uses `8022` by default (not 22).

### Install Tailscale (same account).
Get your old phone’s Tailscale IP:
`tailscale ip -4`
Example: 100.101.50.77

- Install **Termius** and create a new host:

|   Field  | Value                                      |
|:--------:|--------------------------------------------|
| Label    | Old Phone WOL                              |
| Address  | 100.x.y.z (From tailscale)                 |
| Port     | 8022                                       |
| Username | Output of whoami in Termux (e.g., u0_a123) |
| Password | Password you set in Termux                 |

Tap **Save**, then **Connect**. You should get a Termux shell remotely.

# ⚡ Step 3 — One-Tap Wake Shortcut
In **Termius** → **Vaults** → **Snippets**:

1. Add a new snippet called **Wake PC**.
Command:
` bash ~/wolpc.sh`
2. Save
3. Attach it to your “Old Phone WOL” host.

Or go back to Connections, tap your host (old phone), then you can attach the snippet to it.
Next time, instead of typing, you just tap “Wake PC” and it runs automatically.

### 🚀 Result

Now, from **anywhere** (Wi-Fi, 4G, another country):

Open Termius → tap “**Wake PC**” → SSH connects to your old phone via Tailscale → it runs `wolpc.sh` → your PC powers on. ✅

### 🔐 (Optional) Use SSH Keys Instead of Passwords
In Termius on your main phone → generate SSH key → copy **public key**.

On the old phone (Termux):
```
mkdir -p ~/.ssh
chmod 700 ~/.ssh
echo "PASTE_PUBLIC_KEY_HERE" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```
In Termius, assign the private key to your host.
Now it connects instantly without asking for a password.

### 🧩 Troubleshooting

|              Problem              | Fix                                                                                        |
|:---------------------------------:|--------------------------------------------------------------------------------------------|
| Termius says "connection refused" | SSH not running -> run sshd in Termux                                                      |
| Timeout                           | Ensure both device are "Connected" in Tailscale                                            |
| WoL doesn't work                  | Check BIOS/OS settings, correct MAC/BCAST, ensure PC supports waking from that power state |

### 🧰 Commands Cheat Sheet

|       Action       | Command           |
|:------------------:|-------------------|
| Start SSH manually | sshd              |
| Check SSH          | `ps -A            |
| Run WoL            | ./wolpc.sh        |
| Edit script        | nano ~/wolpc.sh   |
| Get username       | whoami            |
| Get Tailscale IP   | tailscale ip -4   |

> Important: If you rebooted the old phone and didn’t set up autostart, you have to run sshd manually each time in termux.