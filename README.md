# Raspberry Pi setup
Download Raspbian (choose whether you want desktop or just CLI, I use the CLI):

**Desktop** - https://downloads.raspberrypi.org/raspbian_latest

**Just CLI** - https://downloads.raspberrypi.org/raspbian_lite_latest

**Download Rufus** - https://github.com/pbatard/rufus/releases/download/v3.8/rufus-3.8p.exe

1) Plug your SD card into your PC
2) Open Rufus
    1) Click the drop down box in the 'Device' section and select your SD card
    2) Click 'SELECT' and browse to your downloaded Raspbian file and select it
    3) Make sure the right device and image selected, then click 'START'
    4) Make sure this completes successfully. We now have a bootable PI image. 
3) Plug this back into your PI

# Getting Prepped

1) Power on your PI and complete any necessary steps to get you logged in.
2) If using a desktop version, find how to start the terminal as this will be where you do all your install / config.
3) Add your own user and give them sudo permissions, run these commands, as is (changing names):

`adduser exampleuser`

`usermod -aG sudo exampleuser`

`vi /etc/sudoers`

Check out some VI commands here if you need some help - https://kb.iu.edu/d/afdc

4) Paste `exampleuser ALL=(ALL) NOPASSWD: ALL` (changing the username to the one you just created) into your /etc/sudoers file so that you can sudo around without a password, it should looks like so:

```# User privilege specification
root    ALL=(ALL:ALL) ALL
exampleuser  ALL=(ALL) NOPASSWD: ALL
# Allow members of group sudo to execute any command
%sudo   ALL=(ALL:ALL) ALL
```

If you can't work out how to use VI, do the exactly the following (when in VI):

1) `/ALL`
2) Press the END key
3) `a`
4) Copy and paste the line here
5) `:wq!`

5) Start the SSH service and enable it on boot:

`sudo service ssh start`

`sudo systemctl enable ssh`

6) From here I do everything as the root user because I'm lazy. To get into the root user type `sudo su`

6) Configure a static IP address for your Raspberry Pi. I've set this to 192.168.0.250, it's high enough that nothing has probably been assigned it, and it's easy to remember. You can do the same or set something slightly different.

`vi /etc/dhcpcd.conf`

Make sure your static IP config looks like so:
```
# Example static IP configuration:
interface eth0
static ip_address=192.168.0.250/24
#static ip6_address=fd51:42f8:caae:d92e::ff/64
static routers=192.168.0.1
static domain_name_servers=192.168.0.1 8.8.8.8 fd51:42f8:caae:d92e::1
```

If you're really stuck and can't edit this file, make a backup and create a new file with the following exact contents (make sure you're not in vi `:q!`):

`mv /etc/dhcpcd.conf /etc/dhcpcd.conf.backup`

`vi /etc/dhcpcd.conf`

```
hostname
clientid
persistent
option rapid_commit
option domain_name_servers, domain_name, domain_search, host_name
option classless_static_routes
option interface_mtu
require dhcp_server_identifier
slaac private
interface eth0
static ip_address=192.168.0.250/24
static routers=192.168.0.1
static domain_name_servers=192.168.0.1 8.8.8.8 fd51:42f8:caae:d92e::1
interface eth0
        static ip_address=192.168.0.250/24
        static routers=192.168.0.1
        static domain_name_servers=127.0.0.1
```

7) You can restart just the networking config, but it's a good idea to do a reboot:

`reboot -n`

# Installing pihole

Actually installing the pihole software is really easy, simply copy and paste the following command into a terminal window and hit enter:

`curl -sSL https://install.pi-hole.net | bash`

This will run through the install script, I can't remember having to answer any questions but just keep an eye on the install, it won't take long.

Once completed there will be a box with all your pihole information, including the admin password.

Lets reset the admin password to something memorable. Run the following in the terminal and input the new password at the prompts:

`pihole -a -p`

# Getting devices to use the pihole

So now it's installed we need to make sure all network devices are using this for DNS resolution. There are two ways:

1) On your BT / Virgin router, login to the admin page, click the DHCP settings and see if you can manually set the DNS resolver (you can't on Virgin), if you can, then great, set these to your Pihole's network address and apply the settings. This means that any device that gets an IP from your router will then send all their DNS requests to the pihole to be blocked

2) Setup your pihole as a DHCP server, this means the pihole will handle all requests for new IP addresses, this will then contain the information to set the DNS resolver to the pihole for blocking.

