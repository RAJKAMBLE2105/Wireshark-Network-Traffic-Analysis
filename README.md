# Task 8 --- Capture Network Traffic with Wireshark

## Objective

Capture live network traffic with **Wireshark**, apply protocol-specific
display filters, inspect packet contents, analyse a TCP three-way
handshake, and document security observations about encrypted
vs. unencrypted traffic.

## Tools / Tech Stack

-   **Wireshark**
-   Local Wi-Fi/network interface
-   Windows host
-   Test traffic generated from the local machine
-   Capture file: `wireshark_capture.pcap`

------------------------------------------------------------------------

## 1. Wireshark Installation

### Installation steps

1.  Download and install Wireshark from the official Wireshark website.
2.  During installation on Windows, allow the **Npcap** component to be
    installed. Npcap allows Wireshark to capture packets from network
    interfaces.
3.  Start Wireshark with the required permissions. On Windows,
    administrator privileges may be required depending on the interface
    and capture configuration.
4.  Select the active **Wi-Fi/local network interface**.
5.  Start the capture and allow normal network activity to run for at
    least two minutes.
6.  Stop the capture and save it as:

``` text
wireshark_capture.pcap
```

> **Safety note:** This analysis was performed on traffic from a
> local/test environment. Packet captures can contain sensitive
> information, so captures should not be shared publicly unless they
> have been reviewed and sanitized.

------------------------------------------------------------------------

## 2. Live Traffic Capture

The capture contains traffic from the local machine, including DNS, TCP,
HTTP, TLS, and QUIC activity.

The screenshots show packet timestamps extending from approximately **77
seconds to more than 206 seconds**, demonstrating that the captured
traffic spans more than the required two-minute period.

![Capture overview](screenshots/capture_overview.png)

------------------------------------------------------------------------

## 3. HTTP Traffic Analysis

### Display filter

``` text
http
```

This filter isolates packets that Wireshark identifies as HTTP traffic.

![HTTP display filter](screenshots/http_filter.png)

### Finding

The capture contains an HTTP request:

``` text
GET /online/ HTTP/1.1
Host: majesticsilverclearrain.neverssl.com
```

The packet uses:

``` text
Source: 10.164.210.21
Destination: 34.223.124.45
Source Port: 64413
Destination Port: 80
```

Port **80** is the standard port associated with HTTP.

------------------------------------------------------------------------

## 4. DNS Traffic Analysis

### Display filter

``` text
dns
```

The DNS filter isolates domain-name resolution traffic.

![DNS display filter](screenshots/dns_filter.png)

### Observations

The capture shows DNS queries and responses for domains including:

-   `content-autofill.googleapis.com`
-   `www.shahucollege...`
-   `chatgpt.com`

The screenshot also shows both **A** and **AAAA** lookups. An A record
resolves a hostname to an IPv4 address, while an AAAA record resolves it
to an IPv6 address.

DNS is important for normal web connectivity because a client commonly
needs to resolve a hostname before connecting to the destination server.

------------------------------------------------------------------------

## 5. TCP Three-Way Handshake

### Display filter

A TCP-specific filter was used to isolate the connection:

``` text
tcp.port == 33456 && tcp.port == 443
```

The relevant connection is between the local IPv6 address and a remote
HTTPS server on port **443**.

![TCP three-way handshake](screenshots/tcp_3_way_handshake.png)

### Handshake sequence

The capture shows the standard TCP connection establishment:

  --------------------------------------------------------------------------
  Step                    Packet                     Meaning
  ----------------------- -------------------------- -----------------------
  1                       `33456 → 443 [SYN]`        Client requests a new
                                                     TCP connection.

  2                       `443 → 33456 [SYN, ACK]`   Server accepts the
                                                     request and
                                                     acknowledges it.

  3                       `33456 → 443 [ACK]`        Client acknowledges the
                                                     server, completing the
                                                     TCP handshake.
  --------------------------------------------------------------------------

After the handshake, the capture shows **TLS 1.3** traffic, including a
Client Hello and Server Hello, followed by encrypted application data.

### Why the handshake matters

The TCP three-way handshake synchronizes sequence numbers and confirms
that both endpoints are ready to communicate. It establishes the
reliable TCP connection before application data is exchanged.

------------------------------------------------------------------------

## 6. Unencrypted HTTP Data

One of the most important security observations is the HTTP GET request
captured in plaintext.

![Unencrypted HTTP GET](screenshots/http_get_unencrypted.png)

The packet exposes information such as:

``` text
GET /online/ HTTP/1.1
Host: majesticsilverclearrain.neverssl.com
Connection: keep-alive
User-Agent: Mozilla/5.0 ...
Referer: http://majesticsilverclearrain.neverssl.com/online/
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
```

The packet also reveals the communicating IP addresses and TCP ports.

### What is visible?

Because this is ordinary HTTP rather than HTTPS, the HTTP request
headers and requested URL are readable directly from the packet capture.
A person able to observe the network could potentially see:

-   The destination hostname
-   The requested path/URL
-   HTTP headers
-   Browser/user-agent information
-   Source and destination IP addresses
-   Source and destination ports
-   Other application data carried over the HTTP connection

No password or other sensitive credential is claimed here; the security
issue demonstrated by the capture is that **application-layer HTTP data
is readable in transit**.

------------------------------------------------------------------------

## 7. Why Unencrypted HTTP Is Dangerous

HTTP does not provide encryption for the application data it transports.

