# Cloudflare-DNS-and-HTTP-Redirect-Lab
Migrating Authoritative DNS from INOS over to Cloudflare, with Zoho Mail, and troubleshooting a Cloudflare edge redirect

**My domain:** `edwinlab.cloud`  
**Redirect destination:** `https://github.com/edwinhchun`

## Overview

This project documents a DNS migration and HTTP redirect implementation. I purchased `edwinlab.cloud` through IONOS, configured Zoho Mail as my mail server, migrated the domain's authoritative DNS to Cloudflare, preserved the email-related DNS records, and created a Cloudflare Redirect Rule that sends visitors to my GitHub profile.

The lab also includes two real troubleshooting scenarios:

- A bad redirect target that created a repeating URL path.
- A macOS DNS resolver-cache issue where `dig` worked while `curl` and `ping` could not resolve the hostname.

## Final Architecture

```text
User enters https://edwinlab.cloud or https://www.edwinlab.cloud
                |
                v
Cloudflare authoritative DNS resolves the hostname
                |
                v
Cloudflare's reverse proxy receives the HTTPS request
                |
                v
Cloudflare Redirect Rule returns HTTP 301
                |
                v
Browser requests https://github.com/edwinhchun
```

## Implementation

### 1. Purchase the Domain through IONOS

I bought `edwinlab.cloud` through IONOS. At this point, IONOS acted as both the registrar and the authoritative DNS provider.

#### Screenshot: IONOS domain overview

![IONOS domain overview](screenshots/01-ionos-domain.png)

> Capture the IONOS domain page showing `edwinlab.cloud` as registered and active. Hide customer numbers, billing details, account identifiers, and unrelated domains.

### 2. Configure Zoho Mail

I configured Zoho Mail before migrating DNS. The email setup required several DNS records:

- **MX records** route inbound email to Zoho Mail.
- **SPF TXT record** identifies systems authorized to send mail for the domain.
- **DKIM TXT record** allows receiving systems to verify Zoho's cryptographic signature.
- **Verification record** proves ownership of the domain to Zoho.

#### Screenshot: Zoho Mail DNS requirements

![Zoho Mail DNS requirements](screenshots/02-zoho-dns-requirements.png)

> Capture Zoho's setup page showing its required MX, SPF, DKIM, or verification records. Do not expose passwords, recovery details, or API keys.

### 3. Add the Domain to Cloudflare

I added the domain to Cloudflare. Cloudflare scanned the existing DNS zone and imported the available records. Before changing nameservers, I reviewed the imported records to confirm that the Zoho Mail records were present.

> [!CAUTION]
> Changing authoritative nameservers changes which provider answers for the entire DNS zone. Missing MX, SPF, or DKIM records can interrupt email delivery or reduce deliverability.

#### Screenshot: Imported DNS records in Cloudflare

![Imported DNS records in Cloudflare](screenshots/03-cloudflare-imported-records.png)

> Show the Zoho MX and TXT records in Cloudflare before changing the nameservers at IONOS. Crop unrelated records and hide sensitive verification values when appropriate.

### 4. Delegate Authoritative DNS to Cloudflare

Cloudflare assigned two authoritative nameservers. I entered those nameservers in the IONOS domain settings. After the change propagated, Cloudflare reported the zone as **Active** and began answering authoritative DNS queries for `edwinlab.cloud`.

Verification command:

```bash
dig NS edwinlab.cloud
```

#### Screenshot: Cloudflare nameserver delegation

![Cloudflare nameserver delegation](screenshots/04-cloudflare-active-nameservers.png)

> Capture either the IONOS nameserver settings showing Cloudflare's nameservers or the Cloudflare Overview page showing the zone as Active. Hide registrar account details.

### 5. Verify Zoho Mail Records after Migration

After Cloudflare became authoritative, I checked the Zoho records again and verified that email still worked. This confirmed that the email provider could remain Zoho while DNS authority moved to Cloudflare.

