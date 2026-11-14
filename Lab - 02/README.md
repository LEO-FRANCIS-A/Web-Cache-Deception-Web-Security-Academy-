# Exploiting Path Delimiters for Web Cache Deception

**Difficulty:** Practitioner  
**Vulnerability:** Web Cache Deception (WCD)  
**Status:** Solved

## 1. Lab Description

This lab demonstrates a **Web Cache Deception vulnerability caused by inconsistent path delimiter interpretation between the cache and the origin server**.

The application contains an authenticated endpoint at `/my-account` that returns sensitive account information, including the user's API key.

The cache uses static-file extension rules to determine which responses should be cached. However, the origin server and the cache interpret certain path characters differently.

In this lab, the semicolon (`;`) is treated as a path delimiter by the origin server but not by the cache.

An attacker can exploit this discrepancy by appending a semicolon and a static file extension to the account endpoint.

The origin server processes the request as `/my-account` and returns the account information, while the cache interprets the URL as a static resource ending in `.js` and stores the response.

When the victim visits the crafted URL, their sensitive account information is stored in the cache and can subsequently be retrieved by the attacker.

## 2. Understanding the Vulnerability

### What is a path delimiter?

A path delimiter is a character that a server or application uses to separate or interpret parts of a URL path.

Different components of a web application may interpret the same character differently.

For example:

```text
/my-account;leo.js
```

The origin server may interpret this as:

```text
/my-account
```

Meanwhile, the cache may interpret the complete path as:

```text
/my-account;leo.js
```

This difference in interpretation creates an opportunity for web cache deception.

### The delimiter discrepancy

The lab involves two important components:

| Component | Interpretation |
|---|---|
| Origin server | Treats `;` as a path delimiter and processes the request as `/my-account`. |
| Cache | Does not treat `;` as a path delimiter and sees a URL ending in `.js`. |

The cache has a rule that allows it to cache responses for static file extensions such as `.js`.

Because the cache does not recognize the semicolon as a delimiter, it incorrectly considers the sensitive account response eligible for caching.

## 3. Exploitation Methodology

### Step 1: Identify a target endpoint

1. Open the lab in Burp Suite's browser.
2. Log in using the provided credentials:

   ```text
   Username: wiener
   Password: peter
   ```

3. Navigate to the account page.
4. Observe that the response contains the authenticated user's API key.
5. Identify `/my-account` as the sensitive endpoint.

### Step 2: Identify path delimiters used by the origin server

1. Open **Proxy → HTTP history**.
2. Find the `GET /my-account` request.
3. Send the request to Burp Repeater.

First, append an arbitrary path segment:

```http
GET /my-account/abc HTTP/2
```

The server responds with `404 Not Found`.

This indicates that the origin server does not simply ignore arbitrary additional path segments.

Next, append an arbitrary string directly to the endpoint:

```http
GET /my-accountabc HTTP/2
```

This also returns `404 Not Found`.

Use this response as a reference when testing possible delimiters.

#### Test possible delimiters using Intruder

1. Send the `/my-accountabc` request to Burp Intruder.
2. Select the **Sniper attack** type.
3. Set the payload position as follows:

   ```http
   GET /my-account§§abc HTTP/2
   ```

4. Add the possible delimiter characters provided by PortSwigger.
5. Disable URL encoding for the payload characters.
6. Start the attack.
7. Sort the results by HTTP status code.

The results show that:

- `;` returns `200 OK`.
- `?` returns `200 OK`.
- Other tested characters return `404 Not Found`.

This indicates that the origin server recognizes `;` and `?` as path delimiters.

### Step 3: Investigate delimiter discrepancies

Now determine whether the cache interprets these delimiters in the same way as the origin server.

Test the question mark delimiter first:

```http
GET /my-account?abc.js HTTP/2
```

The response does not show evidence of caching.

This suggests that the cache also recognizes `?` as a delimiter.

Next, test the semicolon delimiter:

```http
GET /my-account;abc.js HTTP/2
```

The first response contains:

```http
X-Cache: miss
```

Resend the same request.

The response now contains:

```http
X-Cache: hit
```

