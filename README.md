# PC-Raspberry-Pi-network-layer-development-in-IoT-systems

**Installing and updating software**

Open the terminal and update the software list:
```bash
sudo apt update
sudo apt upgrade
```

Raspberry Pi Acceptance Network Signal Strength Test Tool: Wavemon
Type in the Raspberry Pi terminal:
```bash
sudo apt-get install wavemon
wavemon
```

Then you can check parameters including:
1.  signal level
2.	Link Quality: is calculated based on the difference between signal strength and noise level. High Link Quality indicates a more stable and reliable wireless connection.
3.	RX / TX: These are the number and size (in KiB) of packets received (RX) and sent (TX).
4.	Mode: 'Managed' - This indicates that the wireless interface is controlled by the network manager.
5.	Channel: '1 (width: 20 MHz)' - This specifies that the wireless signal is using channel 1 in the 2.4 GHz band with a channel width of 20 MHz.
6.	RX/TX rate: This is the rate at which the wireless network receives/sends data.
7.	Power Mgt: on - Indicates that the power management feature of the wireless interface is on.
8.	TX-Power: This is the transmit power of the wireless interface.
9.	inactive: This indicates the amount of time the wireless interface has been idle since the last time there was a data transfer. If this is a long time, it may mean that the connection is inactive.
10.	retry: This indicates the number of retries when sending packets. If this number is high, it may mean that there is a problem with the wireless connection, causing the packet to take multiple attempts to be sent successfully.
11.	rts/cts (Request to Send/Clear to Send): This is a conflict avoidance mechanism used to reduce packet conflicts in a wireless network. If turned on, a device will send an RTS packet before sending data, and then only begin data transmission when it receives a CTS response. This is useful in environments where there is a lot of network congestion or interference.
12.	frag (Fragmentation threshold): This refers to the wireless network packet fragmentation size threshold. If the packet size exceeds this value, it will be divided into smaller fragments for transmission. This helps to improve transmission reliability in poor quality wireless signal environments.
13.	qlen (Queue Length): This is the length of the queue of packets that the network device is waiting to send. In most cases, this value is the default setting and indicates the maximum length of the packet queue that the network interface can handle.

Sometimes the manufacturer supplied router, signal expander or mesh is not customisable enough. So if you don't want to set up a manufacturer-supplied network device as a gateway in your IoT system, you can choose to develop a Raspberry Pi. 

You can connect both Ethernet and WiFi to the Raspberry Pi, or if you have a USB expansion antenna that fits the Raspberry Pi you can wirelessly connect the Raspberry Pi to your LAN and WiFi at the same time, and then your Raspberry Pi can act as a gateway in the system.

If the Raspberry Pi is configured as a gateway rather than a wireless access point, then your other devices (e.g., smart home devices, computers, etc.) should connect directly to your local area network (LAN) router. The Raspberry Pi serves primarily as a network traffic forwarder and Network Address Translation (NAT) in this configuration, rather than providing a direct wireless network connection.

**Configuring the wireless network**

1. Edit the wpa_supplicant configuration file
```bash
sudo nano /etc/wpa_supplicant/wpa_supplicant.conf
```

Add the configuration for both networks, similarly:
```bash
network={
    ssid="Your_LAN_SSID"
    psk="Your_LAN_Password"
    id_str="lan"
}

network={
    ssid="Your_Internet_SSID"
    psk="Your_Internet_Password"
    id_str="internet"
}
```

2. Assign interfaces to specific networks

Edit the network interface configuration file:
```bash
sudo nano /etc/network/interfaces
```

Add or edit the following to specify the network that each wireless interface should connect to:

```bash
allow-hotplug wlan0
iface wlan0 inet manual
wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
wpa-ssid Your_LAN_SSID
allow-hotplug wlan1
iface wlan1 inet manual
wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
wpa-ssid Your_Internet_SSID
```

3. Apply the new configuration and restart the network service:

```bash
sudo systemctl daemon-reload
sudo service networking restart
```

Check the status of both interfaces to ensure that they are properly connected to the specified network:

```bash
iwconfig
```

**Setting the Raspberry Pi's static IP address**

1. Edit the dhcpcd configuration file:
```bash
sudo nano /etc/dhcpcd.conf
```
Assuming the expansion antenna wlan1 is used to connect to the LAN
```bash
interface wlan1
static ip_address=192.168.x.x/24
static routers=192.168.x.1
static domain_name_servers=192.168.x.1
```
2. Before configuring a static IP, you can use a network scanning tool (such as nmap) or view the router's management interface to ensure that the selected IP address is not in use.
```bash
sudo apt-get install nmap
nmap -sn 192.168.x.0/24
```
This will scan the entire subnet and list all responding IP addresses.

