# DDNS: Using an External Dynamic List to Trigger Updates

![PAN-OS](https://img.shields.io/badge/PAN--OS-Workaround-orange)
![PA-415-5G](https://img.shields.io/badge/PA--415--5G-Tested-blue)
![Status](https://img.shields.io/badge/Status-Unofficial-red)

## Summary

This guide comes from a practical limitation I hit on a PA-415-5G. I needed DDNS for the cellular WAN, but PAN-OS does not support DDNS on cellular interfaces, and the native feature is limited to a small set of providers.

In this workaround, an **External Dynamic List (EDL)** is used as a scheduled HTTP/HTTPS fetcher, so each refresh can act like a DDNS update request sent to a provider endpoint.

This is **not an official** or **fully secure design**. It is a workaround, but it can still help when a firewall needs a DNS record for a dynamic IP used in ACLs, whitelists, peer definitions, reachability, or allow-list use cases.

## How it works

An EDL is a remote text resource that PAN-OS retrieves on a schedule. By setting the EDL source URL to a DDNS provider update endpoint instead of a normal IOC feed, the firewall can generate repeated DDNS update requests without any external script.

The egress path can be controlled with Service Route, so the request can be forced out through the interface of interest, including the 5G path, as long as the routing design supports it. Static routing can also influence the path to the provider, but Service Route is usually the cleaner option for firewall-originated traffic.

## Verified provider patterns

| Provider | Simple GET | Verified endpoint pattern | Auto-detect source IP | Notes |
|---|---|---|---:|---|
| DuckDNS | Yes | `https://www.duckdns.org/update?domains={subname}&token={token}&ip=` | Yes for IPv4 if `ip` is omitted or blank | Very simple and well suited for EDL. Response is `OK` or `KO`. |
| Dynu | Yes | `https://api.dynu.com/nic/update?hostname={hostname}&password={password}` | Yes when IP is omitted, or can be explicitly set | Strong candidate. Returns short status text such as `good <ip-address>`. |
| No-IP | Yes | `https://dynupdate.no-ip.com/nic/update?hostname={hostname}&myip=` | Yes if `myip` is left blank in most cases | Requires credentials or DDNS key. Still simple enough for EDL. |
| FreeDNS / Afraid.org | Yes | `https://freedns.afraid.org/dynamic/update.php?{token}` or `https://sync.afraid.org/u/{token}/` | Yes, using tokenized/direct URL methods | Good fit because tokenized URLs are very simple. |

HTTP is documented by the providers as well, but HTTPS is preferred when available.

## Example URLs

### DuckDNS

```text
https://www.duckdns.org/update?domains=YOURSUBDOMAIN&token=YOURTOKEN&ip=
```

DuckDNS uses a simple HTTPS GET to update the domain and can auto-detect IPv4 if `ip` is not specified. A normal successful reply is `OK`.

### Dynu

```text
https://api.dynu.com/nic/update?hostname=YOURHOSTNAME&password=YOURPASSWORD
```

Dynu exposes a simple HTTP/HTTPS update protocol and supports source-IP based updates when no explicit IP is provided. A successful response typically returns short status text such as `good <ip-address>`.

### No-IP

```text
https://dynupdate.no-ip.com/nic/update?hostname=YOURHOSTNAME&myip=
```

No-IP documents a direct update request URL and states that `myip` can usually be left blank unless a specific IP should be sent.

### FreeDNS / Afraid.org

```text
https://freedns.afraid.org/dynamic/update.php?YOUR_UPDATE_TOKEN
```

or

```text
https://sync.afraid.org/u/YOUR_RANDOMIZED_TOKEN/
```

FreeDNS exposes both tokenized direct URLs and randomized update-token URLs that can be fetched directly. These methods are particularly attractive for this workaround because they avoid complex headers and can rely on the apparent public source IP.

## Parser limitation

PAN-OS expects an EDL to contain a valid plain text IP list, URL list, or domain list. The short success messages returned by DDNS providers, such as DuckDNS `OK` or Dynu `good <ip-address>`, are not valid policy list entries for normal IP, URL, or domain parsing.

That means the firewall may import zero useful entries or flag the content as invalid. For this reason, the EDL should be treated as a scheduler and HTTP trigger, not as a meaningful object for production policy enforcement.

## Important behavior: the EDL must be referenced

A dummy rule is required. In practice, the EDL must be referenced in at least one active rule, otherwise it may not refresh reliably. The dummy rule keeps the EDL active without affecting normal traffic.

A low-impact way to keep the EDL active is:

- EDL type: IP List.
- Source zone: management or a dedicated test zone.
- Source address: one known /32.
- Destination: the EDL object.
- Application: `ping`.
- Service: `application-default`.
- Action: `allow`.

Suggested rule description:

```text
Do not remove: keeps EDL active for DDNS workaround
```

This keeps the rule almost inert while still making the EDL active enough to refresh reliably in practice.

## Screenshots

### EDL Object
<img width="1767" height="837" alt="EDL Object" src="https://github.com/user-attachments/assets/3d8fcdf2-fce2-4651-b3f2-cd613fce2feb" />

### Dummy Rule
<img width="1804" height="237" alt="Dummy Rule" src="https://github.com/user-attachments/assets/2115cd83-295f-4a17-9f2c-ed42ceb678be" />

## Security notes

This method is convenient, but it is not ideal from a security standpoint. The update URL may contain credentials or tokens in the query string.

HTTPS can be used and works with certificate profile set to None. In that case, PAN-OS does not validate the server certificate and will generate a warning at commit time. For full server certificate validation, a certificate profile with the provider's CA chain must be configured.

HTTP is also supported and documented by several providers, but the traffic is sent in clear text.

Because this is not the intended use of the EDL feature, it should always be presented as an unsupported workaround rather than as a supported PAN-OS design.
