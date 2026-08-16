# XLogin

Authentication for [XCore](https://github.com/Xyness/XCore): login and register, premium
auto-login, 2FA, cross-server sessions, proxy support.

## Features

- `/login`, `/register`, `/changepassword`.
- Online mode per player: Netty pipeline injection on a standalone server, proxy-level verification
  on Velocity and BungeeCord. Premium players connect with their Mojang skin and no password. A
  verified profile expires after 30 seconds and is bound to the connection that earned it.
- TOTP two-factor (Google Authenticator, Authy and similar). An accepted code is single-use.
- Sessions stored in the database and shared across servers.
- Proxy support with a role per server (AUTH, LOBBY, GAME) and automatic routing.
- Bedrock players authenticated through Geyser and Floodgate.
- Rate limiting per address, temporary IP bans, IP lock, brute-force protection, password strength
  rules.
- PBKDF2-HMAC-SHA256, 210 000 iterations, salt per account, stored as
  `pbkdf2$iterations$salt$hash`. Older accounts are re-hashed on their next login. Changing or
  resetting a password invalidates every session on that account.
- Players reaching a LOBBY or GAME server without a valid session are kicked.
- One jar for Velocity, BungeeCord and XCore.
- Passwords masked in the console through a Log4j filter.
- Optional restrictions before authentication: blindness, movement, interaction, vehicles, portals,
  teleports.
- Join messages held until authentication, quit message hidden for a player who never logged in.
- Title, action bar and boss bar prompts until the player authenticates.
- Admin tools: force login, reset password, account info, address lookup, AuthMe and JPremium
  import.
- Last login date and address shown after a successful login.
- Update notifications on join (`xlogin.update`).

### Registration captcha

A code drawn on a map, typed back with `/captcha` before `/register` is accepted. Length and number
of attempts are configurable.

### Password quality

Common passwords are refused, from `passwords.txt` in the addon folder. The list is local, nothing
is sent anywhere.

### Connection rate limit

Connections per address per window, refused at pre-login. Separate from the failed-login limit.

### 2FA recovery codes

`/2fa codes` issues eight single-use codes, shown once and stored hashed. `/2fa recover <code>`
restores access.

### UUID mode changes

Switching between offline and online UUIDs moves the account onto the new UUID. Password, sessions
and 2FA follow.

## Requirements

- Paper 1.21.1+ or Folia
- Java 21+
- XCore

## Installation

### Standalone

1. Install XCore in `plugins/`.
2. Put `XLogin.jar` in `plugins/XCore/addons/`.
3. Start the server. The config is written to `plugins/XCore/addons/XLogin/`.
4. Edit `config.yml` and `lang/<code>.yml`.
5. Restart or run `/xlogin reload`.

### Behind a proxy

Put the same jar in two places: the proxy's `plugins/` folder (database, Redis, routing) and
`plugins/XCore/addons/` on each backend (full auth config). The proxy handles premium verification
and routing, the backend handles login, register, restrictions and 2FA.

## Commands

### Players

| Command | Aliases | Description |
|---------|---------|-------------|
| `/login <password>` | `/l` | Log in |
| `/register <password> <confirm>` | `/reg` | Create an account |
| `/changepassword <old> <new>` | `/changepw` | Change your password |
| `/logout` | | Log out and disconnect |
| `/premium` | | Enable premium auto-login |
| `/unpremium` | | Disable it |
| `/2fa setup` | | Set up two-factor authentication |
| `/2fa <code>` | | Enter a code |
| `/2fa codes` | | Issue eight recovery codes |
| `/2fa recover <code>` | | Recover with one |
| `/2fa disable` | | Turn 2FA off |
| `/captcha <code>` | | Answer the registration captcha |
| `/email set <email>`, `/email remove` | | Manage the recovery email |
| `/recover <username> [code] [new password]` | | Password recovery by email |

### Admins

| Command | Permission | Description |
|---------|-----------|-------------|
| `/xlogin reload` | `xlogin.admin` | Reload config and language files |
| `/xlogin setspawn` | `xlogin.admin` | Set the login spawn |
| `/xlogin forcelogin <player>` | `xlogin.admin` | Authenticate a player |
| `/xlogin resetpassword <player> <new>` | `xlogin.admin` | Reset a password |
| `/xlogin info <player>` | `xlogin.admin` | Registration date, addresses, login count |
| `/xlogin accounts <ip>` | `xlogin.admin` | Accounts seen from an address |
| `/xlogin import authme <table>` | `xlogin.admin` | Import from AuthMe |
| `/xlogin import jpremium <table>` | `xlogin.admin` | Import from JPremium |
| `/unregister <player>` | `xlogin.admin` | Delete an account |

| Permission | Description |
|-----------|-------------|
| `xlogin.admin` | Every admin command |
| `xlogin.notify` | Failed logins and registrations |
| `xlogin.update` | Update notifications on join |

## Configuration

```yaml
debug: false

session-timeout: 30        # minutes a session survives a disconnect, 0 = login every time
max-login-attempts: 5
login-timeout: 60          # seconds to log in after joining, 0 = no limit

min-password-length: 6
max-password-length: 32
require-strong-password: false
max-accounts-per-ip: 3     # 0 = no limit

# --- Before login ---
hide-unlogged-players: true
blindness-effect: true
block-movement: true       # head rotation still allowed
block-interactions: true
hide-quit-if-not-logged: true

# /login, /register, /l, /reg and /2fa are always allowed
allowed-commands:
  - "/help"

teleport-to-spawn: false
spawn-location:
  world: "world"
  x: 0.0
  y: 64.0
  z: 0.0
  yaw: 0.0
  pitch: 0.0

name-validation:
  enabled: true
  pattern: "^[a-zA-Z0-9_]{3,16}$"
  blocked-words:
    - "admin"
    - "moderator"
    - "server"
    - "console"

allow-registration: true
force-registration: true   # kick players who do not register in time
show-last-login: true

# --- Security ---
notify-on-failed-login: true
log-events: true
ip-lock: false             # only accept a login from the last known address

security:
  ip-ban-duration: 30       # minutes
  ip-rate-limit-max: 10     # failed attempts from one address
  ip-rate-limit-window: 10  # minutes

# --- Premium auto-login ---
premium:
  enabled: false
  mode: "opt-in"           # strict = every non-Bedrock player verifies, opt-in = /premium only
  uuid-mode: "OFFLINE"     # OFFLINE = UUIDs from the name, REAL = Mojang UUIDs
  proxy-grace-ms: 2000     # backend wait for the proxy profile, 0 disables, max 10000

# --- Messages ---
# LOBBY and GAME only; AUTH never shows them.
messages:
  session-resumed: true
  premium-auto-logged: true
  login-success: true

# Texts live in lang/<code>.yml; timings and colours here.
login-title:
  enabled: true
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
  kick-on-bypass: false    # kick anyone reaching LOBBY or GAME without a session

two-factor:
  enabled: false

bedrock:
  auto-login: true

update:
  check: true
  notifications: true
```

## Network setup

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

Proxy, `plugins/xlogin/config.yml`:

```yaml
database:
  type: MYSQL
  host: localhost
  port: 3306
  database: minecraft    # the same one XCore uses
  username: root
  password: ""

redis:
  enabled: true          # recommended
  host: localhost
  port: 6379
  password: ""

premium-strict-mode: false  # must match the backends
redirect-server: "lobby"
session-timeout: 30         # must match the backends
```

Auth server:

```yaml
premium:
  enabled: true
proxy:
  enabled: true
  role: AUTH
  redirect-server: "lobby"
  redirect-delay: 20
```

Lobby and game servers:

```yaml
premium:
  enabled: true
proxy:
  enabled: true
  role: LOBBY    # or GAME
  auth-server: "auth"
```

Sessions live in `xlogin_sessions`, shared by every server. Redis makes propagation instant but is
not required.

## Premium auto-login

### Standalone

XLogin injects a handler into the Netty pipeline:

1. The player connects to the offline-mode server.
2. The handler intercepts `LoginStart`.
3. XLogin checks whether the name is a premium account, through the API or the database flag.
4. An `EncryptionRequest` goes to the client.
5. The client authenticates with Mojang and answers with `EncryptionResponse`.
6. XLogin decrypts the shared secret and enables AES/CFB8.
7. It verifies with Mojang's `hasJoined` endpoint.
8. On success the Mojang UUID and skin are kept and the login continues.
9. On join the account is created if needed, the skin applied, the player authenticated.

### Behind a proxy

1. The player reaches the proxy.
2. XLogin checks the database for `premium=1`, or asks Mojang in strict mode.
3. Online mode is forced for that connection and the proxy runs the handshake.
4. The verified profile is sent to the backend over Redis and plugin messaging.
5. The player is routed to the lobby.

That profile can reach the backend a few ticks after `PlayerJoinEvent`, so premium-flagged accounts
are held for `premium.proxy-grace-ms`, two seconds by default, before the password prompt. Only
players the proxy is expected to verify are held. A grace period that runs out is logged with the
player's name.

### Strict or opt-in

`premium.mode` lives in the backend config; the proxy carries the same choice as
`premium-strict-mode`.

| Mode | Behaviour |
|------|-----------|
| `strict` | Any name existing as a Mojang account with no account here has to verify. A verified account is flagged premium and verified on every connection. `/premium` and `/unpremium` are disabled. |
| `opt-in` | Only players who ran `/premium` are verified. |

### Standalone setup

1. Set `online-mode=false` in `server.properties`. XLogin runs the handshake itself; a server
   already in online mode runs a second one and the client is disconnected with `Tried to switch to
   AUTHORIZING from ENCRYPTING`. Premium auto-login refuses to start when it detects online mode.
2. Set `premium.enabled: true`.
3. Pick `premium.mode`.
4. Restart.

## Username changes

A premium player changing their name changes their offline UUID. XLogin migrates the account:

1. The player connects under the new name.
2. The handshake returns the same Mojang UUID.
3. XLogin finds the old account: same `mojang_uuid`, different offline UUID.
4. `player_uuid` is updated in `xlogin_accounts`, `xlogin_sessions` and `xlogin_2fa`, in one
   transaction.
5. A `MIGRATE` sync event notifies the other servers.

Password, premium flag, 2FA, email and sessions are kept. Only applies in `uuid-mode: OFFLINE`.

## UUID modes

| Mode | Behaviour |
|------|-----------|
| `OFFLINE` | UUIDs computed from the name. Premium verification still runs for authentication. Default. |
| `REAL` | Verified premium players get their Mojang UUID, set through Paper's `PlayerProfile` API during `AsyncPlayerPreLoginEvent`. Enables cosmetics on Lunar and Badlion. Cracked players keep offline UUIDs. |

Changing `uuid-mode` once players have data is destructive: premium UUIDs change and their data in
other plugins is orphaned.

## Password recovery by email

Set `email-recovery.enabled`, fill in the SMTP settings, reload.

- `/email set <email>`, `/email remove`, both for an authenticated player.
- `/recover <username>` sends a six-digit code.
- `/recover <username> <code> <new password>` resets the password.

```yaml
email-recovery:
  enabled: false
  smtp:
    host: "smtp.gmail.com"
    port: 587
    username: ""
    password: ""
    from: "noreply@example.com"
    tls: true
  code-expiry: 10  # minutes
  cooldown: 5      # minutes between two requests
```

Codes are kept in memory, not in the database.

## Two-factor authentication

Set `two-factor.enabled`. A player runs `/2fa setup`, adds the secret to their app and confirms with
`/2fa <code>`. Logins then ask for the password and the code. `/2fa disable` turns it off,
`/unregister <player>` removes the account and its 2FA.

## Blocked before login

| Action | Behaviour |
|--------|-----------|
| Vision | Blindness, optional |
| Walking | Position locked, head rotation allowed |
| Chat | Blocked |
| Commands | `/login`, `/register`, `/l`, `/reg`, `/2fa`, `/recover`, `/email` and what you allow |
| Breaking and placing blocks | Blocked |
| Interactions | Blocked |
| Inventory | Blocked |
| Dropping and picking up items | Blocked |
| Damage, given and taken | Blocked |
| Vehicles | Blocked |
| Portals | Blocked |
| Teleports not from a plugin | Blocked |
| Swapping hands | Blocked |

## Database

| Table | Contents |
|-------|----------|
| `xlogin_accounts` | UUID, name, password hash, addresses, login count, premium flag, Mojang UUID, email |
| `xlogin_sessions` | Active sessions |
| `xlogin_ip_bans` | Temporarily banned addresses |
| `xlogin_2fa` | TOTP secrets |

Created on first start. Column migrations are applied automatically.

## Importing from AuthMe or JPremium

```
/xlogin import authme <table>
/xlogin import jpremium <table>
```

JPremium's default table is `jp_data`. In both cases:

- The table has to be in the same database as XCore.
- Passwords are not migrated. Imported players reset theirs with `/changepassword`, or an admin uses
  `/xlogin resetpassword`.
- Only players already in XCore's `players` table are imported.
- Accounts already in XLogin are skipped.
- From JPremium, the premium flag and the Mojang UUID are imported.

## Languages

`lang/<code>.yml`, in [MiniMessage](https://docs.advntr.dev/minimessage/format.html), following
XCore's `language` setting. English and French are bundled. The title, boss bar and action bar texts
live there; their timings and colours stay in `config.yml`.

| Placeholder | Used in |
|-------------|---------|
| `{attempts}`, `{max}` | Failed login |
| `{min}`, `{max}` | Password length |
| `{player}` | Admin notifications, info, accounts |
| `{ip}` | Admin notifications, last login, accounts |
| `{date}` | Last login, account info |
| `{time}` | Boss bar countdown |
| `{count}` | Login count, import results |
| `{uuid}` | Account info |
| `{pattern}` | Name validation |
| `{server}` | Proxy redirect |
| `{secret}` | 2FA setup |
| `{last_login}` | Accounts list |
| `{email}` | Email confirmation |

## License

Part of the XCore ecosystem, by [Xyness](https://github.com/Xyness).
