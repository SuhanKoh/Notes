# Create EC2 instance and install OpenVPN

### Link used
- https://www.comparitech.com/blog/vpn-privacy/how-to-make-your-own-free-vpn-using-amazon-web-services/#installopenvpn
---

### Steps
- Go to AWS management dashboard, select *EC2 Instance*
- Click on *Launch Instance*
- Select your AMI (Images, basically OS.)
- Check your instance tier
- Open VPN runs on UDP 1194, so add Custom UDP and select 1194. Allow source from anywhere (Assuming you don't need to whitelist IPs).

## *** Assuming you are creating keypair ***
- Download keypair, save it somewhere safe and "chmod 400 [name].pem" file (Must do this to ssh into the instance.)
- SSH into the instance
  ```
  ssh -i [name].pem [username]@[instance public dns]
  ```
- To update AWS instance, run
  ```
  sudo yum update
  ```
- Paste the following command to set up RSA keys
  ```
  sudo yum install easy-rsa -y --enablerepo=epel
  sudo cp -via /usr/share/easy-rsa/2.0 CA
  ```
- Use root user
  ```
  sudo su
  ```
- Change directory into 'CA'
- Run the following commands:
  ```source ./vars
  ./clean-all
  ./build-ca
  ./build-key-server server
  ./build-dh 2048
  ```
- Generate key for your client(s)
  ```
  ./build-key [name]
  ```
### Generate key for OpenVPN
- Run:
  ```
  cd keys
  openvpn --genkey --secret pfs.key
  ```
  ```
  mkdir /etc/openvpn/keys
  ```
  ```
  for file in server.crt server.key ca.crt dh2048.pem pfs.key; do cp $file /etc/openvpn/keys/; done
  ```
- Set up OpenVPN
  ```
  cd /etc/openvpn
  nano server.conf
  ```
- Copy the following:
  ```
  port 1194
  proto udp
  dev tun
  ca /etc/openvpn/keys/ca.crt
  cert /etc/openvpn/keys/server.crt
  key /etc/openvpn/keys/server.key # This file should be kept secret
  dh /etc/openvpn/keys/dh2048.pem
  cipher AES-256-CBC
  auth SHA512
  server 10.8.0.0 255.255.255.0
  push "redirect-gateway def1 bypass-dhcp"
  push "dhcp-option DNS 8.8.8.8"
  push "dhcp-option DNS 8.8.4.4"
  ifconfig-pool-persist ipp.txt
  keepalive 10 120
  comp-lzo
  persist-key
  persist-tun
  status openvpn-status.log
  log-append  openvpn.log
  verb 3
  tls-server
  tls-auth /etc/openvpn/keys/pfs.key
  ```

### Start OpenVPN
- Run
  ```
  sudo service openvpn start
  ```
- Copy keys out
  ```
  cd /usr/share/easy-rsa/2.0
  chmod 777 keys
  ```
  ```
  cd keys  
  for file in client.crt client.key ca.crt dh2048.pem pfs.key ca.key; do sudo chmod 777 $file; done
  ```
- New command prompt (To download keys out)  
  ``` 
   scp -i [name].pem -r [username]@[instance public dns]:/usr/share/easy-rsa/2.0/keys ./[localpath]
  ```
  
### !! IMPORTANT !!
- Change permission back!
  ```
  for file in client.crt client.key ca.crt dh2048.pem pfs.key; do sudo chmod 600 $file; done
  ```
  - Remove ca.key from the server!
  ```
  rm ca.key
  ```
  ```
  // set 600 for keys folder.
  cd ..
  chmod 600 keys
  ```

# How to VPN!
- Use TunnelBlick on mac
- Load [sample_config.ovpn]

# Add More Client To Your OpenVPN Server
- Redo ./build-key for client
- Set permission /keys/ in /usr/share/easy-rsa/2.0 to 777
- Copy ca.key back to /usr/share/easy-rsa/2.0/keys/
- Set client.crt client.key ca.crt dh2048.pem pfs.key ca.key to 777
- Copy files from /keys/ on the server to your machine
- Remove ca.key on server
- Set files permission back to 600
- Set /keys/ permission to 600