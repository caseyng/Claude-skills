# Web Application Enrichment Checklist

Platform-specific concerns for web applications. Apply after `enrichment-universal.md`.

---

## Authentication and Session Management

**Authentication mechanism**
How do users prove their identity? Options: username/password, OAuth/SSO, magic link, passkeys.
Default: if users have accounts, use an established auth mechanism — do not build password
hashing and session management from scratch. Use a proven library or provider.

**Session handling**
How long do sessions last? What triggers logout? Are sessions invalidatable server-side?
Default: sessions expire after inactivity. Logout invalidates the server-side session,
not just the client cookie. Session tokens are not stored in localStorage.

**CSRF protection**
If the app uses cookie-based sessions, CSRF protection is required.
Default: SameSite=Strict or SameSite=Lax cookies, plus CSRF token for state-changing requests.

---

## Network Security

**HTTPS**
Default: HTTPS everywhere, HTTP Strict Transport Security (HSTS) header.
Redirect all HTTP to HTTPS.

**Content Security Policy**
Prevents XSS by declaring which sources the browser should trust.
Default: implement CSP. Start restrictive and loosen only what the application requires.

**CORS**
If the frontend and backend are on different origins, CORS must be configured.
Default: allow only the specific origins the app needs. Wildcard (`*`) is not acceptable
for authenticated endpoints.

---

## Input and Output

**Input validation and sanitisation**
Every field that accepts user input is a potential injection point (SQL, XSS, command injection).
Default: validate server-side regardless of client validation. Sanitise output when rendering
user-supplied content in HTML.

**File uploads**
If users upload files: validate file type, scan for malware, store outside the web root,
serve with correct Content-Type, enforce size limits.
Default: file uploads require explicit handling of all of the above.

---

## Browser and Compatibility

**Target browsers and minimum versions**
Which browsers and versions must the app support?
Default: modern evergreen browsers (Chrome, Firefox, Safari, Edge — current and prior major version).
Internet Explorer is not supported unless explicitly required.

**Mobile browser support**
Is the app used on mobile devices? Responsive design required?
Default: if any significant portion of users will access on mobile, responsive layout is required.

---

## Performance

**Page load and time-to-interactive**
Are there performance requirements that affect architecture (server-side rendering vs client-side)?
Default: for content-heavy or SEO-critical apps, server-side rendering. For dashboards
and apps where users are already authenticated, client-side rendering is acceptable.

**Caching strategy**
What can be cached and for how long? Static assets, API responses, user data.
Default: cache static assets aggressively (hashed filenames). Do not cache authenticated
or personalised responses without careful cache-key design.

---

## Deployment and Operations

**Hosting environment**
Where does the app run? Cloud provider, VPS, serverless, on-premise?
Default: state the deployment target early — it affects database choices, scaling, and cost.

**Environment configuration**
How are environment-specific values (API keys, database URLs) managed?
Default: environment variables, never committed to source control.
Separate config for development, staging, production.

**Logging and monitoring**
Where do server logs go? How are errors alerted?
Default: structured logging to a persistent store. Error alerting for production failures.

---

## SEO and Discoverability

**Is the app publicly indexed?**
If yes, server-side rendering or static generation is likely required for content to be indexed.
Default: if the app is a SaaS dashboard behind authentication, SEO is not a concern.
If it is a public-facing product, it is.
