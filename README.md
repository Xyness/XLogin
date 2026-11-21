<p align="center">
  <img src="https://celestis.dev/banners/xlogin.png" alt="XLogin">
</p>

<p align="center">
  <b>Login and registration for offline-mode servers, where the players who own the game connect without ever typing a password.</b>
</p>

<p align="center">
  <a href="https://builtbybit.com/resources/xlogin.103391/"><img src="https://img.shields.io/badge/version-2.0-6d9dd8?style=flat-square&labelColor=24292f" alt="Version 2.0"></a>
  <img src="https://img.shields.io/badge/Paper-1.21.1%2B-57606a?style=flat-square&labelColor=24292f" alt="Paper 1.21.1+">
  <img src="https://img.shields.io/badge/Folia-supported-57606a?style=flat-square&labelColor=24292f" alt="Folia supported">
  <img src="https://img.shields.io/badge/Java-21%2B-57606a?style=flat-square&labelColor=24292f" alt="Java 21+">
  <a href="https://github.com/Xyness/XCore"><img src="https://img.shields.io/badge/requires-XCore%202.0-4fb3a0?style=flat-square&labelColor=24292f" alt="Requires XCore 2.0"></a>
</p>

<p align="center">
  <a href="https://builtbybit.com/resources/xlogin.103391/"><b>Get it on BuiltByBit</b></a>&nbsp;&nbsp;&middot;&nbsp;
  <a href="https://celestis.dev/plugins/xlogin/wiki/getting-started">Documentation</a>&nbsp;&nbsp;&middot;&nbsp;
  <a href="https://discord.com/invite/6sRTGprM95">Discord</a>
</p>

<br>

An offline-mode server has to ask everyone who they are, including the players who already own their account. XLogin runs the Mojang handshake itself, one player at a time, so the players who own the game can connect straight in with their own skin while cracked players register as usual. In strict mode, nobody takes a premium name that isn't theirs.

Passwords are hashed with PBKDF2 at 210 000 iterations, brute force runs into limits per address and per account, and one login covers the whole network, Bedrock players included. If XLogin cannot run, the server stays shut: on an offline-mode server there is nothing else between a name and whoever types it.

<p align="center">
  <a href="#what-it-does">What it does</a>&nbsp;&middot;
  <a href="#quick-start">Quick start</a>&nbsp;&middot;
  <a href="#on-a-network">On a network</a>&nbsp;&middot;
  <a href="#commands-and-permissions">Commands</a>&nbsp;&middot;
  <a href="#configuration">Configuration</a>&nbsp;&middot;
  <a href="#dashboard-and-integrations">Dashboard</a>&nbsp;&middot;
  <a href="#under-the-hood">Under the hood</a>
</p>

<br>

## What it does

### Premium players never type a password

The server stays in offline mode. XLogin hooks into the login and runs the encryption handshake for the players who should have one: through the Netty pipeline on a standalone server, at the proxy on a network. A verified player keeps their Mojang skin, set on the profile before they spawn. The verified profile is bound to the connection that earned it and expires after 30 seconds, and the whole thing works behind a PROXY protocol front such as TCPShield, a tunnel or a load balancer.

| `premium.mode` | Who is verified | What it means |
|---|---|---|
| `opt-in` (default) | Players who ran `/premium` | Everybody registers first; premium players then turn auto-login on themselves. |
| `strict` | Any name that exists on Mojang and has no account here | Premium players never see `/register`, and a cracked client cannot take a premium name. A name that already has a password account is never challenged, since its owner may be a cracked player; a premium player in that position runs `/premium`. |

When Mojang's API cannot be reached, a new name is challenged anyway rather than sent to `/register`: a premium client answers the challenge whatever the API says. A cracked client cannot, sees `Invalid session` once, and takes the normal flow on its next connection. The console says why the lookup failed. A name Mojang confirmed as premium, or an account flagged premium, never gets that second chance.