#### Screenshot: Active Zoho records in Cloudflare

![Active Zoho records in Cloudflare](screenshots/05-zoho-records-in-cloudflare.png)

> Show the active MX, SPF, and DKIM records. Crop the image to the relevant rows and exclude account-level controls.

### 6. Create a Proxied Apex Record

I created a proxied A record for the apex hostname. Cloudflare represents the apex domain with `@`.

| Type | Name | IPv4 address | Proxy status |
|---|---|---|---|
| A | `@` | `192.0.2.1` | Proxied — orange cloud |

`192.0.2.1` belongs to a range reserved for documentation. In this lab, the origin was not expected to serve content. Because the record was proxied, Cloudflare received the request and returned the redirect before contacting the placeholder origin.

#### Screenshot: Proxied apex A record

![Proxied apex A record](screenshots/06-proxied-apex-record.png)

> Show the `@` A record pointing to `192.0.2.1` with proxying enabled. Crop unrelated mail and verification records.

### 7. Create the HTTP Redirect Rule

I created a Cloudflare Redirect Rule with the following configuration:

```text
Incoming request condition:
(http.host eq "edwinlab.cloud")

Action:
Static redirect

Destination URL:
https://github.com/edwinhchun

Status code:
301 - Permanent Redirect
```

#### Screenshot: Cloudflare Redirect Rule

![Cloudflare Redirect Rule](screenshots/07-redirect-rule.png)

> Show the hostname condition, complete destination URL, and `301` status code. Hide account IDs, zone IDs, API tokens, and unrelated rules.

## Verification

### Browser Test

Opening `https://edwinlab.cloud` redirected the browser to my GitHub profile. This confirmed the complete user-facing path from DNS resolution to the Cloudflare edge redirect.

#### Screenshot: Successful browser redirect

![Successful browser redirect](screenshots/08-browser-success.png)

> Include the browser address bar when possible. Hide private bookmarks, unrelated tabs, browser profiles, and private GitHub content.

### DNS Verification with `dig`

I used `dig` to verify that the hostname resolved and that the domain was delegated to Cloudflare nameservers.

```bash
# Complete DNS response
dig edwinlab.cloud

# Only returned IP addresses
dig +short edwinlab.cloud

# Authoritative nameservers
dig NS edwinlab.cloud

# Query Cloudflare's public resolver directly
dig @1.1.1.1 edwinlab.cloud
```

#### Screenshot: `dig` results

![dig results](screenshots/09-dig-results.png)

> Show the A-record response and the authoritative NS records. Crop local usernames or paths if they appear in the prompt.

### HTTP Verification with `curl`

I used `curl` to inspect the HTTP behavior independently of the browser.

```bash
# Show response headers only
curl -I https://edwinlab.cloud

# Follow the redirect
curl -L https://edwinlab.cloud

# Display verbose DNS, connection, TLS, request, and response details
curl -v https://edwinlab.cloud

# Show headers while following the complete redirect chain
curl -IL https://edwinlab.cloud
```

| Flag | Meaning | Useful evidence |
|---|---|---|
| `-I` | Sends a HEAD request and displays response headers | HTTP status and `Location` header |
| `-L` | Follows redirect responses automatically | Final destination and response |
| `-v` | Displays connection, TLS, request, and response details | The layer at which a request fails |
| `-IL` | Displays headers while following the redirect chain | Initial `301` followed by the destination response |

Expected redirect evidence:

```http
HTTP/2 301
location: https://github.com/edwinhchun
server: cloudflare
```

#### Screenshot: Successful `curl` response

![Successful curl response](screenshots/10-curl-301-response.png)

> Show the HTTP `301` response and the `Location` header. Review the output for cookies or unique identifiers before publishing.

## Troubleshooting and Root-Cause Analysis

### Issue 1: Attempting to Use a CNAME for a URL Path

My first approach attempted to point a CNAME at `github.com/edwinhchun`.

A CNAME can point to a hostname:

```text
github.com
```

