# Lab 05: Exploiting Exact-Match Cache Rules for Web Cache Deception

![Difficulty](https://img.shields.io/badge/Difficulty-Expert-orange?style=for-the-badge) ![Status](https://img.shields.io/badge/Status-Solved-brightgreen?style=for-the-badge) ![Vulnerability](https://img.shields.io/badge/Vulnerability-Web%20Cache%20Deception-red?style=for-the-badge)

## 1. Lab Information

| Property | Details |
|---|---|
| Platform | PortSwigger Web Security Academy |
| Vulnerability | Web Cache Deception |
| Technique | Exact-Match Cache Rules |
| Difficulty | Expert |
| Status | Solved |
| Target | Administrator Account |
| Target Endpoint | `/my-account` |
| Cache Rule | Exact Filename Matching |
| Cached Filename | `robots.txt` |
| Delimiter | `;` (Semicolon) |
| Cache Normalization | Encoded Path Traversal Resolution |
| Secondary Vulnerability | Cross-Site Request Forgery (CSRF) |
| Primary Payload | `/my-account;%2f%2e%2e%2frobots.txt?leoo` |
| Final Objective | Change Administrator Email Address |

## 2. Lab Objective

The objective was to exploit an exact-match cache rule to retrieve the administrator's CSRF token and subsequently perform a CSRF attack to change the administrator's email address.

Unlike previous Web Cache Deception experiments, this lab required more than retrieving sensitive information.

It involved chaining two vulnerabilities:

1. Web Cache Deception to obtain the administrator's CSRF token.
2. Cross-Site Request Forgery to perform an unauthorized email-change operation.

## 3. Understanding Exact-Match Cache Rules

Web caches commonly store frequently requested files such as:

- `robots.txt`
- `index.html`
- `favicon.ico`

These files generally change infrequently, making them suitable for caching.

Some caching systems use exact filename matching to determine whether a response should be cached.

For example:

```text
/robots.txt
```

The cache may recognize this exact filename and store its response.

Unlike static directory rules, which match an entire directory prefix, exact-match rules specifically target particular filenames.

### How This Becomes Vulnerable

Imagine an application containing a sensitive endpoint:

```text
/my-account
```

The origin server returns account information, including a CSRF token.

If the cache can be tricked into interpreting a manipulated URL as `/robots.txt`, it may store the sensitive account response under an exact-match cache rule.

This creates an opportunity to retrieve another user's private information.

## 4. Understanding the Normalization Discrepancy

This lab exploits a situation where:

- The cache server resolves encoded path traversal sequences.
- The origin server does not resolve the same sequence.
- The origin server recognizes a delimiter that the cache server does not.

The successful payload was:

```text
/my-account;%2f%2e%2e%2frobots.txt?leoo
```

### Payload Breakdown

| Component | Purpose |
|---|---|
| `/my-account` | Sensitive account endpoint |
| `;` | Delimiter recognized by the origin server |
| `%2f` | Encoded forward slash |
| `%2e%2e` | Encoded `..` path traversal sequence |
| `%2f` | Encoded forward slash |
| `robots.txt` | Exact filename targeted by the cache rule |
| `?leoo` | Cache-busting query parameter |

### Origin Server Interpretation

The origin server recognizes the semicolon as a delimiter and processes the request as:

```text
/my-account
```

Consequently, it returns the account page containing the authenticated user's information and CSRF token.

### Cache Server Interpretation

The cache server does not interpret the semicolon as the same delimiter.

It decodes and resolves the encoded path traversal sequence:

```text
/my-account;%2f%2e%2e%2frobots.txt
```

After decoding:

```text
/my-account;/../robots.txt
```

After path normalization:

```text
/robots.txt
```

The cache therefore identifies the request as matching its exact-match `robots.txt` caching rule.

### The Discrepancy

| Component | Interpretation |
|---|---|
| Origin Server | `/my-account` |
| Cache Server | `/robots.txt` |

The origin returns sensitive account information, while the cache treats the response as a request for a commonly cached file.

This allows the administrator's account response to be stored in the cache.

## 5. Retrieving the Administrator's CSRF Token

The first objective was to obtain the CSRF token belonging to the administrator.

I used the following payload:

```text
/my-account;%2f%2e%2e%2frobots.txt?leoo
```

The payload was used to make the administrator's authenticated browser request the manipulated URL.

### Attack Process

1. The administrator visits the malicious URL.
2. The origin server recognizes `;` as a delimiter.
3. The origin processes the request as `/my-account`.
4. The administrator's account page is returned, including the CSRF token.
5. The cache resolves the encoded path traversal sequence.
6. The cache identifies the normalized path as `/robots.txt`.
7. The sensitive account response is stored under the exact-match cache rule.
8. I request the same URL and retrieve the cached administrator response.

From the retrieved response, I extracted the administrator's CSRF token.

The `?leoo` parameter was used to help generate a fresh cache entry.

## 6. Understanding Cross-Site Request Forgery (CSRF)

After obtaining the administrator's CSRF token, the next step was to understand how to use it.

CSRF is a vulnerability that allows an attacker to trick an authenticated user's browser into performing an unwanted action on a website.

This is possible because browsers automatically include applicable authentication cookies when sending requests to the corresponding website.

For example, an authenticated administrator might normally change their email address through a form.

The application expects:

- An authenticated session.
- A valid CSRF token.
- The requested email address.

If an attacker can construct a request containing the required information and cause the administrator's browser to submit it, an unauthorized action may be performed.

In this lab, the administrator's CSRF token was obtained through the Web Cache Deception vulnerability.

This made it possible to construct a valid email-change request.

## 7. Constructing the CSRF Exploit

After retrieving the administrator's CSRF token, I created an HTML form that submitted a POST request to the application's email-change endpoint.

### Final CSRF Payload

```html
<form action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email" method="POST">
    <input type="hidden" name="email" value="attacker@example.com">
    <input type="hidden" name="csrf" value="ADMINISTRATOR-CSRF-TOKEN">
</form>

<script>
    document.forms[0].submit();
</script>
```

### Payload Explanation

| Component | Purpose |
|---|---|
| `action` | Specifies the email-change endpoint. |
| `method="POST"` | Submits the request using HTTP POST. |
| `email` | Specifies the new email address. |
| `csrf` | Contains the administrator's retrieved CSRF token. |
| `document.forms[0].submit()` | Automatically submits the form. |

The form automatically submits when the administrator visits the exploit page.

Because the request targets the vulnerable application's email-change endpoint, the browser can include the administrator's applicable session cookies.

The valid CSRF token allows the application to accept the request.

## 8. Executing the Attack

I used the PortSwigger Exploit Server to host and deliver the CSRF payload.

The attack involved the following sequence:

1. Retrieved the administrator's CSRF token using the Web Cache Deception payload.
2. Created an HTML form containing the new email address and retrieved CSRF token.
3. Configured the form to submit a POST request to `/my-account/change-email`.
4. Added JavaScript to automatically submit the form.
5. Delivered the exploit to the administrator.
6. The administrator's browser submitted the email-change request.
7. The application processed the request and updated the administrator's email address.

**Result: Lab Solved.**

## 9. Key Takeaways

- **Exact-Match Cache Rules:** Caches can store responses based on specific filenames such as `robots.txt`.
- **Delimiter Discrepancies:** The cache and origin server can interpret delimiter characters differently.
- **Cache Server Normalization:** Encoded path traversal sequences can be decoded and resolved by the cache.
- **URL Encoding:** `%2f` represents `/`, while `%2e` represents `.`.
- **Cache Keys:** Query parameters can influence cache entry identification.
- **CSRF:** An authenticated user's browser can be tricked into submitting an unauthorized request.
- **Attack Chaining:** Web Cache Deception can expose security tokens that enable further attacks.
- **Security Impact:** A caching vulnerability can contribute to unauthorized account modifications when combined with another weakness.

## 10. Conclusion

This lab demonstrated how exact-match cache rules can be exploited through URL normalization and delimiter discrepancies.

By combining the origin server's semicolon delimiter handling with the cache server's encoded path normalization, I made the cache interpret a sensitive account request as a request for `robots.txt`.

This allowed me to retrieve the administrator's CSRF token from a cached response.

I then used the token to construct a CSRF attack that submitted an email-change request through the administrator's authenticated browser.

The experiment demonstrated how multiple web security vulnerabilities can be chained together to achieve a more significant security impact.

**Key Lesson:** A vulnerability that exposes sensitive information through caching can become much more dangerous when the exposed information enables further attacks.

## 11. References

- [PortSwigger – Web Cache Deception](https://portswigger.net/web-security/web-cache-deception)
- [PortSwigger – CSRF](https://portswigger.net/web-security/csrf)
- [PortSwigger – Web Cache Deception Delimiter List](https://portswigger.net/web-security/web-cache-deception/wcd-lab-delimiter-list)

---

**Disclaimer:** This repository is intended for educational purposes and authorized security testing only.
