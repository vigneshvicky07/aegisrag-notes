# Eval report: contextual-ollama

- Items: 52 (52 with gold labels)
- Golden set: `eval/golden_set.jsonl`

| Metric | mean | p50 | p95 |
|---|---|---|---|
| Recall@5 | 0.894 | 1.000 | 1.000 |
| Recall@20 | 0.894 | 1.000 | 1.000 |
| MRR | 0.904 | 1.000 | 1.000 |
| Latency (s) | 1.357 | 1.067 | 1.219 |

## Per-query results

| # | Query | R@5 | MRR | latency |
|---|---|---|---|---|
| 1 | Explain the TCP three-way handshake. | 1.00 | 1.00 | 17.49s |
| 2 | How does TCP terminate a connection? | 1.00 | 1.00 | 0.98s |
| 3 | Describe TCP slow start and congestion avoidance. | 1.00 | 1.00 | 1.30s |
| 4 | What is the AIMD algorithm? | 1.00 | 1.00 | 1.16s |
| 5 | How does the sliding window protocol work? | 1.00 | 1.00 | 1.08s |
| 6 | Compare stop-and-wait with sliding window protocols. | 1.00 | 1.00 | 1.15s |
| 7 | What is selective repeat ARQ? | 1.00 | 1.00 | 1.34s |
| 8 | How does Go-Back-N work? | 1.00 | 1.00 | 1.11s |
| 9 | Explain the difference between TCP and UDP. | 1.00 | 1.00 | 0.88s |
| 10 | What is the Nagle algorithm? | 1.00 | 1.00 | 1.06s |
| 11 | Describe the Karn algorithm for RTT estimation. | 1.00 | 1.00 | 1.14s |
| 12 | What is silly window syndrome? | 1.00 | 1.00 | 1.14s |
| 13 | Explain the Dijkstra shortest path algorithm for routing. | 1.00 | 1.00 | 1.12s |
| 14 | How does distance vector routing work? | 1.00 | 1.00 | 1.10s |
| 15 | What is the count-to-infinity problem? | 1.00 | 1.00 | 1.12s |
| 16 | Describe the link state routing protocol. | 1.00 | 1.00 | 1.16s |
| 17 | What is OSPF? | 1.00 | 1.00 | 0.91s |
| 18 | Explain hierarchical routing. | 1.00 | 1.00 | 0.93s |
| 19 | What is CIDR notation? | 1.00 | 1.00 | 1.08s |
| 20 | How does Network Address Translation work? | 1.00 | 1.00 | 0.94s |
| 21 | Describe the IPv4 header format. | 1.00 | 1.00 | 1.10s |
| 22 | What are the differences between IPv4 and IPv6? | 1.00 | 1.00 | 0.77s |
| 23 | How does ARP resolve IP addresses to MAC addresses? | 1.00 | 1.00 | 0.85s |
| 24 | Explain the DHCP protocol. | 1.00 | 1.00 | 1.05s |
| 25 | What is ICMP used for? | 1.00 | 1.00 | 1.13s |
| 26 | Describe the CSMA/CD protocol used in Ethernet. | 1.00 | 1.00 | 1.14s |
| 27 | How does CSMA/CA differ from CSMA/CD? | 0.00 | 0.00 | 1.10s |
| 28 | What is the Ethernet frame format? | 0.00 | 0.00 | 0.70s |
| 29 | Explain the ALOHA protocol. | 1.00 | 1.00 | 1.06s |
| 30 | What is slotted ALOHA? | 1.00 | 1.00 | 1.22s |
| 31 | How does a switch differ from a hub? | 1.00 | 1.00 | 0.87s |
| 32 | What is the spanning tree protocol? | 1.00 | 1.00 | 1.07s |
| 33 | Describe VLAN tagging. | 1.00 | 1.00 | 1.09s |
| 34 | Explain cyclic redundancy check for error detection. | 1.00 | 1.00 | 1.09s |
| 35 | How does Hamming code detect and correct errors? | 1.00 | 1.00 | 0.74s |
| 36 | What is the Nyquist theorem? | 1.00 | 1.00 | 0.97s |
| 37 | State Shannon's channel capacity theorem. | 1.00 | 1.00 | 0.97s |
| 38 | Describe frequency division multiplexing. | 1.00 | 1.00 | 1.09s |
| 39 | What is time division multiplexing? | 1.00 | 1.00 | 1.06s |
| 40 | How do fiber optic cables transmit data? | 1.00 | 1.00 | 1.05s |
| 41 | What is the difference between single-mode and multi-mode fiber? | 0.50 | 1.00 | 1.06s |
| 42 | How does DNS resolve domain names? | 1.00 | 1.00 | 0.89s |
| 43 | What is a DNS recursive resolver? | 0.00 | 0.00 | 0.84s |
| 44 | Describe the HTTP request-response cycle. | 1.00 | 1.00 | 1.19s |
| 45 | What are HTTP persistent connections? | 1.00 | 1.00 | 1.05s |
| 46 | Explain how SMTP transfers email. | 1.00 | 1.00 | 1.09s |
| 47 | What is the purpose of TLS? | 0.00 | 0.00 | 1.05s |
| 48 | Describe public key cryptography. | 1.00 | 1.00 | 0.88s |
| 49 | What is a digital certificate? | 0.00 | 0.00 | 1.02s |
| 50 | Explain Diffie-Hellman key exchange. | 1.00 | 1.00 | 1.04s |
| 51 | What is the OSI reference model? | 1.00 | 1.00 | 1.06s |
| 52 | Compare the OSI model to the TCP/IP model. | 1.00 | 1.00 | 1.07s |