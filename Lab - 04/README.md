# Lab 04: Exploiting Cache Server Normalization for Web Cache Deception

![Difficulty](https://img.shields.io/badge/Difficulty-Practitioner-orange?style=for-the-badge) ![Status](https://img.shields.io/badge/Status-Solved-brightgreen?style=for-the-badge) ![Vulnerability](https://img.shields.io/badge/Vulnerability-Web%20Cache%20Deception-red?style=for-the-badge)

## 1. Lab Information

| Property | Details |
|---|---|
| Platform | PortSwigger Web Security Academy |
| Vulnerability | Web Cache Deception |
| Technique | Cache Server Normalization |
| Difficulty | Practitioner |
| Status | Solved |
| Target Endpoint | `/my-account` |
| Static Directory | `/resources` |
| Delimiter | `#` (`%23` encoded) |
| Cache Normalization | Encoded Path Traversal Resolution |
| Payload | `/my-account%23%2f%2e%2e%2fresources?leo` |

## 2. Lab Objective

Retrieve the API key belonging to the user `carlos` by exploiting differences in URL normalization and delimiter interpretation between the cache server and origin server.

## 3. Vulnerability Explanation

Web Cache Deception can occur when a cache server and an origin server interpret the same URL differently.

In this lab, the cache server performs URL normalization by decoding and resolving encoded path traversal sequences, whereas the origin server does not perform the same normalization.

However, the origin server recognizes a particular delimiter that the cache server does not.

By combining these two behaviors, an attacker can make:

- The **origin server** process the request as a sensitive dynamic endpoint.
- The **cache server** interpret the request as belonging to a cacheable static directory.

This allows sensitive account information to be stored in the cache.

### Understanding the Technique

The general payload structure is:

```text
/<dynamic-path>%2f%2e%2e%2f<static-directory-prefix>
```

The encoded characters represent:

| Encoding | Character | Meaning |
|---|---|---|
| `%2f` | `/` | Forward slash |
| `%2e` | `.` | Dot |
| `%23` | `#` | Hash delimiter |

The encoded path traversal sequence allows the cache to normalize the URL and resolve the path to a static directory.

However, path traversal alone is insufficient. A delimiter recognized by the origin server but not by the cache server is also required.

## 4. Identifying the Target Endpoint

Logged into the application using the provided credentials:

- Username: `wiener`
- Password: `peter`

Accessed the `/my-account` endpoint and identified that the response contained my personal account information, including the API key.

This endpoint became the target for the attack.

## 5. Understanding Cache Server Normalization

The objective was to identify how the cache server processes encoded path traversal sequences.

Unlike origin server normalization, this technique relies on the cache server resolving encoded dot-segments.

The important encoded sequence is:

```text
%2f%2e%2e%2f
```

After decoding, it represents:

```text
/../
```

The cache server resolves this sequence and moves to the parent directory.

By combining this with a static directory prefix, it becomes possible to make the cache interpret a request as belonging to a cacheable location.

## 6. Identifying the Delimiter Discrepancy

Path traversal alone does not guarantee a successful exploit.

The origin server must recognize a delimiter that the cache server does not.

The delimiter is used to make the origin server stop processing the URL path at the sensitive dynamic endpoint, while the cache continues processing and normalizing the remaining path.

In this lab, the successful delimiter was:

```text
#
```

Its URL-encoded representation is:

```text
%23
```

The encoded delimiter allowed the two components to interpret the same URL differently.

| Component | Behavior |
|---|---|
| Origin Server | Recognizes `#` as a delimiter and processes the dynamic endpoint. |
| Cache Server | Does not apply the same delimiter interpretation and continues normalization. |

This discrepancy made it possible to combine the sensitive endpoint with the static directory caching rule.

## 7. Constructing the Exploit

After identifying the required normalization behavior and delimiter, I combined them into a single payload.

### Successful Payload

```text
/my-account%23%2f%2e%2e%2fresources?leo
```

### Payload Breakdown

| Component | Purpose |
|---|---|
| `/my-account` | Sensitive dynamic endpoint |
| `%23` | Encoded hash delimiter |
| `%2f%2e%2e%2f` | Encoded path traversal sequence |
| `resources` | Static directory used by the cache |
| `?leo` | Cache-busting query parameter |

The payload combines two important behaviors:

1. The origin server recognizes the encoded delimiter and processes the request as the `/my-account` endpoint.
2. The cache server continues processing the encoded path traversal sequence and normalizes the request toward the `/resources` directory.

As a result, the origin returns sensitive account information while the cache treats the response as belonging to a static directory.

## 8. Exploiting the Victim's Session

After constructing the successful payload, I used the PortSwigger Exploit Server to make Carlos visit the malicious URL.

### Exploit Server Payload

```html
<script>
document.location="https://YOUR-LAB-ID.web-security-academy.net/my-account%23%2f%2e%2e%2fresources?leo"
</script>
```

### Attack Flow

1. Delivered the JavaScript through the Exploit Server.
2. Carlos visited the exploit page.
3. His browser navigated to the malicious URL.
4. The cache server processed the encoded path traversal sequence.
5. The origin server interpreted the delimiter and processed the request as `/my-account`.
6. The application returned Carlos's authenticated account information.
7. The cache stored the sensitive response under the normalized cache location.

## 9. Retrieving Carlos's API Key

After delivering the exploit to Carlos, I requested the same malicious URL.

The cache returned the previously stored response containing Carlos's account information.

I extracted his API key and submitted it through the lab's Submit Solution functionality.

**Result: Lab Solved.**

## 10. Key Takeaways

- **Cache Server Normalization:** A cache server can decode and resolve encoded path traversal sequences before applying caching rules.
- **Origin Server Normalization:** The origin server may process URL paths differently from the cache.
- **Delimiter Discrepancies:** Different interpretations of delimiter characters can create opportunities for Web Cache Deception.
- **URL Encoding:** `%23` represents an encoded hash character.
- **Path Traversal:** `%2f%2e%2e%2f` represents an encoded `/../` sequence.
- **Static Directory Rules:** Caches may use directory prefixes such as `/resources` to identify cacheable content.
- **Cache Busting:** A query parameter can help generate a fresh cache entry when it is included in the cache key.

## 11. Conclusion

This lab demonstrated how cache server normalization combined with delimiter discrepancies can introduce a Web Cache Deception vulnerability.

Unlike techniques that rely on the origin server performing path normalization, this approach exploits normalization performed by the cache server.

By combining an encoded delimiter with an encoded path traversal sequence, I successfully made the origin server return sensitive account information while the cache interpreted the request as belonging to a static directory.

This allowed me to retrieve Carlos's API key through the cached response.

**Key Lesson:** Differences in URL normalization and delimiter interpretation between caching infrastructure and origin servers can expose sensitive dynamic content.

## 12. References

- [PortSwigger – Web Cache Deception](https://portswigger.net/web-security/web-cache-deception)
- [PortSwigger – Web Cache Deception Delimiter List](https://portswigger.net/web-security/web-cache-deception/wcd-lab-delimiter-list)

---

**Disclaimer:** This repository is intended for educational purposes and authorized security testing only.