This confirms that the cache is storing the response.

The important observation is that the origin server recognizes `;` as a delimiter, but the cache does not.

The `.js` extension triggers the cache's static-file caching rule even though the origin server processes the request as the sensitive `/my-account` endpoint.

### Step 4: Craft the exploit

Open the **Exploit Server**.

In the Body section, submit the following HTML:

```html
<script>
document.location="https://YOUR-LAB-ID.web-security-academy.net/my-account;leo.js"
</script>
```

Replace `YOUR-LAB-ID` with your actual lab domain.

The URL uses:

- `/my-account` — the sensitive account endpoint.
- `;` — the delimiter recognized by the origin server.
- `leo` — an arbitrary string that helps create a unique cache key.
- `.js` — the static file extension that triggers the cache rule.

Click **Deliver exploit to victim**.

The victim, Carlos, visits the crafted URL while authenticated. The origin server returns Carlos's account information, and the cache stores the response.

### Step 5: Retrieve the cached response

After delivering the exploit, navigate to:

```text
https://YOUR-LAB-ID.web-security-academy.net/my-account;leo.js
```

The cached response contains Carlos's account information, including his API key.

Copy the API key and submit it using the lab's **Submit solution** button.

The lab is solved.

## 4. Payload Breakdown

### Exploit Server payload

```html
<script>
document.location="https://YOUR-LAB-ID.web-security-academy.net/my-account;leo.js"
</script>
```

| Component | Explanation |
|---|---|
| `document.location` | Redirects the victim's browser to the crafted URL. |
| `/my-account` | The authenticated endpoint containing sensitive account information. |
| `;` | Path delimiter recognized by the origin server but not by the cache. |
| `leo` | Arbitrary path string used to create a unique cache key. |
| `.js` | Static file extension that matches the cache's caching rule. |

### Why use a unique string?

The arbitrary string helps create a unique cache key.

For example:

```text
/my-account;abc.js
/my-account;leo.js
```

These are different URLs from the cache's perspective.

Using a unique value avoids receiving an existing cached response and allows Carlos's account information to be stored under the crafted URL.

## 5. Root Cause

The root cause is inconsistent URL path interpretation between the cache and the origin server.

The origin server treats the semicolon (`;`) as a delimiter and processes the request as `/my-account`.

The cache does not recognize the semicolon as a delimiter. Instead, it evaluates the complete URL and identifies the `.js` extension.

As a result, the cache applies its static-file caching rule to a response containing sensitive, user-specific account information.

This allows private account data to be stored in a shared cache.

## 6. Remediation

### 1. Ensure consistent URL normalization

The cache and origin server should interpret URL paths consistently.

Normalize and canonicalize paths before applying caching rules.

### 2. Configure cache rules carefully

Do not rely exclusively on file extensions to determine whether a response is safe to cache.

A URL ending in `.js` does not necessarily represent a static JavaScript resource.

### 3. Prevent caching of sensitive responses

Authenticated endpoints containing private information should return appropriate cache-control directives, such as:

```http
Cache-Control: no-store
```

### 4. Separate static and dynamic resources

Use explicit routing and caching policies to distinguish static resources from dynamic, authenticated endpoints.

## 7. Key Takeaways

- Web Cache Deception can occur when the cache and origin server interpret URL paths differently.
- Path delimiters can create discrepancies between these components.
- The semicolon (`;`) was recognized by the origin server but not by the cache.
- The `.js` extension triggered the cache's static-file caching rule.
- The origin server returned sensitive account information while the cache stored it as a static resource.
- A unique path string helps create a separate cache key.
- Sensitive authenticated responses should not be stored in shared caches.

## 8. References

- [PortSwigger – Web Cache Deception](https://portswigger.net/web-security/web-cache-deception)
- [PortSwigger – Web Cache Deception Lab: Exploiting Path Delimiters](https://portswigger.net/web-security/web-cache-deception/lab-wcd-exploiting-path-delimiters)
- [PortSwigger – Web Cache Deception Lab Delimiter List](https://portswigger.net/web-security/web-cache-deception/wcd-lab-delimiter-list)
