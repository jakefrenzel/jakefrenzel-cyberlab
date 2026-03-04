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