`/premium` asks twice before it flags an account, because from then on only the Mojang owner of the name can open it, and a player on a cracked launcher would lock themselves out. It also warns when the switch gives the player their Mojang UUID (`uuid-mode: REAL`, or behind a proxy). `/unpremium` clears the account's link to its Mojang UUID.

> [!NOTE]
> Premium auto-login ships switched off. To turn it on, keep `online-mode=false` in `server.properties`, set `premium.enabled: true`, pick a `premium.mode` and restart. XLogin runs the handshake itself, and a server already in online mode would run a second one: the client is disconnected with `Tried to switch to AUTHORIZING from ENCRYPTING`. XLogin detects online mode and refuses to start auto-login rather than break every login. When a premium player is asked for a password anyway, set `debug: true`: every step of the handshake is logged as `[Premium]`.

`premium.uuid-mode` decides which UUID a verified player plays under:

| Mode | Behaviour |
|---|---|
| `OFFLINE` (default) | UUIDs computed from the name. Premium verification still runs for authentication. |
| `REAL` | Verified premium players get their Mojang UUID, set through Paper's `PlayerProfile` API during `AsyncPlayerPreLoginEvent`. Enables cosmetics on Lunar and Badlion. Cracked players keep offline UUIDs. |

Switching between offline and online UUIDs moves the account onto the new UUID: XCore recognises the player as they log in, and XLogin moves the account before they join, password, sessions and 2FA included. An account tied to a Mojang account only moves onto a connection Mojang has verified.

> [!WARNING]
> Pick the UUID mode before players have data. Changing `uuid-mode` later changes premium UUIDs, and while XCore, XLogin, XBans and XAuctionHouse move their data onto the new ones, what other plugins keep by UUID, inventories included, is orphaned.

<details>
<summary><b>The handshake, step by step</b></summary>

<br>

**On a standalone server**, XLogin injects a handler into the Netty pipeline:

1. The player connects to the offline-mode server.
2. The handler intercepts `LoginStart`. Bedrock players (Floodgate) are left alone.
3. XLogin checks whether the name is a premium account: the database flag first, then Mojang's API for a name with no account here (`strict` mode). Two official endpoints are tried and the answer is cached.
4. An `EncryptionRequest` goes to the client.
5. The client authenticates with Mojang and answers with `EncryptionResponse`.
6. XLogin decrypts the shared secret and enables AES/CFB8.
7. It verifies with Mojang's `hasJoined` endpoint.
8. On success the Mojang UUID and skin are kept and the login continues. The skin is set on the profile before the player spawns.
9. On join the account is created if needed and the player authenticated.

**Behind a proxy:**

1. The player reaches the proxy.
2. XLogin checks the database for `premium=1`, or asks Mojang in strict mode.
3. Online mode is forced for that connection and the proxy runs the handshake.
4. The verified profile is sent to the backend over Redis and plugin messaging.
5. The player is routed to the lobby.

</details>

<details>
<summary><b>Strict mode, in detail</b></summary>

<br>

- `premium.mode` lives in the backend config; the proxy carries the same choice as `premium-strict-mode`.
- `/register` asks Mojang again: a premium name is refused, and so is a registration while Mojang cannot answer (`register-name-unverified`). A connection whose check takes more than 15 seconds is refused. This is on purpose, since a name registered with a password is never challenged again; the price is that a server whose address Mojang throttles or blocks takes no new registrations for as long as that lasts, and the console warns about it at most once every five minutes.
- A verified account is flagged premium and verified on every connection.
- `/unpremium` is disabled.

</details>

<details>
<summary><b>Username changes</b></summary>

<br>

A premium player changing their name changes their offline UUID. XLogin migrates the account:

1. The player connects under the new name.
2. The handshake returns the same Mojang UUID.
3. XLogin finds the old account: same `mojang_uuid`, different offline UUID.
4. `player_uuid` is updated in `xlogin_accounts`, `xlogin_sessions` and `xlogin_2fa`, in one transaction.
5. A `MIGRATE` sync event notifies the other servers.

