# Web Cache Deception (WCD)

## 1. Introduction

Imagine logging into a website and opening your personal account page. It displays your name, email address, account details, and other private information.

Now imagine that the website accidentally saves a copy of your private account page in a shared storage system. Another person visits that same location and receives your personal information instead of their own.

This is the fundamental idea behind **Web Cache Deception (WCD)**.

Web Cache Deception is a web security vulnerability in which an attacker tricks a web caching system into storing sensitive information that should never have been publicly accessible.

The vulnerability occurs because the caching system and the actual web application interpret the same URL differently.

To understand how this happens, we first need to understand web caching.

---

## 2. What Is Web Caching?

When you visit a website, your browser requests information from a web server. This information might include:

- HTML pages
- Images
- CSS stylesheets
- JavaScript files
- Other website resources

Websites often receive thousands or even millions of requests. Processing every request directly through the main server can consume significant resources and increase loading times.

To solve this problem, websites use **caching**.

A web cache is an intermediate storage system that temporarily saves copies of responses so they can be delivered faster when requested again.

### A Simple Example

Imagine a website has an image called `logo.png`.

Without caching:

1. A visitor requests `logo.png`.
2. The request reaches the main server.
3. The server processes the request and returns the image.
4. Another visitor requests the same image.
5. The main server processes the request again.

With caching:

1. A visitor requests `logo.png`.
2. The request reaches the cache.
3. The cache doesn't have a copy, so it requests the image from the main server.
4. The cache stores a copy of the image.
5. Another visitor requests `logo.png`.
6. The cache already has the image and delivers it directly.

The second visitor receives the image without the main server having to generate another response.

This improves performance and reduces the workload on the origin server.

### Cache Miss and Cache Hit

Two important terms describe this process:

- **Cache Miss:** The requested content is not available in the cache, so the request is forwarded to the origin server.
- **Cache Hit:** The requested content is already stored in the cache, so it is delivered directly without requesting it from the origin server.

### Where Are Caches Used?

Caching is commonly implemented through:

- Content Delivery Networks (CDNs)
- Reverse proxies
- Web application infrastructure
- Browser caching systems

CDNs are particularly important because they store copies of website resources across geographically distributed servers. This allows users to receive content from a server closer to their location.

---

## 3. Understanding the Origin Server and Cache Server

To understand Web Cache Deception, it is important to distinguish between two components.

**Origin Server:**

The origin server is the actual backend server responsible for processing requests and generating responses.

For example, when you log into a website and request your account information, the origin server retrieves the appropriate information and returns it.

**Cache Server:**

The cache server sits between the user and the origin server. It stores selected responses and delivers them to users when appropriate.

Normally, these two components work together.

However, problems occur when they interpret the same request differently.

That difference is what makes Web Cache Deception possible.

---

## 4. What Is Web Cache Deception?

Web Cache Deception occurs when an attacker exploits differences in how a cache server and an origin server interpret a URL.

Consider a website with the following account endpoint:

```http
https://example.com/my-account
```

When an authenticated user visits this URL, the origin server returns their private account information.

Now imagine an attacker creates a modified URL:

```http
https://example.com/my-account/profile.css
```

Something interesting can happen here.

The origin server might interpret this request as a request to `/my-account` and return the user's account information, ignoring the additional `profile.css` portion.

However, the cache server might see the `.css` extension and assume that the requested resource is a static CSS file.

Since caches are commonly configured to store static resources, the cache may decide to save the response.

The problem is that the response is not actually a CSS file. It contains private account information.

The cache has been tricked into storing sensitive content.

### Why Is This Dangerous?

An attacker can persuade an authenticated victim to visit the specially crafted URL.

The victim's browser sends the request with their active session. The origin server returns the victim's private information, and the cache stores the response.

The attacker can then request the same URL and potentially retrieve the cached response without having the victim's authentication credentials.

This results in unauthorized disclosure of private information.

---

## 5. How Does a Web Cache Deception Attack Work?

