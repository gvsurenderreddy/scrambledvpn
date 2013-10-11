# This repo provides a patched version of openvpn, that needs to be 
# compiled again
# From Web post http://scramblevpn.wordpress.com/
# To create an openvpn server which is patched on Raspberry PI

sudo apt-get update
sudo apt-get install gcc make automake autoconf dh-autoreconf file patch perl dh-make debhelper devscripts gnupg lintian quilt libtool pkg-config libssl-dev liblzo2-dev libpam0g-dev libpkcs11-helper1-dev -y


cd $HOME
git clone https://github.com/john564/scrambledvpn.git
cd scrambledvpn
sudo cp -rf ./openvpn /etc/openvpn/
cd /etc/openvpn/
sudo autoreconf -i -v -f
sudo ./configure --prefix=/usr
sudo make
sudo make install
sudo wget https://gist.github.com/john564/6765292/raw/0a97df1237a138a5a941bbec45b6cd41e973f840/etc+init.d+openvpn -O /etc/init.d/openvpn
sudo chmod +x /etc/init.d/openvpn
sudo update-rc.d openvpn defaults

# Now we set up the server keys and certs
# TIP: You must answer y to Sign the certificate? [y/n]:y
# TIP: You must answer y to commit? [y/n]y
# everything else just keep pressing return
cd /etc/openvpn
sudo git clone git://github.com/OpenVPN/easy-rsa.git
sudo cp -r ./easy-rsa/easy-rsa/2.0/* ./easy-rsa/
sudo chown -R $USER /etc/openvpn/easy-rsa/
cd /etc/openvpn/easy-rsa/
source vars
./clean-all
./build-ca
./build-key-server server
./build-dh
./build-key client1

cd /etc/openvpn/easy-rsa/keys
sudo cp ca.crt ca.key dh2048.pem server.crt server.key /etc/openvpn
sudo mkdir $HOME/openvpn-client-files
sudo cp ca.crt client1.crt client1.key $HOME/openvpn-client-files
sudo openvpn --genkey --secret /etc/openvpn/ta.key
sudo cp /etc/openvpn/ta.key $HOME/openvpn-client-files

# Now we create the OpenVPN client configuration on the Raspberry PI
sudo nano $HOME/openvpn-client-files/raspberrypi-client-scrambled.ovpn


    client
    dev tun
    scramble obfuscate test
    proto udp
    remote change_this_to_server_address 34557
    resolv-retry infinite
    nobind
    persist-key
    persist-tun
    ca ca.crt
    cert client1.crt
    key client1.key
    tls-auth ta.key 1
    ns-cert-type server
    cipher AES-256-CBC
    comp-lzo
    verb 3

# Now we merge client certs and keys into the client script
sudo wget https://gist.github.com/john564/6763098/raw/9e3e42fc9c171e238a08c62a64cf2e0ec5c50c73/combine.sh -O $HOME/openvpn-client-files/combine.sh
cd $HOME/openvpn-client-files/
sudo chmod +x $HOME/openvpn-client-files/combine.sh
sudo $HOME/openvpn-client-files/combine.sh
sudo chown $USER $HOME/openvpn-client-files/raspberrypi-client-scrambled.ovpn

# Now transfer combined client script raspberrypi-client-scrambled.ovpn
# in $HOME/openvpn-client-files to your client PC
# Due to permissions, I had to transfer it to C:\
# then in windows, copy the file(s)
# to C:\Program Files (x86)\OpenVPN\config
# or windows 32bit
# C:\Program Files\OpenVPN\config

# Back to Raspberry PI, Now we create file for server config
# Below is my OpenVPN server configuration saved as /etc/openvpn/server.conf
sudo nano /etc/openvpn/server.conf

    port 34557
    proto udp
    dev tun
    scramble obfuscate test
    ca ca.crt
    cert server.crt
    key server.key
    tls-auth ta.key 0
    dh dh2048.pem
    server 10.8.0.0 255.255.255.0
    cipher AES-256-CBC
    comp-lzo
    persist-key
    persist-tun
    user nobody
    group nogroup
    status openvpn-status.log
    verb 3
    tun-mtu 1500
    tun-mtu-extra 32
    mssfix 1450
    push "redirect-gateway def1"
    push "dhcp-option DNS 208.67.222.222"
    push "dhcp-option DNS 208.67.220.220"
    keepalive 5 30

# uncomment to allow data redirect
sudo nano /etc/sysctl.conf

    net.ipv4.ip_forward=1

# Make file for firewall setting
sudo nano /usr/local/bin/firewall.sh

    #!/bin/bash
    iptables -t filter -F
    iptables -t nat -F
    iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
    iptables -A FORWARD -s "10.8.0.0/24" -j ACCEPT
    iptables -A FORWARD -j REJECT
    iptables -t nat -A POSTROUTING -s "10.8.0.0/24" -j MASQUERADE

# Make firewall script executable, run it and check
sudo chmod +x /usr/local/bin/firewall.sh
sudo /usr/local/bin/firewall.sh
sudo iptables --list

# add new text line into file /etc/rc.local
# before ‘exit 0′ to ensure the firewall rules are run at reboot or power up.
sudo nano /etc/rc.local

    /usr/local/bin/firewall.sh

# reboot the pi
sudo reboot
