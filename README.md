# Raspberry-WIFI-hotspot
A Raspberry WIFI access point (hotspot) and WIFI client at the same time

##1. Install hostapd and dnsmasq
Install the hostapd access point daemon and the dnsmasq dhcp service.

sudo apt-get install hostapd dnsmasq 

##2. Edit /etc/dhcpd.conf
Set a static IP address (192.168.50.1)

interface uap0
	static ip_address=192.168.50.1/24
        nohook wpa_supplicant

##2. Create a new /etc/dnsmasq.conf and add the following to it:

interface=lo,uap0               #Use interfaces lo and uap0
bind-interfaces                 #Bind to the interfaces
server=8.8.8.8                  #Forward DNS requests to Google DNS
domain-needed                   #Don't forward short names
bogus-priv                      #Never forward addresses in the non-routed address spaces
# Assign IP addresses between 192.168.50.2 and 192.168.50.150 with a 12-hour lease time
dhcp-range=192.168.50.2,192.168.50.150,12h

##3. Create file /etc/hostapd/hostapd.conf and add the following:

Important Note: The channel written here MUST match the channel of the wifi that you connect to in client mode (via wpa-supplicant). If the channels for your AP and STA mode services do not match, then one or both of them will not run. This is because there is only one physical antenna. It cannot cover two channels at once.

# Set the channel (frequency) of the host access point
channel=1
# Set the SSID broadcast by your access point (replace with your own, of course)
ssid=yourSSIDhere
# This sets the passphrase for your access point (again, use your own)
wpa_passphrase=passwordBetween8and64charactersLong
# This is the name of the WiFi interface we configured above
interface=uap0
# Use the 2.4GHz band (I think you can use in ag mode to get the 5GHz band as well, but I have not tested this yet)
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

##4. Add a line to /etc/default/hostapd
DAEMON_CONF="/etc/hostapd/hostapd.conf"