Password, premium flag, 2FA, recovery codes, email and sessions are kept. In `uuid-mode: REAL` the UUID never changes with the name; what moves there is an account that was registered with a password before its owner was verified, from its offline UUID onto the Mojang one.

</details>

### Hard to break into, harder to impersonate

| Someone tries to | What happens |
|---|---|
| Guess a password from one address | After 10 failures in 10 minutes the address is banned for 30 minutes. IPv6 addresses are counted, limited and banned per /64. |
| Guess it from many addresses | Past 10 failures on one account in the window, only the owner's last address is accepted. |
| Flood the server with bots | Six connections per address per minute, counted apart from failed logins. Switch on the captcha and `/register` also needs a code drawn on a map. |
| Use `password123` | Refused. Common passwords come from `passwords.txt` in `plugins/XLogin/`, a local list: nothing is sent anywhere. |
| Join as `steve` while `Steve` has an account | Refused, with the registered spelling in the message. |
| Kick you by joining under your name | Refused: a second connection under the name of a logged-in player is turned away. |
| Change your password from a session left open | Refused: changing the password, the email, the 2FA or the premium flag needs the password typed on this connection, or a premium or Bedrock auto-login. |
| Read passwords in the console | Masked by a Log4j filter. |

Passwords are hashed with PBKDF2-HMAC-SHA256, 210 000 iterations and a salt per account, stored as `pbkdf2$iterations$salt$hash`. Older accounts are re-hashed on their next login, with no reset, and changing or resetting a password closes every session on that account. A password can hold any printable character but a space, `%` and `$` included. A wrong password, 2FA code or recovery code tells the player where they stand:

> `Wrong password! (2/5 attempts)`

The spelling rule exists because an offline UUID is derived from the exact letters of a name: `steve` and `Steve` are two different players to the server and one name to everybody reading the chat. Premium players verified by Mojang and Bedrock players are not concerned, and a premium player is held to Mojang's spelling instead, with the right one in the message. On top of that come a limit of accounts per address (three by default), an optional IP lock, and name validation with a pattern and a list of blocked words.

An account created without a password (premium, Bedrock, imported) cannot be opened with one: `/login` says so, and an admin sets one with `/xlogin resetpassword`.

<details>
<summary><b>The map captcha</b></summary>

<br>

A short code drawn on a map, typed back with `/captcha` before `/register` is accepted. A person reads it without thinking; a headless bot cannot read it at all. Length (five characters by default) and the number of attempts are configurable, and ambiguous characters such as O and 0 are never used.

The map goes in an empty hotbar slot. Whatever it displaces moves to a free slot or, with a full inventory, is kept on the server side with the player's data, never inside the map, and comes back when the map goes, including after a crash. The map cannot be used on blocks or entities, and crafting with it gives nothing. Map ids are pooled and reused across restarts, so a bot flood does not leave thousands of maps in the world folder.

The captcha ships switched off: `captcha.enabled: true` turns it on.

</details>

### Two-factor and recovery

TOTP, so Google Authenticator, Authy and the rest. A player runs `/2fa setup`, adds the secret to their app and confirms with `/2fa <code>`; logins then ask for the password and the code. A code is valid for up to 90 seconds, and once accepted it cannot be replayed inside that window.

`/2fa codes` issues eight recovery codes, shown once and stored hashed. A player who loses their phone gets back in alone with `/2fa recover <code>`, without asking staff to switch 2FA off. A recovery code stands in for the 2FA code, never for the password: it is only accepted once the password has been typed. Wrong 2FA and recovery codes count towards `max-login-attempts`.

Password recovery by email is there too. `/email set <email>` mails a code to the address and `/email confirm <code>` links it; `/recover <username>` then sends a reset code of eight letters and digits, at most five mails per account per day, and a code is dropped after five wrong answers. Two-factor and email recovery both ship switched off (`two-factor.enabled`, `email-recovery.enabled`).

<details>
<summary><b>Setting up email recovery</b></summary>

<br>

Set `email-recovery.enabled`, fill in the SMTP settings, reload.

