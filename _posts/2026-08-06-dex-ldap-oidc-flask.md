---
layout: post
title:  "Dex + LDAP: One Directory, Any App, via OpenID Connect"
summary: "A practical, tested walkthrough of running Dex against an LDAP directory with Docker Compose, wiring up the connector, and letting a small Flask app log a real LDAP user in over OIDC — no password ever touches the app."
author: vijay
date: '2026-08-06 14:30:00 +0000'
category: devops
thumbnail: /assets/img/posts/dex-ldap-oidc.svg
keywords: dex, ldap, oidc, openid connect, docker compose, flask, authlib, sso, identity broker
permalink: /blog/dex-ldap-oidc-flask/
usemathjax: false
---

Every company past a certain size has the same directory problem. There is one real source of truth for "who works here and what's their password" — usually LDAP or Active Directory — and then there is every application built since 2015, none of which wants to speak LDAP. Modern frameworks, modern client libraries, and modern security reviewers all want OpenID Connect: a redirect, a token, a signed set of claims.

[Dex](https://dexidp.io/) is the piece that sits between those two worlds. It's an OIDC provider that doesn't own any users itself — it delegates authentication to a *connector*, and one of the connectors it ships with talks directly to LDAP. Point Dex at your directory, register your app as an OIDC client, and every app you write from that point on authenticates the exact same way — regardless of whether the identity behind it lives in LDAP, Active Directory, GitHub, Google, or SAML. Swap the connector later and the app doesn't change a line of code.

This post covers all of it, and everything in it is tested against real running containers, not just documentation: Dex + a real LDAP server via Docker Compose, the exact connector configuration to wire them together, what to change in that configuration to point at a real corporate directory instead, and a small Flask app that logs a user in and prints their LDAP `uid` back at them.

<div class="wif-fig" markdown="0"><style>
.wif-fig { margin: 1.75rem 0; }
.wif-fig figure { margin: 0; }
.wif-fig .wif-scroll { overflow-x: auto; -webkit-overflow-scrolling: touch; border: 1px solid #e2e8f0; border-radius: 0.75rem; background: #ffffff; padding: 0.75rem; }
.dark .wif-fig .wif-scroll { border-color: #334155; background: #0f172a; }
.wif-fig svg { display: block; width: 100%; min-width: 720px; height: auto; }
.wif-fig figcaption { margin-top: 0.6rem; font-size: 0.85rem; color: #64748b; font-style: italic; text-align: center; }
.dark .wif-fig figcaption { color: #94a3b8; }
</style>
<figure>
<div class="wif-scroll">
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1140 620" class="dexd" role="img" aria-label="Architecture of Dex bridging LDAP to OIDC for a Flask application">
  <style>
    .dexd .lane-app  { fill: #f8fafc; stroke: #cbd5e1; }
    .dexd .lane-dex  { fill: #f5f3ff; stroke: #ddd6fe; }
    .dexd .lane-ldap { fill: #fffbeb; stroke: #fde68a; }
    .dexd .card      { fill: #ffffff; stroke: #cbd5e1; }
    .dexd .card-dex  { fill: #ffffff; stroke: #a78bfa; }
    .dexd .card-ldap { fill: #ffffff; stroke: #f59e0b; }
    .dexd .chip      { fill: #e2e8f0; }
    .dexd .chip-dex  { fill: #ede9fe; }
    .dexd .chip-ldap { fill: #fef3c7; }
    .dexd .lane-t    { fill: #334155; font-weight: 700; }
    .dexd .lane-t-dex{ fill: #6d28d9; font-weight: 700; }
    .dexd .lane-t-ldap{ fill: #b45309; font-weight: 700; }
    .dexd .h         { fill: #0f172a; font-weight: 700; }
    .dexd .p         { fill: #475569; }
    .dexd .mono      { fill: #be185d; font-family: 'JetBrains Mono', Menlo, monospace; }
    .dexd .mono-v    { fill: #6d28d9; font-family: 'JetBrains Mono', Menlo, monospace; }
    .dexd .mono-a    { fill: #b45309; font-family: 'JetBrains Mono', Menlo, monospace; }
    .dexd .arrow     { stroke: #6366f1; fill: none; }
    .dexd .arrow-h   { fill: #6366f1; }
    .dexd .alab      { fill: #4f46e5; font-weight: 600; }
    .dexd .note      { fill: #64748b; font-style: italic; }
    .dexd text       { font-family: Inter, system-ui, -apple-system, sans-serif; }
    .dark .dexd .lane-app  { fill: #0f172a; stroke: #334155; }
    .dark .dexd .lane-dex  { fill: #1e1b3a; stroke: #6d28d9; }
    .dark .dexd .lane-ldap { fill: #2a2210; stroke: #a16207; }
    .dark .dexd .card      { fill: #1e293b; stroke: #475569; }
    .dark .dexd .card-dex  { fill: #241f45; stroke: #8b5cf6; }
    .dark .dexd .card-ldap { fill: #2e2610; stroke: #d97706; }
    .dark .dexd .chip      { fill: #334155; }
    .dark .dexd .chip-dex  { fill: #4c1d95; }
    .dark .dexd .chip-ldap { fill: #78350f; }
    .dark .dexd .lane-t    { fill: #cbd5e1; }
    .dark .dexd .lane-t-dex{ fill: #c4b5fd; }
    .dark .dexd .lane-t-ldap{ fill: #fcd34d; }
    .dark .dexd .h         { fill: #f1f5f9; }
    .dark .dexd .p         { fill: #cbd5e1; }
    .dark .dexd .mono      { fill: #f472b6; }
    .dark .dexd .mono-v    { fill: #c4b5fd; }
    .dark .dexd .mono-a    { fill: #fcd34d; }
    .dark .dexd .arrow     { stroke: #818cf8; }
    .dark .dexd .arrow-h   { fill: #818cf8; }
    .dark .dexd .alab      { fill: #a5b4fc; }
    .dark .dexd .note      { fill: #94a3b8; }
  </style>
  <defs>
    <marker id="dah" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M0 0 L10 5 L0 10 z" class="arrow-h"/>
    </marker>
  </defs>
  <!-- LANES -->
  <rect class="lane-app"  x="14"  y="52" width="326" height="530" rx="16" stroke-width="1.5"/>
  <rect class="lane-dex"  x="368" y="52" width="384" height="530" rx="16" stroke-width="1.5"/>
  <rect class="lane-ldap" x="780" y="52" width="346" height="530" rx="16" stroke-width="1.5"/>
  <rect class="chip"      x="30"  y="20" width="180" height="28" rx="14"/>
  <text class="lane-t"    x="120" y="39" text-anchor="middle" font-size="14">Browser + Flask app</text>
  <rect class="chip-dex"  x="384" y="20" width="200" height="28" rx="14"/>
  <text class="lane-t-dex" x="484" y="39" text-anchor="middle" font-size="14">Dex (OIDC provider)</text>
  <rect class="chip-ldap" x="796" y="20" width="220" height="28" rx="14"/>
  <text class="lane-t-ldap" x="906" y="39" text-anchor="middle" font-size="14">LDAP directory</text>
  <!-- COLUMN A -->
  <g>
    <rect class="card" x="34" y="76" width="286" height="118" rx="12" stroke-width="1.5"/>
    <text class="h"    x="52" y="102" font-size="15">Your Flask app</text>
    <text class="p"    x="52" y="124" font-size="12">Doesn't know LDAP exists.</text>
    <text class="p"    x="52" y="142" font-size="12">Speaks only OpenID Connect,</text>
    <text class="p"    x="52" y="160" font-size="12">via a standard library</text>
    <text class="mono" x="52" y="178" font-size="12">(Authlib)</text>
  </g>
  <g>
    <rect class="card" x="34" y="210" width="286" height="140" rx="12" stroke-width="1.5"/>
    <text class="h"    x="52" y="236" font-size="15">/login</text>
    <text class="p"    x="52" y="258" font-size="12">Redirects the browser to</text>
    <text class="mono" x="52" y="276" font-size="11.5">/dex/auth?client_id=…</text>
    <text class="p"    x="52" y="298" font-size="12">&amp;scope=openid+profile+email</text>
    <text class="note" x="52" y="320" font-size="11">the app never sees a password</text>
  </g>
  <g>
    <rect class="card" x="34" y="366" width="286" height="150" rx="12" stroke-width="1.5"/>
    <text class="h"    x="52" y="392" font-size="15">/callback</text>
    <text class="p"    x="52" y="414" font-size="12">Exchanges the code for tokens,</text>
    <text class="p"    x="52" y="432" font-size="12">reads the ID token claims:</text>
    <text class="mono" x="52" y="454" font-size="11.5">preferred_username: "fry"</text>
    <text class="mono" x="52" y="470" font-size="11.5">email: fry@planetexpress.com</text>
    <text class="mono" x="52" y="486" font-size="11.5">groups: ["ship_crew"]</text>
  </g>
  <!-- COLUMN B -->
  <g>
    <rect class="card-dex" x="386" y="76" width="348" height="150" rx="12" stroke-width="1.5"/>
    <text class="h"        x="404" y="102" font-size="15">OIDC endpoints</text>
    <text class="mono-v"   x="404" y="124" font-size="11.5">/.well-known/openid-configuration</text>
    <text class="mono-v"   x="404" y="142" font-size="11.5">/dex/auth</text>
    <text class="mono-v"   x="404" y="160" font-size="11.5">/dex/token</text>
    <text class="mono-v"   x="404" y="178" font-size="11.5">/dex/keys</text>
    <text class="p"        x="404" y="200" font-size="12">Standard OIDC — any client library works.</text>
  </g>
  <g>
    <rect class="card-dex" x="386" y="240" width="348" height="130" rx="12" stroke-width="1.5"/>
    <text class="h"        x="404" y="266" font-size="15">LDAP connector</text>
    <text class="p"        x="404" y="288" font-size="12">config: type: ldap</text>
    <text class="p"        x="404" y="306" font-size="12">Renders the login form, binds</text>
    <text class="p"        x="404" y="324" font-size="12">to the directory with the user's</text>
    <text class="p"        x="404" y="342" font-size="12">own submitted credentials.</text>
  </g>
  <g>
    <rect class="card-dex" x="386" y="384" width="348" height="132" rx="12" stroke-width="1.5"/>
    <text class="h"        x="404" y="410" font-size="15">Static client registry</text>
    <text class="mono-v"   x="404" y="432" font-size="11.5">staticClients:</text>
    <text class="mono-v"   x="418" y="448" font-size="11.5">- id: flask-demo</text>
    <text class="mono-v"   x="418" y="464" font-size="11.5">redirectURIs:</text>
    <text class="mono-v"   x="432" y="480" font-size="11.5">- localhost:5000/callback</text>
    <text class="note"     x="404" y="500" font-size="11">issues the ID token, signs it with its own key</text>
  </g>
  <!-- COLUMN C -->
  <g>
    <rect class="card-ldap" x="798" y="76" width="312" height="130" rx="12" stroke-width="1.5"/>
    <text class="h"         x="816" y="102" font-size="15">Directory tree</text>
    <text class="mono-a"    x="816" y="124" font-size="11.5">dc=planetexpress,dc=com</text>
    <text class="mono-a"    x="830" y="140" font-size="11.5">ou=people</text>
    <text class="mono-a"    x="844" y="156" font-size="11">uid=fry, mail, cn …</text>
    <text class="mono-a"    x="844" y="172" font-size="11">uid=leela, mail, cn …</text>
    <text class="p"         x="816" y="194" font-size="11.5">any LDAP or Active Directory works</text>
  </g>
  <g>
    <rect class="card-ldap" x="798" y="220" width="312" height="118" rx="12" stroke-width="1.5"/>
    <text class="h"         x="816" y="246" font-size="15">Bind #1 — service account</text>
    <text class="mono-a"    x="816" y="268" font-size="11">bindDN + bindPW</text>
    <text class="p"         x="816" y="288" font-size="12">Read-only. Used only to</text>
    <text class="p"         x="816" y="306" font-size="12">search for the user's DN.</text>
    <text class="note"      x="816" y="324" font-size="10.5">never sees the end user's password</text>
  </g>
  <g>
    <rect class="card-ldap" x="798" y="352" width="312" height="130" rx="12" stroke-width="1.5"/>
    <text class="h"         x="816" y="378" font-size="15">Bind #2 — the real check</text>
    <text class="p"         x="816" y="400" font-size="12">Dex re-binds AS that DN,</text>
    <text class="p"         x="816" y="418" font-size="12">using the password the user</text>
    <text class="p"         x="816" y="436" font-size="12">just typed into Dex's form.</text>
    <text class="note"      x="816" y="458" font-size="11">success = correct password</text>
    <text class="note"      x="816" y="474" font-size="11">failure = invalid credentials</text>
  </g>
  <!-- ARROWS -->
  <path class="arrow" d="M320 280 H384" stroke-width="2.5" marker-end="url(#dah)"/>
  <text class="alab" x="352" y="272" text-anchor="middle" font-size="11">browser redirect</text>
  <path class="arrow" d="M734 154 V128 M560 226 V240" stroke-width="0" opacity="0"/>
  <path class="arrow" d="M562 226 V240" stroke-width="2.5" marker-end="url(#dah)"/>
  <path class="arrow" d="M734 305 H796" stroke-width="2.5" marker-end="url(#dah)"/>
  <text class="alab" x="765" y="297" text-anchor="middle" font-size="10.5">search bindDN</text>
  <path class="arrow" d="M734 411 H796" stroke-width="2.5" marker-end="url(#dah)"/>
  <text class="alab" x="765" y="403" text-anchor="middle" font-size="10.5">re-bind as user</text>
  <path class="arrow" d="M384 440 H318" stroke-width="2.5" marker-end="url(#dah)"/>
  <text class="alab" x="352" y="432" text-anchor="middle" font-size="11">code exchange</text>
</svg>
</div>
<figcaption>Dex sits between your app and the directory. The app only ever speaks OIDC; only Dex speaks LDAP.</figcaption>
</figure>
</div>

Three things worth fixing in your head before the config starts flying:

- **The LDAP bind DN and password never leave your infrastructure.** They live in Dex's config, not in the app, not in the browser.
- **Dex binds twice.** Once as a read-only service account to *find* the user's directory entry, and a second time as that user's own DN with the password they just typed, to actually check it. That second bind succeeding *is* the authentication.
- **The app speaks one protocol forever.** However many identity sources you plug into Dex over the years, the Flask app in this post never changes — it only ever talks OIDC to Dex.

---

## 1. Running Dex against LDAP with Docker Compose

For the directory, I'm using [`rroemhild/docker-test-openldap`](https://github.com/rroemhild/docker-test-openldap) — a disposable OpenLDAP server pre-loaded with a small cast of test users (it uses the Planet Express crew from *Futurama* as sample data), so there's no LDIF to write and no schema to design before you can see a real login work. Section 5 covers exactly what to change to point this at your actual corporate LDAP or Active Directory.

```yaml
# docker-compose.yml
services:
  openldap:
    image: ghcr.io/rroemhild/docker-test-openldap:master
    container_name: openldap
    ports:
      - "10389:10389"   # LDAP
      - "10636:10636"   # LDAPS
    restart: unless-stopped

  dex:
    image: ghcr.io/dexidp/dex:v2.45.1
    container_name: dex
    depends_on:
      - openldap
    ports:
      - "5556:5556"
    volumes:
      - ./dex-config.yaml:/etc/dex/config.yaml:ro
    command: ["dex", "serve", "/etc/dex/config.yaml"]
    restart: unless-stopped
```

Two things about that file that aren't obvious until you hit them:

- **Dex's own image expects its config at `/etc/dex/config.docker.yaml` by default.** The upstream `Dockerfile`'s `CMD` is `dex serve /etc/dex/config.docker.yaml`. I mount the config to a plainer path and override `command` explicitly so there's no ambiguity about which file is actually loaded.
- **The test LDAP image listens on 10389, not the standard 389**, both inside and outside the container — that's baked into its `Dockerfile` (`EXPOSE 10389 10636`), not a Compose quirk. Keep that in mind if you swap in a different LDAP image that uses the standard ports.

Now the file that actually does the work — `dex-config.yaml`, sitting next to the compose file:

```yaml
# dex-config.yaml
issuer: http://localhost:5556/dex

storage:
  type: memory

web:
  http: 0.0.0.0:5556

connectors:
- type: ldap
  id: ldap
  name: Planet Express LDAP
  config:
    host: openldap:10389
    insecureNoSSL: true

    # A read-only account Dex uses only to *search* for the user's DN.
    bindDN: cn=admin,dc=planetexpress,dc=com
    bindPW: GoodNewsEveryone

    usernamePrompt: "Email or Username"

    userSearch:
      baseDN: ou=people,dc=planetexpress,dc=com
      filter: "(objectClass=inetOrgPerson)"
      username: uid
      idAttr: uid
      emailAttr: mail
      nameAttr: cn
      preferredUsernameAttr: uid

    groupSearch:
      baseDN: ou=people,dc=planetexpress,dc=com
      filter: "(objectClass=Group)"
      userMatchers:
      - userAttr: DN
        groupAttr: member
      nameAttr: cn

staticClients:
- id: flask-demo
  name: "Flask Hello App"
  secret: flask-demo-secret
  redirectURIs:
  - "http://localhost:5000/callback"
```

Bring it up:

```bash
docker compose up -d
```

Confirm Dex is alive and has picked up the connector:

```bash
curl -s http://localhost:5556/dex/.well-known/openid-configuration | jq .issuer
# "http://localhost:5556/dex"

docker compose logs dex | grep connector
# level=INFO msg="config connector" connector_id=ldap
```

`storage: memory` is deliberate for this walkthrough — nothing about the LDAP connector needs a database, and it means there's no SQLite file permissions to fight with. For a Dex you intend to keep running, switch to `sqlite3`, `postgres`, or `etcd` (all documented in Dex's [`config.yaml.dist`](https://github.com/dexidp/dex/blob/master/config.yaml.dist)) so `staticClients` added later via the gRPC API and refresh tokens survive a restart. Static, file-defined clients and connectors — the only kind this post uses — are read fresh from the YAML every time regardless of storage backend.

---

## 2. How the pieces actually fit together

<div class="wif-fig" markdown="0">
<figure>
<div class="wif-scroll">
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1160 860" class="dexs" role="img" aria-label="Sequence of a login through Dex against LDAP for a Flask application">
  <style>
    .dexs .actor-app { fill: #f1f5f9; stroke: #94a3b8; }
    .dexs .actor-dex { fill: #f5f3ff; stroke: #a78bfa; }
    .dexs .actor-ldap{ fill: #fffbeb; stroke: #f59e0b; }
    .dexs .life      { stroke: #cbd5e1; stroke-dasharray: 4 6; }
    .dexs .an        { fill: #0f172a; font-weight: 700; }
    .dexs .req       { stroke: #4f46e5; fill: none; }
    .dexs .req-h     { fill: #4f46e5; }
    .dexs .res       { stroke: #059669; fill: none; stroke-dasharray: 7 5; }
    .dexs .res-h     { fill: #059669; }
    .dexs .lreq      { fill: #4338ca; font-weight: 600; }
    .dexs .lres      { fill: #047857; font-weight: 600; }
    .dexs .mono      { fill: #be185d; font-family: 'JetBrains Mono', Menlo, monospace; }
    .dexs .noteb     { fill: #fffbeb; stroke: #fcd34d; }
    .dexs .notet     { fill: #78350f; }
    .dexs .band      { fill: #f8fafc; }
    .dexs .bandt     { fill: #64748b; font-weight: 700; letter-spacing: 0.06em; }
    .dexs .foot      { fill: #64748b; font-style: italic; }
    .dexs text       { font-family: Inter, system-ui, -apple-system, sans-serif; }
    .dark .dexs .actor-app { fill: #1e293b; stroke: #64748b; }
    .dark .dexs .actor-dex { fill: #241f45; stroke: #8b5cf6; }
    .dark .dexs .actor-ldap{ fill: #2e2610; stroke: #d97706; }
    .dark .dexs .life      { stroke: #475569; }
    .dark .dexs .an        { fill: #f1f5f9; }
    .dark .dexs .req       { stroke: #818cf8; }
    .dark .dexs .req-h     { fill: #818cf8; }
    .dark .dexs .res       { stroke: #34d399; }
    .dark .dexs .res-h     { fill: #34d399; }
    .dark .dexs .lreq      { fill: #a5b4fc; }
    .dark .dexs .lres      { fill: #6ee7b7; }
    .dark .dexs .mono      { fill: #f472b6; }
    .dark .dexs .noteb     { fill: #2a2210; stroke: #a16207; }
    .dark .dexs .notet     { fill: #fcd34d; }
    .dark .dexs .band      { fill: #172033; }
    .dark .dexs .bandt     { fill: #94a3b8; }
    .dark .dexs .foot      { fill: #94a3b8; }
  </style>
  <defs>
    <marker id="dsrq" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M0 0 L10 5 L0 10 z" class="req-h"/>
    </marker>
    <marker id="dsrs" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M0 0 L10 5 L0 10 z" class="res-h"/>
    </marker>
  </defs>
  <!-- phase bands -->
  <rect class="band" x="0" y="80"  width="1160" height="130" rx="8"/>
  <rect class="band" x="0" y="210" width="1160" height="210" rx="8"/>
  <rect class="band" x="0" y="420" width="1160" height="150" rx="8"/>
  <rect class="band" x="0" y="570" width="1160" height="150" rx="8"/>
  <rect class="band" x="0" y="720" width="1160" height="100" rx="8"/>
  <rect class="band" x="8" y="86"  width="150" height="16"/>
  <text class="bandt" x="14" y="98" font-size="10">1 · START LOGIN</text>
  <rect class="band" x="8" y="216" width="190" height="16"/>
  <text class="bandt" x="14" y="228" font-size="10">2 · LDAP LOGIN FORM</text>
  <rect class="band" x="8" y="426" width="230" height="16"/>
  <text class="bandt" x="14" y="438" font-size="10">3 · CONSENT + REDIRECT BACK</text>
  <rect class="band" x="8" y="576" width="170" height="16"/>
  <text class="bandt" x="14" y="588" font-size="10">4 · CODE EXCHANGE</text>
  <rect class="band" x="8" y="726" width="130" height="16"/>
  <text class="bandt" x="14" y="738" font-size="10">5 · RENDER</text>
  <!-- actors -->
  <g>
    <rect class="actor-app" x="18"  y="14" width="220" height="50" rx="10" stroke-width="1.5"/>
    <text class="an" x="128" y="35" text-anchor="middle" font-size="13">Browser</text>
    <text class="an" x="128" y="52" text-anchor="middle" font-size="13">(the user)</text>
    <path class="life" d="M128 64 V806" stroke-width="1.5"/>
  </g>
  <g>
    <rect class="actor-app" x="278" y="14" width="220" height="50" rx="10" stroke-width="1.5"/>
    <text class="an" x="388" y="35" text-anchor="middle" font-size="13">Flask app</text>
    <text class="an" x="388" y="52" text-anchor="middle" font-size="13">(Authlib client)</text>
    <path class="life" d="M388 64 V806" stroke-width="1.5"/>
  </g>
  <g>
    <rect class="actor-dex" x="538" y="14" width="220" height="50" rx="10" stroke-width="1.5"/>
    <text class="an" x="648" y="35" text-anchor="middle" font-size="13">Dex</text>
    <text class="an" x="648" y="52" text-anchor="middle" font-size="13">(OIDC provider)</text>
    <path class="life" d="M648 64 V806" stroke-width="1.5"/>
  </g>
  <g>
    <rect class="actor-ldap" x="798" y="14" width="220" height="50" rx="10" stroke-width="1.5"/>
    <text class="an" x="908" y="35" text-anchor="middle" font-size="13">LDAP server</text>
    <text class="an" x="908" y="52" text-anchor="middle" font-size="13">(the directory)</text>
    <path class="life" d="M908 64 V806" stroke-width="1.5"/>
  </g>
  <!-- ① -->
  <text class="lreq" x="258" y="122" text-anchor="middle" font-size="12">① GET /login</text>
  <path class="req" d="M128 132 H378" stroke-width="2.2" marker-end="url(#dsrq)"/>
  <text class="lreq" x="518" y="150" text-anchor="middle" font-size="12">② 302 to /dex/auth?client_id=flask-demo&amp;scope=openid+profile+email+groups&amp;…</text>
  <path class="req" d="M388 160 H638" stroke-width="2.2" marker-end="url(#dsrq)"/>
  <text class="lres" x="388" y="184" text-anchor="middle" font-size="12">③ Dex renders its LDAP login form</text>
  <path class="res" d="M648 192 H138" stroke-width="2.2" marker-end="url(#dsrs)"/>
  <!-- ② band -->
  <text class="lreq" x="258" y="252" text-anchor="middle" font-size="12">④ user types uid + password into Dex's own page</text>
  <path class="req" d="M128 262 H638" stroke-width="2.2" marker-end="url(#dsrq)"/>
  <text class="lreq" x="778" y="288" text-anchor="middle" font-size="12">⑤ bind as bindDN, search userSearch.baseDN for uid=&lt;input&gt;</text>
  <path class="req" d="M648 298 H898" stroke-width="2.2" marker-end="url(#dsrq)"/>
  <text class="lres" x="778" y="316" text-anchor="middle" font-size="12">⑥ returns the entry's DN</text>
  <path class="res" d="M908 324 H658" stroke-width="2.2" marker-end="url(#dsrs)"/>
  <text class="lreq" x="778" y="352" text-anchor="middle" font-size="12">⑦ re-bind as that DN, using the password just submitted</text>
  <path class="req" d="M648 362 H898" stroke-width="2.2" marker-end="url(#dsrq)"/>
  <rect class="noteb" x="700" y="374" width="330" height="42" rx="10" stroke-width="1.5"/>
  <text class="notet" x="716" y="392" font-size="11">This bind is the actual password check —</text>
  <text class="notet" x="716" y="407" font-size="11">Dex never hashes or stores it itself.</text>
  <!-- ③ band -->
  <text class="lres" x="388" y="460" text-anchor="middle" font-size="12">⑧ first login: Dex shows a one-time "Grant Access" consent screen</text>
  <path class="res" d="M648 470 H138" stroke-width="2.2" marker-end="url(#dsrs)"/>
  <text class="lreq" x="258" y="490" text-anchor="middle" font-size="12">⑨ user clicks "Grant Access"</text>
  <path class="req" d="M128 500 H638" stroke-width="2.2" marker-end="url(#dsrq)"/>
  <text class="lres" x="388" y="524" text-anchor="middle" font-size="12">⑩ 302 to /callback?code=&lt;authorization code&gt;&amp;state=…</text>
  <path class="res" d="M648 534 H138" stroke-width="2.2" marker-end="url(#dsrs)"/>
  <text class="notet" x="388" y="552" text-anchor="middle" font-size="10.5" font-style="italic">the code is single-use and expires in seconds</text>
  <!-- ④ band -->
  <text class="lreq" x="518" y="612" text-anchor="middle" font-size="12">⑪ POST /dex/token — code + client_id + client_secret</text>
  <path class="req" d="M388 622 H638" stroke-width="2.2" marker-end="url(#dsrq)"/>
  <text class="lres" x="518" y="646" text-anchor="middle" font-size="12">⑫ id_token + access_token (server-to-server call)</text>
  <path class="res" d="M648 656 H398" stroke-width="2.2" marker-end="url(#dsrs)"/>
  <rect class="noteb" x="700" y="580" width="330" height="80" rx="10" stroke-width="1.5"/>
  <text class="notet" x="716" y="600" font-size="11.5" font-weight="700">Authlib does this for you:</text>
  <text class="notet" x="716" y="618" font-size="11">verifies the id_token's signature</text>
  <text class="notet" x="716" y="634" font-size="11">against Dex's /dex/keys (JWKS),</text>
  <text class="notet" x="716" y="650" font-size="11">then exposes the claims as token["userinfo"]</text>
  <!-- ⑤ band -->
  <text class="lres" x="258" y="762" text-anchor="middle" font-size="12">⑬ 302 to / — Flask reads preferred_username from the claims</text>
  <path class="res" d="M388 772 H138" stroke-width="2.2" marker-end="url(#dsrs)"/>
  <text class="foot" x="580" y="806" text-anchor="middle" font-size="11">Flask never saw a password. LDAP never saw an HTTP request from the browser.</text>
</svg>
</div>
<figcaption>The full login sequence. Steps &#9312;&#8211;&#9313; never involve LDAP directly, and steps &#9314;&#8211;&#9317; never involve the browser.</figcaption>
</figure>
</div>

Walk through what each numbered step is really doing:

**① – ③ Kick off.** Your app redirects the browser to Dex's `/dex/auth` endpoint with the usual OAuth 2.0 authorization-code parameters. Because this Dex has exactly one connector configured, it skips the "choose your identity provider" screen and renders the LDAP connector's login form directly.

**④ – ⑦ The two binds.** The user types their LDAP username and password into a form served *by Dex*, not by your app. Dex's LDAP connector then does exactly what any LDAP client would:

1. Binds as `bindDN` (the service account) and searches `userSearch.baseDN` for an entry matching `userSearch.filter` where `userSearch.username` equals what was typed.
2. Takes the DN of whatever it found, and re-binds to the LDAP server *as that DN*, using the password the user just submitted.

If bind #2 succeeds, the user is who they say they are. That's the entire authentication — Dex holds no password hash of its own for LDAP users, checks nothing itself, and simply asks the directory the same question it would ask for any other client.

**⑧ – ⑩ Consent and the redirect back.** The first time a given user authorizes a given client, Dex shows a one-time "Grant Access" screen listing the scopes requested (profile, email, groups). Approve it, and Dex 302s the browser back to your app's redirect URI with a short-lived, single-use authorization `code`.

**⑪ – ⑫ The code exchange.** This step never touches the browser. Your app's backend calls Dex's `/dex/token` endpoint directly, authenticating as itself with `client_id` + `client_secret`, and trades the code for an `id_token` and `access_token`.

**⑬ Render.** The ID token is a signed JWT. Your OIDC client library verifies its signature against Dex's published keys and hands you a plain dictionary of claims — `preferred_username`, `email`, `groups`, whatever scopes you asked for. That's where `"fry"` comes from.

Nowhere in that sequence does your application process see a password, and nowhere does the browser talk to LDAP. That separation is the entire point of putting Dex in the middle.

---

## 3. What Dex actually needs from your directory

The `connectors[].config` block above is the part worth understanding field by field, because it's what you'll actually be editing when you point this at something real.

| Field | What it's for | Value used above |
|---|---|---|
| `host` | LDAP server address, `host:port` | `openldap:10389` |
| `insecureNoSSL` | Allow a plaintext connection | `true` (demo only — see §6) |
| `bindDN` / `bindPW` | The read-only service account used to *search* for users | `cn=admin,dc=planetexpress,dc=com` |
| `usernamePrompt` | Label on Dex's login form | `"Email or Username"` |
| `userSearch.baseDN` | Where under the tree to look for people | `ou=people,dc=planetexpress,dc=com` |
| `userSearch.filter` | Extra LDAP filter narrowing the search | `(objectClass=inetOrgPerson)` |
| `userSearch.username` | Attribute compared against what the user typed | `uid` |
| `userSearch.idAttr` | Attribute (or literal `DN`) used as the stable internal ID | `uid` |
| `userSearch.emailAttr` | Attribute mapped to the OIDC `email` claim | `mail` |
| `userSearch.nameAttr` | Attribute mapped to the OIDC `name` claim | `cn` |
| `userSearch.preferredUsernameAttr` | Attribute mapped to `preferred_username` | `uid` |
| `groupSearch.baseDN` / `.filter` | Where to look for group objects | `ou=people,…`, `(objectClass=Group)` |
| `groupSearch.userMatchers` | How a user's DN maps onto a group's member list | `userAttr: DN`, `groupAttr: member` |

`preferredUsernameAttr` is the field doing the most work for this specific use case — it's what puts the *bare* LDAP `uid` (`fry`, not a DN, not an email) into the token as `preferred_username`, which is exactly what the Flask app below prints.

Two things Google's and every other vendor's docs will tell you if you keep going down this road, worth internalizing now: **map any attribute before you reference it** — you can't use a claim in `userSearch` or `groupSearch` that isn't named there — and **an unset `groupSearch` block is fine**; groups are additive, not required for basic login to work.

---

## 4. The Flask app

This is deliberately small: log in, read one claim, print it.

```
dex-demo/
├── app.py
├── requirements.txt
```

```text
# requirements.txt
Flask==3.1.3
Authlib==1.7.2
requests==2.32.3
```

That `requests` line matters more than it looks — Authlib's Flask integration imports it eagerly even though `pip install Authlib` alone doesn't always pull it in as a hard dependency. Leave it out and the app fails at import time with `ModuleNotFoundError: No module named 'requests'`, not at request time, which makes it a confusing first error to hit.

```python
# app.py
import os

from authlib.integrations.flask_client import OAuth
from flask import Flask, redirect, session, url_for

app = Flask(__name__)
app.secret_key = os.environ.get("FLASK_SECRET_KEY", "dev-only-change-me")

oauth = OAuth(app)
oauth.register(
    name="dex",
    server_metadata_url="http://localhost:5556/dex/.well-known/openid-configuration",
    client_id="flask-demo",
    client_secret="flask-demo-secret",
    client_kwargs={"scope": "openid profile email groups"},
)


@app.route("/")
def index():
    user = session.get("user")
    if not user:
        return '<p>Not logged in.</p><a href="/login">Log in with Dex</a>'
    return (
        f"<h1>Hello, {user['preferred_username']}!</h1>"
        f"<p>Email: {user.get('email')}</p>"
        f"<p>Groups: {', '.join(user.get('groups', [])) or '(none)'}</p>"
        f'<a href="/logout">Log out</a>'
    )


@app.route("/login")
def login():
    redirect_uri = url_for("callback", _external=True)
    return oauth.dex.authorize_redirect(redirect_uri)


@app.route("/callback")
def callback():
    token = oauth.dex.authorize_access_token()
    session["user"] = token["userinfo"]
    return redirect(url_for("index"))


@app.route("/logout")
def logout():
    session.pop("user", None)
    return redirect(url_for("index"))


if __name__ == "__main__":
    app.run(host="localhost", port=int(os.environ.get("PORT", 5000)), debug=True)
```

Walking through the parts that matter:

- **`server_metadata_url`** is the only endpoint you have to hand-configure. Authlib fetches Dex's discovery document from it and learns every other endpoint (`/dex/auth`, `/dex/token`, `/dex/keys`) on its own — this is the entire point of OIDC discovery.
- **`client_kwargs={"scope": "openid profile email groups"}`** determines which claims come back. Drop `profile` and `preferred_username` disappears from the token; drop `groups` and so does the group list. `openid` is non-negotiable — without it you get a bare OAuth 2.0 token, no ID token at all.
- **`oauth.dex.authorize_redirect(redirect_uri)`** builds the entire `/dex/auth?...` URL — client ID, scope, state, and PKCE challenge — and sends the browser there. You never construct that URL by hand.
- **`token = oauth.dex.authorize_access_token()`** is doing steps ⑪ and ⑫ from the diagram in one call: it exchanges the code, fetches Dex's signing keys, verifies the ID token's signature, and returns the decoded claims as `token["userinfo"]`.
- **`session["user"] = token["userinfo"]`** is deliberately the only thing this demo persists. In anything beyond a demo, treat this as a starting point for your own session/user model, not the final design.

Install and run it:

```bash
pip install -r requirements.txt
python app.py
```

Open `http://localhost:5000` and log in as any of the test directory's users — `fry`, `leela`, `bender`, `hermes`, `professor`, `zoidberg`, or `amy` — using the username as the password too (`fry` / `fry`). You'll land on Dex's login page, not Flask's; that's correct, and it's the whole design. After granting access once, you're bounced back to:

```
Hello, fry!
Email: fry@planetexpress.com
Groups: ship_crew
```

That's a real LDAP bind, through a real OIDC exchange, printing a claim that traces straight back to a `uid` attribute in the directory.

> **Browse to `localhost`, not `127.0.0.1`.** The redirect URI registered with Dex is `http://localhost:5000/callback` exactly. Flask's `url_for(..., _external=True)` builds that URL from the host header of the incoming request, so if you open the app via `127.0.0.1:5000` instead, the generated redirect URI won't match what's registered and Dex will reject it. Pick one hostname and use it everywhere — browser, `redirectURIs`, and `server_metadata_url` alike.

---

## 5. Pointing this at a real LDAP or Active Directory

Everything above uses a disposable test directory so the whole chain works from a clean checkout with zero setup. Swapping in your real infrastructure only ever touches the `connectors[].config` block — nothing in the Flask app changes, because it never talks to LDAP at all.

| Setting | Test directory (this post) | Typical real OpenLDAP | Typical Active Directory |
|---|---|---|---|
| `host` | `openldap:10389` | `ldap.yourco.internal:636` | `dc01.yourco.internal:636` |
| `insecureNoSSL` | `true` | `false` | `false` |
| TLS | none | `rootCAData` with your CA, or `startTLS: true` on 389 | same |
| `bindDN` | `cn=admin,dc=planetexpress,dc=com` | a dedicated read-only service account, never `cn=admin` | a service account with just "read" AD rights |
| `userSearch.baseDN` | `ou=people,dc=planetexpress,dc=com` | your people OU | your Users OU or a narrower sub-OU |
| `userSearch.filter` | `(objectClass=inetOrgPerson)` | `(objectClass=inetOrgPerson)` | `(&(objectClass=person)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))` — excludes disabled accounts |
| `userSearch.username` / `idAttr` | `uid` | `uid` | `sAMAccountName` |
| `groupSearch.filter` | `(objectClass=Group)` | `(objectClass=groupOfNames)` | `(objectClass=group)` |

A production-shaped LDAPS block looks like this:

```yaml
host: ldap.yourco.internal:636
insecureNoSSL: false
insecureSkipVerify: false
rootCAData: "<base64 of your CA cert>"
```

Get that value with:

```bash
base64 -w 0 your-ca.crt
```

Never run `insecureNoSSL: true` or `insecureSkipVerify: true` against a directory holding real credentials — that combination sends every user's password to your LDAP server in clear text, and skips validating that the server on the other end is actually the one you think it is.

The [Dex LDAP connector reference](https://dexidp.io/docs/connectors/ldap/) documents every field, including Kerberos/SPNEGO single sign-on if you want to skip the login form entirely for domain-joined machines.

---

## 6. Hardening before this goes anywhere real

- **Never bind as a domain admin or `cn=admin`.** Create a dedicated service account for `bindDN` with read-only rights, scoped to the OU it needs to search.
- **Turn on TLS.** `insecureNoSSL: true` and `insecureSkipVerify: true` exist for exactly the disposable-test-container situation in this post. Treat them as demo flags, not defaults.
- **Rotate `staticClients[].secret` per environment**, and don't commit real ones to git — the `flask-demo-secret` above is deliberately obvious.
- **Use `sqlite3`/`postgres` storage, not `memory`,** for any Dex you expect to survive a restart with refresh tokens intact.
- **Put Dex behind HTTPS** in anything beyond localhost — the `issuer` value is part of what gets checked when a client validates a token, and `http://` issuers are a footgun the moment this leaves your laptop.
- **Scope `groupSearch` narrowly.** A wide-open group search leaks group membership of your whole directory into every client that asks for the `groups` scope.

---

## Wrapping up

The value of Dex here isn't really "LDAP now has a web login" — LDAP already had one. It's that the *app* stops caring where identity comes from. The Flask app in this post is thirty lines, doesn't import an LDAP library, and would work completely unchanged if you deleted the `ldap` connector tomorrow and replaced it with `google`, `github`, or `saml`. That's the trade Dex is making: a small broker in the middle, in exchange for every application behind it speaking one protocol, forever.

Everything in this post — the compose file, the connector config, the Flask app — is exactly what I ran to verify it, against a real LDAP bind, not just documentation.

Questions or corrections, reach me at [vijay@mevijay.com](mailto:vijay@mevijay.com) or on [GitHub](https://github.com/sharmavijay86).

---

### References

- [Dex](https://dexidp.io/) — project site and docs
- [Dex LDAP connector reference](https://dexidp.io/docs/connectors/ldap/)
- [dexidp/dex](https://github.com/dexidp/dex) — source, `examples/ldap/`, `config.yaml.dist`
- [rroemhild/docker-test-openldap](https://github.com/rroemhild/docker-test-openldap) — the disposable test directory used in this post
- [Authlib Flask OAuth client](https://docs.authlib.org/en/latest/client/flask.html)