If a user sends information over plain HTTP on a network that an
attacker can monitor, the attacker may be able to inspect the traffic
and read or modify the transmitted content. On an untrusted network,
this can enable eavesdropping, traffic manipulation, session theft when
credentials/cookies are exposed, and other man-in-the-middle attacks.

The screenshot demonstrates the basic problem: the HTTP request is
visible as readable text inside Wireshark.

------------------------------------------------------------------------

## 8. How HTTPS Prevents This Type of Eavesdropping

HTTPS is HTTP transported through **TLS (Transport Layer Security)**.

In the capture, the connection to port **443** shows TLS 1.3 messages
such as:

``` text
Client Hello
Server Hello
Application Data
```

TLS provides encryption and integrity protection for application data
after the secure session is established. As a result, a passive network
observer can still see metadata such as IP addresses, ports, packet
sizes, and timing, but normally cannot simply read the protected HTTP
request/response contents from the captured TLS application-data
packets.

HTTPS also uses server certificates to authenticate the server and helps
protect against attackers impersonating the intended server.

### HTTP vs HTTPS

  -----------------------------------------------------------------------
  Feature                 HTTP                    HTTPS
  ----------------------- ----------------------- -----------------------
  Default port            80                      443

  Application data        No                      Yes, using TLS
  encrypted                                       

  Readable HTTP headers   Generally yes           Protected after TLS
  in a packet capture                             encryption

  Integrity protection    No built-in TLS         Yes, provided by TLS
                          protection              

  Protection from passive Poor                    Stronger
  eavesdropping                                   
  -----------------------------------------------------------------------

> HTTPS does **not** make all network metadata invisible. DNS activity,
> destination information, packet timing, and other metadata can still
> be observable depending on the network and protocols in use.

------------------------------------------------------------------------

## 9. Security Observations

1.  **Plain HTTP exposes application data.**\
    The captured `GET /online/` request demonstrates that HTTP requests
    can be read directly from a packet capture.

2.  **DNS reveals queried domains.**\
    DNS packets can reveal which domain names a device is resolving,
    even when the eventual website connection uses HTTPS.

3.  **TCP establishes the connection before application traffic.**\
    The SYN → SYN-ACK → ACK sequence provides a clear example of TCP
    connection establishment.

4.  **TLS protects application data.**\
    The port 443 traffic in the capture proceeds into TLS 1.3 and
    encrypted application data rather than exposing the HTTP request as
    readable plaintext.

5.  **Packet captures can contain sensitive information.**\
    PCAP files should be handled carefully because they may contain
    addresses, hostnames, URLs, headers, and potentially credentials or
    other private data.

------------------------------------------------------------------------

## 10. Wireshark Display Filters Used

``` text
http
```

Used to isolate HTTP traffic.

``` text
dns
```

Used to isolate DNS queries and responses.

``` text
tcp.port == 33456 && tcp.port == 443
```

Used to isolate the TCP connection between the local ephemeral port
`33456` and HTTPS port `443` for analysing the three-way handshake.

------------------------------------------------------------------------

## 11. Capture File

The final packet capture should be saved in the repository as:

``` text
wireshark_capture.pcap
```

Recommended project structure:

``` text
.
├── README.md
├── wireshark_capture.pcap
└── screenshots/
    ├── capture_overview.png
    ├── http_filter.png
    ├── dns_filter.png
    ├── tcp_3_way_handshake.png
    └── http_get_unencrypted.png
```

------------------------------------------------------------------------

## 12. Feature Checklist

-   [x] Install Wireshark and document installation/permission
    requirements
-   [x] Capture more than 2 minutes of local network traffic
-   [x] Apply `http` display filter and document the results
-   [x] Apply `dns` display filter and document the results
-   [x] Apply TCP filter and analyse a complete three-way handshake
-   [ ] Export/copy the final capture as `wireshark_capture.pcap` into
    the GitHub repository
-   [x] Identify an unencrypted HTTP GET request
-   [x] Document information visible in the unencrypted request
-   [x] Explain the security risks of HTTP
-   [x] Explain how HTTPS/TLS protects application data
-   [x] Include a glossary

> **Repository note:** The screenshots supplied for this README are
> included in the accompanying `screenshots/` folder. Add the actual
> `wireshark_capture.pcap` file to the repository before marking the
> final checklist item as complete.

------------------------------------------------------------------------

## 13. Glossary

### Packet

A **packet** is a small unit of network data sent from one device to
another. It contains control information such as addresses and protocol
information along with some of the data being transmitted.

### Protocol

A **protocol** is a set of rules that devices follow to communicate.
Examples in this task include HTTP, DNS, TCP, and TLS.

### Port

A **port** is a numbered logical endpoint used to identify a network
service or application on a device. For example, HTTP commonly uses port
`80`, while HTTPS commonly uses port `443`.

### Payload

A **payload** is the actual data carried by a packet or protocol
message, excluding the addressing and control information needed to
deliver and process it.

### Handshake

A **handshake** is an exchange of messages used by two network endpoints
to establish or negotiate communication. In TCP, the three-way handshake
is **SYN → SYN-ACK → ACK**.

------------------------------------------------------------------------

## Conclusion

This Wireshark exercise demonstrates how packet analysis can reveal the
behaviour of common network protocols. DNS traffic shows domain
resolution, TCP shows reliable connection establishment, and TLS
protects application data on HTTPS connections. Most importantly, the
captured HTTP GET request demonstrates why unencrypted HTTP should not
be used for sensitive communication: its application-layer contents can
be inspected directly from network traffic.

For secure web communication, **HTTPS/TLS should be preferred over plain
HTTP**, especially when credentials, personal information, session
tokens, or other sensitive data are involved.
