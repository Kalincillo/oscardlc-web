# Migration Context — Flask Portfolio → ASP.NET Core

> **How to use this file:** drop it at the root of the new C# repo. If you're working with the
> Claude extension in VS Code, also create a `CLAUDE.md` whose first line is
> `Read MIGRATION-CONTEXT.md before doing anything.` This file is the source of truth for
> architecture, naming, URL preservation and deployment decisions.

---

## 1. Goal

Port an existing Python/Flask personal portfolio (`www.oscardlc.com`, currently hosted on
PythonAnywhere) to C# / ASP.NET Core, with **identical public URLs**, **identical visual
output**, and **cheaper or equal hosting cost**.

This is a *port*, not a redesign. Do not restyle, rename routes, or "improve" the UX unless
explicitly asked. Any change to a public URL is a regression.

---

## 2. Current system (verified from source)

### 2.1 Stack

| Concern | Current |
|---|---|
| Framework | Flask 2.2.2 |
| Templating | Jinja2 3.1.2 |
| Forms | Flask-WTF 1.0.1 + WTForms 3.0.1 + email-validator |
| Config | python-dotenv, `.env` file (git-ignored) |
| ORM | Flask-SQLAlchemy 2.5.1 + SQLAlchemy 1.4.40 — **installed but never imported anywhere in the code** |
| Database | None. No models, no migrations, no connection string. |
| Host | PythonAnywhere, path `/home/kalincillo/my_blog/`, WSGI file `/var/www/kalincillo_pythonanywhere_com_wsgi.py` |
| Runtime | Python 3.10 |
| Storage in use | ~312 MB of a 1 GB quota (≈31%) — almost certainly the `static/` folder (images / media) |

**Key conclusion: this is a static-content site.** No database, no auth, no user state. The only
dynamic piece is a contact form, and that form is currently commented out.

### 2.2 File layout (inferred)

```
my_blog/
├── flasky.py               # entry point: create_app('default'), app.run()
├── config.py               # Config / DevConfig / ProdConfig + config dict
├── requirements.txt
├── .env                    # git-ignored, holds SECRET_KEY
├── .gitignore              # ignores .env, .idea, __pycache__, .pdf
└── app/
    ├── __init__.py         # create_app factory, registers `main` blueprint
    └── main/
        ├── __init__.py     # Blueprint('main', __name__); imports views, errors, forms
        ├── views.py        # all routes
        ├── errors.py       # 404 / 500 app_errorhandler
        ├── forms.py        # ContactForm
        ├── templates/      # ← NOT YET PROVIDED
        └── static/         # ← NOT YET PROVIDED
```

### 2.3 Routes to preserve — exact strings, non-negotiable

All are `GET`-only and all just render a template with no context variables.

| URL | Flask endpoint | Template |
|---|---|---|
| `/` | `main.index` | `index.html` |
| `/about` | `main.about` | `about.html` |
| `/music` | `main.music` | `music.html` |
| `/projects` | `main.projects` | `projects.html` |
| `/hardware` | `main.hardware` | `hardware.html` |
| `/software` | `main.software` | `software.html` |
| `/cane` | `main.cane` | `cane.html` |
| `/wordcloud` | `main.wordcloud` | `wordcloud.html` |
| `/music_and_death` | `main.music_and_death` | `music_and_death.html` |
| `/music_and_death_power_bi` | `main.music_and_death_power_bi` | `music_and_death_power_bi.html` |
| `/pedal` | `main.pedal` | `pedal.html` |
| `/rudi` | `main.rudi` | `rudi.html` |
| `/burger` | `main.burger` | `burger.html` |
| `/strength` | `main.strength` | `strength.html` |
| `/contact` | `main.contact` — **commented out**, GET + POST | `contact.html` |

Error handlers: `404 → 404.html`, `500 → 500.html`.

⚠️ Several URLs use **snake_case with underscores** (`/music_and_death_power_bi`). Razor Pages
derives routes from file names, and a file named `MusicAndDeathPowerBi.cshtml` would produce
`/musicandmeathpowerbi` — wrong. **Every page must declare an explicit route template** (see §4.3).

### 2.4 The contact form (currently disabled)

`ContactForm` (WTForms):