```yaml
email-recovery:
  enabled: false
  smtp:
    host: "smtp.gmail.com"
    port: 587
    username: ""
    password: ""
    from: "noreply@example.com"
    tls: true      # STARTTLS, required when on; port 465 uses implicit TLS
  code-expiry: 10  # minutes
  cooldown: 5      # minutes between two requests
```

- `/email set`, `/email confirm` and `/email remove` need a confirmed login on this connection.
- A player can only recover the account they are connected as; the console can recover any.
- `/recover <username> <code> <new password>` resets the password. The new password follows the same rules as at registration.
- Codes are kept in memory, not in the database. With `tls: true`, a mail server that does not offer STARTTLS is sent nothing, rather than the password and the code in clear, and its certificate has to match the host name.
- The subjects and bodies of the mails are language keys (`recover-mail-*`, `email-confirm-mail-*`).

</details>

### Locked until authenticated

Until a player logs in, the world holds them. Their position is locked (head rotation and falling still work), chat is blocked both ways, and blocks, items, inventory, damage, vehicles, portals and interaction are all off, with optional blindness and the player hidden from everybody else. The prompt stays on screen as a title, an action bar, a boss bar counting down and a chat line; each one can be switched off, down to a single title in the middle of the screen.

Join messages are held until the player authenticates, and the quit message of someone who never logged in is hidden. After a successful login the player sees the date and address of their last one, so a session that wasn't theirs gets noticed. A session only resumes from the address that opened it.

<details>
<summary><b>Everything blocked before login</b></summary>

<br>

| Action | Behaviour |
|---|---|
| Vision | Blindness, optional |
| Walking | Position locked, head rotation allowed, falling allowed |
| Chat | Blocked, both ways: the chat is not shown to them either |
| Commands | `/login`, `/register`, `/l`, `/reg`, `/captcha`, `/2fa`, `/recover`, `/email` and what you allow |
| Breaking and placing blocks | Blocked |
| Interactions | Blocked, entities, armour stands and item frames included |
| Inventory | Blocked |
| Dropping and picking up items | Blocked |
| Damage, given and taken | Blocked |
| Vehicles | Blocked |
| Portals | Blocked |
| Teleports not from a plugin | Blocked |
| Swapping hands | Blocked |

</details>

### Bedrock, and coming from another plugin

Bedrock players are logged in through Geyser and Floodgate. A handshake announcing a Bedrock player only counts when Floodgate is installed, and a connection claiming it without being a Floodgate player is refused.

Coming from AuthMe or JPremium, `/xlogin import authme <table>` and `/xlogin import jpremium <table>` bring the accounts across. Passwords cannot come with them, because the hash formats are not compatible: imported accounts have no password, `/login` tells the player so, and an admin gives each one with `/xlogin resetpassword`. Premium players are unaffected and log in automatically. From JPremium, the premium flag and the Mojang UUID come across too.

<details>
<summary><b>Importing, in detail</b></summary>

<br>

- The table has to be in the same database as XCore. JPremium's default table is `jp_data`.
- Only players already in XCore's `players` table are imported.
- Accounts already in XLogin are skipped.
- The console says how many accounts were created without a password.

