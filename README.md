# Raspberry-WIFI-hotspot
A Raspberry WIFI access point (hotspot) and WIFI client at the same time

## 1. Install hostapd and dnsmasq

Install the hostapd access point daemon and the dnsmasq dhcp service.
```
sudo apt-get install hostapd dnsmasq 
```
## 2. Edit /etc/dhcpcd.conf

Set a static IP address (192.168.50.1)
```
interface uap0
	static ip_address=192.168.50.1/24
        nohook wpa_supplicant
```
## 3. Create a new /etc/dnsmasq.conf and add the following to it:
```
interface=lo,uap0               #Use interfaces lo and uap0
bind-interfaces                 #Bind to the interfaces
server=8.8.8.8                  #Forward DNS requests to Google DNS
domain-needed                   #Don't forward short names
bogus-priv                      #Never forward addresses in the non-routed address spaces
# Assign IP addresses between 192.168.50.2 and 192.168.50.150 with a 12-hour lease time
dhcp-range=192.168.50.2,192.168.50.150,12h
```
## 4. Create file /etc/hostapd/hostapd.conf and add the following:

Important Note: The channel written here MUST match the channel of the wifi that you connect to in client mode (via wpa-supplicant). If the channels for your AP and STA mode services do not match, then one or both of them will not run. This is because there is only one physical antenna. It cannot cover two channels at once.
```
# Set the channel (frequency) of the host access point
channel=1
# Set the SSID broadcast by your access point (replace with your own, of course)
ssid=wifipi
# This sets the passphrase for your access point (again, use your own)
wpa_passphrase=raspberry
# This is the name of the WiFi interface we configured above
interface=uap0
# Use the 2.4GHz band (but I have not tested 5Ghz band yet)
hw_mode=g
# Accept all MAC addresses
macaddr_acl=0
# Use WPA authentication
auth_algs=1
# Require clients to know the network name
ignore_broadcast_ssid=0
# Use WPA2
wpa=2
# Use a pre-shared key
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
driver=nl80211
# I commented out the lines below in my implementation, but I kept them here for reference.
# Enable WMM
#wmm_enabled=1
# Enable 40MHz channels with 20ns guard interval
#ht_capab=[HT40][SHORT-GI-20][DSSS_CCK-40]
```
## 5. Add a line to /etc/default/hostapd
```
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```
## 6. Create a startup script.

Add a new file /usr/local/bin/wifi_hotspot_start (or whatever you like)
```
#!/bin/bash

# Redundant stops to make sure services are not running
echo "Stopping network services (if running)..."
systemctl stop hostapd.service
systemctl stop dnsmasq.service
systemctl stop dhcpcd.service

#Make sure no uap0 interface exists (this generates an error; we could probably use an if statement to check if it exists first)
echo "Removing uap0 interface..."
iw dev uap0 del

#Add uap0 interface (this is dependent on the wireless interface being called wlan0, which it may not be in Stretch)
echo "Adding uap0 interface..."
iw dev wlan0 interface add uap0 type __ap

#Modify iptables (these can probably be saved using iptables-persistent if desired)
echo "IPV4 forwarding: setting..."
sysctl net.ipv4.ip_forward=1
echo "Editing IP tables..."
iptables -t nat -A POSTROUTING -s 192.168.50.0/24 ! -d 192.168.50.0/24 -j MASQUERADE

# Bring up uap0 interface. Commented out line may be a possible alternative to using dhcpcd.conf to set up the IP address.
#ifconfig uap0 192.168.50.1 netmask 255.255.255.0 broadcast 192.168.50.255
ifconfig uap0 up

# Start hostapd. 10-second sleep avoids some race condition, apparently. It may not need to be that long. (?) 
echo "Starting hostapd service..."
systemctl start hostapd.service
sleep 10

#Start dhcpcd. Again, a 5-second sleep
echo "Starting dhcpcd service..."
systemctl start dhcpcd.service
sleep 5

echo "Starting dnsmasq service..."
systemctl start dnsmasq.service
echo "wifi_hotspot_start DONE"
```
## 7. Start at boot

Add the following to your /etc/rc.local script above the exit 0 line (note the spacing between "/bin/bash" and "/usr/local/bin/wifi_hotspot_start"):

```
/bin/bash /usr/local/bin/wifi_hotspot_start
```
