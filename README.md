# Bitcoin-FullNode
Run a BTC full node for fun


A Huge thanks to Ministry of Nodes and his precious video tutorial. What you will find here is nothing but an updated 
step by step guide to that video tutorial. I strongly reccomend you to visit the channel and watch his content.

[Ministry of Nodes](https://www.youtube.com/c/MinistryofNodes)



# Dedicated HDD preparation and migration of /home

```
$ fdisk /dev/sdb
```

- n
- p
- enter
- enter
- t
- 31
- w

> We are going to use the entire disk, so nothing fancy.

```bash
$ pvcreate /dev/sdb1
$ pvs
$ vgcreate vg_home /dev/sdb1
$ lvcreate -l 100%FREE -n lv_home vg_home
$ mkfs.xfs /dev/vg_home/lv_home
$ mount /dev/vg_home/lv_home /mnt
$ cp -pr /home/* /mnt
$ mv /home /home.orig
$ mkdir /home
$ chmod 1777 /home
$ chwon root:root /home
$ nano /etc/fstab (append to the file)
```

```
/dev/vg_home/lv_home /home                xfs     defaults,noatime 1 2
```

```
sudo reboot
```

# Install TOR if you did not

```
$ sudo apt install tor 
$ sudo usermod -a -G debian-tor YOUR_USER
```


# Download Files


[BitcoinCore](https://bitcoincore.org/en/download/)

```
$ wget https://bitcoincore.org/bin/bitcoin-core-22.0/SHA256SUMS
$ wget https://bitcoincore.org/bin/bitcoin-core-22.0/SHA256SUMS.asc
$ wget https://bitcoincore.org/bin/bitcoin-core-22.0/bitcoin-22.0-x86_64-linux-gnu.tar.gz
```

>if you are upgrading the version and want to verify the new packege, consider to run gpg --keyserver hkp://keyserver.ubuntu.com --refresh-keys



# Verify the Package

```
$ cd Downloads/
$ sha256sum --ignore-missing --check SHA256SUMS
```

_In the output produced by the above command, you can safely ignore any warnings and failures, but you must ensure the output lists "OK" after the name of the release file you downloaded_

```
$ gpg --keyserver hkp://keyserver.ubuntu.com --recv-keys E777299FC265DD04793070EB944D35F9AC3DB76A
```

_The output of the command above should say that one key was imported, updated, has new signatures, or remained unchanged._

```
$ gpg --verify SHA256SUMS.asc
```

_Search for a line with "gpg: Good signature"_


# Install Bitcoind

```
$ sudo install -m 0755 -o root -g root -t /usr/local/bin bitcoin-22.0/bin/*
$ mkdir /home/satoshi/.bitcoin
$ cd /home/.bitcoin
$ nano bitcoin.conf
```

# Confiure Bitcoind 

```bitcoin.conf

server=1
blockfilterindex=1
txindex=1
rpcport=8332
rpcbind=0.0.0.0
rpcallowip=127.0.0.1
rpcallowip=10.0.0.0/8
rpcallowip=172.0.0.0/8
rpcallowip=192.0.0.0/8
zmqpubrawblock=tcp://0.0.0.0:28332
zmqpubrawtx=tcp://0.0.0.0:28333
zmqpubhashblock=tcp://0.0.0.0:28334
rpcuser=YOUR_USER
rpcpassword=YOUR_PASSWD
```



# Configure the service

```
$ cd /etc/systemd/system/
$ sudo nano bitcoind.service
```

> Note: replace "***" with yours

```bitcoind.service

# It is not recommended to modify this file in-place, because it will
# be overwritten during package upgrades. If you want to add further
# options or overwrite existing ones then use
# $ systemctl edit bitcoind.service
# See "man systemd.service" for details.

# Note that almost all daemon options could be specified in
# /etc/bitcoin/bitcoin.conf, but keep in mind those explicitly
# specified as arguments in ExecStart= will override those in the
# config file.

[Unit]
Description=Bitcoin daemon
Documentation=https://github.com/bitcoin/bitcoin/blob/master/doc/init.md

# https://www.freedesktop.org/wiki/Software/systemd/NetworkTarget/
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/local/bin/bitcoind -daemonwait \
                            -pid=/run/bitcoind/bitcoind.pid \
                            -conf=/home/***/.bitcoin/bitcoin.conf \
                            -datadir=/home/***/.bitcoin

# Make sure the config directory is readable by the service user
#PermissionsStartOnly=true
#ExecStartPre=/bin/chgrp bitcoin /etc/bitcoin

# Process management
####################

Type=forking
PIDFile=/run/bitcoind/bitcoind.pid
Restart=on-failure
TimeoutStartSec=infinity
TimeoutStopSec=600

# Directory creation and permissions
####################################

# Run as bitcoin:bitcoin
User=***
Group=***

# /run/bitcoind
RuntimeDirectory=bitcoind
RuntimeDirectoryMode=0710

# /etc/bitcoin
ConfigurationDirectory=bitcoin
ConfigurationDirectoryMode=0710

# /var/lib/bitcoind
StateDirectory=bitcoind
StateDirectoryMode=0710

# Hardening measures
####################

# Provide a private /tmp and /var/tmp.
PrivateTmp=true

# Mount /usr, /boot/ and /etc read-only for the process.
#ProtectSystem=full

# Deny access to /home, /root and /run/user
#ProtectHome=true

# Disallow the process and all of its children to gain
# new privileges through execve().
NoNewPrivileges=true

# Use a new /dev namespace only populated with API pseudo devices
# such as /dev/null, /dev/zero and /dev/random.
PrivateDevices=true

# Deny the creation of writable and executable memory mappings.
MemoryDenyWriteExecute=true

[Install]
WantedBy=multi-user.target
```

```
$ sudo systemctl enable bitcoind.service
$ sudo systemctl start bitcoind
$ tail -f /home/satoshi/.bitcoin/debug.log
```

_If you did not made mistakes, everything should be allright._


# Bitcoin Sync

_Now that you set up everything what we needed, we have to wait the end of block sync process, it will takes a while, depending from bandwith, PC hardware limitation, TOT blocks numbers_

_you can monitor the status and healty of the process with_

```
$ bitcoin-cli getblockchaininfo
$ bitcoin-cli getconnectioncount
```

Ex JSON Resp of getblockchaininfo:

```json
"chain": "main", 
"blocks": 503527, 
"headers": 707440,
```

> Note: When Blocks and Headers will match in number, you will be done. You can also monitoring the process by tail the debug.log file



# Connect to Tor


$ sudo apt install tor
$ sudo vim /usr/share/tor/tor-service-defaults-torrc


```tor-service-defaults-torrc
DataDirectory /var/lib/tor
PidFile /run/tor/tor.pid
RunAsDaemon 1
User debian-tor

ControlPort 9051
ControlSocket /run/tor/control GroupWritable RelaxDirModeCheck
ControlSocketsGroupWritable 1
SocksPort unix:/run/tor/socks WorldWritable
SocksPort 9050

CookieAuthentication 1
CookieAuthFileGroupReadable 1
CookieAuthFile /run/tor/control.authcookie

Log notice syslog
```

```
$ sudo /etc/init.d/tor restart
$ grep User /usr/share/tor/tor-service-defaults-torrc
$ sudo usermod -a -G debian-tor <username>

Open a new terminal and exec:

$ id <username>

--> uid=1000(username) gid=100(users) groups=100(users),121(debian-tor)

```

# bitcoin service changes

``` /etc/systemd/system/bitcoind.service

ExecStart=/usr/local/bin/bitcoind -daemonwait -debug=tor\
                            -pid=/run/bitcoind/bitcoind.pid \
                            -conf=/home/<user>/.bitcoin/bitcoin.conf \
                            -datadir=/home/<user>/.bitcoin
```

# bitcoin.conf changes

_create a new python file with:

```
#!/usr/bin/env python3
# Copyright (c) 2015-2021 The Bitcoin Core developers
# Distributed under the MIT software license, see the accompanying
# file COPYING or http://www.opensource.org/licenses/mit-license.php.

from argparse import ArgumentParser
from base64 import urlsafe_b64encode
from getpass import getpass
from os import urandom

import hmac

def generate_salt(size):
    """Create size byte hex salt"""
    return urandom(size).hex()

def generate_password():
    """Create 32 byte b64 password"""
    return urlsafe_b64encode(urandom(32)).decode('utf-8')

def password_to_hmac(salt, password):
    m = hmac.new(bytearray(salt, 'utf-8'), bytearray(password, 'utf-8'), 'SHA256')
    return m.hexdigest()

def main():
    parser = ArgumentParser(description='Create login credentials for a JSON-RPC user')
    parser.add_argument('username', help='the username for authentication')
    parser.add_argument('password', help='leave empty to generate a random password or specify "-" to prompt for password', nargs='?')
    args = parser.parse_args()

    if not args.password:
        args.password = generate_password()
    elif args.password == '-':
        args.password = getpass()

    # Create 16 byte hex salt
    salt = generate_salt(16)
    password_hmac = password_to_hmac(salt, args.password)

    print('String to be appended to bitcoin.conf:')
    print('rpcauth={0}:{1}${2}'.format(args.username, salt, password_hmac))
    print('Your password:\n{0}'.format(args.password))

if __name__ == '__main__':
    main()    
```


> Note: You need to execute to execute the python scrypt in order to retrive the string you have to insert into your bitocin.conf file


```bitcoin.conf

server=1
blockfilterindex=1
rpcport=8332
rpcbind=0.0.0.0
rpcallowip=127.0.0.1
rpcallowip=127.0.0.0/8
rpcallowip=172.0.0.0/8
rpcallowip=192.0.0.0/8
rpcuser=YOUR_USER_HERE
rpcpassword=YOUR_PASSWD_HERE


# [core]
# Maintain a full transaction index (improves lnd performance)
txindex=1
daemon=1
disablewallet=0
maxuploadtarget=1000


# [rpc]
# Accept command line and JSON-RPC commands.
server=1
rpcauth="YOUR_USER_HERE":"STRING_GENERATED_VIA_rpcauth.py"


# [zeromq]
# Enable publishing of transactions to [address]
zmqpubrawtx=tcp://127.0.0.1:28333
# Enable publishing of raw block hex to [address].
zmqpubrawblock=tcp://127.0.0.1:28332
# Not Clear
zmqpubhashblock=tcp://0.0.0.0:28334

# Privacy
proxy=127.0.0.1:9050
#bind=127.0.0.1
bind=127.0.0.1:8333
# Allow DNS lookups for -addnode, -seednode and -connect values.
dns=0
# Query for peer addresses via DNS lookup, if low on addresses.
dnsseed=0
# Specify your own public IP address.
externalip="STRING_GENERATED_VIA_getnetworkinfo".onion
# Use separate SOCKS5 proxy to reach peers via Tor
onion=127.0.0.1:9050
proxy=127.0.0.1:9050
proxyrandomize=1
# Only connect to peers via Tor.
onlynet=onion
listenonion=1
listen=1
# helps bootstrap peers for initial sync
addnode=gyn2vguc35viks2b.onion
addnode=kvd44sw7skb5folw.onion
addnode=nkf5e6b7pl4jfd4a.onion
addnode=yu7sezmixhmyljn4.onion
addnode=3ffk7iumtx3cegbi.onion
addnode=3nmbbakinewlgdln.onion
addnode=4j77gihpokxu2kj4.onion
addnode=546esc6botbjfbxb.onion
addnode=5at7sq5nm76xijkd.onion
addnode=77mx2jsxaoyesz2p.onion
addnode=7g7j54btiaxhtsiy.onion
addnode=a6obdgzn67l7exu3.onion
addnode=ab64h7olpl7qpxci.onion
addnode=am2a4rahltfuxz6l.onion
addnode=azuxls4ihrr2mep7.onion
addnode=bitcoin7bi4op7wb.onion
addnode=bitcoinostk4e4re.onion
addnode=bk7yp6epnmcllq72.onion
addnode=bmutjfrj5btseddb.onion
addnode=ceeji4qpfs3ms3zc.onion
addnode=clexmzqio7yhdao4.onion
addnode=gb5ypqt63du3wfhn.onion
addnode=h2vlpudzphzqxutd.onion
```

# Service Restart

```
$ sudo systemct daemon-reload
$ sudo systemct restart bitcoind.service
$ tail -f ~/.bitcoin/debug.log | grep tor

You should see output like so:

torcontrol thread start
tor: Successfully connected!
tor: Connected to Tor version 0.2.8.6
Supported authentication method: COOKIE
Supported authentication method: SAFECOOKIE
Using SAFECOOKIE authentication, reading cookie authentication from /var/run/tor/control.authcookie
SAFECOOKIE authentication challenge successful
AUTHCHALLENGE ServerHash af7da689d08a67d5b4e789a98c76a6eacbabaa32baefc223ab0d7b1f46c3d
Authentication successful
ADD_ONION successful
Got service ID w4bh4cputinlptzm, advertising service w4bh4cputinlptzm.onion:8333
Cached service private key to /home/username/.bitcoin/onion_private_key
AddLocal(w4bh4cputinlptzm.onion:8333,4)
```

_Some useful Cmd:_

```
$ bitcoin-cli getnetworkinfo
$ bitcoin-cli getconnectioncount
$ bitcoin-cli getpeerinfo
```

For a fully walk through in parameters and command used to generate the needed values, please refer to:


[Tor Guide](https://blog.lopp.net/tor-only-bitcoin-lightning-guide/)
[Tor Hidden Service](https://en.bitcoin.it/wiki/Setting_up_a_Tor_hidden_service)


# Install Electrum server


```
$ sudo apt install clang cmake build-essential librocksdb-dev
$ sudo apt install -y libgflags-dev libsnappy-dev zlib1g-dev libbz2-dev liblz4-dev libzstd-dev
$ git clone -b v6.11.4 --depth 1 https://github.com/facebook/rocksdb && cd rocksdb
$ make shared_lib -j $(nproc) && sudo make install-shared
$ cd .. && rm -r rocksdb

$ git clone https://github.com/romanz/electrs
$ cd electrs
$ ROCKSDB_INCLUDE_DIR=/usr/local/include ROCKSDB_LIB_DIR=/usr/local/lib cargo build --locked --release
```


> Note: We need to modify one more time the bitcoind conf in order to allow electrs to indexs and do the magic by append to the bitcoin.conf

```
# Electrs Server
prune=0
whitelist=download@127.0.0.1

$ sudo systemctl restart bitcoind.service

Now it's time to create the electrs toml file. Guys do not blame on me but I had several problem during the service start due to the missing
configuration file. Actually I have the same file in two different paths, do the same or take the time I did not and experiment by yourself 
which is the right path 

/home/<user>/electrs/target/release/electrs.toml
/home/<user>/electrs/electrs.toml
```

```electrs.toml
# DO NOT EDIT THIS FILE DIRECTLY - COPY IT FIRST!
# If you edit this, you will cry a lot during update and will not want to live anymore!

# This is an EXAMPLE of how configuration file should look like.
# Do NOT blindly copy this and expect it to work for you!
# If you don't know what you're doing consider using automated setup or ask an experienced friend.

# This example contains only the most important settings.
# See docs or electrs man page for advanced settings.

# File where bitcoind stores the cookie, usually file .cookie in its datadir
#cookie_file = "/var/run/bitcoin-mainnet/cookie"

# The listening RPC address of bitcoind, port is usually 8332
daemon_rpc_addr = "127.0.0.1:8332"

# The listening P2P address of bitcoind, port is usually 8333
daemon_p2p_addr = "127.0.0.1:8333"

# Directory where the index should be stored. It should have at least 70GB of free space.
db_dir = "/home/<user>/electrs/target/release/db"

# bitcoin means mainnet. Don't set to anything else unless you're a developer.
network = "bitcoin"

# The address on which electrs should listen. Warning: 0.0.0.0 is probably a bad idea!
# Tunneling is the recommended way to access electrs remotely.
electrum_rpc_addr = "127.0.0.1:50001"

# How much information about internal workings should electrs print. Increase before reporting a bug.
log_filters = "INFO"

auth="<bitcoinduser>:<bitcoindpasswd>"
```

# First electrs execution and indexing

```
$ cd /home/<user>/electrs/target/release
$ ./electrs --log-filters INFO --db-dir ./db --electrum-rpc-addr="127.0.0.1:50001"
```

_In case of needed you could also increase the log level to DEBUG

_As soon as the indexing process will be terminated:

```
2021-11-03T20:00:36.068Z INFO  electrs::db] finished full compaction 
```


# Run Electrs as a service

```
$ sudo nano /etc/systemd/system/electrs.service
```



```
electrs.service

[Unit]
Description=Electrs
After=bitcoind.service

[Service]
WorkingDirectory=/home/<user>/electrs
ExecStart=/home/<user>/electrs/target/release/electrs --index-batch-size=10 --electrum-rpc-addr="127.0.0.1:50001"
User=<user>
Group=<group>
Type=simple
KillMode=process
TimeoutSec=60
Restart=always
RestartSec=60

[Install]
WantedBy=multi-user.target
```

```
$ sudo systemctl enable electrs.service
$ sudo systemctl start electrs.service
```



# Expose the Electrum on Tor ( Optional )

```
$ sudo nano /etc/tor/torrc
```

```
HiddenServiceDir /var/lib/tor/electrs_hidden_service/
HiddenServiceVersion 3
HiddenServicePort 50001 127.0.0.1:50001
```

```
$ sudo systemctl restart tor
$ sudo cat /var/lib/tor/electrs_hidden_service/hostname

<your-onion-address>.onion
```

_On your client machine, run the following command (assuming Tor proxy service runs on port 9050):

```
$ electrum --oneserver --server <your-onion-address>.onion:50001:t --proxy socks5:127.0.0.1:9050
```

For more details:

[See](http://docs.electrum.org/en/latest/tor.html)



# Electrum Wallet

> on Arch linux, by enabling AUR repository it is possible to obtain the git version, 
> but in order to use it in conjunction
> with our hardware wallet we need to execute the following commands:

```
$ git clone https://github.com/spesmilo/electrum.git
$ cd electrum
$ sudo groupadd plugdev
$ sudo usermod -aG plugdev $(whoami)
$ sudo cp contrib/udev/*.rules /etc/udev/rules.d/
$ sudo udevadm control --reload-rules && sudo udevadm trigger
```

_o connect to your node, use the following string:_


>YOUR_NODE_IP:50001:t


_You are now readz to add your device and create a wallet




# Btc Rpc Explorer

_let's bring back txindex parameter to 1 in bitcoin.conf and restart the service.
_I have reason to believe that the only config change needed for Electrs was the whitelist= parameter 
_and that I did not need to set the txindex to 0, anyway since our electrs is now fully sync we do not care 

```
$ curl -fsSL https://deb.nodesource.com/setup_17.x | sudo -E bash -
$ sudo apt-get install -y nodejs gcc g++ make

$ git clone https://github.com/janoside/btc-rpc-explorer
$ cd btc-rpc-explorer
$ npm install
$ npm up

$ cp .env-sample ~/.config/btc-rpc-explorer.env
$ nano ~/.config/btc-rpc-explorer.env
```

```btc-rpc-explorer.env
BTCEXP_HOST=127.0.0.1
BTCEXP_BITCOIND_USER=YOUR_BITCOIND_USER
BTCEXP_BITCOIND_PASS=YOUR_BITCOIND_PASSWORD
BTCEXP_ADDRESS_API=electrumx
BTCEXP_ELECTRUM_SERVERS=tcp://127.0.0.1:50001
BTCEXP_SLOW_DEVICE_MODE=false
BTCEXP_NO_RATES=false
```

```
$ cd ~/btc-rpc-explorer
$ nano config.json

--> {}

$ npm start
```

> http://YOU_IP_NODE:3002/


```
$ sudo nano /etc/systemd/system/btcexplorer.service
```


``` btcexplorer.service
[Unit]
Description=BTC RPC Explorer
After=network.target bitcoind.service

# If you use an Electrum server, uncomment the following line and make sure to use the correct the service
After=electrs.service

[Service]
WorkingDirectory=/home/satoshi/btc-rpc-explorer       # !! CHANGE YOUR HOME !!
ExecStart=/usr/bin/npm start
User=satoshi   # !! CHANGE USER !!

# Restart on failure but no more than 2 time every 10 minutes (600 seconds). Otherwise stop
Restart=on-failure
StartLimitIntervalSec=600
StartLimitBurst=2

[Install]
WantedBy=multi-user.target
```

```
$ sudo systemct enable /etc/systemd/system/btcexplorer.service
$ sudo systemctl start btcexplorer.service
```

Note: Consider to always connect yout Electrum, Wasabi, Ledger and Trezor wallet to your own BTC node. Expose your server on 
      Tor will makes your server reachable from whereever you are.

A few Resources which will help you to setup HW Wallet as well wasabi and/or ledger


[1](https://support.ledger.com/hc/en-us/articles/360017551659-Setting-up-your-Bitcoin-full-node?docs=true)
[2](https://armantheparman.com/connect-electrum-desktop-wallet-to-your-bitcoin-node/)
[3](https://blog.trezor.io/connecting-your-wallet-to-a-full-node-edf56693b545)
[4](https://blog.lopp.net/how-to-run-bitcoin-as-a-tor-hidden-service-on-ubuntu/)
[5](https://bitnodes.io/)
[6](https://armantheparman.com/tor/)
[7](https://mynodebtc.github.io/tor/electrum.html)
[8](https://github.com/romanz/electrs/blob/master/doc/config.md#tor-hidden-service)
