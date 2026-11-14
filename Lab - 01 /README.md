# Lab 01: Exploiting Path Mapping for Web Cache Deception

**Difficulty:** Apprentice  
**Status:** Solved  
**Platform:** PortSwigger Web Security Academy  
**Vulnerability:** Web Cache Deception (WCD)

## 1. Lab Objective

The objective of this lab was to exploit a path mapping discrepancy between the origin server and the web cache to retrieve the API key belonging to the user `carlos`.

Credentials provided:

- Username: `wiener`
- Password: `peter`

## 2. Understanding the Vulnerability

A web application has two components involved in processing requests:

- **Origin Server:** Processes requests and returns application data, including sensitive information.
- **Cache Server:** Stores responses that match its caching rules and serves them to subsequent requests.

In this lab, the origin server and cache server interpret URL paths differently.

The origin server identifies the actual application endpoint, whereas the cache identifies resources based on URL patterns, particularly static file extensions.

This difference allows us to make the origin server return sensitive information while making the cache believe the response is a static resource.

## 3. Vulnerability Identification

After logging in as `wiener`, I accessed the `/my-account` endpoint.

The response contained my personal account information, including an API key.

I sent the request to Burp Repeater and started modifying the URL.

### Step 1: Test Path Mapping

Original request:

```http
GET /my-account HTTP/2
```

Modified request:

```http
GET /my-account/abc HTTP/2
```

Even after appending an arbitrary path segment, the origin server continued returning the account information.

**Observation:** The origin server was mapping both URLs to the same application endpoint.

This indicated a path mapping discrepancy that could potentially be exploited.

### Step 2: Identify Cache Behavior

I appended a static file extension to the URL:

```http
GET /my-account/abc.js HTTP/2
```

The response contained:

```http
X-Cache: miss
Cache-Control: max-age=30
```

The observations were:

- `X-Cache: miss` indicated that the response was not retrieved from an existing cache entry.
- `Cache-Control: max-age=30` indicated that the response could remain fresh for 30 seconds.

I sent the same request again within that period.

This time, the response contained:

```http
X-Cache: hit
```

This confirmed that the response was being served from the cache.

**Conclusion:** The origin server interpreted the URL as `/my-account`, while the cache treated it as a static JavaScript resource because of the `.js` extension.

The sensitive account response was successfully cached.

## 4. Exploitation

The next step was to make the victim, `carlos`, visit a URL that triggered the vulnerable behavior.

I used the PortSwigger Exploit Server to deliver the following JavaScript:

```javascript
<script>
    document.location="https://YOUR-LAB-ID.web-security-academy.net/my-account/wcd.js"
</script>
```

### Why document.location?

`document.location` redirects the victim's browser to the specified URL.

Since Carlos is authenticated, his browser sends the request with his active session.

The origin server returns Carlos's account information, and the cache stores the response because the URL ends in `.js`.

I used a different path segment (`wcd`) to ensure that the victim's request generated a fresh cache entry instead of reusing my previously cached response.

### Attack Flow

1. I delivered the JavaScript through the Exploit Server.
2. Carlos visited the exploit page.
3. His browser navigated to `/my-account/wcd.js`.
4. The origin server returned his private account information.
5. The cache stored the response.
6. I requested the same URL.
7. The cache returned the previously stored response.
8. I retrieved Carlos's API key.

The response contained the API key required to complete the lab.

## 5. Key Observations

| Observation | What it revealed |
|---|---|
| `/my-account/abc` returned account information | Origin server ignored the additional path segment. |
| `.js` extension returned account information | Origin server treated the URL as the account endpoint. |
| `X-Cache: miss` | No existing cached response was available. |
| `Cache-Control: max-age=30` | Response had a 30-second freshness lifetime. |
| `X-Cache: hit` | Response was successfully retrieved from the cache. |

## 6. What I Learned

- Web Cache Deception can occur when the cache and origin server interpret URL paths differently.
- The origin server can return sensitive dynamic content even when the URL appears to reference a static resource.
- Static file extension-based caching rules can introduce security risks when configured incorrectly.
- `X-Cache` and `Cache-Control` headers are useful when investigating caching behavior.
- An attacker can exploit a victim's authenticated session to populate the cache with private information.
- A cache hit can allow another user to retrieve information that originally belonged to an authenticated victim.

## 7. Mitigation

- Prevent sensitive responses from being stored in shared caches.
- Apply appropriate `Cache-Control: no-store` directives to sensitive responses.
- Ensure consistent URL interpretation between the cache and origin server.
- Avoid relying exclusively on file extensions when deciding whether a response is cacheable.
- Separate static resources from dynamic application endpoints.

## 8. References

- [PortSwigger — Web Cache Deception](https://portswigger.net/web-security/web-cache-deception)
- [PortSwigger — Exploiting Path Mapping for Web Cache Deception](https://portswigger.net/web-security/web-cache-deception/lab-wcd-exploiting-path-mapping)

---

**Final Takeaway:** The vulnerability was not caused by the origin server exposing the API key directly to an unauthenticated attacker. It was caused by the cache incorrectly storing an authenticated user's sensitive response and subsequently serving it to another user.
