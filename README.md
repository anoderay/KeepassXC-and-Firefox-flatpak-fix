# Getting Firefox and KeePassXC flatpaks to talk!
Fix to get KeepassXC and Firefox to communicate via native messaging when running both as flatpak and using the KeePassXC browser extension. 

**IMPORTANT CAVEAT**: For this fix to work KeePassXC has to be launched before Firefox.

````
#!/bin/bash

# IMPORTANT CAVEAT: For this fix to work KeePassXC has to be launched before Firefox.
# Taken and translated from https://linuxnews.de/keepassxc-2-7-10/

# Set flatpak-override, to connect KeePassXC and Firefox
echo "Set Flatpak-Override, to connect KeePassXC to Firefox..."
flatpak override --user \
--filesystem={/var/lib,xdg-data}/flatpak/{app/org.keepassxc.KeePassXC,runtime/org.kde.Platform}:ro \
--filesystem=xdg-run/app/org.keepassxc.KeePassXC:create \
org.mozilla.firefox
echo "Flatpak-Override set."

# Create wrapper-script for keepassxc-proxy...
echo "Create Wrapper-Script for keepassxc-proxy..."
WRAPPER_DIR="$HOME/.var/app/org.mozilla.firefox/.local/bin"
WRAPPER_SCRIPT="$WRAPPER_DIR/keepassxc-proxy-wrapper.sh"
mkdir -p "$WRAPPER_DIR"

echo "writing wrapper-script..."
cat << 'EOF' > "$WRAPPER_SCRIPT"
#!/bin/bash
APP_REF="org.keepassxc.KeePassXC/x86_64/stable"
for inst in "$HOME/.local/share/flatpak" "/var/lib/flatpak"; do
    if [ -d "$inst/app/$APP_REF" ]; then
        FLATPAK_INST="$inst"
        break
    fi
done
[ -z "$FLATPAK_INST" ] && exit 1
APP_PATH="$FLATPAK_INST/app/$APP_REF/active"
RUNTIME_REF=$(awk -F'=' '$1=="runtime" { print $2 }' < "$APP_PATH/metadata")
RUNTIME_PATH="$FLATPAK_INST/runtime/$RUNTIME_REF/active"
exec flatpak-spawn \
    --env=LD_LIBRARY_PATH=/app/lib \
    --app-path="$APP_PATH/files" \
    --usr-path="$RUNTIME_PATH/files" \
    -- keepassxc-proxy "$@"
EOF
echo "wrapper-script created"

# Make wrapper-script executable
echo "Setze Berechtigungen für Wrapper-Skript..."
chmod +x "$WRAPPER_SCRIPT"
echo "Wrapper-Skript ist nun ausführbar."

# Create native messaging host JSON
echo "Create native messaging host JSON-File for Firefox..."
NATIVE_MESSAGING_DIR="$HOME/.var/app/org.mozilla.firefox/.mozilla/native-messaging-hosts"
NATIVE_MESSAGING_FILE="$NATIVE_MESSAGING_DIR/org.keepassxc.keepassxc_browser.json"
mkdir -p "$NATIVE_MESSAGING_DIR"

echo "Writing native messaging JSON-File..."
cat <<EOF > "$NATIVE_MESSAGING_FILE"
{
    "allowed_extensions": [
        "keepassxc-browser@keepassxc.org"
    ],
    "description": "KeePassXC integration with native messaging support",
    "name": "org.keepassxc.keepassxc_browser",
    "path": "$WRAPPER_SCRIPT",
    "type": "stdio"
}
EOF
echo "Native messaging JSON-File created"

echo "Setup completed. Please restart KeePassXC and Firefox to apply the changes."

````
