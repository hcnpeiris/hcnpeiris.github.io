# Angular SSRF Explained

Angular SSR applications have had multiple reported SSRF vulnerabilities, including CVE-2025-62427 and CVE-2026-27739. This article explains how these vulnerabilities work and how Angular fixed them.

The issue occurs in Angular SSR's request URL construction logic. The `createRequestUrl` function builds an absolute URL by combining the request protocol, hostname, and request path, then converts the resulting string into a JavaScript URL object.

When Angular processes a relative request URL, the URL object uses the host from the newly created URL as the base origin. Under normal conditions, this ensures that relative paths are requested from the application's own host. However, if an attacker can manipulate the URL construction process and influence the host value of the generated URL object, a relative request can be resolved against an attacker-controlled host instead. This causes Angular SSR to send server-side requests to an unintended destination, resulting in server-side request forgery (SSRF).

## CVE-2025-62427

The issue originates from how Angular constructs the request URL:

```js
return new URL(originalUrl ?? url, `${protocol}://${hostnameWithPort}`);
```

According to the [Node.js documentation](https://nodejs.org/api/url.html#new-urlinput-base):

`new URL(input, base)`

- If `input` is relative, it is resolved using the `base`.
- If `input` is absolute, the `base` is ignored.

For a normal relative path:

```
node -e "const u = new URL('/some-page', 'http://localhost:4200'); console.log(u.href)"
```

Result:

```
http://localhost:4200/some-page
```

The host remains `localhost:4200` because the path is resolved against the base.

But there is a confusing case when the input is schema-relative. Although the input is considered relative, the base URL is not fully used. Only the scheme from the base URL is inherited, while the host is taken from the input.

```
node -e "const u = new URL('//some-page', 'http://localhost:4200'); console.log(u.href)"
```

Result:

```
http://some-page
```

Here, the host from the input (`attacker-domain.com`) overrides the base host.

In Angular, `originalUrl ?? url` is attacker-controlled. If a request is made like:

```
http://localhost:4200//attacker-domain.com/some-page
```

then:

- `input` = `//attacker-domain.com/some-page`
- `base` = `http://localhost:4200`

The resulting URL becomes:

```
http://attacker-domain.com/some-page
```

Angular now treats `attacker-domain.com` as the origin. Any subsequent relative request, such as `/api/data`, is resolved against this host:

```
http://attacker-domain.com/api/data
```

This results in server-side request forgery (SSRF).
