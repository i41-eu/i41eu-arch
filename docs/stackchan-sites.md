# Stackchan online: a manager and separate apps

**Status:** live: the pages on s.i41.eu, the manager on m.i41.eu, each app on `<app>.q41.eu`
(raw.q41.eu, pet.q41.eu, ...).

## 1. Hostnames

`s.i41.eu`, `m.i41.eu` and the apps' `<app>.q41.eu` are Stackchan's. The apps have a domain
of their own, so browsers keep them apart from the pages and the manager (cookies).

| Host | What |
|---|---|
| `s.i41.eu` | index and project page (static) |
| `m.i41.eu` | **the Stackchan manager** online (private code; the home manager [i41eu-manager](https://github.com/i41-eu/i41eu-manager) is its public part): sign-in, your robots, the apps you approve for them, one-click setup over USB, their tokens, the official firmware |
| `raw.q41.eu` | the raw dashboard ([q41eu-raw](https://github.com/i41-eu/q41eu-raw)): every sensor and command, an app like any other |
| `pet.q41.eu` | the pet ([q41eu-pet](https://github.com/i41-eu/q41eu-pet)) |
| `dev.q41.eu` | Dev ([q41eu-dev](https://github.com/i41-eu/q41eu-dev)): the robot for programs (HTTP) and AI agents (MCP) with a key per robot; runs at home, not deployed yet |
| `*.q41.eu` | more apps, one host each |

**Every app has its own host,** because the browser isolates by origin (host), not by path:
pages of one app cannot read another's storage (e.g. the end-to-end encryption keys), cookies
or sign-in, and cannot call its API as the user. One wildcard certificate covers `*.q41.eu`.

## 2. The manager sets up the apps you approve

1. You sign in on m.i41.eu (through Dex: GitHub, Google).
2. You pick the apps for your robot from the manager's catalog (raw, pet, …) and the one it
   starts with.
3. One click with the robot on USB: the manager installs the official firmware (if needed) and
   writes every approved app into the robot, each with **a token of its own** (`provision`
   with `servers` and `pin`); the robot asks on its screen before its start app is set.
4. The robot connects to its start app; the app switcher (the QR screen) switches between the approved apps.

Removing a robot (or an app, by setting it up again) on m.i41.eu revokes the tokens: an app
accepts a revoked token for at most a minute more (it caches the manager's answer), and a robot
connected right now stays until it reconnects.

## 3. Robot tokens: issued by the manager, checked by each app

- One token **per robot per app**; the manager keeps only the hash.
- An app checks a robot's token when the robot connects: it asks the manager
  (`POST m.i41.eu/api/robot-auth {app, robot, token}`, authenticated as that app) and gets the
  robot's owner and whether it is private or public. The answer is cached for a few minutes.
- Apps hold no robot tokens and no account database. Later the manager signs the tokens and apps
  verify them offline ([trust design](implementations/stackchan-trust.md)).

## 4. People, pairing, privacy

- One sign-in for all sites: each site is its own client of Dex, and an account is the
  same everywhere (the manager's user id, which may have several sign-ins: [accounts](accounts.md)).
- A robot is private to its owner by default in every app: the app learns the owner from the
  manager and pairs only them (and whom they allow). End-to-end encryption stays per app.
- Tiers and limits ([accounts](accounts.md)) apply per app, decided by the manager.

## 5. Steps

1. **Done: DNS and certificates:** `s.i41.eu`, `m.i41.eu`, wildcard `*.q41.eu` (GitOps).
2. **Done: the manager and the raw app.** The manager became a repo of its own (robots, the app
   catalog, setup, firmware, `robot-auth`; later split into the public home manager,
   [i41eu-manager](https://github.com/i41-eu/i41eu-manager), and m.i41.eu's private part with
   sign-in and users); q41eu-raw kept the raw app (robot connections,
   pairing, the dashboard, the end-to-end relay), which checks tokens with the manager.
   Deployed as m.i41.eu and raw.q41.eu.
3. **Done: pet.q41.eu:** the pet checks tokens with the manager too, with the Go package
   both apps share (`github.com/i41-eu/q41eu-raw/robotauth`).
4. **Done: s.i41.eu:** the index page.
5. **dev.q41.eu:** Dev exists (q41eu-dev, at home); online it needs a catalog entry at the
   manager, sign-in through it, and its state in Postgres.

## 6. Open questions

1. **Done: apps added later:** the owner changes a robot's apps on the manager's page (if
   allowed at the USB setup); the manager sends the signed list on the robot's own connection to
   it ([manager-channel.md](manager-channel.md)), and the robot checks the signature.
2. **Tiers across apps:** the manager answers per account, or the apps read the same tiers file.
3. **Home servers:** a home keeps its own servers and [app catalog](app-catalog.md); the public
   manager is for the public apps. Should a home server be able to use m.i41.eu for setup?
4. **The pet in public:** pet.q41.eu is live, with pairing by each robot's QR code and a
   parent PIN instead of sign-in. Its leaderboard photos are stored on the public server and
   served only to browsers paired with that robot. Is that enough?
