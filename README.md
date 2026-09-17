# azvpn-cli

Headless Azure point-to-site VPN client for **Entra-authenticated** gateways.
Python 3 stdlib driving `openvpn3` — no pip packages, no GUI, **no sudo**.

**Linux only.** See [Why openvpn3](#why-openvpn3-and-not-openvpn).

> **Unaffiliated** with Microsoft or OpenVPN Inc. Not endorsed by either.
> "Azure", "Entra" and "OpenVPN" are trademarks of their respective owners.
> The protocol details here were derived from Azure's own published VPN profile
> format and from the *retired* Azure VPN Client for Linux, for
> interoperability. Use at your own risk.

**Nothing organisation-specific is embedded.** You supply your own gateway
profile with `--profile`. Safe to share as-is.

## Contents

- [Why](#why)
- [How it works](#how-it-works)
- [Requirements](#requirements)
- [Usage](#usage)
- [Files it writes](#files-it-writes)
- [Why openvpn3 and not openvpn](#why-openvpn3-and-not-openvpn)
- [Split DNS](#split-dns)
- [Verified / not verified](#verified--not-verified)
- [Gotchas](#gotchas)
- [Troubleshooting](#troubleshooting)
- [Scope and contributions](#scope-and-contributions)

## Why

The Azure VPN Client hardcodes its OAuth **`client_id`** in a per-cloud table
and ignores the profile's `<audience>`. If your gateway sits behind a **custom**
Entra app, the stock client is refused:

```
AADSTS650057: Invalid resource. The client has requested access to a resource
which is not listed in the requested permissions...
```

…because Microsoft's first-party app is not permitted to request your custom
resource. On Linux that hardcoded id can be byte-patched. **On macOS the App
Store build is signed and cannot be** — so for Mac users a CLI is not a
convenience, it's the only option.

`azvpn-cli` avoids the problem entirely: it does the OAuth itself, using your
gateway's own `<audience>` as the `client_id`.

It also **outlives the client's 2026-08-31 retirement**, because it speaks to
the gateway directly rather than through a vendor binary.

## How it works

Azure P2S with Entra auth is plain OpenVPN where the **password is an Entra
access token**. Everything else comes out of the profile XML:

| Profile element | Becomes |
|---|---|
| `<fqdn>` | `remote <fqdn> 443 tcp-client` |
| `<serversecret>` | 512 hex chars = OpenVPN Static key V1 → `<tls-auth>` |
| `<audience>` | OAuth `client_id`, and scope `<audience>/.default` |
| `<tenant>` | authority |
| `<dnsserver>` / `<dnssuffix>` | split DNS via the up/down hook |

The OAuth flow mirrors the GUI client exactly — parameters read out of its
binary: v2 endpoint, PKCE, loopback redirect on `http://localhost:2023`, scope
`<audience>/.default offline_access`, `prompt=select_account`. Refresh tokens
are cached, so only the first connect opens a browser.

## Requirements

- Python 3.9+
- **`openvpn3`** (openvpn3-linux) — required, not optional. It is not in the
  Ubuntu/Debian archives; add OpenVPN Inc.'s repository first.
  Install guide: <https://openvpn.net/as-docs/openvpn3-linux.html>
- No sudo — openvpn3 runs the tunnel through its own D-Bus service
- The gateway root CA. The bundled `DigiCert_Global_Root_CA.crt` is what Azure
  gateways chain to. Override with `--ca` if yours differs.
- Your gateway's **`AzureVPN/azurevpnconfig.xml`** — Azure portal → *Virtual
  network gateway* → *Point-to-site configuration* → *Download VPN client*,
  then unzip.

  > Use the file from the `AzureVPN/` folder, **not** `Generic/VpnSettings.xml`
  > — the Generic one has no `<serversecret>`, so no tls-auth key.

## Usage

```bash
azvpn-cli up      --profile ~/vpn/eu.xml     # sign in + connect
azvpn-cli status  --profile ~/vpn/eu.xml
azvpn-cli logs -f --profile ~/vpn/eu.xml
azvpn-cli down    --profile ~/vpn/eu.xml
```

To skip `--profile` every time, set `AZVPN_PROFILE=~/vpn/eu.xml` or drop the
file at `~/.config/azvpn-cli/profile.xml`.

**Several tunnels?** One XML per gateway, anywhere you like — point `--profile`
at whichever you want. The connection name comes from the profile's `<name>`,
so tokens, configs, logs and pidfiles are kept separate automatically. No
registry, no import step.

```bash
azvpn-cli up --profile ~/vpn/eu.xml
azvpn-cli up --profile ~/vpn/apac.xml
```

All commands:

| Command | Does |
|---|---|
| `login` | Acquire/refresh the token. `--force-login` ignores the cache |
| `logout` | Drop the cached token |
| `gen` | Render the `.ovpn` and stop (inspection / use with your own runner) |
| `up` | login + gen + start openvpn as a daemon |
| `down` | Stop the tunnel |
| `status` | Profile, token, tunnel, interface, route count |
| `logs [-f]` | Tail the openvpn log |

Other flags: `--name` (override the connection name), `--ca`, `--verify-name`.

## Files it writes

| Path | Mode | Contents |
|---|---|---|
| `~/.config/azvpn-cli/<name>.ovpn` | 0600 | **contains the tls-auth key — do not share** |
| `~/.cache/azvpn-cli/token-<audience>.json` | 0600 | access + refresh token |

The access token is never written to disk as a credential file — it is piped to
`openvpn3 session-start` on stdin. Session state, logs and the tun device are
owned by openvpn3, not by this tool (`openvpn3 sessions-list`).

Share **the tool**, never the generated `.ovpn` or anything under `~/.cache`.

> `.gitignore` in this repo excludes `*.ovpn`, `azurevpnconfig*.xml` and token
> files, because all three carry the gateway's tls-auth key or your credentials.
> Check before you commit anyway.

## Why openvpn3 and not openvpn

Stock `openvpn` 2.x **cannot** connect to an Entra-authenticated Azure gateway.
It packs random material, options, `IV_*` peer-info, username and password into
a single **2048-byte** control-channel buffer, and an Entra access token is
~2.1 kB, so it fails before sending anything:

```
TLS Error: Key Method #2 write failed
```

That is client-side serialisation, not a rejection. Ruled out by direct tests:

| Attempt | Result |
|---|---|
| default settings | `Key Method #2 write failed` |
| `--max-packet-size 2048` (documented maximum) | still fails |
| 1-character username instead of the 30-char UPN | still fails |
| short dummy password | write succeeds → confirms length is the cause |

The token is not trimmable either — no `groups` claim, payload 1620 B and
signature 341 B are ordinary. `--max-packet-size` caps control *packet* size
(154–2048); it does not enlarge that plaintext buffer.

**OpenVPN 3 Core has no such ceiling** and carries the same token fine. It is
also the lineage Microsoft's own Azure VPN Client is built on. Hence openvpn3
is required, not preferred.

`openvpn3-linux` is **Linux only**. There is no openvpn3 CLI for macOS, and
stock `openvpn` there hits the identical 2048-byte wall — so macOS has no
supported path through this tool yet.

## Split DNS

Azure pushes DNS **servers** but no **domains** — the suffixes exist only in the
profile XML, which openvpn3 never sees. So `--dns-scope tunnel` on its own
leaves zero routing domains and private zones silently resolve via public DNS
(you get the public endpoint IP and an HTTP 403 instead of the private one).

`azvpn-cli` fixes this by emitting each `<dnssuffix>` as a client-side
`dhcp-option DOMAIN` in the generated config, then setting
`--dns-scope tunnel`. Result — matching the GUI client:

```
DNS Servers: 10.x.x.4 10.x.x.5
DNS Domain:  blob.core.windows.net vaultcore.azure.net …   (routing-only)
```

Only those suffixes go to the VPN resolver; the rest of your DNS is untouched.

## Verified / not verified

Verified on Ubuntu 22.04 against a live Azure gateway:

```
TLS: Initial packet from <gw>:443, sid=…              <- tls-auth key correct
VERIFY OK: depth=2, CN=DigiCert Global Root CA
VERIFY OK: depth=1, CN=DigiCert SHA2 Secure Server CA
VERIFY EKU OK
VERIFY X509NAME OK: CN=<vnet-id>.vpn.azure.com
VERIFY OK: depth=0, …
```

- ✅ profile parsing, `.ovpn` generation, tls-auth key derivation
- ✅ certificate chain and cert-CN pinning against the real gateway
- ✅ OAuth end to end — PKCE loopback sign-in, token cached, silent refresh
- ✅ **connect end to end** via openvpn3: a pushed `10.x.x.x/25` address, 65 routes,
  `Client connected`
- ✅ split DNS — private endpoint resolves to its `10.x` address while public
  names keep using normal DNS
- ✅ L7 — authenticated Azure Storage download through a private endpoint
- ✅ `up` / `down` / `status` / `logs` idempotent, teardown removes `tun0`
- ❌ **macOS: not supported.** openvpn3-linux is Linux only.

## Gotchas

- **The gateway cert CN is not the connection FQDN.** You connect to
  `azuregateway-<vnet-id>-<instance>.vpn.azure.com`, but the certificate is
  issued to `<vnet-id>.vpn.azure.com`. `azvpn-cli` derives the CN from the
  FQDN; `--verify-name` overrides it. Pinning the FQDN fails with
  `VERIFY X509NAME ERROR`.
- **The `<instance>` suffix changes when Azure reprovisions the gateway.**
  Re-download the profile if the FQDN stops resolving. The vnet id is stable,
  which is why the cert CN is pinned to that instead.
- **Port 2023 must be free during sign-in** — the Azure VPN Client GUI uses the
  same loopback redirect. Quit it first.
- **`http://localhost:2023` must be a registered redirect URI** on the Entra app
  registration. It is for the GUI client; if your custom app was registered
  without it, sign-in fails with `AADSTS50011` and an Entra admin has to add it.
- **openvpn caps username/password length.** Entra access tokens are large; if
  yours is rejected on length, that limit is the reason.
- **The gateway is silent on an idle tunnel, and that used to reconnect it every
  53 seconds.** It pushes no `keepalive` and answers nothing, not even the
  client's own pings: `openvpn3 session-stats` shows `PACKETS_OUT` climbing
  while `PACKETS_IN` sits still. OpenVPN 3 Core's built-in 50-second receive
  timeout then fires (`Session invalidated: KEEPALIVE_TIMEOUT`), the session
  restarts 2 s later, and the desktop notifier pops "Session is reconnecting"
  about 68 times an hour. The generated profile ships `ping-restart 0` for that
  reason; raising the value only changes the interval. The trade-off: with no
  restart timer, a path that dies silently (suspend, Wi-Fi switch) is left to
  TCP to notice, which is untested here. Re-run `azvpn-cli up` if the tunnel
  goes quiet.
- **The access token is handed to openvpn3 once, at `session-start`.** openvpn3
  caches it and replays it on every reconnect, so once the token expires (about
  70 minutes) any reconnect fails and openvpn3 retries every 5 s forever while
  still reporting the session UP. `azvpn-cli status` shows `token : EXPIRED`
  with `tunnel : UP — Connection, Client reconnect`. Fix: `azvpn-cli up`.
- **Don't run this and the GUI client at the same time** against the same
  gateway — two tunnels, conflicting routes.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `AADSTS650057` | wrong `client_id` | Shouldn't happen here — the audience *is* the client_id. Check `<audience>` in the profile |
| `AADSTS50011` redirect mismatch | `http://localhost:2023` not registered | Entra admin adds it to the app registration |
| `cannot bind http://localhost:2023` | GUI client running | Quit the Azure VPN Client |
| `VERIFY X509NAME ERROR` | CN pin wrong | `--verify-name <cn from the error message>` |
| `VERIFY ERROR: unable to get local issuer` | root CA missing | pass `--ca`, or add the root to the OS trust store |
| Silent hang, no TLS handshake | tls-auth key wrong | Check `<serversecret>` is 512 hex chars |
| `Key Method #2 write failed` | you are on stock `openvpn`, not `openvpn3` | install openvpn3 — see [Why openvpn3](#why-openvpn3-and-not-openvpn) |
| `AUTH_FAILED` | token rejected | `azvpn-cli login --force-login` |
| "Session is reconnecting" notification every ~53 s | gateway sends nothing when idle, core keepalive timeout fires | already handled by `ping-restart 0` in the generated profile; check it survived a re-`gen` |
| Endless reconnect every 5 s, `status` says `token : EXPIRED` | access token expired mid-session; openvpn3 replays the cached credential | `azvpn-cli up` (silent refresh) |
| Connects, private names resolve to public IPs | routing domains missing | `resolvectl status tun0` — expect the profile's suffixes under `DNS Domain`; if absent, re-run `azvpn-cli gen` |
| `Maximum option line length (256) exceeded` | a long generated line | Report it — the suffix list is already externalised |

## Scope and contributions

Built to solve one problem: connecting Linux to an Azure point-to-site gateway
that uses **a custom Entra app as its audience**, where the official client
refuses with `AADSTS650057`. It works for standard first-party-audience
gateways too, but that case is already served by the official client.

Issues and PRs welcome, particularly:

- macOS support — needs an OpenVPN 3 Core CLI; stock `openvpn` cannot carry the
  token (see [Why openvpn3](#why-openvpn3-and-not-openvpn))
- Gateways whose profile shape differs from the ones tested here
- Certificate-authenticated (non-Entra) profiles, which are not handled at all