Impact of not setting static IP addresses:
Dynamic IP address change issues: If the Raspberry Pi uses a dynamic IP address assigned by a DHCP server (such as your router), this address may change on reboots or network changes. This means that other devices on the network will need to constantly update the IP address they use to access the gateway (Raspberry Pi).
Increased network management complexity: Dynamic IPs can lead to increased complexity in network management, especially in troubleshooting and device configuration.
Potential network instability: If the IP address of the gateway changes, devices within the network may temporarily lose connectivity until they obtain a new gateway address.

**Enable IP Forwarding**

Edit the /etc/sysctl.conf file to enable IP forwarding:
```bash
sudo nano /etc/sysctl.conf
```

Find or add the following lines:
```bash
net.ipv4.ip_forward=1
```
In nano, you can save a file by pressing Ctrl+O and then Ctrl+X to exit.
To make these changes take effect immediately, run the following command:
```bash
sudo sysctl -p
```

**Configuring NAT (Network Address Translation)**

Use ‘iptables’ to set up NAT rules to allow the Raspberry Pi to forward traffic. Assume that wlan0 is the interface connected to the Internet and wlan1 is the interface connected to the LAN:
```bash
sudo apt install iptables
sudo iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
sudo iptables -A FORWARD -i wlan0 -o wlan1 -m state –state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i wlan1 -o wlan0 -j ACCEPT
```

Install iptables-persistent to save your rules:
```bash
sudo apt install iptables-persistent
sudo netfilter-persistent save
```

Viewing NAT Table Rules
```bash
sudo iptables -t nat -L -v -n
```

Look for the MASQUERADE rule in the output. This is usually located in the POSTROUTING chain. Make sure that the MASQUERADE rule is applied to the correct egress interface (usually the one connected to the Internet).

View chain-specific rules, for example FORWARD: 
```bash
sudo iptables -L FORWARD -v -n
```

**Setting up the Raspberry Pi as a DHCP server**

Setting up your Raspberry Pi as a DHCP server means that your Raspberry Pi will be responsible for assigning network configuration information such as IP addresses, subnet masks, default gateways, and DNS servers to devices connected to it.

IP address assignment: A DHCP (Dynamic Host Configuration Protocol) server automatically assigns IP addresses, usually temporary, called "leases," to devices on the network. This eliminates the need for devices to manually configure network settings and allows them to plug and play.

Network management is simplified: In a network without a DHCP server, you would need to manually set static IP addresses for each device, which is impractical in large networks. A DHCP server automates this process, making network management simpler.

Dynamic and flexible: A DHCP server can dynamically assign and reclaim IP addresses based on devices joining and leaving, making IP address management more efficient and flexible.

Install dnsmasq:
```bash
sudo apt-get update
sudo apt-get install dnsmasq
```
Configure dnsmasq:
```bash
sudo nano /etc/dnsmasq.conf
```
Add or modify the following configuration in the file to reflect your network settings:
```bash
# Set the IP address range and lease period for DHCP service
dhcp-range=192.168.x.x,192.168.x.y,24h
# Set the IP address of the Raspberry Pi to be used as the gateway.
dhcp-option=option:router,192.168.x.z
# Optional: set the DNS server address
dhcp-option=option:dns-server,192.168.x.z
```
where x is your network segment, 192.168.x.x and 192.168.x.y are IP address ranges (e.g. 192.168.1.50 to 192.168.1.150), and 192.168.x.z is the static IP address of the Raspberry Pi.
```bash
sudo systemctl restart dnsmasq
```

However, If your local area network (LAN) router is already acting as a DHCP server, there is usually no need for additional DHCP server configuration on the Raspberry Pi. The router will be responsible for assigning IP addresses, gateways and DNS information to the devices on the LAN. In this case, the Raspberry Pi acts as a gateway primarily responsible for network traffic forwarding and NAT (Network Address Translation).

**Setting up a network on other LAN-only Raspberry Pi:**

Set the default gateway to point to gateway Raspberry Pi:

If the LAN interface IP on gateway Raspberry Pi is 192.168.x.y, set the default gateway and DNS servers on other LAN-only Raspberry Pi:
```bash
sudo ip route add default via 192.168.x.y
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf > /dev/null
```

