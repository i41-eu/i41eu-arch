# Security: the aim, the state today, the backlog

**Status:** 2026-10-13. **We aim for i41 to be secure, but there is a known backlog, and nothing has been independently reviewed or verified yet.** The code is written with AI agents and has not been
reviewed by humans. This page says what you can do
now, what we aim for, what is known to be missing, and what was fixed; it changes as items are
fixed. To report a problem privately: [SECURITY.md](https://github.com/i41-eu/.github/blob/main/SECURITY.md).

## 1. What you can do now

- **Keep the robot and its backup files private.** Anyone with the robot and a USB cable, or
  with a backup file the setup saved, can read your Wi-Fi password and the robot's tokens (§4.1).
- **If a robot is lost, change your Wi-Fi password.**
- **Run the home manager only on a home network whose devices and guests you trust** (§4.5).
- **Sign phones in to the home manager only on your own network, and sign them out when you are
  done:** a phone's sign-in does not end on its own (§4.6).
- **For the camera and the microphone, use an app with end-to-end encryption** (Raw data at
  raw.q41.eu has it; at home, mark Raw data `"e2e": true` in the manager's catalog). Without it,
  the app's server sees what the robot sends.
- **Close the manager's page when a setup is done** (§4.13).

## 2. The aim

We aim for i41 to be secure and private by design:

- **Local first.** The robot, the home manager and the home's apps work on the home network;
  m.i41.eu and the apps on q41.eu help set robots up and reach them, and are optional
  ([principles](principles.md)).
- **Each app on an address of its own**, so the browser keeps the apps apart, and a token per
  robot per app: an app gets only what the robot's owner approved for it
  ([Stackchan sites](stackchan-sites.md)).
- **End-to-end encryption** for what the robot sees and hears when it goes through a relay: the
  relay carries only ciphertext ([e2ee](e2ee.md)); the browser's side still runs in a page the
  relay sends (§4.7).
- **Releases anyone can check:** the firmware builds reproducibly, the owner rebuilds each
  release and signs its hashes, and anyone can rebuild it ([device setup](device-setup.md) §6,
  [independent rebuild](independent-rebuild.md)); not all of it is in place yet (§4.14).

That is the aim, not a promise: i41 is a proof of concept, written with AI agents and tested at
home, and there is a known backlog (§4).

## 3. Independent review: none yet

The code has not been reviewed by humans, and no one outside the project has reviewed the
design; there was no audit and no penetration test. The reproducible builds (§2) check that a
release was built from its source, not that the source is safe, and the rebuilds are ours too
(GitHub Actions, the owner's laptop, the project's own cloud machine:
[independent rebuild](independent-rebuild.md)); no one outside the project has rebuilt a release
yet.

We would welcome a review of these, the most useful first:

1. **The end-to-end encryption:** the design ([e2ee](e2ee.md)) and its code (q41eu-raw's `e2e`
   package, the dashboard's `e2e.js`, Embody Mode's `e2e.cpp`).
2. **Sign-in and rights:** m.i41.eu's one sign-in for every app (q41eu-raw's `sso` package),
   and how the home manager decides who is the owner (the computer it runs on, or a phone
   signed in at a robot).
3. **The setup over USB:** what a page holding the robot's USB port can do, and what the robot
   confirms itself ([device setup](device-setup.md) §2, §3.3).
4. **The firmware:** its network code and what it stores.
5. **The release process:** builds, hashes, signatures.

Please send findings privately ([SECURITY.md](https://github.com/i41-eu/.github/blob/main/SECURITY.md)).

## 4. The backlog: known limits

These are limits of the public design and code; we list them so you can protect yourself (§1).
A bug that can be exploited and is not public yet stays private until a fix is released (§6).
Each item: what, where, why it matters, and what would fix it.

### 4.1 The robot's flash can be read

- **What:** the robot keeps the Wi-Fi password, its apps' and managers' tokens and its
  end-to-end key in its flash unencrypted; flash encryption and secure boot are off.
- **Where:** the firmware ([i41eu-StackChan](https://github.com/i41-eu/i41eu-StackChan)
  `firmware/sdkconfig.defaults*`); NVS namespaces `wifi`, `embody` and `embody_e2e`.
- **Why it matters:** anyone with the robot and a USB cable can read them, and the firmware
  backup the setup page saves to your computer holds them too.
- **Fix:** flash encryption, NVS encryption and secure boot, opt-in for each robot, because they
  burn the chip's eFuses for good ([device setup](device-setup.md) §6.6). Until then, keep the
  robot and its backup files private, and change the Wi-Fi password if a robot is lost.

### 4.2 A shared robot token is not tied to one robot

- **What:** a server may give all of its own robots one shared token (a server's own robots, or
  a home app set up with a token file); m.i41.eu gives each robot a token of its own per app.
- **Where:** q41eu-raw and the apps that check tokens the same way; the home manager's catalog
  (`token_file`).
- **Why it matters:** a shared token works for any robot id on that server, so whoever reads it
  from one robot (§4.1) can act as any of that server's robots; only a new token on that server
  takes it back.
- **Fix:** no shared tokens on public servers; a token per robot at home too.

### 4.3 You trust the server that sends the page

- **What:** the setup page is whatever the manager's server sends, and an app's page is whatever
  the app's server sends. Setting a robot up at m.i41.eu, you trust that server with what you
  type (the Wi-Fi password) and with the robot's USB port, which can also install firmware
  ([device setup](device-setup.md) §2).
- **Where:** m.i41.eu's manager (not public), [i41eu-manager](https://github.com/i41-eu/i41eu-manager),
  the apps on q41.eu.
- **Why it matters:** a compromised m.i41.eu could serve a page that leaks what you type or sets
  the robot up as it likes. End-to-end encryption does not help against a page from the server
  you use (§4.7).
- **Fix:** the firmware installer as a static page published with each release
  ([device setup](device-setup.md) §3.2). The home manager's setup page comes from code you built
  or downloaded yourself.

### 4.4 The pages set no Content-Security-Policy

- **What:** the managers' main page (where robots are set up) and the apps' pages send no
  Content-Security-Policy (s.i41.eu does).
- **Where:** m.i41.eu's manager (not public), [i41eu-manager](https://github.com/i41-eu/i41eu-manager),
  the apps on q41.eu.
- **Why it matters:** without one, a single injection bug (a name or a text shown on the page)
  could let a script read what you type or use the robot's USB port.
- **Fix:** a strict Content-Security-Policy on every page.

### 4.5 The home manager talks plain HTTP and WebSocket on the home network

- **What:** the home manager serves its page on port 8790 over plain HTTP, and the robot's
  connection to it (the manager channel) is a plain `ws://` WebSocket on the home network. That
  channel carries the robot's tokens and the one-time code the robot's Manager screen shows for
  signing a phone in. The manager can also serve HTTPS (`-tls-listen`, with a self-signed
  certificate made once for the computer's host name and the addresses it had then), but phones
  still get the `http://` address unless `-page-url` (or an `https://` `-public-url`) gives the
  HTTPS one.
- **Where:** [i41eu-manager](https://github.com/i41-eu/i41eu-manager) (the page, the robot
  channel, the phone sign-in); [manager channel](manager-channel.md) §6.
- **Why it matters:** anyone who can read your home network's traffic can take over the home
  manager and the robots it manages.
- **Fix:** HTTPS for the page by default, and a channel that no longer carries usable codes and
  tokens in clear (an encrypted channel whose certificate the USB setup pins, or codes the robot
  makes and the manager only checks).

### 4.6 Phone sign-ins at the home manager do not end

- **What:** a phone signs in to the home manager by scanning the code on a robot's Manager screen.
  The manager never ends that sign-in: only the phone signing out does (its cookie lasts a year).
  It has the owner's rights over the robots (since i41eu-manager v0.1.1 not the setup and not the
  link, §5), and the page cannot list or sign out phones.
- **Where:** [i41eu-manager](https://github.com/i41-eu/i41eu-manager) (`internal/manager`: the
  phone sessions and the owner check).
- **Why it matters:** anyone who once signed in at a robot (a visitor, a babysitter), or got a code
  or a cookie (§4.5), stays the owner until the manager's state is reset; a lost phone stays signed
  in.
- **Fix:** phone sign-ins that expire, a list of them on the computer's page with Remove, each tied
  to the robot it signed in at.

### 4.7 End-to-end encryption: what it does not cover yet

- **What:** the design in [e2ee](e2ee.md) has limits today:
  - the relay also serves the page and its encryption code, so the encryption holds only while
    the relay serves the published code;
  - each session numbers its messages under a 32-bit random prefix with a long-lived key, so over
    very long use two sessions may repeat a nonce, which weakens AES-GCM;
  - a browser does not check that what the robot sends is new, so a relay could send old frames
    again;
  - removing a lost phone from the robot relies on the app reporting which browser it is.
- **Where:** q41eu-raw (the relay, the dashboard's `e2e.js`), Embody Mode's `e2e.cpp`.
- **Why it matters:** the relay is meant to be untrusted, and these limits leave it trusted in
  part.
- **Fix:** the page from a separate origin pinned by hash; 96-bit random nonces or per-session
  keys; a replay check in the browser; a way to forget browsers on the robot itself (e2ee §8).

### 4.8 The home manager keeps sign-ins and tokens unencrypted in its state

- **What:** the home manager keeps phone sign-ins, its link token to m.i41.eu and the robots'
  m.i41.eu app tokens in its state (a file, or a database) as they are.
- **Where:** [i41eu-manager](https://github.com/i41-eu/i41eu-manager) (its state file).
- **Why it matters:** whoever can read that file or database can act as a signed-in owner, as the
  home towards m.i41.eu, or as its robots.
- **Fix:** store only hashes of sign-ins, and keep tokens only until the robot has them (or encrypt
  them).

### 4.9 Release firmware lets an app restart the robot or start other firmware apps

- **What:** the release build has the robot's automation commands on: any app may restart the
  robot or start another app of the firmware's launcher (with end-to-end encryption, through its
  page in an enrolled browser).
- **Where:** [i41eu-StackChan](https://github.com/i41-eu/i41eu-StackChan)
  `firmware/sdkconfig.defaults.release` (`CONFIG_STACKCHAN_EMBODY_AUTOMATION`).
- **Why it matters:** the firmware's other apps do not apply Embody Mode's camera and microphone
  setting or its questions on the screen.
- **Fix:** off in release builds, or only from the robot's manager, or with a question on the
  robot's screen.

### 4.10 A USB setup can change the robot's manager without a tap on the robot

- **What:** the robot asks on its screen only when a setup changes the app it starts with; a new
  manager, given over USB, is taken without a question.
- **Where:** the firmware's USB setup (`usb_setup.cpp`); [device setup](device-setup.md) §3.3.
- **Why it matters:** whoever holds the robot's USB port can give it another manager (and, without
  secure boot, other firmware anyway: §4.1, §4.13).
- **Fix:** a question on the robot's screen when its manager changes.

### 4.11 The robot does not check certificate dates

- **What:** the robot checks a server's certificate chain and host name, but not its dates.
- **Where:** the firmware's TLS settings (`CONFIG_MBEDTLS_HAVE_TIME_DATE` off).
- **Why it matters:** an expired certificate is still accepted, so the key of an old certificate
  keeps working against robots.
- **Fix:** check the dates once the robot's clock is set (it has network time and a clock), or pin
  the servers' keys.

### 4.12 The Bluetooth Wi-Fi setup logs the password (M5Stack's code)

- **What:** the Bluetooth Wi-Fi setup in M5Stack's firmware writes the Wi-Fi name and password to
  the serial log.
- **Where:** `handle_set_wifi` in
  [`firmware/main/hal/hal_ble.cpp`](https://github.com/i41-eu/i41eu-StackChan/blob/a4646d7/firmware/main/hal/hal_ble.cpp#L467)
  (M5Stack's code, kept in the fork).
- **Why it matters:** little today: whoever reads the robot's USB console while its Wi-Fi is set
  over Bluetooth sees the password, but someone with the cable can read it from the flash anyway
  (§4.1). It matters once the flash is encrypted. Embody Mode's setup over USB does not use that
  path, and the SmartConfig log line in the hotspot library is never reached: our firmware never
  starts SmartConfig.
- **Fix:** take the password out of that log line in the fork, and tell M5Stack.

### 4.13 The browser keeps the manager's page's access to the robot's USB port

- **What:** after a setup, the browser (Chrome or Edge) remembers that the manager's page may use
  the robot's USB port, and while the page is open it looks for the robot when the page opens and
  whenever the robot is plugged in.
- **Where:** the setup page of [i41eu-manager](https://github.com/i41-eu/i41eu-manager)
  (`internal/manager/ui/index.html`: the page's `connect` listener and its check at load;
  `usb.js`) and m.i41.eu's setup page: they never call `port.forget()`.
- **Why it matters:** only the manager's own page gets the port again without asking, and it can
  do anything to the robot over USB ([device setup](device-setup.md) §2); that matters if the page
  is not honest (§4.3).
- **Fix (planned):** a setup option "let this site use the robot's USB only for this setup", on
  by default at m.i41.eu: the page forgets the port when the setup ends, and the browser asks
  again next time.

### 4.14 Releases are published before the owner's approval, and not checked by the managers

- **What:** the release workflow publishes a firmware release on its tag; the owner rebuilds it
  and signs its hashes afterwards, and the managers install the latest release without checking
  that signature.
- **Where:** [device setup](device-setup.md) §6.1, §6.2;
  [mj41cz-approved](https://gitlab.com/mj41cz/mj41cz-approved).
- **Why it matters:** a release that someone else managed to publish would be installed before
  anyone noticed.
- **Fix:** publishing only after the owner's approval, and the managers checking the owner's
  signature before installing (device setup §6.1, §6.2).

## 5. Fixed

| Date | What was wrong | Where | Fixed in |
|---|---|---|---|
| 2026-10-13 | A phone signed in at a robot could set robots up again (getting their tokens) and start, change or end the home's link to m.i41.eu; now only the page on the computer the manager runs on can | the home manager | [i41eu-manager](https://github.com/i41-eu/i41eu-manager) v0.1.1 |
| 2026-10-13 | The home manager gave the computer's Wi-Fi password to phones signed in at a robot; now only the page on the computer itself gets it | the home manager | [i41eu-manager](https://github.com/i41-eu/i41eu-manager) v0.1.0. m.i41.eu was not affected |

## 6. Reporting

Report problems privately: in the repository concerned, **Security** → **Report a
vulnerability** ([SECURITY.md](https://github.com/i41-eu/.github/blob/main/SECURITY.md)). A bug
that can be exploited stays in a private advisory until a fix is released; then it is listed in
§5.