| Field | Type | Validators |
|---|---|---|
| `name` | StringField | required |
| `email` | StringField | valid email ("Not a valid email address."), required |
| `body` | StringField | required, min length 4 ("Your message is too short.") |
| `submit` | SubmitField | — |

The view validated on POST and redirected to `main.index` on success — **it never actually sent
anything**. So the C# version must *implement* delivery, not port it. See §7.

### 2.5 Config quirks worth knowing

- `DevConfig` and `ProdConfig` **do not inherit from `Config`**. `DevConfig` therefore has no
  `SECRET_KEY` and no folder settings at all. This is a latent bug.
- `flasky.py` hardcodes `create_app('default')`, so `DevConfig`/`ProdConfig` are dead code — the
  app always runs on base `Config` with `DEBUG` unset (i.e. off). Fine in production, but it means
  the config system was never really exercised. **Don't faithfully reproduce this structure.**
- `SECRET_KEY` is the only real secret. In Flask it protected WTForms CSRF tokens and the session.

---

## 3. Target stack — decisions already made

| Decision | Choice | Why |
|---|---|---|
| Framework | **ASP.NET Core 10 (LTS)** | Current LTS; supported by the target host. |
| UI model | **Razor Pages** | 1 page = 1 route = 1 file, mirroring the Flask view/template pairing. MVC adds a controller layer with no benefit for content pages. |
| Not Blazor Server | — | Needs a persistent WebSocket circuit; wasteful and fragile for static content. |
| Not Blazor WASM | — | Ships a multi-MB runtime and hurts SEO for a portfolio. |
| Not a Web API + SPA | — | Nothing here is an API. |
| Database | **None** | Do not add EF Core, do not add SQL Server. There is nothing to persist. |
| CSS/JS build | **None initially** | Reuse the existing hand-written CSS/JS as-is. No npm, no bundler, unless the current site already uses one. |
| Auth | **None** | |

### Target project layout

```
OscarDlc/                     (repo root)
├── MIGRATION-CONTEXT.md
├── CLAUDE.md
├── .gitignore
├── OscarDlc.slnx
└── OscarDlc.Web/
    ├── OscarDlc.Web.csproj
    ├── Program.cs
    ├── appsettings.json
    ├── appsettings.Development.json
    ├── Pages/
    │   ├── _ViewImports.cshtml
    │   ├── _ViewStart.cshtml
    │   ├── Shared/
    │   │   ├── _Layout.cshtml        ← from templates/base.html
    │   │   ├── _Nav.cshtml
    │   │   └── _Footer.cshtml
    │   ├── Index.cshtml
    │   ├── About.cshtml
    │   ├── Music.cshtml
    │   ├── Projects.cshtml
    │   ├── Hardware.cshtml
    │   ├── Software.cshtml
    │   ├── Contact.cshtml (+ .cshtml.cs)
    │   ├── Cane.cshtml
    │   ├── WordCloud.cshtml
    │   ├── MusicAndDeath.cshtml
    │   ├── MusicAndDeathPowerBi.cshtml
    │   ├── Pedal.cshtml
    │   ├── Rudi.cshtml
    │   ├── Burger.cshtml
    │   ├── Strength.cshtml
    │   └── Error.cshtml (+ .cshtml.cs)
    └── wwwroot/                       ← from app/main/static/
        ├── css/
        ├── js/
        ├── img/
        └── files/
```

**Pages with no logic get no `.cshtml.cs` file.** Only `Contact` and `Error` need a PageModel.

---

## 4. Translation reference

### 4.1 Concept mapping