**Raspberry Pi Internet Connection Check**

```bash
ping -I wlan0 -c 4 8.8.8.8
# DNS check
ping -I wlan0 -c 4 www.google.com
# Checking the dnsmasq Configuration
sudo nano /etc/dnsmasq.conf
```


**Gateway_set.sh:** Automatically select the best gateway Pi after the Raspberry Pi is switched on.

Run 
```bash 
chmod +x /home/pi/Gateway_set.sh 
```
to give the script execute permission.

Use 
```bash
sudo nano /etc/systemd/system/gateway_check.service
```
to create a new service file.

```bash
[Unit]
Description=Set the best gateway on startup

[Service]
ExecStart=/home/pi/gateway_set.sh
Type=oneshot

[Install]
WantedBy=multi-user.target
```

Run 
```bash
sudo systemctl enable gateway_check.service
sudo systemctl start gateway_set.service
```
to enable the service and start the service.

Confirm the service status and script execution by checking the output of 
```bash
systemctl status gateway_check.service
```

Run 
```bash
cat /tmp/script_log.txt
```
to track down script runtime errors.


**Gateway_check.sh:** A script that runs at regular intervals to compare and select the best gateway.

Use the same step to make the script executable and track down script runtime errors.

Type 
```bash
crontab -e
```
Add
```bash
0 * * * * * /home/pi/Gateway_check.sh
```
at the end of the file to run the script every hour.

Run 
```bash
grep CRON /var/log/syslog
```
to view the logs of cron jobs and confirm that your scripts are executing as expected.


**SelectBestGateway.ps1:** Script to automatically set the best gateway for laptops. Write the code in Notepad and run it using the "Terminal (Administrator)".

By default, Windows may not allow execution of unsigned scripts. You need to change the execution policy of PowerShell. Enter the following command in PowerShell:
```bash
Set-ExecutionPolicy Bypass -Scope Process
```

If you wish to run it regularly on a Windows PC:

In "Task Scheduler", select "Create Task...". Fill in the General tab with the task name and description, and select Run with highest privileges in the Security options section. Switch to the Triggers tab and click "New..." to create a new trigger, e.g. you can choose to run it at a specific time of day or on login. Switch to the "Actions" tab and click on "New...". Select "Start a program" from the "Actions" drop-down menu. Type powershell.exe in "Program/script". In "Add arguments" type -ExecutionPolicy Bypass -File "C:\path\to\your\SelectBestGateway.ps1" (make sure to replace it with the actual path of the script). If desired, you can make additional configurations for the task in the Conditions and Settings tabs.

In addition, to ensure that the network system works properly, sending work status verification messages via telegram is necessary.

Step 1: Get Telegram Bot Token and Chat ID
Create a Telegram Bot: Create a new bot by talking to BotFather and get the token.
Get Chat ID: add your bot to a group or send a message to it, then visit https://api.telegram.org/bot<YourBOTToken>/getUpdates to find your chat_id.

Step 2: Write a script to send Telegram messages
Create Script File: Create a new script file on the Raspberry Pi, such as send_telegram.sh.

Script content: add the following bash code to the script for sending messages via Telegram. Remember to replace <YourBOTToken> and <YourChatID> with the actual token and chat_id.

**send_telegram.sh**: Send a message via Telegram API to prove that the work status is normal.

Step 3: Timing Script Execution with Cron


If you want to build an app using Firebase as a backend and need to use the data from the Raspberry Pi, you can follow the below method:

1．	Preparing data on the Raspberry Pi
For example, you can design a simple data structure to represent this information:

````bash
event_data = {
    "Device ID": "1",
    "Event ID": "1234567890123", 
    "Action": "xxxxxxx",
}
````

2．	Installing and initialising the Firebase SDK

````bash
pip3 install firebase-admin
````

You then need to generate a new private key file (usually a JSON file) in the Firebase console and save that file to the Raspberry Pi.
	In the Firebase console, find and open your project.
	In the sidebar, click on Project Settings (the gear next to the Settings icon).
	Select the "Service Accounts" tab.
	Scroll down to the "Firebase Admin SDK" section and click "Generate new private key". A dialogue box will pop up, confirm and the private key file (JSON file) will be downloaded to your computer.

Then, write a script and use the following code to initialize Firebase:

**transfer_data_script.py**: Used on the Raspberry Pi to transfer data to Firebase.
