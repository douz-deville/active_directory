# 06 - Configure DNS Role

## 1. Phase Objectives

* Verify that the DNS service is functioning correctly.
* Implement reverse DNS resolution using a **Reverse Lookup Zone**.
* Reduce the attack surface by configuring **secure DNS Forwarders**.
* Prevent Man-in-the-Middle (MitM) attacks through **Secure Dynamic Updates**.
* Implement **Aging & Scavenging** to automatically remove stale DNS records.

## 2. DNS Role Verification

```powershell
# Verify the DNS service
Get-Service DNS
```

## 3. Reverse Lookup Zone

The reverse lookup zone allows an IP address to be resolved to an FQDN through **PTR records**. This is useful for network troubleshooting (`nslookup`) and security auditing.

* **Zone type:** Primary Zone
* **Storage:** Active Directory-integrated (*Store the zone in Active Directory*)
* **Replication scope:** All DNS servers running on domain controllers in this domain
* **Network ID:** `192.168.197` (according to the lab's network range)

## 4. DNS Forwarders Configuration and Hardening

### Configuration Applied

1. Open `dnsmgmt.msc` → Right-click the DC → **Properties** → **Forwarders** tab.
2. Replace the default gateway/router with security-focused public DNS resolvers:

   * **Quad9 (Primary):** `9.9.9.9`
   * **Cloudflare Security:** `1.1.1.2`
3. **Uncheck**:

   * *Use root hints if no forwarders are available*

### Security Hardening Rationale

* **C2 / Phishing Filtering:** Quad9 can block the resolution of domains associated with malicious activity.
* **Network Isolation:** Domain clients should query the DC for DNS resolution. Outbound DNS traffic (port 53) to the Internet should be blocked by the firewall for all clients except the DC.
* **Root Hints Disabled:** Prevents the DNS server from bypassing the configured security filtering if the forwarders become temporarily unavailable.

## 5. Securing Dynamic Updates

For both the forward lookup zone (`techcorp.local`) and the reverse lookup zone:

* Right-click the zone → **Properties** → **General** tab.
* Set **Dynamic updates** to **Secure only**.

### Security Rationale

* Only authenticated, domain-joined systems can create or modify DNS records.
* Prevents unauthenticated attackers from injecting malicious IP addresses into DNS records to redirect traffic toward rogue systems.
* Uses **GSS-TSIG** to associate DNS record ownership with the corresponding computer account in Active Directory.

## 6. DNS Record Lifecycle Management (Aging & Scavenging)

### Interval Configuration

* **No-refresh interval:** `7 days`
  Prevents the timestamp from being refreshed when the IP address has not changed, reducing unnecessary Active Directory replication traffic.
* **Refresh interval:** `7 days`
  Defines the period during which the client must refresh its DNS record.
* **Total validity period:** `14 days`
  After this period, an unchanged record becomes eligible to be considered **stale**.

### Two-Step Activation

1. **At the Zone level:**
   Right-click the zone → **Properties** → **Aging** → Enable *Scavenge stale resource records*.

2. **At the DNS Server level:**
   Right-click the DC → **Set Aging/Scavenging for All Zones** → Enable scavenging and configure the scavenging interval, for example, every `7 days`.

### Security Rationale

* **Prevention of DNS Stale Record Takeover:** Helps prevent situations where DHCP reassigns an old IP address to another system while a stale DNS record still points to that address.
* **Reduction of LLMNR / NBT-NS Fallback:** Keeping DNS records accurate reduces failed DNS resolutions that may cause clients to fall back to local name-resolution protocols vulnerable to NTLM relay attacks, such as those exploited by tools like Responder.
* **Active Directory Mapping Integrity:** Removes obsolete DNS information that could otherwise provide inaccurate or misleading information during network reconnaissance, including tools such as BloodHound.
