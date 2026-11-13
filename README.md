# IoT Network Protocol Analysis

Network traffic analysis lab using Wireshark to capture, dissect, and validate
secure IoT communication protocols across a full CDA-GDA-Cloud architecture.

**Key result:** with 10% packet loss, MQTT QoS 2 (exactly once) latency rose from 0.47 ms to 1.88 ms, while QoS 0
stayed flat at about 0.27 ms. More handshake steps = more delay when packets are lost.
Full write-ups: [`Project-MQTT-CoAP.pdf`](Project-MQTT-CoAP.pdf) (TELE 6510 project) and [`WiresharkTask1.pdf`](WiresharkTask1.pdf).

## Architecture

<p align="center"><img src="docs/architecture.svg" width="100%" alt="Two Ubuntu VMs on QEMU/KVM: a device VM running coap-client and mosquitto_pub, and an application VM at 192.168.122.30 running a custom CoAP server with a /test resource, the MQTT broker side and Wireshark. Packet loss of 0, 3, 5 and 10 percent is added on the access side. Messages per mode: CoAP CON GET then ACK 2.05; CoAP NON GET then 2.05; MQTT QoS 0 PUBLISH; QoS 1 PUBLISH, PUBACK; QoS 2 PUBLISH, PUBREC, PUBREL, PUBCOMP. Average latency from 0 to 10 percent loss: CoAP CON 0.24 to 0.34 ms, NON 0.14 to 0.15, QoS 0 0.26 to 0.27, QoS 1 0.31 to 0.97, QoS 2 0.47 to 1.88."></p>

**Why this architecture**

| Choice | Why |
|---|---|
| Two VMs (device and application) | Traffic really crosses a network link, so Wireshark sees every packet between them. |
| Wireshark timestamps for latency | Measures request → response on the wire, not in the client tools. |
| Loss added on the access side at 0, 3, 5 and 10% | The same loss levels for every protocol make the results comparable. |
| Small payload (5-byte "hello" for CoAP, `Temp=…` for MQTT) | The task asked for a readout of 10 bytes or less, so size doesn't hide protocol overhead. |
| 30 transactions per setting, averaged | One request is too noisy at sub-millisecond times. |

## Analysis Coverage

| Protocol | What Was Captured |
|----------|------------------|
| MQTT | All 14 control packet types including CONNECT, PUBLISH, SUBSCRIBE |
| TLS/SSL | Full handshake sequence and certificate exchange |
| TCP/IP | Connection establishment, data transfer, teardown |
| CoAP | Constrained device request/response patterns |

## Results

All numbers are copied from [`Project-MQTT-CoAP.pdf`](Project-MQTT-CoAP.pdf). Each value is the average of 30
transactions, read from Wireshark timestamps.

**Part I: CoAP confirmable vs non-confirmable** (avg ms)

| Packet loss | CoAP CON | CoAP NON |
|---|---|---|
| 0% | 0.22 | 0.14 |
| 3% | 0.29 | 0.13 |
| 5% | 0.32 | 0.14 |
| 10% | 0.36 | 0.15 |

**Part II: CoAP vs MQTT QoS levels** (avg ms)

| Packet loss | CoAP CON | MQTT QoS 0 (at most once) | MQTT QoS 1 (at least once) | MQTT QoS 2 (exactly once) |
|---|---|---|---|---|
| 0% | 0.24 | 0.26 | 0.31 | 0.47 |
| 3% | 0.27 | 0.27 | 0.42 | 0.83 |
| 5% | 0.31 | 0.27 | 0.54 | 1.02 |
| 10% | 0.34 | 0.27 | 0.97 | 1.88 |

- CoAP CON latency went up 64% from 0% to 10% loss in Part I (retransmissions after a missing ACK); NON stayed flat
  because lost NON messages are never resent.
- QoS 1 and QoS 2 grow fastest because each lost PUBACK / PUBREC / PUBREL / PUBCOMP forces a retry.
- [`WiresharkTask1.pdf`](WiresharkTask1.pdf): Wireshark set up on an Ubuntu VM (needs `sudo` to see interfaces),
  traffic captured on `ens33`, filtered for HTTP; frame 16545 shows a `GET` from 192.168.203.129 answered with
  `HTTP/1.1 200 OK`, TTL 64.

## What worked and what didn't

| | |
|---|---|
| ✅ | Every mode's message exchange is visible in the captures: CON/ACK, NON, and the 1-, 2- and 4-step MQTT handshakes. |
| ✅ | The results show a clear pattern: more acknowledgement steps = more delay under loss (QoS 2 went up 4×, QoS 0 not at all). |
| ⚠️ | The loss setting was checked with short ping runs (6, 38, 10 and 12 pings), which read 0%, 5%, 10% and 17% loss. No ping check is shown for 3%. Small samples make the real loss rate uncertain. |
| ⚠️ | The CoAP CON numbers differ slightly between the two parts (0.22–0.36 ms vs 0.24–0.34 ms), so differences of a few hundredths of a ms are within run-to-run noise. |
| ⚠️ | NON and QoS 0 latency only counts messages that arrived; how many were lost was not measured. |
| ⚠️ | The MQTT vs HTTP comparison (part II.3) is a written discussion, not a measurement. |
| ⚠️ | Only screenshots are in the repo; the raw `.pcap` capture files are not included. |

## 🔧 Technologies Used
`Wireshark` `MQTT` `TLS/SSL` `TCP/IP` `CoAP` `Mosquitto` `libcoap` `QEMU/KVM`

## 🎯 Skills Demonstrated
- Deep packet inspection and protocol dissection
- Validating end-to-end encrypted IoT communication
- Network traffic analysis for security verification
- IoT protocol behavior across constrained and gateway devices