## Configuring the Pihole DHCP server

We need to enable the pihole DHCP server along with disabling the DHCP server on the router.

1) Login to your pihole admin interface
2) Click 'Settings'
3) Click 'DHCP' along the top
4) Configure the DHCP server
    1) From `192.168.0.100`
    2) To `192.168.0.180'
    3) Router `192.168.0.1`
5) Click 'Enable'
6) DO NOT SAVE YET, WE WILL DO THIS SHORTLY

**Disable DHCP on your router**
1) Login to your router admin interface, IP is usually 192.168.0.1
2) Navigate to the DHCP settings - for virgin routers it's under 'Advanced Settings'
3) Disable DHCP server checkbox
4) Save

**On the Pihole**
1) Save the DHCP settings and make sure it's enabled.

**NOTE**

If for some reason you have a really shitty lease time on your PC you may lose network access to your pihole / router, if so you need to manually assign an IP address so you can turn on your pihole DHCP server. This is all done on your PC.

1) Open 'Network and sharing center'
2) Click 'Change adapter settings'
3) Right click your 'Local area connection' and click 'Properties'
4) Double click 'Internet Protocol Version 4 (TCP/IPV4)'
5) Click the 'Use following address' radio button
    1) IP address `192.168.0.99`
    2) Subnet mask `255.255.255.0`
    3) Default gateway `192.168.0.1`
    
You should now be able to navigate to the Pihole IP address (192.168.0.250) in your browser to enable DHCP.

Once re-enabled, make sure to remove the static IP config we set previously, just select the 'Obtain IP address automatically' radio button.

# Add some blocklists to the Pihole

By default pihole comes with a few blocklists, we need to bulk this out. We can use some other lists available on the internet.

On the Pihole admin page, navigate to:
1) Settings
2) On the top, Blocklists
3) Remove the ones that are there (you re-add them below)
4) Copy the below list and paste into the blocklist area, you can do it all in one hit as they're on a new line

```
https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts
https://mirror1.malwaredomains.com/files/justdomains
http://sysctl.org/cameleon/hosts
https://s3.amazonaws.com/lists.disconnect.me/simple_tracking.txt
https://s3.amazonaws.com/lists.disconnect.me/simple_ad.txt
https://adaway.org/hosts.txt
https://reddestdream.github.io/Projects/MinimalHosts/etc/MinimalHostsBlocker/minimalhosts
https://raw.githubusercontent.com/StevenBlack/hosts/master/data/add.Spam/hosts
https://raw.githubusercontent.com/StevenBlack/hosts/master/data/KADhosts/hosts
https://v.firebog.net/hosts/AdguardDNS.txt
https://raw.githubusercontent.com/anudeepND/blacklist/master/adservers.txt
https://hosts-file.net/ad_servers.txt
https://v.firebog.net/hosts/Easylist.txt
https://pgl.yoyo.org/adservers/serverlist.php?hostformat=hosts;showintro=0
https://raw.githubusercontent.com/StevenBlack/hosts/master/data/UncheckyAds/hosts
https://www.squidblacklist.org/downloads/dg-ads.acl
https://v.firebog.net/hosts/Easyprivacy.txt
https://v.firebog.net/hosts/Prigent-Ads.txt
https://gitlab.com/quidsup/notrack-blocklists/raw/master/notrack-blocklist.txt
https://raw.githubusercontent.com/StevenBlack/hosts/master/data/add.2o7Net/hosts
https://raw.githubusercontent.com/crazy-max/WindowsSpyBlocker/master/data/hosts/spy.txt
https://raw.githubusercontent.com/Perflyst/PiHoleBlocklist/master/android-tracking.txt
https://raw.githubusercontent.com/Perflyst/PiHoleBlocklist/master/SmartTV.txt
https://v.firebog.net/hosts/Airelle-trc.txt
```

You can find some more at - https://firebog.net/

5) Hit 'Save and update' - FYI this will take a little while
6) That's it, you've got your Pihole all setup!

If you want to have your phone routing through the pihole wherever you are, we just need to setup a VPN, tell it to use the pihole and get pihole to monitor the traffic. You can see how to do this below.

## Add some regex blocking

We all know advertisers like to change sites and stuff regularly so we can setup some regex blocking to stop these. I've pulled these from the pihole subreddit. Sadly you can't input these all at the same time and need to do it one by one.

1) Navigate to 'Blacklist'
2) Copy and paste the below, one by one, and click 'Add (regex)' after each one

```
(^|\.)xn--.*$
^(.+[-_.])??adse?rv(er?|ice)?s?[0-9]*[-.]
^(.+[-_.])??m?ad[sxv]?[0-9]*[-_.]
^adim(age|g)s?[0-9]*[-_.]
^adtrack(er|ing)?[0-9]*[-.]
^advert(s|is(ing|ements?))?[0-9]*[-_.]
^aff(iliat(es?|ion))?[-.]
^analytics?[-.]
^banners?[-.]
^beacons?[0-9]*[-.]
^count(ers?)?[0-9]*[-.]
^pixels?[-.]
^stat(s|istics)?[0-9]*[-.]
^telemetry[-.]
^track(ers?|ing)?[0-9]*[-.]
^traff(ic)?[-.]
```

## Add a test site

To make sure I'm going through my pihole I setup a test site that I've blocked (milkfish.com - sadly not mine). So if I can reach this on a device I know it's not being piholed, if I can't reach it, then we're all good.

1) Navigate to 'Blacklist'
2) Add `milkfish.com` and click 'Add (exact)'

You should now see the following error:

```
This site can’t be reached
milkfish.com’s server IP address could not be found.
Try running Windows Network Diagnostics.
DNS_PROBE_FINISHED_NXDOMAIN
```

Now we know all my requests are going through the pihole.

-------------------------

# PiVPN

The guys at PiVPN have made it extremely easy to setup a VPN server on your Pi. Just run the following command and go through the prompts:

`curl -L https://install.pivpn.io | bash`