| Flask / Jinja2 | ASP.NET Core / Razor |
|---|---|
| `create_app()` factory | `Program.cs` with `WebApplicationBuilder` |
| Blueprint | Not needed — one small app. (Areas exist but are overkill.) |
| `@main.route('/x')` + `render_template` | `Pages/X.cshtml` with `@page "/x"` |
| `templates/` | `Pages/` |
| `static/` | `wwwroot/` (served by `app.UseStaticFiles()`) |
| `config.py` + `.env` | `appsettings.json` + `dotnet user-secrets` (dev) + env vars (prod) |
| `SECRET_KEY` | Not needed for CSRF — antiforgery is automatic. Only needed if you add Data Protection key persistence. |
| WTForms + validators | POCO + `System.ComponentModel.DataAnnotations` attributes |
| `form.validate_on_submit()` | `if (!ModelState.IsValid) return Page();` |
| CSRF token (`form.hidden_tag()`) | Automatic on any `<form method="post">` with the tag helper |
| `redirect(url_for('main.index'))` | `return RedirectToPage("/Index");` |
| `@app.errorhandler(404)` | `app.UseStatusCodePagesWithReExecute("/Error", "?code={0}")` |
| `@app.errorhandler(500)` | `app.UseExceptionHandler("/Error")` |
| `app.run()` | `app.Run()` |

### 4.2 Jinja2 → Razor syntax

| Jinja2 | Razor |
|---|---|
| `{% extends "base.html" %}` | `_ViewStart.cshtml` sets `Layout = "_Layout";` (usually nothing per-page) |
| `{% block content %}…{% endblock %}` | `@RenderBody()` in the layout |
| `{% block title %}` | `ViewData["Title"] = "…";` + `@ViewData["Title"]` in layout |
| `{% block scripts %}` | `@RenderSection("Scripts", required: false)` + `@section Scripts { … }` |
| `{% include "nav.html" %}` | `<partial name="_Nav" />` |
| `{{ url_for('main.about') }}` | `<a asp-page="/About">` (never hardcode `href="/about"`) |
| `{{ url_for('static', filename='css/style.css') }}` | `<link rel="stylesheet" href="~/css/style.css" asp-append-version="true" />` |
| `{{ value }}` (auto-escaped) | `@value` (auto-escaped) |
| `{{ value \| safe }}` | `@Html.Raw(value)` — use only for the Power BI iframe if needed |
| `{% for x in items %}` | `@foreach (var x in items)` |
| `{% if cond %}` | `@if (cond)` |
| `{# comment #}` | `@* comment *@` |

**Razor gotcha:** a literal `@` in HTML or CSS (e.g. `@media`, `@font-face`, an email address) must
be escaped as `@@`. This *will* bite when pasting existing templates. Inline `<style>` blocks are
the usual culprit.

### 4.3 Route preservation — mandatory pattern

Every page declares its URL explicitly, so the file name never dictates the route:

```cshtml
@page "/music_and_death_power_bi"
@{ ViewData["Title"] = "Music and Death — Power BI"; }
```

Also set case-insensitive, lowercase URL generation in `Program.cs`:

```csharp
builder.Services.AddRouting(o => { o.LowercaseUrls = true; });
```

**Acceptance test:** after migration, every URL in the §2.3 table must return HTTP 200. Write a
smoke test (xUnit + `WebApplicationFactory`) that asserts exactly that. It's ~20 lines and it is
the single most valuable test in this project.

---

## 5. `Program.cs` — target shape

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddRazorPages();
builder.Services.AddRouting(o => o.LowercaseUrls = true);
builder.Services.AddResponseCompression();
// Contact form only:
builder.Services.AddRateLimiter(/* fixed window, e.g. 5 requests / 10 min per IP */);

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Error");
    app.UseHsts();
}

app.UseStatusCodePagesWithReExecute("/Error", "?code={0}");
app.UseHttpsRedirection();
app.UseStaticFiles();          // serves wwwroot
app.UseResponseCompression();
app.UseRouting();
app.UseRateLimiter();
app.MapRazorPages();

