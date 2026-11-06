# setup




cat > ~/setup.sh <<'SCRIPT'
set -e
say(){ echo; echo "👉 $*"; }
say "Installing everything. This takes a few minutes. Type your VM password if asked (it won't show, that's normal)."
sudo apt update -qq && sudo apt upgrade -y -qq
sudo apt install -y -qq git curl ca-certificates openssl nano
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash - >/dev/null
sudo apt install -y -qq nodejs
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main' | sudo tee /etc/apt/sources.list.d/cloudflared.list >/dev/null
sudo apt update -qq && sudo apt install -y -qq cloudflared
mkdir -p ~/.ssh && chmod 700 ~/.ssh
[ -f ~/.ssh/streamertts ] || ssh-keygen -q -t ed25519 -C streamertts-proxmox -f ~/.ssh/streamertts -N ""
grep -q 'IdentityFile ~/.ssh/streamertts' ~/.ssh/config 2>/dev/null || printf 'Host github.com\n  IdentityFile ~/.ssh/streamertts\n  IdentitiesOnly yes\n' >> ~/.ssh/config
ssh-keyscan github.com >> ~/.ssh/known_hosts 2>/dev/null
clear
echo "========== STEP 1 of 3 =========="
echo "Copy this line:"
echo
cat ~/.ssh/streamertts.pub
echo
echo "Then on GitHub: your streamertts repo > Settings > Deploy keys > Add deploy key"
echo "Name: proxmox. Paste the line. Leave 'Allow write access' OFF. Click Add key."
until ssh -T git@github.com 2>&1 | grep -q successfully; do read -p "Press Enter when done... " < /dev/tty; done
say "GitHub works. Downloading StreamerTTS..."
[ -d ~/streamertts ] || git clone -q git@github.com:KylieCreate/streamertts.git ~/streamertts
cd ~/streamertts && npm install --silent
clear
echo "========== STEP 2 of 3 =========="
echo "On your LAPTOP, open a terminal and run:"
echo
echo "   sudo systemctl cat cloudflared | grep -o 'eyJ[A-Za-z0-9_=-]*'"
echo
read -p "Paste the long eyJ... code it prints here, then press Enter: " TUNNEL < /dev/tty
clear
echo "========== STEP 3 of 3 =========="
echo "On your LAPTOP, open the .env file in your StreamerTTS folder and copy ALL of it."
echo "When you press Enter, an editor opens here. Paste it in,"
echo "then press Ctrl+O, Enter, Ctrl+X to save and close."
read -p "Press Enter to open the editor... " < /dev/tty
nano .env < /dev/tty
SYNC=$(openssl rand -hex 24)
sed -i '/^\(NODE_NAME\|SYNC_TOKEN\|TUNNEL_TOKEN\|TTS_ERROR_LOG_GITHUB\)=/d' .env
printf '\nNODE_NAME=proxmox\nSYNC_TOKEN=%s\nTUNNEL_TOKEN=%s\nTTS_ERROR_LOG_GITHUB=0\n' "$SYNC" "$TUNNEL" >> .env
chmod 600 .env
sudo tee /etc/systemd/system/streamertts.service >/dev/null <<EOF
[Unit]
Description=StreamerTTS
After=network-online.target
Wants=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/streamertts
ExecStart=$(which node) failover.js primary
Restart=always
RestartSec=5
TimeoutStopSec=40
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload && sudo systemctl enable -q streamertts
clear
echo "✅ ALL DONE. The VM is ready."
echo
echo "Save this password, your laptop needs it:"
echo
echo "   $SYNC"
SCRIPT
bash ~/setup.sh
