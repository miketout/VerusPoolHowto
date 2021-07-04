# S-NOMP for Verus

Operating a mining pool requires you to know about systems administration, IT security, databases, software development, coin daemons and other more or less related stuff. Running a production pool can literally be more work than a full-time job.

**NOTE** When you are done please message `englal#8861` on the [Verus discord](https://discord.gg/VRKMP2S)) with your poolwallet IP so he can `addnode` it around his platform, which contributes to network stability. `Done` in this case means at least full setup procedure completed, pool running, a block was found and paid out. Thank you.

A VPS with 4GB of RAM, anything above 20GB **SSD** storage and 1 CPU core which knows about AES-NI is the absolute minimum requirement. Generally, having more RAM is more important than having more CPU power here. Additionally, the hypervisor of your VPS _must_ pass through the original CPU designation from it's host. See below for an example that will likely lead to trouble.

```bash
lscpu|grep -i "model name"
Model name:            QEMU Virtual CPU version 2.5+
```

Basically, anything in there that is not a real CPU name _may_ cause NodeJS to behave funny despite the `Virtual CPU` having all necessary CPU flags. Be aware and ready to switch servers and/or hosting companies if need be. Start following the guide while logged in as `root`.


## Operating System

This guide tailored to and tested on `Debian 10 "Buster"` but should probably also work on Debian-ish derivatives like `Devuan` or `Ubuntu` and others. Before starting, please install the latest updates and prerequisites.

```bash
apt update
apt -y upgrade
apt -y install libgomp1 redis-server git libboost-all-dev libsodium-dev build-essential
```

## Poolwallet

Create a user for the poolwallet, switch to that account:

```bash
useradd -m -d /home/verus -s /bin/bash verus
su - verus
```

Download the [official Verus binaries](https://github.com/VerusCoin/VerusCoin/releases) for the **current release** and unpack them (v0.6.2-1used in this example):

```bash
wget https://github.com/VerusCoin/VerusCoin/releases/download/v0.6.2-1/Verus-CLI-Linux-v0.6.2-1-amd64.tar.gz https://github.com/VerusCoin/VerusCoin/releases/download/v0.6.2-1/Verus-CLI-Linux-v0.6.2-1-amd64.tar.gz.txt
# if this next command doesn't return `true`, please re-download and/or report to discord. thank you.
# note, currently the txt file isn't valid json, you'll need to add a `,` at the end of the `signer` line.
verus verifyfile "$(cat Verus-CLI-Linux-v0.6.2-1-amd64.tar.gz.txt | jq -r .signer)" "$(cat Verus-CLI-Linux-v0.6.2-1-amd64.tar.gz.txt | jq -r .signature)" $(pwd)/Verus-CLI-Linux-v0.6.2-1-amd64.tar.gz
tar xf Verus-CLI-Linux-v0.6.2-1-amd64.tar.gz
```

Create a `~/bin` directory, move over all executable files.

```bash
mkdir ~/bin
mv verus-cli/verus* verus-cli/fetch-params ~/bin
echo export PATH=\"${PATH}:/home/verus/bin\" >> ~/.bashrc
```

Log out of the `verus` account and back into it to get `~/bin` into the `PATH`:

```bash
exit
su - verus
```

Now, get the `zcparams` data:

```bash
fetch-params
```

Now, lets create the data and wallet export directory. Then, get the bootstrap and unpack it there.

```bash
mkdir ~/export
mkdir -p ~/.komodo/VRSC
cd ~/.komodo/VRSC
wget --continue --retry-connrefused --waitretry=3 --timeout=15 https://bootstrap.veruscoin.io/VRSC-bootstrap.tar.gz https://bootstrap.veruscoin.io/VRSC-bootstrap.tar.gz.sha256sum

# if that doesn't work out OK please redownload or abort and report to discord!
sha256sum -c VRSC-bootstrap.tar.gz.sha256sum

tar xf VRSC-bootstrap.tar.gz
rm VRSC-bootstrap.tar.gz
```

It's time to do the wallet config. A resonably secure `rpcpassword` can be generated using this command:   

```bash
cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 32 | head -n 1
```

Create `~/.komodo/VRSC/VRSC.conf` and include the parameters listed below, adapt the ones that need adaption.

```conf
##
## default recommended pool wallet config for verus/s-nomp
## see https://github.com/VerusCoin/VerusServicesSetup/blob/master/S-NOMP.md
##

# network options
listen=1
listenonion=0
port=27485
maxconnections=1024

# rpc options
server=1
rpcport=27486
rpcuser=verus
rpcpassword=rpcpassword
rpcbind=127.0.0.1
rpcallowip=127.0.0.1
rpcthreads=256
rpcworkqueue=1024

# logging options
logtimestamps=1
logips=1

# debug options
shrinkdebugfile=0
debug=addrman
debug=alert
debug=bench
debug=coindb
debug=db
debug=estimatefee
#debug=http
debug=libevent
debug=lock
debug=mempool
#debug=net
debug=partitioncheck
debug=pow
debug=proxy
debug=prune
debug=rand
debug=reindex
#debug=rpc
debug=selectcoins
debug=tor
#debug=zmq
debug=zrpc
debug=zrpcunsafe

# miscellaneous options
banscore=64
checkblocks=64
checklevel=4

# wallet related
exportdir=/home/verus/export
spendzeroconfchange=0

# blocknotify
#blocknotify=

# seednodes
# all seeds, bacloud servers
seednode=seeds.veruscoin.io:27485
seednode=185.25.48.236:27487
seednode=185.64.105.111:27487
seednode=185.25.48.72:27487

# addnodes
# vrsc0..2
addnode=185.25.48.236:27485
addnode=185.25.48.236:27487
addnode=185.64.105.111:27485
addnode=185.64.105.111:27487
addnode=185.25.48.72:27485
addnode=185.25.48.72:27487

# EOF
```

Afterwards, start the verus daemon and let it sync the rest of the blockchain. We'll also watch the debug log for a moment:

```bash
cd ~/.komodo/VRSC; verusd -daemon 1>/dev/null 2>&1; sleep 1; tail -f debug.log
```

Press `ctrl-c` to exit `tail` if it looks allright. To check the status and know when the initial sync has been completed, issue

```bash
verus getinfo
```

When it has synced up to height, the `blocks` and `longestchain` values will be at par. Additionally, you should verify against [the explorer](https://explorer.veruscoin.io) that you are in fact not on a fork. Edit the `crontab` using `crontab -e` and include the line below to autostart the poolwallet:

```crontab
@reboot cd /home/verus/.komodo/VRSC; /home/verus/bin/verusd -daemon 1>/dev/null 2>&1
```

**HINT** if you can't stand `vi`, do `EDITOR=nano crontab -e` ;-)

## Redis

Switch back to the `root` account by typing `exit` or hitting `ctrl-d`. In your `/etc/redis/redis.conf` file, make sure it contains this (and none of it is commented out):

```conf
bind 127.0.0.1
appendonly yes
```

Set amount of connections to 1024 (or 65535 if you think you need it) instead of the standard 128:

```bash
echo 'net.core.somaxconn = 1024' >> /etc/sysctl.conf
```
And use the following command to activate it immediately
```bash
sysctl net.core.somaxconn=1024
```


Set the overcommit_memory feature to 1, to avoid loss of data in case of not enough memory:
```bash
echo 'vm.overcommit_memory = 1' >> /etc/sysctl.conf
```
And use the following command to activate it immediately
```bash
sysctl vm.overcommit_memory=1
```

Finally disable Transparent Huge Page:
```bash
nano /etc/default/grub.d/no_thp.cfg
```
add this to the empty file:
```conf
GRUB_CMDLINE_LINUX_DEFAULT="$GRUB_CMDLINE_LINUX_DEFAULT transparent_hugepage=never"
```
Update `GRUB` and reboot.
```bash
update-grub
shutdown -r now
```

Wait for the reboot to finish and log back in as `root`. Set `redis-server` to start at bootup and start it manually:

```bash
update-rc.d redis-server enable
/etc/init.d/redis-server start
```

## Node.js

Create a new user account to run the pool from. Switch to that user to setup `nvm.sh`:


```bash
useradd -m -d /home/pool -s /bin/bash pool
su - pool
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.35.1/install.sh | bash
```

Log out and back in to activate `nvm.sh`

```bash
exit
su - pool
```

Now, install `NodeJS v8` via `nvm.sh` as well as `redis-commander` and [PM2](http://pm2.keymetrics.io) via `npm`.   
**NOTE** Node v10 won't work. You _will_ have to use Node v8!

```bash
nvm install 8
npm install -g redis-commander pm2
```

Because `nvm.sh` comes without it, we need to add one symlink into it's bindir for our installed NodeJS.

```bash
which node
/home/pool/.nvm/versions/node/v8.16.2/bin/node
```

Change to the resulting directory and create a symlink like below.

```bash
cd /home/pool/.nvm/versions/node/v8.16.2/bin
ln -s node nodejs
exit
```

## S-NOMP

Nake sure you're in the `pool` account and clone the S-NOMP from our main repository:

```bash
su - pool
git clone https://github.com/veruscoin/s-nomp
cd s-nomp
```

Next, install all dependencies using `npm`:

```bash
npm update
npm install
```

## Configuration Instructions

Shielding is no longer required for mined VerusCoins. We will need one public address for this. Switch to the `veruscoin` user and generate the addresses:

```bash
verus getnewaddress
```

Next, we will dump the private keys of these addresses for safety reasons. For the transparent addresses, use

```bash
verus dumpprivkey <Verus T-Address>
```

**Save the data in an offline location, not on your computer!**

Now, switch to the `pool` account. First, copy `/home/pool/s-nomp/config_example.json` to `/home/pool/s-nomp/config.json`. Edit it to reflect the changes listed below.

  * Set both `host` and `stratumHost` to the external IP or DNS name of your server.

Now create a pool config. Copy `/home/pool/s-nomp/pool_configs/examples/vrsc.json` to `/home/pool/s-nomp/pool_configs/vrsc.json`. Edit it to reflect the changes listed below.

 * Set `enabled` to `true`.
 * Set `coin` to `vrsc.json`.
 * Set `address` to one of the public addresses generated before.
 * Set `rewardRecipients` to your fee address and fee percentage. Remove `"": 0.2` if you want 0% fee.
 * Set `paymentInterval` to `180`
 * Set `minimumPayment` to `2`.
 * Set `maxBlocksPerPayment` to `8`.
 * Both `rewardRecipients` and `invalidAddress` are set to a Verus Foundation address per default, should you like to keep them intact.
 * **Otherwise make sure you do not use an address from the poolwallet for either `rewardRecipients` or `invalidAddress`**
 * Set `paymentInterval` (in Seconds) and `minimumPayment` (in VRSC) according to your planned scenario.
 * There are 2 occurences of `user`, `password` and `port` each. Use the `rpcuser`, `rpcpassword` and `rpcport` values from `/home/verus/.komodo/VRSC/VRSC.conf`.
 * Set `diff` to `131072`.
 * Set `minDiff` to `16384`.
 * Set `maxDiff` to `2147483648`

Edit the file `~/s-nomp/coins/vrsc.json` to reflect the following setting:
 * make sure `"requireShielding":false,` is set.

We are almost done now. Using the command mentioned at the beginning of this document, check if the blockchain has finished syncing. If not, wait for it to complete before continuing.

Now switch to the `verus` user, stop the wallet once more.

```bash
verus stop
```

To determine the location of you `node` binary, switch to user `pool`, do this and record your result. We'll need it for the next step.

```bash
which node
/home/pool/.nvm/versions/node/v8.16.1/bin/node
```

Switch back to user `verus` and edit `~/.komodo/VRSC/VRSC.conf` to enable the blocknotify command as seen below, using the location you just got from using `which node` before:

```conf
blocknotify=/home/pool/.nvm/versions/node/v8.16.1/bin/node /home/pool/s-nomp/scripts/cli.js blocknotify verus %s
```

Restart the wallet using the command already listed above. If you are not using `STDOUT`/`STDERR`-redirection, you will see errors about blocknotify. These are expected, because the pool is not running yet and thus the blocknotify script cannot complete successfully.


Switch to the `pool` user. Then start the pool using `pm2`:

```bash
cd ~/s-nomp
pm2 start init.js --name veruspool
```

Use `pm2 log` to check for S-NOMP startup errors.

If you completed all steps correctly, the web dashboard on your pool can be reached via port `8080` on the external IP or the DNS name of your server.

### S-Nomp Autostart

Edit your crontab using `crontab -e` and shove in this line at the bottom:

```crontab
@reboot /bin/sleep 300 && cd /home/pool/s-nomp && /usr/bin/pm2 start init.js --name veruspool
```

**HINT** if you can't stand `vi`, do `EDITOR=nano crontab -e` ;-)


## Further considerations

None of the topics below is strictly necessary, but most of them are recommended.

### Useful DNS resolvers

Empty your `/etc/resolv.conf` and replace it with this:

```conf
# google, https://developers.google.com/speed/public-dns/docs/using
nameserver 8.8.4.4
nameserver 8.8.8.8
nameserver 2001:4860:4860::8844
nameserver 2001:4860:4860::8888

# verisign, https://publicdnsforum.verisign.com/discussion/13/verisign-public-dns-set-up-configuration-instructions
nameserver 64.6.64.6
nameserver 64.6.65.6
nameserver 2620:74:1b::1:1
nameserver 2620:74:1c::2:2

# quad9, https://www.quad9.net/faq/
nameserver 9.9.9.9
nameserver 149.112.112.112
nameserver 2620:fe::fe
nameserver 2620:fe::9

# cloudflare/apnic, https://1.1.1.1/de/
nameserver 1.1.1.1
nameserver 1.0.0.1
nameserver 2606:4700:4700::1111
nameserver 2606:4700:4700::1001

# opendns, https://use.opendns.com, https://www.opendns.com/about/innovations/ipv6/
nameserver 208.67.222.222
nameserver 208.67.220.220
nameserver 2620:119:35::35
nameserver 2620:119:53::53

# see 'man 5 resolv.conf'
options rotate timeout:1 attempts:5
```

Thank you.

### Improving SSH security

If you remember the good old `rand=4; // chosen by fair dice roll` comic, you're probably doing this anyways. If you don't go google the comic, you might have missed a laugh there!

As `root`, generate a proper `/etc/ssh/moduli` like this:

```bash
ssh-keygen -G "/root/moduli.candidates" -b 4096
mv /etc/ssh/moduli /etc/ssh/moduli.old
ssh-keygen -T /etc/ssh/moduli -f "/root/module.candidates"
rm "/root/moduli.candidates"
```

Add the recommended changes from [CiperLi.st](https://cipherli.st) to `/etc/ssh/sshd_config`, also make sure that `PermitRootLogin` is at least set to `without-password`. Then remove and re-generate your host keys like this:

```bash
cd /etc/ssh
rm ssh_host_*key*
ssh-keygen -t ed25519 -f ssh_host_ed25519_key < /dev/null
ssh-keygen -t rsa -b 4096 -f ssh_host_rsa_key < /dev/null
```

To finish, restart the ssh server:

```bash
/etc/init.d/sshd restart
```

### Reverse-proxying S-NOMP behind `nginx`

As `root`, install `nginx` and enable it on boot using these commands:

```bash
apt -y install nginx
update-rc.d enable nginx
```

Create `/etc/nginx/blockuseragents.rules` with these contents:

```conf
map $http_user_agent $blockedagent {
default         0;
~*malicious     1;
~*bot           1;
~*backdoor      1;
~*crawler       1;
~*bandit        1;
}
```

Edit `/etc/nginx/sites-available/default` to look like this:

```conf
include /etc/nginx/blockuseragents.rules;
server {
	if ($blockedagent) {
		return 403;
	}
	if ($request_method !~ ^(GET|HEAD)$) {
		return 444;
	}

	listen 80 default_server;
	listen [::]:80 default_server;
	charset utf-8;
	root /var/www/html;
	index index.html index.htm index.nginx-debian.html;

	location / {
		proxy_pass http://127.0.0.1:8080/;
		proxy_set_header X-Real-IP $remote_addr;
	}

	location /admin {
		rewrite ^/.* /stats permanent;
	}

}
```

Restart `nginx`:

```bash
/etc/init.d/nginx restart
```

Switch to the `pool` user, edit `/home/pool/s-nomp/config.json` to bind the web interface to `127.0.0.1:8080`:

```conf
[...]
"website": {
    "enabled": true,
    "host": "127.0.0.1",
    "port": 8080,
[...]

```

Restart the pool:

```bash
pm2 restart veruspool
```

If you've followed the above steps correctly, your pool's webdashboard is now proxied behind nginx.

### Disable unused webdashboard pages

Change to the `pool` account. Edit `/home/pool/libs/website.js` to have the `pageFiles` array look like below:

```conf
var pageFiles = {
    'index.html': 'index',
    'home.html': '',
    'manual.html': 'manual',
    'stats.html': 'stats',
    'tbs.html': 'tbs',
    'workers.html': 'workers',
    'api.html': 'api',
    'miner_stats.html': 'miner_stats',
    'payments.html': 'payments'
}
```

### Link to the `payments` page

Change to the `pool` user account. Edit `/home/pool/website/index.html` to include a new link at the right position, which is somewhere in between lines `30-70`:

```html
<header>
[...]
    <li class="{{? it.selected === 'payments' }}pure-menu-selected{{?}}">
        <a class="hot-swapper" href="/payments">
            <i class="fa fa-usd"></i>&nbsp;
            Payments
        </a>
    </li>
[...]
</header>
```

### Enable `logrotate`

As `root` user, create a file called `/etc/logrotate.d/veruspool` with these contents:

```conf
/home/verus/.komodo/VRSC/debug.log
/home/pool/.pm2/logs/veruspool-out.log
/home/pool/.pm2/logs/veruspool-error.log
{
  rotate 14
  daily
  compress
  delaycompress
  copytruncate
  missingok
  notifempty
}
```

### Increase open files limit

Add this to your `/etc/security/limits.conf`:

```conf
* soft nofile 1048576
* hard nofile 1048576
```

Reboot to activate the changes. Alternatively you can make sure all running processes are restarted from within a shell that has been launched _after_ the above changes were put in place, which usually is a huge pain. Just reboot.


### Networking optimizations

If your pool is expected to receive a lot of load, consider implementing below changes, all as `root`:

Enable the `tcp_bbr` kernel module:

```bash
modprobe tcp_bbr
echo tcp_bbr >> /etc/modules
```

Edit your `/etc/sysctl.conf` to include below settings:

```conf
net.ipv4.tcp_congestion_control=bbr
net.core.rmem_default = 1048576
net.core.wmem_default = 1048576
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
net.ipv4.udp_rmem_min = 16384
net.ipv4.udp_wmem_min = 16384
net.core.netdev_max_backlog = 262144
net.ipv4.tcp_max_orphans = 262144
net.ipv4.tcp_max_syn_backlog = 262144
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_tw_buckets = 2000000
net.ipv4.ip_local_port_range = 16001 65530
net.core.somaxconn = 20480
net.ipv4.tcp_low_latency = 1
net.ipv4.tcp_slow_start_after_idle = 0
net.ipv4.tcp_mtu_probing = 1
net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_limit_output_bytes = 131072
```

Run below command to activate the changes, alternatively reboot the machine:


```bash
sysctl -p /etc/sysctl.conf
```

### Change swapping behaviour

If your system has a lot of RAM, you can change the swapping behaviour to only swap when necessary. Edit `/etc/sysctl.conf` to include this setting:

```conf
vm.swappiness=1
```

The range is `1-100`. The *lower* the number, the *later* the system will start swapping stuff out. Run below command to activate the change, alternatively reboot the machine:

```bash
sysctl -p /etc/sysctl.conf
```

### Install `molly-guard`

As a last sanity check before reboots, `molly-guard` will prompt you for the hostname of the system you're about to reboot. Install it like this:

```bash
apt -y install molly-guard
```

Check `/etc/molly-guard/rc` for more options.