app.Run();
```

Notes:
- `UseHttpsRedirection()` behind a reverse proxy (IIS/nginx) needs `UseForwardedHeaders` or the
  host's integration, or you get redirect loops. On MonsterASP (IIS in-process) this is handled.
- Add long-lived cache headers for `wwwroot` — 312 MB of images benefits a lot:
  `app.UseStaticFiles(new StaticFileOptions { OnPrepareResponse = ctx => ctx.Context.Response.Headers.CacheControl = "public,max-age=31536000,immutable" });`
  paired with `asp-append-version="true"` for cache busting.

---

## 6. Configuration & secrets

| Setting | Dev | Prod |
|---|---|---|
| Everything non-secret | `appsettings.json` (committed) | same |
| Environment overrides | `appsettings.Development.json` | env var `ASPNETCORE_ENVIRONMENT=Production` |
| Secrets (SMTP password / API key) | `dotnet user-secrets set "Email:ApiKey" "…"` | environment variables in the host control panel |

Rules:
- **Never** commit a secret to `appsettings.json`. There is no `.env` equivalent and you don't need one.
- Bind config to strongly-typed options: `builder.Services.Configure<EmailOptions>(builder.Configuration.GetSection("Email"));`
- Do not port `SECRET_KEY`. Antiforgery uses ASP.NET Data Protection, which generates its own keys.
  On a single-instance host, the default file-based key ring is fine. If keys land in a directory
  wiped on deploy, users get sporadic "antiforgery token invalid" errors after each deploy — if that
  happens, persist the key ring to a stable path via `PersistKeysToFileSystem`.

`.gitignore` for the new repo: use the standard `dotnet new gitignore` output (covers `bin/`,
`obj/`, `.vs/`, `*.user`), plus keep `.idea` from the old one if you still use Rider.

---

## 7. Contact form — implement properly this time

The Flask version validated and then discarded the message. The C# version should actually deliver it.

**Model** (`Contact.cshtml.cs`):

```csharp
public class ContactInput
{
    [Required, StringLength(100)]
    public string Name { get; set; } = "";

    [Required, EmailAddress(ErrorMessage = "Not a valid email address.")]
    public string Email { get; set; } = "";

    [Required, MinLength(4, ErrorMessage = "Your message is too short.")]
    [StringLength(4000)]
    public string Body { get; set; } = "";

