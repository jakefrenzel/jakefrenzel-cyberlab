# VPN - Firewall Encrypted Traffic

## Nord VPN
1. Navigate to your Nord VPN dashboard

2. Select `Setup Nord VPN Manually`

3. Select `Get Setup Config`

4. Choose OpenVPN and `Download UDP`

5. Also grab service credentials

## Add A Certificate Authority
```pfsense
System -> Certificates -> Authorities
```

1. Press `+Add`

2. Give a description of `NordVPN_CA`

3. Change the method to `Import and existing Certificate Authority`

4. Open the .ovpn file you downloaded and copy everything in between the `<ca></ca>` including the `---BEGIN CERT---`

5. Paste it in and save

## Add a VPN Client
```pfsense
VPN -> OpenVPN -> Clients
```

1. Press `+Add`

2. Set a description: `NordVPN`

3. Add server host/address, it should look like `us1234.nordvpn.com

4. Change port to `1194`

5. Keep everything else the same

6. Paste in the username and password from the service credentials you saved earlier

## Cryptographic Configuration

1. Disable automatic TLS key generation'

2. Paste key there just like before using the text between `<tls-auth>` excluding the #'s

3. Set the key direction to 1

4. For **Peer Certificate Authority** select `NordVPN_CA`

5. Switch **Auth Digest Algorithm** to `SHA512`

6. Save

## Verify
```pfsense
status -> OpenVPN
```

1. You should see **Connected (success)**
