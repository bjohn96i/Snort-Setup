# Snort-Setup

This is the setup process for Snort 3 on Ubuntu 24.04

## Pre-Requisites

_Ensure your Linux Distro is updated and the following packages are installed_

```
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install libhwloc15=2.9.0-1
sudo apt-get install -y build-essential autotools-dev libdumbnet-dev libluajit-5.1-dev libpcap-dev zlib1g-dev pkg-config libhwloc-dev cmake liblzma-dev openssl libssl-dev cpputest libsqlite3-dev libtool uuid-dev git autoconf bison flex libcmocka-dev libnetfilter-queue-dev libunwind-dev libmnl-dev ethtool libjemalloc-dev

# We Are Also Going To Create an Directory To Download and Build All The Components for Snort
mkdir ~/snort
```

## Installation

### 1. Install pcre

```
cd ~/snort 
wget https://sourceforge.net/projects/pcre/files/pcre/8.45/pcre-8.45.tar.gz
tar -xzvf pcre-8.45.tar.gz
cd pcre-8.45
./configure 
make 
sudo make install
```

### 2. Install gperftools

```
cd ~/snort 
wget https://github.com/gperftools/gperftools/releases/download/gperftools-2.9.1/gperftools-2.9.1.tar.gz
tar xzvf gperftools-2.9.1.tar.gz
cd gperftools-2.9.1 
./configure 
make 
sudo make install
```

### 3. Install Ragel

```
cd ~/snort 
wget http://www.colm.net/files/ragel/ragel-6.10.tar.gz
tar -xzvf ragel-6.10.tar.gz 
cd ragel-6.10 
./configure 
make 
sudo make install
```

### 4. Download, but don’t install, Boost C++ Libraries

```
cd ~/snort 
wget https://boostorg.jfrog.io/artifactory/main/release/1.77.0/source/boost_1_77_0.tar.gz
tar -xvzf boost_1_77_0.tar.gz
```

### 5. Install Hyperscan

```
cd ~/snort 
wget https://github.com/intel/hyperscan/archive/refs/tags/v5.4.2.tar.gz 
tar -xvzf v5.4.2.tar.gz
mkdir ~/snort/hyperscan-5.4.2-build
cd hyperscan-5.4.2-build/
cmake -DCMAKE_INSTALL_PREFIX=/usr/local -DBOOST_ROOT=~/snort/boost_1_77_0/ ../hyperscan-5.4.2 
make 
sudo make install
```

### 6. Install flatbuffers - Were here!

```
cd ~/snort 
wget https://github.com/google/flatbuffers/archive/refs/tags/v2.0.0.tar.gz -O flatbuffers-v2.0.0.tar.gz
tar -xzvf flatbuffers-v2.0.0.tar.gz
mkdir flatbuffers-build
cd flatbuffers-build
cmake ../flatbuffers-2.0.0 
make 
sudo make install
```

### 7. Install Data Acquistion (DAQ) from Snort

```
cd ~/snort
wget https://github.com/snort3/libdaq/archive/refs/tags/v3.0.13.tar.gz
tar -xzvf v3.0.13.tar.gz
cd libdaq-3.0.13
./bootstrap
./configure
make
sudo make install
```

### 8. Update shared libraries

`sudo ldconfig`

### 9. Install Snort 3

```
cd ~/snort
wget https://github.com/snort3/snort3/archive/refs/tags/3.1.74.0.tar.gz -O snort3-3.1.74.0.tar.gz 
tar -xzvf snort3-3.1.74.0.tar.gz 
cd snort3-3.1.74.0 
./configure_cmake.sh --prefix=/usr/local --enable-jemalloc 
cd build 
make 
sudo make install
```

### 10. Test Snort3 Installation

`snort -V`
**Troubleshooting**: If the command is not found. It should be located in `/usr/local/bin/snort`

### 11. Let's validate the default config:

`snort -c /usr/local/etc/snort/snort.lua`

## Disable Generic and Large Receive Offloads

1. Get Network Adapter Name using `ip a`. It would look something like _eno18_
2. Check to see if GRO and LRO are enabled. If they are both disabled, skip to step 12.
   `sudo ethtool -k <network adapter name> | grep receive-offload`
3. Let's create a service to disable these kernels. ens18
   1. `sudo nano /lib/systemd/system/ethtool.service`
   2. Paste in the Script at the bottom of this section, replace the network adapter text, and save using `CTRL-X`
   3. Enable the service using `sudo systemctl enable ethtool.service`
   4. Start the service using `sudo systemctl start ethtool.service`
4. Validate GRO and LRO are now disabled.
   `sudo ethtool -k <network adapter> | grep receive-offload`
   ***
   > ethtool.service

```
[Unit] 
Description=Ethtool Configration for Network Interface 
[Service] 
Requires=network.target 
Type=oneshot 
ExecStart=/sbin/ethtool -K <network adapter> gro off 
ExecStart=/sbin/ethtool -K <network adapter> lro off 
[Install] 
WantedBy=multi-user.target
```

---

## Creating a Rule

### Create A Test Rule

1. Create a directory to store your rules
   `sudo mkdir /usr/local/etc/rules`
2. Create a rule file
   `sudo nano /usr/local/etc/rules/local.rules`
3. Lets create a rule that listens for pings
   `alert icmp any any -> any any ( msg:"ICMP Detected"; sid:1000001;)`
4. Save and exit Nano `Ctrl-X`
5. Validate rule creation
   `snort -c /usr/local/etc/snort/snort.lua -R /usr/local/etc/rules/local.rules`
