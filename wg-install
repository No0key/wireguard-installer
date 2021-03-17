#!/bin/bash

if [[ $(whoami) != "root" ]]
then
        echo "Run script as root"
        exit 1
fi

check_prerequisites(){
        if [[ -d /etc/wireguard ]]
        then
                echo "Wireguard server already installed."
                exit 0
        else
                mkdir /etc/wireguard
                mkdir /opt/wireguard_clients
        fi
}

#Return IP address for new client
get_new_ip_addr(){
    LAST_IP_ADDRESS=$1
    FIRST_OCTET=$(echo "${LAST_IP_ADDRESS}" | awk -F. '{print $1}')
    SECOND_OCTET=$(echo "${LAST_IP_ADDRESS}" | awk -F. '{print $2}')
    THIRD_OCTET=$(echo "${LAST_IP_ADDRESS}" | awk -F. '{print $3}')
    LAST_OCTET=$(echo "${LAST_IP_ADDRESS}" | awk -F. '{print $4}')
    if [[ ${LAST_OCTET} == 250 ]]
    then
        FIRST_OCTET=$(echo "${LAST_IP_ADDRESS}" | awk -F. '{print $1}')
        SECOND_OCTET=$(echo "${LAST_IP_ADDRESS}" | awk -F. '{print $2}')
        THIRD_OCTET=$((THIRD_OCTET+1))
        LAST_OCTET=1
        echo "${FIRST_OCTET}.${SECOND_OCTET}.${THIRD_OCTET}.${LAST_OCTET}"
    else
        LAST_OCTET=$((LAST_OCTET+1))
        echo "${FIRST_OCTET}.${SECOND_OCTET}.${THIRD_OCTET}.${LAST_OCTET}"
    fi
}

create_new_client_config(){
    if [[ ! -d /etc/wireguard ]]
     then
                echo "Wireguard server not installed."
                exit 0
    fi
    until [[ "${CLIENT_NAME}" =~ ^[a-zA-Z]{1}\.[a-zA-Z]{1,10}$ ]]
    do
            read -rp "Enter client name[a-zA-Z]{1}.[a-zA-Z]{1,10}. Example - b.obama : " CLIENT_NAME
    done
    NEW_CLIENT_PRIV_KEY=$(wg genkey)
    NEW_CLIENT_PUBLIC_KEY=$(echo "${NEW_CLIENT_PRIV_KEY}" | wg pubkey)
    SRV_PUB_KEY=$(cat /etc/wireguard/server_pubkey)
    LAST_PEER_ADDRESS=$(grep -oE "[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}" /etc/wireguard/wg0.conf  | tail -1)
    NEW_PEER_ADDRESS="$(get_new_ip_addr "${LAST_PEER_ADDRESS}")"
    GLOBAL_IP=$(curl ifconfig.me)
    echo "[Peer]
# Client - ${CLIENT_NAME}
PublicKey = ${NEW_CLIENT_PUBLIC_KEY}
AllowedIPs = ${NEW_PEER_ADDRESS}/32" >> /etc/wireguard/wg0.conf
    #reload wg0 interface after editing wg0.conf
    /bin/wg syncconf wg0 <(wg-quick strip wg0)
    echo -e "[Interface]
#Client config
Address = ${NEW_PEER_ADDRESS}/16
PrivateKey = ${NEW_CLIENT_PRIV_KEY}
DNS = 84.200.70.40, 1.1.1.1
[Peer]
#Server config
PublicKey = ${SRV_PUB_KEY}
Endpoint = ${GLOBAL_IP}:51820
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25" > /opt/wireguard_clients/"${CLIENT_NAME}".conf

    echo "New configuration file ${CLIENT_NAME}.conf successfully created in /opt/wireguard_clients/"
}

#Install wg server if not installed
install_wireguard_server() {
        check_prerequisites

        echo "Checking that kernel module is loaded..."

        if [[ $(lsmod | grep wireguard) != "0" ]]
        then
                modprobe wireguard &> /dev/null
                echo -e "\nWireguard kernel module loaded."
        fi

        echo "___________________"
        echo "Installing packages"
        echo "___________________"
        /bin/apt-get update && /bin/apt-get install -y wireguard wireguard-tools iptables iproute2 curl &> /dev/null


        echo "Generating private and public keys"
        /bin/wg genkey | tee /etc/wireguard/server_privatekey | /bin/wg pubkey > /etc/wireguard/server_pubkey
        /bin/chmod 600 -R /etc/wireguard/

        #Enable IP forwarding
        if [[ $(cat /proc/sys/net/ipv4/ip_forward) != "1" ]]
        then
                echo -e "\nEnabling IPv4 forwarding "
                sysctl -w net.ipv4.ip_forward=1
                echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
        fi
    SRV_PRIV_KEY=$(cat /etc/wireguard/server_privatekey)
    SRV_PUB_KEY=$(cat /etc/wireguard/server_pubkey)
    SRV_IFACE_NAME=$(ip -o -4 route show to default | awk '{print $5}')
    echo -e "[Interface]
Address = 192.168.0.1/16
SaveConfig = true
ListenPort = 51820
PrivateKey = ${SRV_PRIV_KEY}
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o ${SRV_IFACE_NAME} -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o ${SRV_IFACE_NAME} -j MASQUERADE" > /etc/wireguard/wg0.conf
    wg-quick up wg0
    systemctl enable wg-quick@wg0.service
    echo "___________________________________________________"
    echo "Wireguard server successfully installed and running"
}


echo "______________________________"
echo "1) Install wireguard server."
echo "2) Add new client config file."
echo "3) Remove client"
echo "4) Exit."
echo "______________________________"
until [[ ${ACTION_BUTTON} =~ ^[1-4]$ ]]
do
        read -rp "Select an option [1-4]: " ACTION_BUTTON
done

case "${ACTION_BUTTON}" in
1)
        install_wireguard_server
        ;;
2)
    create_new_client_config
        ;;
3)
    #remove_client
    ;;
4)
        exit 0
        ;;
esac

#TODO
#1) Menu entry to remove client
#2) Recursive DNS resolver
