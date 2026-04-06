## DHCP (Dynamic Host Configuration Protocol)

### The Effect of DHCP

When a new device wants to join a network, DHCP provides it with basic network configuration (IP address, subnet mask, gateway, DNS server, etc.).

## DNS (Domain Name System)

### What is DNS?
DNS is a distributed database that translates human-friendly domain names (e.g., `www.google.com`) into machine-readable IP addresses. It acts as the "phonebook" of the Internet.

### Hierarchical Structure
DNS uses a hierarchical tree structure to manage domain names:
1. **Root DNS Servers**: The top of the hierarchy.
2. **Top-Level Domain (TLD) Servers**: Responsible for `.com`, `.org`, `.edu`, and country codes like `.cn`.
3. **Authoritative DNS Servers**: Provide the actual IP mapping for a specific organization's hosts.

### Resolution Process
*   **Recursive Query**: The client (resolver) asks the **Local DNS Server** to find the answer. The Local DNS Server does all the work and returns the final result.
*   **Iterative Query**: The contacted server replies with the address of the next DNS server in the hierarchy for the client to ask.

### Common DNS Records (Resource Records)
*   **A**: Maps a hostname to an IPv4 address.
*   **AAAA**: Maps a hostname to an IPv6 address.
*   **CNAME**: An alias (canonical name) for another domain.
*   **MX**: Specifies the mail servers for the domain.
*   **NS**: Specifies the authoritative name servers for the domain.

### DNS Caching
To reduce latency, DNS results are cached at various levels (Browser, OS, Local DNS Server) for a duration specified by the **TTL (Time To Live)** value.