6. Run snort and listen to traffic
   `sudo snort -c /usr/local/etc/snort/snort.lua -R /usr/local/etc/rules/local.rules -i <network adapter> -A alert_fast`
7. Run ping from another machine and monitor traffic. You should see alerts triggered by this rule.

### Adding Rule File To Default Configuration

1. Modify the Snort Configuration
   `sudo nano /usr/local/etc/snort/snort.lua`
2. Find the ips field and modify it so that it looks like the following:

```
ips = {
  -- use this to enable decoder and inspector alerts
  enable_builtin_rules = true
  -- use include for rules files; be sure to set your path
  -- note that rules files can include other rules files
  -- (see also related path vars at the to of snort_defaults.lua)
  include = "/usr/local/etc/rules/local.rules",
  variables = default_variables
}
```

3. Save and exit Nano `Ctrl-X`
4. Run snort and listen to traffic
   `snort -c /usr/local/etc/snort/snort.lua -i <network adapter> -A alert_fast`
5. Run ping from another machine and monitor traffic. You should see alerts triggered by this rule.

## Installing PulledPork3

PulledPork3 is a Snort3 Rules Management Command-Line Interface. You can use this to pull rules from [Snort3 Community Repository](https://www.snort.org/downloads). **Note**: You will need an user account to gain access to these rule-sets.

[Link to Repo](https://github.com/shirkdog/pulledpork3)

### 1. Download and Install the Repo

```
cd ~/snort
git clone https://github.com/shirkdog/pulledpork3.git 
cd pulledpork3

sudo mkdir /usr/local/etc/pulledpork3 
sudo cp etc/pulledpork.conf /usr/local/etc/pulledpork3/

sudo mkdir /usr/local/bin/pulledpork3
sudo cp pulledpork.py /usr/local/bin/pulledpork3/
sudo cp -r lib/ /usr/local/bin/pulledpork3 
sudo chmod +x /usr/local/bin/pulledpork3/pulledpork.py 
```

2. Test and validate the version: `pulledpork.py -V`
   **Troubleshooting**: If the command is not found. It should be located in `/usr/local/bin/pulledpork3/pulledpork.py`
3. Modify the Configuration file and import rule-sets
   1. Open PulledPork Configuration file
      `sudo nano /usr/local/etc/pulledpork3/pulledpork.conf`
   2. Enable the following settings:
      1. `registered_ruleset = true`
      2. `oinkcode = <This Code Is In Your User Profile Settings>`
      3. `snort_blocklist = true`
      4. Uncomment and Ensure this line is correct.`snort_path = /usr/local/bin/snort`
      5. Uncomment and Ensure this line is correct.`local_rules = /usr/local/etc/rules/local.rules`. Delete the other placeholder
   3. Save and exit Nano `Ctrl-X`
   4. Create a so_rules directory
      `sudo mkdir /usr/local/etc/so_rules`
      `sudo mkdir /usr/local/etc/lists`
   5. There is a typo in the PulledPork3 Python File.
      1. Open PulledPork Python file
         `sudo nano /usr/local/bin/pulledpork3/pulledpork.py`
      2. Modify the following line:
         `RULESET_URL_SNORT_REGISTERED = 'https://snort.org/rules/snortrules-snapshot-<VERSION>.tar.gz'`
         to
         `RULESET_URL_SNORT_REGISTERED = 'https://snort.org/rules/snortrules-snapshot-31470.tar.gz'`
      3. Save and exit Nano `Ctrl-X`
   6. Run PulledPork with the config we just setup.
      `sudo /usr/local/bin/pulledpork3/pulledpork.py -c /usr/local/etc/pulledpork3/pulledpork.conf`
4. Modify the Snort Configuration
   `sudo nano /usr/local/etc/snort/snort.lua`
5. Find the ips field and modify it so that it looks like the following:

```
ips = {
  -- use this to enable decoder and inspector alerts
  enable_builtin_rules = true,
  -- use include for rules files; be sure to set your path
  -- note that rules files can include other rules files
  -- (see also related path vars at the to of snort_defaults.lua)
  include = "/usr/local/etc/rules/pulledpork.rules",
  variables = default_variables
}
```

6. View downloaded rules
   `cat /usr/local/etc/rules/pulledpork.rules | less`
7. Save and exit Nano `Ctrl-X`
8. Validate the configuration returns with no errors
   `snort -c /usr/local/etc/snort/snort.lua --plugin-path "/usr/local/etc/so_rules/"`
9. Run new Snort Rules on Network Adapter
   `sudo snort -c /usr/local/etc/snort/snort.lua --plugin-path "/usr/local/etc/so_rules/" -i <network adapter> -A alert_fast`

## Exercise: Analyze A PCAP From A Security Incident

[Exercise Link](https://www.malware-traffic-analysis.net/2022/02/23/index.html)

1. Download and unzip PCAP

```
wget https://www.malware-traffic-analysis.net/2022/02/23/2022-02-23-traffic-analysis-exercise.pcap.zip

# Password for file is infected
unzip 2022-02-23-traffic-analysis-exercise.pcap.zip
```

2. Read the PCAP
   `snort -c /usr/local/etc/snort/snort.lua --plugin-path /usr/local/etc/so_rules/ -r 2022-02-23-traffic-analysis-exercise.pcap -A alert_fast -q`
3. **Answer The Following Questions**:
   - What do you see?
   - What hosts/user account names are active on this network?
   - What type of malware are they infected with?