The attack can be understood through five stages.

### Stage 1: Identify Sensitive Information

The attacker identifies a website endpoint that returns private information.

For example:

```http
https://example.com/my-account
```

This endpoint might return:

- Username
- Email address
- Account information
- Personal details

The endpoint normally requires authentication.

### Stage 2: Create a Modified URL

The attacker modifies the URL to make it appear like a static resource.

For example:

```http
https://example.com/my-account/profile.css
```

The intention is to make the cache and origin server interpret the URL differently.

### Stage 3: Trick the Victim's Browser

The attacker persuades an authenticated victim to visit the modified URL.

Because the victim is already logged in, their browser sends the request with the relevant session information.

The origin server processes the request and returns the victim's private account information.

### Stage 4: The Cache Stores the Response

The cache sees the `.css` extension and assumes that the response is a static resource.

If its caching rules permit it, the cache stores the response.

This is the critical mistake: sensitive dynamic content has been stored as though it were a public static resource.

### Stage 5: Retrieve the Cached Information

The attacker requests the same URL.

If the cache serves the previously stored response without requiring the victim's authentication session, the attacker can access the victim's private information.

**The attack succeeds because the cache stores a response that should have remained private.**

---

## 6. Why Does Web Cache Deception Happen?

The main reason is a difference in how the cache and origin server process URLs.

There are several ways these differences can occur.

### 6.1 Path Mapping Discrepancies

Path mapping determines how a server interprets a URL and identifies the resource being requested.

Different web servers and frameworks can process URL paths differently.

For example:

```http
https://example.com/user/123/profile/wcd.css
```

An origin server using REST-style routing might interpret this as:

```text
/user/123/profile
```

It may ignore the additional `wcd.css` portion and return the user's profile information.

Meanwhile, a cache might interpret the complete URL as a request for a CSS file.

If the cache stores responses ending in `.css`, it may cache the private profile information.

### 6.2 Delimiter Discrepancies

A delimiter is a character used to separate or distinguish different parts of a URL.

The cache and origin server might interpret certain delimiter characters differently.

For example, one component might treat a character as the end of a resource path, while another might continue processing the remaining characters.

These differences can create opportunities to make the origin server return sensitive information while the cache believes it is handling a static resource.

### 6.3 URL Normalization Discrepancies

URL normalization is the process of converting a URL into a standardized format.

Different components may normalize URL paths differently.

For example, a cache might simplify or decode certain URL characters differently from the origin server.

When their interpretations do not match, an attacker may be able to manipulate the URL to trigger unintended caching behavior.

---

## 7. Understanding Cache Keys and Cache Rules

Two important concepts help explain how caches decide what to store and retrieve.

### Cache Keys

A cache key is an identifier generated from parts of an HTTP request.

It helps the cache determine whether a requested resource already exists in storage.

A cache key commonly contains:

- URL path
- Query parameters
- Hostname
- Selected HTTP headers

If two requests produce the same cache key, the cache may consider them equivalent and serve the same stored response.

### Cache Rules

Cache rules determine which responses should be stored and how long they should remain in the cache.

Websites commonly configure caches to store static resources because they are frequently reused and generally do not contain user-specific information.

Some common caching rules include:

| Cache Rule | Example |
|---|---|
| Static file extension | `.css`, `.js`, `.png` |
| Static directory | `/static/`, `/assets/` |
| Specific filenames | `robots.txt`, `favicon.ico` |

The problem occurs when a dynamic endpoint accidentally matches one of these rules.

A cache may believe it is storing a harmless static resource when it is actually storing sensitive user information.

---

## 8. Web Cache Deception vs. Web Cache Poisoning

These two vulnerabilities are related because both involve web caching, but their objectives are different.

| Feature | Web Cache Deception | Web Cache Poisoning |
|---|---|---|
| Main objective | Access private information | Inject malicious content into cached responses |
| Main weakness | Incorrect application of cache rules | Improper cache key construction and handling of unkeyed inputs |
| Attacker's goal | Make the cache store sensitive content | Make the cache store an attacker-controlled response |
| Primary security impact | Confidentiality | Integrity |