    // Honeypot: must stay empty. Hidden via CSS, not type="hidden".
    public string? Website { get; set; }
}
```

Keep the original error message strings — they may already be styled/tested in the template.

**Delivery options** (pick one, cheapest first):
1. **Resend** — free tier ~3,000 emails/month, simple REST API, no SMTP config.
2. **SMTP via MailKit** (`MailKit` NuGet) against Gmail/Zoho with an app password.
3. **Formspree/Getform** — no backend code at all, but the form posts off-site.

**Anti-spam, all three cheap and effective:** honeypot field, `AddRateLimiter` per IP, and a
minimum time-to-submit check. Skip CAPTCHA for a portfolio.

Route: `@page "/contact"`, handlers `OnGet()` and `OnPostAsync()`, redirect to `/Index` (or better,
a `?sent=1` state on `/contact`) on success.

---

## 8. Error pages

Single `Error.cshtml` page that reads `?code=` and renders the right message, or two pages
(`/Error/404`, `/Error/500`) mirroring the current `404.html` / `500.html`. Either is fine — the
existing templates decide. Do **not** leak stack traces in Production; only
`app.UseDeveloperExceptionPage()` in Development.

---

## 9. Hosting — options and real prices

Constraint: the site must serve **`www.oscardlc.com` over HTTPS**. That single requirement
eliminates most "free" tiers.

| Option | Price | Custom domain + HTTPS | Verdict |
|---|---|---|---|
| **MonsterASP.NET — Premium Single** | **$1.95/mo first year, $2.50/mo renewal**, billed annually | ✅ free Let's Encrypt | **Recommended.** 25 GB storage, 512 MB dedicated RAM, .NET 10, deploy from VS / GitHub / FTP / SFTP, daily backups, EU or Salt Lake City DC. Storage covers your 312 MB with huge headroom. |
| MonsterASP.NET — Free tier | $0 | ❌ subdomain only (`*.runasp.net`) | Perfect **staging environment**. Deploy here first, verify all 15 URLs, then upgrade. 5 GB / 256 MB RAM, EU only. |
| Azure App Service **F1** | $0 | ❌ no custom domain, 60 CPU-min/day, sleeps when idle | Dev toy only. Rules itself out. |
| Azure App Service **B1** | ~$13/mo | ✅ | ~5× the cost for a static portfolio. Only worth it if you want Azure on your CV. |
| Render / Railway / Fly.io | ~$5–7/mo | ✅ | Linux + Docker. More DevOps to learn; more portable. Free tiers sleep. |
| Hetzner / Contabo VPS | ~€4–5/mo | ✅ (you configure it) | Cheapest at scale but you own OS patching, nginx, certbot, systemd. |
| Cloudflare Pages / GitHub Pages | $0 | ✅ | **Only if the site becomes fully static.** See below. |

### The escape hatch worth considering

Your site has **zero server-side logic** apart from a contact form that doesn't exist yet. You could:

- build it in ASP.NET Core Razor Pages exactly as specced,
- pre-render all 15 pages to static HTML at build time,
- host on **Cloudflare Pages for $0** with a free cert and a global CDN,
- and handle the contact form with a Cloudflare Worker or Formspree.

That's **$0/month forever** and faster than any shared host. The tradeoff is a slightly unusual
build step, and you lose the "it's a real running C# app" story if that's part of the point of the
exercise. Decide this **before** starting — it changes nothing about the Razor code but changes the
deployment pipeline entirely.

**Default recommendation:** MonsterASP free tier for staging → Premium Single ($1.95–2.50/mo) for
production. Compare against what PythonAnywhere currently bills you: a custom domain there requires
a paid plan, so this is likely a small saving as well as a stack change.

---

## 10. Deployment

For MonsterASP (Windows / IIS, in-process hosting):

```bash
dotnet publish OscarDlc.Web -c Release -o ./publish
```

Then FTP/Web Deploy the contents of `./publish`. Include a `web.config` if the SDK doesn't generate
one (it normally does for `Microsoft.NET.Sdk.Web`). Optionally set up GitHub Actions:
build → test (the URL smoke test) → publish → FTP deploy on push to `main`.

Set `ASPNETCORE_ENVIRONMENT=Production` in the host's control panel.

### 10.1 DNS — domain is registered at GoDaddy

GoDaddy is the **registrar** only. It stays the registrar; nothing about the migration requires
transferring the domain. You only repoint records.

**Known GoDaddy limitation:** its DNS supports no `ALIAS`, no `ANAME`, and no CNAME flattening.
Since the DNS spec forbids a `CNAME` at the zone apex, the naked `oscardlc.com` can only be an
`A` record pointing at a literal IP. This matters *only* if the host hands you a hostname instead
of an IP.

| Host | Apex (`oscardlc.com`) | `www` | Blocked by GoDaddy? |
|---|---|---|---|
| MonsterASP | `A` → server IP from control panel | `CNAME` → assigned host | No — they give a real IP |
| Azure App Service B1 | `A` → inbound IP + `TXT` `asuid` verification | `CNAME` → `app.azurewebsites.net` | No |
| Cloudflare Pages | ✋ needs flattening | `CNAME` → `project.pages.dev` | **Yes** — see workaround |
| VPS (Hetzner etc.) | `A` → VPS IP | `A` or `CNAME` | No |

**Cloudflare Pages workaround (pick one):**
- GoDaddy *Domain Forwarding*: `oscardlc.com` → `https://www.oscardlc.com`, permanent (301).
- Or move nameservers to Cloudflare (free). Registration stays at GoDaddy; Cloudflare handles the
  zone and flattens the apex automatically. Also gives faster DNS and a free proxy/CDN.

**Records currently in place:** there is almost certainly a `CNAME` for `www` pointing at
`something.pythonanywhere.com` (or an `A` record to a PythonAnywhere IP). That's the record being
replaced. Leave MX, TXT/SPF, and any Google/Search Console verification records untouched.

### 10.2 Cutover checklist

1. **~24h before:** in GoDaddy DNS, lower the TTL on the `www` and `@` records to `600` seconds.
   The default is 1 hour, which makes mistakes slow to undo.
2. Deploy to the MonsterASP **free** subdomain and verify all 15 URLs there first. Do not touch DNS
   until this passes.
3. Upgrade to Premium and add `oscardlc.com` + `www.oscardlc.com` in the host control panel. It will
   display the exact records to create.
4. In GoDaddy: update `A` for `@`, update `CNAME` for `www`. Save.
5. Wait for resolution — check with `nslookup www.oscardlc.com` or `dig +short www.oscardlc.com`
   from a machine that hasn't cached the old value.
6. **Only then** issue the Let's Encrypt certificate. ACME validation requires DNS to already point
   at the new host; requesting it early just fails and may hit rate limits.
