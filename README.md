The purpose of this is to emulate a simple client <==> server model with a router inbetween. Different network impairments will be applied to show examples of various network failure scenarios.

While this is simply a Linux container routing between two other containers ()"client" and "server") the concepts apply to Access Points as well; or any network element responsible for carrying traffic between two hosts.
The main impairments we'll be analyzing are:
* Deley
* Packet Loss
* QoS Buffer overflow

# Requirements
* docker

# Quickstart
* git clone this repo
* docker-compose up
* [in 3 terminals] docker attach client|server|router

# Scenario 1 [Upstream Network Delay]
This example will apply delay at the routers outgoing interface towards the server.
In the real world this could happen for any number of reasons, including link capacity, AP/Router CPU, software-but etc.

On the router container lets introduce 250ms of delay on the interface headed towards the Server:
```
tc qdisc add dev eth2 root netem delay 250ms
```
## Verify it's broken
Ping the Server **from the Client**
```
root@33a60d9be0a7:/home/simpleclient# ping 172.18.1.5
PING 172.18.1.5 (172.18.1.5) 56(84) bytes of data.
64 bytes from 172.18.1.5: icmp_seq=2 ttl=63 **time=252 ms**
64 bytes from 172.18.1.5: icmp_seq=2 ttl=63 **time=251 ms**
64 bytes from 172.18.1.5: icmp_seq=3 ttl=63 **time=251 ms**
64 bytes from 172.18.1.5: icmp_seq=4 ttl=63 **time=250 ms**
```
## Analyze the PCAP
When looking at packet captures, choosing the location (or device) is critical. Here we'll run TCPDUMP on the Routers interfaces to isolate where they delay is occuring.

First, on the interface headed towards the client:
```
root@d9fbc3167c1e:/home/router# tcpdump -n -i eth0
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
15:40:23.161935 IP 172.18.0.5 > 172.18.1.5: ICMP echo request, id 22, seq 1, length 64
15:40:23.412485 IP 172.18.1.5 > 172.18.0.5: ICMP echo reply, id 22, seq 1, length 64
```
You can subtract the timestamps to see the same amount of latency being reported by the ping command. Eg; 23.412485 - 23.161935 = 0.25055 (~250ms). Alternatively, you can also print the timestamps as a delta from previously captured packet (no need for the math). For example
```
root@d9fbc3167c1e:/home/router# tcpdump -n -i eth0 -ttt
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
 00:00:00.000000 IP 172.18.0.5 > 172.18.1.5: ICMP echo request, id 29, seq 1, length 64
 **00:00:00.250685** IP 172.18.1.5 > 172.18.0.5: ICMP echo reply, id 29, seq 1, length 64
 00:00:00.750663 IP 172.18.0.5 > 172.18.1.5: ICMP echo request, id 29, seq 2, length 64
 **00:00:00.250717** IP 172.18.1.5 > 172.18.0.5: ICMP echo reply, id 29, seq 2, length 64
 00:00:00.750206 IP 172.18.0.5 > 172.18.1.5: ICMP echo request, id 29, seq 3, length 64
 **00:00:00.250473** IP 172.18.1.5 > 172.18.0.5: ICMP echo reply, id 29, seq 3, length 64
 ```

That just shows us what we already knew; so lets try to isolate it further. Since the delay is apparent on the interface heading towards the client, lets look at the next step upstream, the interface headed towards the server
```
root@d9fbc3167c1e:/home/router# tcpdump -n -i eth2 -ttt
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth2, link-type EN10MB (Ethernet), capture size 262144 bytes
 00:00:00.000000 IP 172.18.0.5 > 172.18.1.5: ICMP echo request, id 30, seq 1, length 64
 **00:00:00.000247** IP 172.18.1.5 > 172.18.0.5: ICMP echo reply, id 30, seq 1, length 64
 00:00:01.003734 IP 172.18.0.5 > 172.18.1.5: ICMP echo request, id 30, seq 2, length 64
 **00:00:00.000207** IP 172.18.1.5 > 172.18.0.5: ICMP echo reply, id 30, seq 2, length 64
 00:00:00.999918 IP 172.18.0.5 > 172.18.1.5: ICMP echo request, id 30, seq 3, length 64
 **00:00:00.000394** IP 172.18.1.5 > 172.18.0.5: ICMP echo reply, id 30, seq 3, length 64
```
Looking at the timestamps, the delay is NOT apparent here. This points at the delay occurring somewhere between the incoming interface FROM the Client and the outgoing interface TO the Server; eg the Router.

But what if we didn't know which direction the delay was being incurred on? Since Ping reports the RTT, let's single out whether the delay is occurring in the upstream or downstream. Perhaps you don't have access to the router, or intermediate network element, but you do have access to the Client and Server. Or maybe you have access to the Client, Server and first-hop router (or Access Point), but do not have access to the next upstream router. For this example, we'll capture on both the Client and Server and take a closer look at the two pcaps to identify where the direction of the incurred delay.
```
$docker ps -f name=client
CONTAINER ID        IMAGE               COMMAND                   CREATED             STATUS              PORTS               NAMES
**33a60d9be0a7**        mike909/client:v1   "/bin/sh -c '\"/home/…"   3 hours ago         Up 3 hours                              client
$docker exec -it 33a60d9be0a7 bash
root@33a60d9be0a7:/home/simpleclient# tcpdump -n -i eth0 -w /mnt/hostdir/client_pings_delayed.pcap
# And on the Server:
root@3c2c5b896825:/home/server# tcpdump -n -i eth0 -w /mnt/hostdir/server_pings_delayed.pcap
tcpdump: listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes

root@33a60d9be0a7:/home/simpleclient# ping 172.18.1.5 -c3
```
Now, in our hosts direectory we have the two pcaps; one from Client and other from Server. In this contrived example we can simply open them both in Wireshark and analyze separately; however in a more complex situation, it may be beneficial to combone both pcaps into a single file, and analyze together. Open one pcap, then File-->Merge and select the other.
Note, Wireshark won't be able to 'follow' the ICMP Echo/Reply conversation from the Client, however you can use the Sequence numbers yourself to follow it. For example:
!(img/combined1.png "foo")
!(img/conbined2.png)

Note, I've changed the preferences to show Source and Dest as MAC instead of IP here to make it easier to read (Preferences-->Columns). I've also modified the address resolution to rename the MACs to Client, Server and Router instead of the actual MAC address. Do this in View-->Name Resolution-->Edit Resolved Name; or populate a file named 'ethers' in your config directory (see Help-->About Wireshark-->Folders). For example, my ethers file:
```
02:42:ac:12:00:05 Client
02:42:ac:12:00:01 Router_eth0
02:42:ac:12:01:01 Router_eth2
02:42:ac:12:01:05 Server
```



# Scenario 2 [Downstream Network Delay]
This is much like Scneario 1, difference being the delay is being applied ONLY to
downstream packets going FROM server TO cliient; egressing router interface eth0.
If this router was an AP, this could be due to contention/congestion, or excessive 802.11 retries; from the POV of the clients packet capture. Clearly here we're doing the pcap on the Client interface, so no 802.11 (where you'd see something like excessive 802.11 retries quite clearly.)