The full guides: [Migrating from AuthMe](https://celestis.dev/plugins/xlogin/wiki/migration#authme) and [from JPremium](https://celestis.dev/plugins/xlogin/wiki/migration#jpremium).

</details>

<br>

## Quick start

1. **Install [XCore 2.0](https://github.com/Xyness/XCore)**, the free framework XLogin runs on, and keep `online-mode=false` in `server.properties`: XLogin is the authentication of an offline-mode server.
2. **Put `XLogin.jar` in `plugins/`**, next to XCore. On a network, the same jar goes on the proxy as well: see [On a network](#on-a-network).
3. **Start the server.** XLogin writes `config.yml`, its languages and `passwords.txt` to `plugins/XLogin/` and creates its tables. New players are asked to `/register`, returning ones to `/login`. Edit `config.yml`, then run `/xlogin reload`.

> [!IMPORTANT]
> If XLogin or XCore cannot start (the database unreachable at boot, a wrong password), XLogin stays up with nothing but a lockdown. Every connection is refused with *Authentication is unavailable on this server right now* instead of letting players in unchecked, and the console says:
>
> `Every connection is refused until XLogin starts correctly (security.refuse-connections-on-failure in config.yml).`
>
> XCore also refuses connections at boot until XLogin is running, so a missing or broken XLogin jar does not leave the server open either. Both follow `security.refuse-connections-on-failure` (default `true`). To open the server without XLogin on purpose, set `addons.XLogin: false` in `plugins/XCore/addons.yml`.

### Upgrading from 1.x

> [!WARNING]
> Never run XLogin 2.0 on XCore 1.x. It does not load there, and XCore 1.x does not close the server in its place: players would get in with no authentication at all. Update XCore to 2.0 in the same stop as XLogin, XCore first.

XLogin used to live inside XCore as an addon. Since 2.0 it is a Paper plugin of its own: put its jar in `plugins/`, take the old one out of `plugins/XCore/addons/`, and update XCore in the same stop. On its first start XLogin moves its folder by itself, settings included:

> `Moved plugins/XCore/addons/XLogin/ to plugins/XLogin/: XLogin is a plugin since XCore 2.0, and its settings and data come with it.`

The database does not change. Commands keep their names; only the namespaced form becomes `/xlogin:login` instead of `/xcore:login`. On a network, update the proxy and every backend together. The steps in full and how to roll back are on **[Upgrading to 2.0](https://celestis.dev/plugins/xcore/wiki/upgrading)**.

<br>

## On a network

The same jar goes in two places: the proxy's `plugins/` folder, and `plugins/` on each backend. The proxy verifies premium players during its own login phase and routes everyone; the backends handle login, registration, restrictions and 2FA, each in a role.

```
  Velocity / BungeeCord
  + XLogin.jar (proxy mode)
         |
         ├─ Premium player? → LOBBY, auth skipped
         ├─ Valid session?  → LOBBY, auth skipped
         └─ Otherwise       → AUTH
                                |
    +-----------+-----------+---+
    |           |           |
  AUTH       LOBBY        GAME
  + XLogin   + XLogin    + XLogin
  /login     session     session
  /register  auto-login  auto-login
```

| Role | What the server does |
|---|---|
| `AUTH` | Players `/login` and `/register` here, then are sent on to the lobby. It never shows the auto-login messages. |
| `LOBBY` | The hub. A valid session logs the player in without a word. |
| `GAME` | Every other server. Session auto-login only. |

| Mechanic | Behaviour |
|---|---|
| Going around the proxy | A player who reaches a LOBBY or GAME server without a valid session is kicked. |
| Sessions | Stored in `xlogin_sessions`, shared by every server, refreshed every 30 seconds while the player is online and again at shutdown. Redis makes propagation instant but is not required. |
| `session-timeout: 0` | LOBBY and GAME still accept a network login from the last 90 seconds, the time a server switch takes. |
| Logouts | With XCore's cross-server sync on, a session ended or an account deleted on one server sends its player back to the prompt on the others. |
| Late profiles | The verified profile can reach the backend a few ticks after the player joins, so premium-flagged accounts are held for `premium.proxy-grace-ms` (two seconds) before the password prompt. Only the players the proxy is expected to verify are held, and a grace period that runs out is logged with the player's name. |
| Database down | A proxy that cannot reach its database at startup tries again every 30 seconds, and until then neither verifies nor routes anybody. Once running, an account whose premium flag cannot be read is treated as premium: Mojang verifies it, and a cracked player is refused until the database answers again. |

A profile sent by plugin message is signed. A proxy relays what clients write on a plugin channel too, so an unsigned "Mojang vouched for this player" could be written by anybody. The secret lives in the shared database (`xlogin_secrets`), created by a backend on its first start and read by the proxy, so there is nothing to configure. The backend also checks that the message is recent and arrived on the connection of the player it names, and the proxy drops anything a client sends on `xlogin:premium`.

> [!NOTE]
> The proxy and the backends run the same XLogin version. A backend refuses the unsigned profiles of an older proxy, and says so in its console.

<details>
<summary><b>The configuration of each server</b></summary>

<br>

**The proxy**, in its own `config.yml` (`plugins/XLogin/` on BungeeCord, `plugins/xlogin/` on Velocity):

```yaml
database:
  type: MYSQL            # or POSTGRESQL; SQLite cannot be shared with a proxy
  host: localhost
  port: 3306
  database: minecraft    # the same one XCore uses
  username: root
  password: ""
  ssl: false             # encrypted connection, same as database.ssl in XCore

redis:
  enabled: true          # recommended, not required
  host: localhost
  port: 6379
  password: ""

premium-strict-mode: false  # must match premium.mode on the backends (true = strict)
auth-server: "auth"         # where unauthenticated players are sent
redirect-server: "lobby"    # where premium and session players go; empty = nobody skips the auth server
session-timeout: 30         # must match the backends
```

**The auth server:**

```yaml
premium:
  enabled: true
proxy:
  enabled: true
  role: AUTH
  redirect-server: "lobby"
  redirect-delay: 20
```

**Lobby and game servers:**

```yaml
premium:
  enabled: true
proxy:
  enabled: true
  role: LOBBY    # or GAME
  auth-server: "auth"
```

The full walkthrough: [Proxy setup](https://celestis.dev/plugins/xlogin/wiki/proxy#xlogin-proxy).

</details>

<br>

## Commands and permissions

**Players**

| Command | Aliases | What it does |
|---|---|---|
| `/login <password>` | `/l` | Log in |
| `/register <password> <confirm>` | `/reg` | Create an account |
| `/changepassword <old> <new>` | `/changepw`, `/cp` | Change your password |
| `/logout` | `/logoff` | Log out and disconnect |
| `/premium` | | Turn premium auto-login on, asked twice, with a warning when your UUID will change |
| `/unpremium` | | Turn it off and clear the Mojang link (`opt-in` mode only) |
| `/2fa setup` | | Set up two-factor authentication |
| `/2fa <code>` | | Enter a code |
| `/2fa codes` | | Issue eight recovery codes |
| `/2fa recover <code>` | | Recover with one |
| `/2fa disable` | | Turn 2FA off |
| `/captcha <code>` | | Answer the registration captcha |
| `/email set <email>`, `/email confirm <code>`, `/email remove` | | Manage the recovery email: a code is mailed to it, and it is linked once the code is typed back |
| `/recover <username> [code] [new password]` | | Password recovery by email |

`/changepassword`, `/email set|confirm|remove`, `/2fa setup|disable|codes`, `/premium` and `/unpremium` need a confirmed login on this connection: the password typed on it, a premium auto-login verified by Mojang, or a Bedrock login confirmed by Floodgate. A player who came back through a session, or was logged in by an admin, runs `/login <password>` once first, which asks for the 2FA code too when 2FA is on. Wrong passwords there count as failed logins.

**Admins**

| Command | What it does |
|---|---|
| `/xlogin reload` | Reload config and language files |
| `/xlogin setspawn` | Set the login spawn |
| `/xlogin forcelogin <player>` | Authenticate a player |
| `/xlogin logout <player>` | End a player's session on every server and disconnect them |
| `/xlogin resetpassword <player> <new>` | Reset a password; a connected player is disconnected the same way |
| `/xlogin info <player>` | Registration date, addresses, login count |
| `/xlogin accounts <ip>` | Accounts seen from an address |
| `/xlogin import authme <table>` | Import from AuthMe |
| `/xlogin import jpremium <table>` | Import from JPremium |
| `/unregister <player>` | Delete an account, online or not, with its sessions, 2FA and recovery codes |

**Permissions**

| Permission | Grants |
|---|---|
| `xlogin.admin` | Every admin command above, and the account actions of the dashboard |
| `xlogin.notify` | Failed logins and registrations |
| `xlogin.update` | Update notifications on join |

Every command also answers to its namespaced form, `/xlogin:login` and so on.

<br>

## Configuration

Everything lives in `plugins/XLogin/config.yml`, and `/xlogin reload` applies a change. The texts of the prompts live in `lang/<code>.yml`; their timings and colours stay here.

<details>
<summary><b>The main settings, with their defaults</b></summary>

<br>

```yaml
debug: false               # true logs every step of the premium handshake as [Premium]

session-timeout: 30        # minutes a session survives a disconnect, 0 = login every time
                           # (with 0, LOBBY and GAME still accept a network login from the last 90 s)
max-login-attempts: 5
login-timeout: 60          # seconds to log in after joining, 0 = no limit

min-password-length: 6
max-password-length: 32
require-strong-password: false   # uppercase, lowercase and a digit
password:
  refuse-common: true      # passwords.txt, one per line; empty the file to disable

captcha:
  enabled: false
  length: 5                # 3 to 8
  max-attempts: 3

connection-limit:
  enabled: true
  max-per-ip: 6            # connections from one address...
  window: 60               # ...in this many seconds

max-accounts-per-ip: 3     # 0 = no limit
ip-lock: false             # only accept a login from the last known address

# --- Before login ---
hide-unlogged-players: true
blindness-effect: true
block-movement: true       # head rotation still allowed
block-interactions: true
hide-quit-if-not-logged: true

# /login, /register, /l, /reg, /captcha, /2fa, /recover and /email are always allowed
allowed-commands:
  - "/help"

teleport-to-spawn: false
spawn-location:            # set in game with /xlogin setspawn
  world: "world"
  x: 0.5
  y: 64.0
  z: 0.5
  yaw: 0.0
  pitch: 0.0

name-validation:
  enabled: true
  pattern: "^[a-zA-Z0-9_]{3,16}$"
  blocked-words: [admin, moderator, server, console, staff, owner, operator, hack, cheat]

allow-registration: true
force-registration: true   # false = registration is optional, players without an account just play
show-last-login: true

# --- Security ---
notify-on-failed-login: true
log-events: true

security:
  ip-ban-duration: 30         # minutes
  ip-rate-limit-max: 10       # failed attempts from one address
  ip-rate-limit-window: 10    # minutes
  account-rate-limit-max: 10  # failed logins on one account in that window, from any address;
                              # past it only the owner's last address is accepted. 0 = off
  refuse-connections-on-failure: true  # XLogin could not start: refuse everyone (auth-unavailable)

# --- Premium auto-login ---
premium:
  enabled: false
  mode: "opt-in"           # strict = every non-Bedrock player verifies, opt-in = /premium only
  uuid-mode: "OFFLINE"     # OFFLINE = UUIDs from the name, REAL = Mojang UUIDs
  proxy-grace-ms: 2000     # backend wait for the proxy profile, 0 disables, max 10000

# --- Messages ---
# LOBBY and GAME only; AUTH never shows them.
messages:
  login-prompt: true       # the chat line asking for /login or /register on join
  session-resumed: true
  premium-auto-logged: true
  bedrock-auto-logged: true
  bedrock-auto-registered: true
  login-success: true

login-title:
  enabled: true
  keep-until-login: true   # kept on screen until the player authenticates; false = shown once for `stay`
  fade-in: 10
  stay: 100
  fade-out: 10

login-bossbar:
  enabled: true
  color: RED

login-actionbar:
  enabled: true

# --- Proxy ---
proxy:
  enabled: false
  role: AUTH               # AUTH, LOBBY or GAME
  redirect-server: "lobby"
  auth-server: "auth"
  redirect-delay: 20       # ticks

two-factor:
  enabled: false

bedrock:
  auto-login: true

update:
  check: true
  notifications: true
```

</details>

Every key, explained: [config.yml](https://celestis.dev/plugins/xlogin/wiki/configuration#config-yml).

<br>

## Dashboard and integrations

### Web dashboard

XLogin adds accounts, sessions and statistics to XCore's dashboard, with password reset (you type the new password), 2FA removal, session invalidation and account deletion, applied live from the browser.

<details>
<summary><b>Dashboard permissions</b></summary>

<br>

None of these nodes is registered, so they all default to op: ops keep everything.

- **Reading.** Every page needs `xcore.dashboard.xlogin`. The accounts and sessions pages need `xlogin.admin` as well, and a player's card on their dashboard profile needs `xlogin.admin` and `xcore.dashboard.players`.
- **Acting.** Resetting a password, disabling 2FA, deleting an account and invalidating a session need `xlogin.admin`, like the commands.
- **Saving the configuration** needs `xcore.dashboard.config`, `xcore.dashboard.config.save` and `xlogin.admin`. The file decides whether anyone has to log in at all, so saving it is trusted like the console, and every save is logged with its author. Secrets such as the SMTP password are shown as `<hidden>`; saving `<hidden>` unchanged keeps the stored value.

Every node of the dashboard: [Dashboard roles](https://celestis.dev/plugins/xcore/wiki/configuration#dashboard-roles).

</details>

### Works with

| With | What for |
|---|---|
| **Velocity and BungeeCord** | The same jar, in proxy mode, with premium verification and routing |
| **Geyser and Floodgate** | Bedrock players logged in automatically |
| **TCPShield, tunnels, load balancers** | Premium auto-login behind a PROXY protocol front |
| **Redis** | Instant delivery of sessions and verified profiles; optional |
| **Lunar and Badlion** | Cosmetics, with `uuid-mode: REAL` |
| **Any SMTP server** | Password recovery by email, over STARTTLS or implicit TLS |
| **AuthMe and JPremium** | Account import |

<br>

## Under the hood

**Languages.** `lang/<code>.yml`, in [MiniMessage](https://docs.advntr.dev/minimessage/format.html), following XCore's `language` setting. English and French are bundled. The title, boss bar and action bar texts live there; their timings and colours stay in `config.yml`.

**Updates.** Players with `xlogin.update` are told on join when a new version is out.

<details>
<summary><b>Language placeholders</b></summary>

<br>

| Placeholder | Used in |
|---|---|
| `{attempts}`, `{max}` | Wrong password, 2FA code or recovery code |
| `{min}`, `{max}` | Password length |
| `{player}` | Admin notifications, info, accounts |
| `{ip}` | Admin notifications, last login, accounts |
| `{date}` | Last login, account info |
| `{time}` | Boss bar countdown |
| `{count}` | Login count, import results |
| `{uuid}` | Account info |
| `{pattern}` | Name validation |
| `{name}` | Name registered under another spelling |
| `{server}` | Proxy redirect |
| `{secret}` | 2FA setup |
| `{last_login}` | Accounts list |
| `{email}` | Email confirmation |
| `{code}`, `{minutes}` | Recovery and confirmation mails |

</details>

<details>
<summary><b>Database tables</b></summary>

<br>

| Table | Contents |
|---|---|
| `xlogin_accounts` | UUID, name, password hash, addresses, login count, premium flag, Mojang UUID, email |
| `xlogin_sessions` | Active sessions |
| `xlogin_ip_bans` | Temporarily banned addresses |
| `xlogin_2fa` | TOTP secrets |
| `xlogin_recovery_codes` | 2FA recovery codes, hashed |
| `xlogin_secrets` | The secret the proxy signs verified profiles with (proxy mode only) |

Created on first start. Column migrations are applied automatically.

</details>

<br>

## Support

Questions, bug reports and ideas go to the [Discord](https://discord.com/invite/6sRTGprM95). Every command, setting and screen is documented on [celestis.dev](https://celestis.dev/plugins/xlogin/wiki/getting-started).

<p align="center">
  <sub>XLogin is part of the XCore family, with <a href="https://celestis.dev/plugins/xbans">XBans</a>, <a href="https://celestis.dev/plugins/xauctionhouse">XAuctionHouse</a> and <a href="https://celestis.dev/plugins/xantilag">XAntiLag</a>. Made by <a href="https://github.com/Xyness">Xyness</a>.</sub>
</p>