## Directing VPN traffic to pihole

We need to make a few changes so that traffic is directed to the pihole

1) `vi /etc/openvpn/server.conf`
2) Make sure the `push "dhcp-option` setting is configured to point to the pihole, it should look like so:

`push "dhcp-option DNS 192.168.0.250"`

3) Remove any other `push "dhcp-option` that do not match `push "dhcp-option DNS 192.168.0.250"` - In vi you can use `dd` to delete a whole line

## Allowing pihole to monitor VPN traffic

We need to add the VPN interface (tun0) to the pihole config in two places, `setupVars.conf` and `01-pihole.conf`. Run the exact commands in the code blocks.

1) `vi /etc/pihole/setupVars.conf`
    1) `i`
    2) Press ENTER
    3) Paste the following `piholeInterface=tun0`
    4) `:wq`
    
2) `vi /etc/dnsmasq.d/01-pihole.conf`
    1) `/interface`
    2) Press END
    3) `i`
    4) Press ENTER
    5) Paste `interface=tun0`
    6) `:wq`
    
Lets reboot the Raspberry Pi - `reboot -n`


## Adding a new user

Once PiVPN has been installed you get a nice utility to add and remove users. We need to create a new user for our phone which is done with the `pivpn -a` command. Follow the prompts here and it should create the certificate and ovpn profile, it will tell you where these are like so:

```
========================================================
Done! exampleuserv2.ovpn successfully created!
exampleuserv2.ovpn was copied to:
  /home/exampleuser/ovpns
for easy transfer. Please use this profile only on one
device and create additional profiles for other devices.
========================================================
```

You need to get this off your pi and onto your phone. I used WinSCP - https://winscp.net/download/WinSCP-5.15.4-Portable.zip

1) Open WinSCP
    1) Hostname (IP of Pi) - 192.168.0.250
    2) User (The user we first created) - exampleuser
    3) Password - The password for this user
2) You will now be in the home directory, if this isn't where the ovpn profile was put you can navigate to where it previously told you.
3) Upload to google drive or copy over to your phone

Setup the new profile on your phone:

1) Download and install OpenVPN
2) Open OpenVPN and click 'OVPN Profile'
3) Find the .ovpn file you copied over to your phone and select it
4) Input the password and your connected!!

As a final step, try to go to your test site (milkfish.com). If you can reach it, something's not configured right, if you can't, then boom, ad / tracker free browsing on your phone, anywhere.
