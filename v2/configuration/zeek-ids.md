# Intrusion Detection System - Zeek

## Prerequisite
- [x] Complete installation, testing, and configuration for [Suricata IDS](suricata-ids.md)

## Zeek Setup
1. Add noble updates to upgrade mismatch versions
```shell
sudo nano /etc/apt/sources.list.d/ubuntu.sources
```

2. In the first section where it says `Suites: noble` switch it to `Suites: noble noble-updates`

3. Save and exit

4. Re-sync the system
```shell
sudo apt update
```
```shell
sudo apt full-upgrade -y
```
```shell
sudo apt autoremove -y
```
```shell
sudo apt autoclean
```
```shell
sudo apt upgrade
```

5. Install the dependencies
```shell
sudo apt install -y \
  cmake make gcc g++ flex bison \
  libpcap-dev libssl-dev \
  zlib1g-dev python3-dev \
  libmaxminddb-dev \
  libsqlite3-dev git curl
```

6. Go to source directory
```shell
cd /usr/local/src
```

7. Install swig
```shell
sudo apt install swig
```

8. Download Zeek
```shell
sudo curl -LO https://download.zeek.org/zeek-8.0.6.tar.gz
sudo tar -xzf zeek-8.0.6.tar.gz
sudo chown -R $USER:$USER zeek-8.0.6
cd zeek-8.0.6

./configure
```

9. Compile it
```shell
make -j4
```

10. Install
```shell
sudo make install
```

11. Add to path
```shell
echo 'export PATH=/usr/local/zeek/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

12. Verify
```shell
zeek --version
```

13. Run a quick test
```shell
sudo zeek -i eth0
```

14. Create Zeek as a service
```shell
sudo nano /etc/systemd/system/zeek.service
```
15. Paste and save
```shell
[Unit]
Description=Zeek Network Security Monitor
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/zeek/bin/zeekctl deploy
ExecStop=/usr/local/zeek/bin/zeekctl stop
ExecReload=/usr/local/zeek/bin/zeekctl deploy
User=root

[Install]
WantedBy=multi-user.target
```

16. Enable it
```shell
sudo systemctl daemon-reload
```
```shell
sudo systemctl enable zeek
```

17. Start the service
```shell
sudo systemctl start zeek
```

18. Check status
```shell
sudo systemctl status zeek
```

### Enable JSON logging in Zeek
1. Edit the config file
```shell
sudo nano /usr/local/zeek/share/zeek/site/local.zeek
```

2. Add to the bottom of the file
```shell
@load policy/tuning/json-logs.zeek
```

3. Add this as well for Zeek Intel Framework (We set this up later)
```shell
@load frameworks/intel/seen
@load frameworks/intel/do_notice
redef Intel::read_files += { "/usr/local/zeek/share/zeek/site/intel/malicious_ips.dat" };
```

3. Redeploy
```shell
sudo /usr/local/zeek/bin/zeekctl deploy
```

### Enable Zeek Intel Framework
1. Create Intel folder
```shell
sudo mkdir -p /usr/local/zeek/share/zeek/site/intel
```

2. Create first threat feed
```shell
sudo nano /usr/local/zeek/share/zeek/site/intel/malicious_ips.dat
```

3. Add text for test intel indicator
```shell
#fields indicator indicator_type meta.source
8.8.8.8 Intel::ADDR test-feed
```

4. Redeploy
```shell
sudo /usr/local/zeek/bin/zeekctl deploy
```

5. Run a test and verify it's working
```shell
curl http://8.8.8.8
```

6. Then check for: `intel.log` and `notice.log`
> You may only see notice.log in this test scenario
```shell
ls /usr/local/zeek/logs/current
```

Future:
Later you can add real threat feeds like:
- EmergingThreats
- AbuseIPDB
- AlienVault OTX
- Malware domains

🧠 Zeek Scan Detection Framework
This lets Zeek detect:
- Nmap scans
- host discovery scans
- horizontal scans
- vertical scans
