#!/bin/bash

# Quick Install Script for Proxy
# Usage: curl -s URL | sudo bash -s [squid|dante|all] username password

TYPE=$1
USERNAME=$2
PASSWORD=$3

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

# Check root
if [[ $EUID -ne 0 ]]; then
    echo -e "${RED}Cần quyền root!${NC}"
    exit 1
fi

# Check parameters
if [ -z "$TYPE" ] || [ -z "$USERNAME" ] || [ -z "$PASSWORD" ]; then
    echo -e "${RED}Sử dụng: $0 [squid|dante|all] username password${NC}"
    exit 1
fi

# Get public IP
IP=$(curl -s ifconfig.me || hostname -I | awk '{print $1}')

# Install Squid
install_squid() {
    echo -e "${YELLOW}Cài đặt Squid HTTP Proxy...${NC}"
    apt update && apt upgrade -y
    apt install -y squid apache2-utils
    
    # Backup
    cp /etc/squid/squid.conf /etc/squid/squid.conf.backup
    
    # Configure
    sed -i '/# INSERT YOUR OWN RULE(S) HERE TO ALLOW ACCESS FROM YOUR CLIENTS/a\
auth_param basic program /usr/lib/squid/basic_ncsa_auth /etc/squid/passwords\
auth_param basic realm proxy\
acl authenticated proxy_auth REQUIRED' /etc/squid/squid.conf
    
    sed -i '/http_access deny all/i\
http_access allow localhost\
http_access allow authenticated' /etc/squid/squid.conf
    
    # Create user
    htpasswd -cb /etc/squid/passwords "$USERNAME" "$PASSWORD"
    
    # Start service
    ufw allow 3128/tcp
    systemctl restart squid
    systemctl enable squid
    
    echo -e "${GREEN}✓ Squid đã cài đặt!${NC}"
    echo -e "${GREEN}Proxy: $IP:3128:$USERNAME:$PASSWORD${NC}"
}

# Install Dante
install_dante() {
    echo -e "${YELLOW}Cài đặt Dante SOCKS5 Proxy...${NC}"
    apt update
    apt install -y dante-server
    
    # Get interface
    INTERFACE=$(ip route | grep default | awk '{print $5}' | head -1)
    
    # Configure
    cat > /etc/danted.conf << EOF
logoutput: syslog
user.privileged: root
user.notprivileged: nobody

internal: 0.0.0.0 port=1080
external: $INTERFACE

socksmethod: username
clientmethod: none

client pass {
    from: 0.0.0.0/0 to: 0.0.0.0/0
    log: error connect disconnect
}

socks pass {
    from: 0.0.0.0/0 to: 0.0.0.0/0
    log: error connect disconnect
}
EOF
    
    # Create user
    useradd -r -s /bin/false "$USERNAME"
    echo "$USERNAME:$PASSWORD" | chpasswd
    
    # Start service
    ufw allow 1080/tcp
    systemctl restart danted
    systemctl enable danted
    
    echo -e "${GREEN}✓ Dante đã cài đặt!${NC}"
    echo -e "${GREEN}Proxy: $IP:1080:$USERNAME:$PASSWORD${NC}"
}

# Main
ufw --force enable

case $TYPE in
    squid)
        install_squid
        ;;
    dante)
        install_dante
        ;;
    all)
        install_squid
        echo ""
        install_dante
        ;;
    *)
        echo -e "${RED}Type không hợp lệ! Chọn: squid, dante, hoặc all${NC}"
        exit 1
        ;;
esac

echo -e "${YELLOW}Check proxy tại: https://proxy6.net/checker${NC}"
