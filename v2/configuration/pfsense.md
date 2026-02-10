# Initial pfSense Configuration

## Access the Web GUI

1. Open a browser and go to the default IP address of the pfSense firewall.

```pfsense
192.168.1.1
```

2. Sign in with default credentials: admin | pfsense
3. Go to step 2 in the pfSense setup and add DNS primary DNS server
```pfsense
1.1.1.1
```
4. Add a secondary DNS
```pfsense
8.8.8.8
```
5. Uncheck overide DNS

> You can also change the hostname and domain if you would like

## Configure WAN Interface
1. Set time zone and go to step 4 of 9
2. Make sure WAN interface type is DHCP

## Configure LAN Interface
1. Switch LAN IP address
```pfsense
10.27.10.1
```

## Set Admin Password
1. Go to step 6 of 9
2. Set a **secure** admin password
3. Click next and finish!

> You will now have to access the web GUI from the new IP address
```pfsense
10.27.20.1
```