In simple terms:

- **Web Cache Deception:** Tricks the cache into storing something private.
- **Web Cache Poisoning:** Tricks the cache into storing something malicious.

---

## 9. Security Impact

A successful Web Cache Deception attack can expose sensitive information belonging to authenticated users.

Potential consequences include:

- **Personal Information Disclosure:** Names, email addresses, and other private information may be exposed.
- **Account Information Exposure:** Private account details may become accessible to unauthorized users.
- **Sensitive Data Leakage:** API responses and other confidential information may be disclosed.
- **Privacy Violations:** Information belonging to one user may be delivered to another user.
- **Account Security Risks:** If sensitive authentication-related information is exposed, it may create further security risks.

The actual impact depends on what information the vulnerable endpoint returns.

---

## 10. Detecting Web Cache Deception

Security testers can investigate caching behavior by examining HTTP requests, responses, and response headers.

Some useful HTTP response headers include:

| Header | Purpose |
|---|---|
| `Cache-Control` | Specifies caching instructions. |
| `Age` | Indicates how long a response has been stored in a shared cache. |
| `X-Cache` | Often indicates whether a response came from a cache. |
| `Vary` | Specifies which request headers influence cache selection. |
| `Expires` | Specifies when a response becomes stale. |
| `ETag` | Identifies a particular version of a resource. |

### Understanding X-Cache

Some caching systems use the `X-Cache` header to indicate cache behavior.

Common values include:

- `X-Cache: hit` — The response was served from the cache.
- `X-Cache: miss` — No matching cached response was found.
- `X-Cache: dynamic` — The response was generated dynamically and generally isn't suitable for caching.
- `X-Cache: refresh` — The cached response required refreshing or revalidation.

Not every caching system uses these headers, so their absence does not mean caching is not occurring.

Response time differences can also provide clues. A cached response may be delivered faster than a response generated by the origin server.

---

## 11. Prevention and Mitigation

Web Cache Deception can be prevented by ensuring that sensitive responses are never accidentally stored or shared.

### 1. Use Appropriate Cache-Control Headers

Sensitive and user-specific responses should have appropriate caching restrictions.

For example:

```http
Cache-Control: no-store
```

This instructs caches not to store the response.

### 2. Configure Cache Rules Carefully

Do not rely exclusively on file extensions to decide whether a response is safe to cache.

A URL ending in `.css` does not necessarily contain a CSS file.

### 3. Maintain Consistent URL Interpretation

The CDN, reverse proxy, web server, and backend application should interpret URL paths consistently.

### 4. Separate Static and Dynamic Resources

Static resources and dynamic application endpoints should have clearly defined routing and caching policies.

### 5. Prevent Cross-User Data Exposure

User-specific responses should never be shared between different authenticated users.

### 6. Regularly Test Caching Behavior

Developers and security teams should verify that authenticated responses cannot accidentally enter shared caches.

---

## 12. Conclusion

Web Cache Deception demonstrates how a seemingly small difference in URL interpretation can create a serious security vulnerability.

Web caching is designed to improve performance, but incorrect caching rules and inconsistent URL processing can expose information that should remain private.

The important lesson is that a cache should not decide whether content is safe to store based solely on how a URL looks. The actual nature and sensitivity of the response must also be considered.

Understanding how the cache and origin server communicate, interpret requests, and handle responses is essential to identifying and preventing this vulnerability.

---

## References

- PortSwigger Web Security Academy — Web Cache Deception
- PortSwigger Research — Gotta Cache 'em All: Bending the Rules of Web Cache Exploitation
- RFC 9111 — HTTP Caching

---

**Disclaimer:** This repository is intended for educational purposes and authorized security testing only. All practical security testing should be performed in controlled environments or on systems where explicit permission has been granted.