A CNAME cannot point to a complete URL or URL path:

```text
https://github.com/edwinhchun
github.com/edwinhchun
```

DNS does not understand the `https://` scheme or the `/edwinhchun` path.

**Resolution:** I used DNS to make `edwinlab.cloud` reach Cloudflare, then used an HTTP Redirect Rule to send the browser to the complete GitHub URL.

### Issue 2: Redirect Target Interpreted as a Relative Path

The redirect initially produced a rapidly expanding URL similar to:

```text
https://edwinlab.cloud/github.com/github.com/github.com/...
```

The destination had been entered without a complete URL scheme. Cloudflare interpreted it as a relative path on `edwinlab.cloud`. Because the rule continued to match the same hostname, each request appended another copy of the path.

**Resolution:** I changed the destination to the fully qualified URL:

```text
https://github.com/edwinhchun
```

Including `https://` tells Cloudflare that `github.com` is a separate hostname rather than a path on the current domain.

#### Screenshot: Redirect error and corrected rule

![Redirect error and corrected rule](screenshots/11-redirect-loop-fix.png)

> A before-and-after image can show the malformed expanding URL and the corrected destination. Crop browser history and unrelated account information.

### Issue 3: `dig` Worked while `curl` and `ping` Failed

During terminal testing, `dig` returned valid DNS answers, but `ping` and `curl` reported that the hostname could not be resolved.

```text
ping: cannot resolve edwinlab.cloud: Unknown host
curl: (6) Could not resolve host: edwinlab.cloud
```

This happened because `dig` performs a direct DNS query, while `ping` and `curl` normally use the macOS system resolver. The browser could also continue working because it had a cached result or used a different resolver path.

I flushed the macOS DNS cache and restarted the resolver service:

```bash
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

Afterward, `ping` and `curl` successfully resolved the hostname.

#### Screenshot: macOS resolver troubleshooting

![macOS resolver troubleshooting](screenshots/12-macos-dns-troubleshooting.png)

> Show three stages: `dig` resolving, `curl` or `ping` failing, and `curl` succeeding after the cache flush. Crop local usernames, computer names, and unrelated terminal history.

## What Each Test Proved

| Test | Primary question | What success proved |
|---|---|---|
| `dig` | Does DNS publish an answer for this hostname? | The authoritative DNS zone and record were available |
| `dig NS` | Which nameservers are authoritative? | IONOS delegated DNS to Cloudflare |
| `ping` | Can the OS resolve the hostname, and does the target answer ICMP? | System name resolution worked; lack of an ICMP reply alone would not prove the website was down |
| `curl -I` | What status and headers does the HTTPS endpoint return? | Cloudflare returned the intended redirect and `Location` header |
| `curl -L` | Where does the redirect chain end? | The client reached the GitHub destination |
| `curl -v` | Where in DNS, TCP, TLS, or HTTP does the request fail? | The detailed connection sequence could be inspected |
| Browser | Does the complete user experience work? | A normal client could resolve, connect, receive the redirect, and load the destination |

## Key Lessons Learned

- The registrar and authoritative DNS provider can be different companies.
- Changing nameservers changes the authoritative source for the entire DNS zone.
- Email can remain hosted by Zoho while authoritative DNS moves to Cloudflare.
- MX, SPF, DKIM, and verification records must be preserved during a DNS migration.
- An apex record represented by `@` applies to `edwinlab.cloud`, not automatically to `www.edwinlab.cloud`.
- CNAME flattening allows an apex hostname to reference another hostname, but it does not make DNS understand URL paths.
- A CNAME is a DNS alias, not an HTTP redirect.
- Cloudflare Redirect Rules operate at the HTTP layer and require requests to reach a proxied hostname.
- A fully qualified redirect target must include the URL scheme, such as `https://`.
- Troubleshooting should proceed layer by layer: DNS, connectivity, TLS, and HTTP.
- Browser success does not always prove the operating-system resolver is healthy because caches and resolver paths can differ.
