---
layout: post
title:  "Nahamcon CTF 22: Project Circuitbreaker"
date:   2022-12-17 10:02:59 +0100
categories: jekyll update
---
I tried finding some time to do Nahamcon this year and I found a few hours between kids going to be and kids birthday party. 
This is a writeup about:
[https://www.projectcircuitbreaker.com/nahamconeu-2022/](https://www.projectcircuitbreaker.com/nahamconeu-2022/)

Checking out the packet capture. I started out in wireshark and then went to tshark when I needed to get the data out.

```sh
tshark -r projectcircuitbreaker.pcap
    1   0.000000  10.10.1.101 → 10.10.1.10   UDP 118 10110 → 10110 Len=76
    2 528029.000000  10.10.1.101 → 10.10.1.10   UDP 121 10110 → 10110 Len=79
    3 -302543.000000  10.10.1.101 → 10.10.1.10   UDP 118 10110 → 10110 Len=76
    4 -989857.000000  10.10.1.101 → 10.10.1.10   UDP 118 10110 → 10110 Len=76
    5 -667123.000000   10.10.1.10 → 10.10.1.50   Syslog 221 DAEMON.INFO: Oct 08 01:26:18 flightcomputer telemetryd ㅏйc派Mㅐㅅaзi5*ㅎCx9ㅕи갉ㅐ11фиㅍ оЯㅇх01т쭈飞очᆷㅔㅅA0ㅅ咖йㅌㅅㅊE@x:ㅋ41ㅂㅃㅕㅌㅎийн가
    6 131814.000000  10.10.1.101 → 10.10.1.10   UDP 121 10110 → 10110 Len=79
    7 -1229906.000000   10.10.1.10 → 10.10.1.50   Syslog 126 DAEMON.INFO: Oct 01 13:06:35 flightcomputer telemetryd FWC/ECAM CRC mismatch, frame discarded
    8 -2061137.000000  10.10.1.101 → 10.10.1.10   UDP 118 10110 → 10110 Len=76
    9 -2176367.000000  10.10.1.101 → 10.10.1.10   UDP 118 10110 → 10110 Len=76
   10 -2233144.000000  10.10.1.101 → 10.10.1.10   UDP 121 10110 → 10110 Len=79
```

First we note that the packets are not in time order. This will be important.
There seems to be two types of packet. Syslogs and some other packets.
The syslogs seem to be mostly avionics stuff like:
```
DAEMON.INFO: Oct 22 11:19:21 flightcomputer telemetryd ECAM: Check LGCIU-PHC 5 interface (INTMT)
DAEMON.INFO: Oct 01 13:06:35 flightcomputer telemetryd FWC/ECAM CRC mismatch, frame discarded
DAEMON.INFO: Oct 10 18:09:14 flightcomputer telemetryd ECAM: Fuel consumption -1.66% off predicted consumption, recalculating range
DAEMON.INFO: Oct 22 11:07:41 gps gpsd Interference on ant 0 cleared

```
Checking out the other udp packets they are 

```sh
...
$GPRMC,115613.00,A,4532.2863352,N,12257.6437174,W,000.0,000.0,190918,000.0,W*5D
$GPGGA,115613.00,4532.2863352,N,12257.6437174,W,1,5,1.5,100.01,M,0.00,M,,*72
...
```

what appears to be NMEA 0183 GPS-data.
I sorted the data and took only the NMEA data using the following python script
```python
from scapy.all import *
packets = []

for packet in PcapReader('projectcircuitbreaker.pcap'):
    packets.append(packet)

packets.sort(key=lambda x: x.time)
j
for packet in packets:
    data = packet[UDP].load.decode()
    if data[0] == "$":
        print(data)
```

and used [https://nmeagen.org](nmeagen.org) to plot it. It was a lot of points, it lagged badly and it looked like garbage on the map.

About when I was starting to lose hope I thought about the gps interference message. and in wireshark I found:
![](/assets/interference.png)
Looking at that it's kinda obvious that the bad gps packets also have bad checksum.

Added the following code, also could probably have looked at checksum as well:
```python
    if "on ant 0 cleared" in data:
        INTERFERENCE = False
    elif "on ant 0" in data:
        INTERFERENCE = True
    if not INTERFERENCE:
        if data[0] == "$":
            print(data)

```

This gives us the following map:

![](/assets/circuitbreaker.png)
