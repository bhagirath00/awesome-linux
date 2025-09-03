# Chapter 43: Network Performance - Bandwidth Monitoring and Benchmarking (iperf3)

## 1. Introduction: Measuring Network Throughput

Monitoring bandwidth and benchmarking network throughput is essential for verifying cloud network guarantees (e.g., verifying an AWS EC2 instance actually achieves its 10Gbps or 25Gbps network bandwidth spec), detecting switch port duplex mismatches, and identifying traffic spikes.

This chapter covers the three premier Linux network performance tools:
1. **`iperf3`**: The gold standard for synthetic active network throughput benchmarking.
2. **`iftop`**: Real-time interactive bandwidth monitoring by network connection.
3. **`nload`**: Real-time visual incoming/outgoing bandwidth graph.

---

## 2. Active Throughput Benchmarking with `iperf3`

`iperf3` performs point-to-point bandwidth testing between a **Server** and a **Client**:

```
[ iperf3 Client ] ======= 10 Gbps Ethernet Network =======> [ iperf3 Server ]
(Generates TCP streams)                                    (Measures Throughput)
```

### Step 1: Start the Server Node
On the receiving target server, start `iperf3` in listener mode (default port `5201`):
```bash
iperf3 -s
```
```
-----------------------------------------------------------
Server listening on 5201 (test #1)
-----------------------------------------------------------
```

### Step 2: Run the Benchmark from the Client Node
On the initiating client machine:
```bash
# Basic 10-second TCP bandwidth test
iperf3 -c 192.168.1.100
```

```
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-1.00   sec  1.12 GBytes  9.62 Gbits/sec    0             
[  5]   1.00-2.00   sec  1.15 GBytes  9.88 Gbits/sec    0             
[  5]   2.00-3.00   sec  1.15 GBytes  9.88 Gbits/sec    0             
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  11.4 GBytes  9.80 Gbits/sec    0             sender
[  5]   0.00-10.00  sec  11.4 GBytes  9.78 Gbits/sec                  receiver
```
The test confirms the line achieves a solid **9.80 Gbps** line rate with **0 TCP Retransmissions (`Retr`)**.

### Advanced iperf3 Testing Flags:
```bash
# Parallel streams (-P 4): Saturates multi-core network queues
iperf3 -c 192.168.1.100 -P 4

# Reverse test (-R): Measures server-to-client download speed
iperf3 -c 192.168.1.100 -R

# UDP throughput & packet loss jitter testing (-u -b 1G):
iperf3 -c 192.168.1.100 -u -b 1G
```

---

## 3. Real-Time Connection Monitoring with `iftop`

While `iperf3` creates artificial synthetic traffic, **`iftop`** monitors live, real-world traffic on an interface, displaying bandwidth consumed per individual IP conversation pair:

```bash
# Install iftop
sudo apt install -y iftop   # Ubuntu
sudo dnf install -y iftop   # RHEL (EPEL)

# Monitor eth0 (with -n to disable slow reverse DNS lookups)
sudo iftop -n -i eth0
```

```
                 19.1Mb              38.1Mb              57.2Mb              76.3Mb
+-------------------+-------------------+-------------------+-------------------+
10.0.1.15     => 198.51.100.22      14.2Mb  12.8Mb  11.5Mb
              <=                     1.2Mb   1.1Mb   1.0Mb
10.0.1.15     => 172.16.4.88         2.4Mb   2.1Mb   1.9Mb
              <=                     8.5Mb   7.8Mb   6.9Mb
---------------------------------------------------------------------------------
TX:             cum:   42.1MB   peak:   18.5Mb            rates:   16.6Mb  14.9Mb  13.4Mb
RX:                    18.4MB            9.8Mb                      9.7Mb   8.9Mb   7.9Mb
TOTAL:                 60.5MB           28.3Mb                     26.3Mb  23.8Mb  21.3Mb
```
You can immediately identify that external IP `198.51.100.22` is consuming 14.2 Mbps of upload bandwidth.

---

## 4. Visual Throughput Graphs with `nload`

`nload` provides a clean, ASCII graph of incoming and outgoing bandwidth:

```bash
sudo apt install -y nload   # Ubuntu
sudo nload eth0
```

```
Device eth0 [10.0.1.15] (1/1):
================================================================================
Incoming:
            Curr: 8.45 MBit/s
            Avg:  7.80 MBit/s
            Min:  1.20 MBit/s
            Max: 12.40 MBit/s
            Ttl: 42.10 GByte
   .||||....|||...||||.||
Outgoing:
            Curr: 45.20 MBit/s
            Avg:  42.10 MBit/s
            Min:  10.00 MBit/s
            Max:  52.10 MBit/s
            Ttl: 184.20 GByte
   ||||||||||||||||||||||
```

---

## 5. Summary Takeaways

- Benchmark network speed and verify throughput SLAs using `iperf3`.
- Use `iperf3 -P 4` to test parallel multi-stream capability.
- Use `sudo iftop -n -i <interface>` to isolate which IP connection is hoarding bandwidth.
- Use `nload` for a quick graphical overview of incoming and outgoing traffic.