7. Keep PythonAnywhere running for **at least 48h** after the switch. Some resolvers ignore low TTLs.
   Cancel only once traffic has fully moved.
8. Verify `http://` → `https://` redirect and that `oscardlc.com` (no www) resolves too.

**Watch out:** GoDaddy will try to upsell hosting and an SSL certificate during this. You need
neither — the cert is free via Let's Encrypt at the host.

---

## 11. Migration plan

**Phase 0 — Decide** (before writing code)
- [ ] Static-export + Cloudflare Pages, or hosted ASP.NET Core app? (§9)
- [ ] Is the contact form in scope for v1?

**Phase 1 — Scaffold**
- [ ] Visual Studio → ASP.NET Core **Empty** template, solution `OscarDlc`, project `OscarDlc.Web`, .NET 10 (build Razor Pages up by hand)
- [ ] Add Razor Pages to `Program.cs` by hand (Empty template has none)
- [ ] `Program.cs` per §5

**Phase 2 — Assets**
- [ ] Copy `app/main/static/**` → `wwwroot/**`, preserving subfolder names
- [ ] Audit size: 312 MB is a lot. Compress/convert large images to WebP where possible — this is the one "improvement" worth making, because it directly reduces hosting storage and load time.

**Phase 3 — Layout**
- [ ] Convert `base.html` → `_Layout.cshtml`; extract nav/footer into partials
- [ ] Create `_ViewStart.cshtml` and `_ViewImports.cshtml` (`@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers`)
- [ ] Replace every `url_for` and escape every stray `@`

**Phase 4 — Pages** (one at a time, verify in browser after each)
- [ ] 14 content pages with explicit `@page "/…"` routes
- [ ] Error page(s)

**Phase 5 — Contact form** (§7)

**Phase 6 — Verify**
- [ ] Smoke test: all 15 URLs return 200
- [ ] Visual diff against the live PythonAnywhere site, page by page
- [ ] Check the Power BI embed still loads (iframe CSP / `X-Frame-Options` can break embeds — if you add security headers, allow the Power BI origin)
- [ ] Lighthouse pass

**Phase 7 — Deploy**
- [ ] MonsterASP free tier, full re-verification
- [ ] Upgrade to Premium, point DNS for `oscardlc.com` + `www`, issue Let's Encrypt cert
- [ ] Keep PythonAnywhere running until DNS has fully propagated, then cancel

---

## 12. Still needed from the repo

This context file was written from 9 files. To finish the spec, the following are required:

1. **`app/main/templates/*.html`** — all 17 templates. Especially `base.html`: it defines the
   layout, nav, and block structure everything else depends on.
2. **`app/main/static/`** — a directory listing (`tree` or `ls -R`) plus total size per subfolder.
3. **`app/__init__.py` vs `app/main/__init__.py`** — two files named `__init__.py` were provided;
   confirm which is which and the exact package nesting.
4. **The PythonAnywhere WSGI file** — reveals how the app is actually bootstrapped in prod.
5. **Confirmation** that `templates/` and `static/` live under `app/main/` and not under `app/`.
6. Whether any page uses **client-side JS frameworks**, external APIs, or the Power BI JS SDK
   (vs. a plain iframe embed).

## 13. Open questions

- Is the contact form meant to be revived, or intentionally retired?
- Is `www.oscardlc.com` registered somewhere that lets you edit DNS records freely?
- What is the current PythonAnywhere monthly bill (for a real cost comparison)?
- Any analytics, sitemap, `robots.txt`, or Search Console verification files that must carry over?
- Is preserving the exact current design a hard requirement, or is a refresh welcome later?

---

## 14. Rules for the coding agent

1. **Never change a public URL.** §2.3 is a contract.
2. **Do not add a database, ORM, or migration framework.** SQLAlchemy in `requirements.txt` was
   unused; do not "helpfully" reintroduce it as EF Core.
3. **Do not restyle.** Reuse the existing CSS verbatim. Image optimisation is the only exception.
4. Prefer `asp-page` tag helpers over hardcoded `href` values.
5. One page at a time; verify in the browser before moving on.
6. No secrets in committed files, ever.
7. If a Jinja template uses something with no clean Razor equivalent, stop and ask rather than
   inventing a workaround.
