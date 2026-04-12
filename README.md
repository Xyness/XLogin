# XLogin

Advanced authentication addon for [XCore](https://github.com/Xyness/XCore). Provides a complete login/register system with premium auto-login, 2FA, cross-server sessions, and proxy support.

---

## Features

- **Login / Register** - `/login`, `/register`, `/changepassword`
- **Premium Auto-Login** - Per-player online-mode via Netty pipeline injection. Premium players connect seamlessly with their real Mojang skin, no `/login` needed. Cracked players using a premium name are automatically rejected.
- **Two-Factor Authentication (2FA)** - TOTP support (Google Authenticator, Authy, etc.)
- **Cross-Server Sessions** - Database-backed sessions shared across all servers. Login once, authenticated everywhere.
- **Proxy Support (BungeeCord / Velocity)** - Role-based server configuration (AUTH, LOBBY, GAME) with automatic player routing.
- **Bedrock Auto-Login** - Automatic authentication for Geyser/Floodgate players.
- **Security** - IP rate limiting, temporary IP bans, IP lock, brute-force protection, password strength requirements.
- **Player Restrictions** - Blindness, movement blocking, interaction blocking, vehicle/portal/teleport blocking until authenticated.
- **Join/Quit Message Control** - Join messages are hidden until authentication, quit messages hidden if never logged in.
- **Persistent Title/ActionBar/BossBar** - Configurable login prompts that persist until the player authenticates.
- **Admin Tools** - Force login, reset password, account info, IP lookup, AuthMe import.
- **Last Login Info** - Shows last login date and IP after successful authentication.

---

## Requirements

- **Paper 1.21.1+** (or Folia)
- **Java 21+**
- **XCore 1.0.0+** - The core framework (must be installed as a plugin)

---

## Installation

1. Install **XCore** in your `plugins/` folder.
2. Place the **XLogin** JAR in `plugins/XCore/addons/`.
3. Start the server. XLogin will generate its config files in `plugins/XCore/addons/XLogin/`.
4. Edit `config.yml` and `lang.yml` to your needs.
5. Restart or `/xlogin reload`.

---

## Commands

### Player Commands

| Command | Aliases | Description |
|---------|---------|-------------|
| `/login <password>` | `/l` | Login to your account |
| `/register <password> <confirm>` | `/reg` | Register a new account |
| `/changepassword <old> <new>` | `/changepw` | Change your password |
| `/logout` | | Logout and disconnect |
| `/premium` | | Enable premium auto-login for your account |
| `/unpremium` | | Disable premium auto-login |
| `/2fa setup` | | Set up two-factor authentication |
| `/2fa <code>` | | Verify a 2FA code (during login or setup) |
| `/2fa disable` | | Disable two-factor authentication |

### Admin Commands

| Command | Permission | Description |
|---------|-----------|-------------|
| `/xlogin reload` | `xlogin.admin` | Reload configuration and language files |
| `/xlogin setspawn` | `xlogin.admin` | Set the login spawn point to your current location |
| `/xlogin forcelogin <player>` | `xlogin.admin` | Force-authenticate a player |
| `/xlogin resetpassword <player> <new>` | `xlogin.admin` | Reset a player's password |
| `/xlogin info <player>` | `xlogin.admin` | View account info (register date, IPs, login count) |
| `/xlogin accounts <ip>` | `xlogin.admin` | View all accounts from a given IP |
| `/xlogin import authme <table>` | `xlogin.admin` | Import accounts from an AuthMe database table |
| `/unregister <player>` | `xlogin.admin` | Delete a player's account |

### Permissions

| Permission | Description |
|-----------|-------------|
| `xlogin.admin` | Access to all admin commands |
| `xlogin.notify` | Receive admin notifications (failed logins, registrations) |

---

## Configuration

### config.yml

```yaml
# Enable debug logging
debug: false

# ============================================
# Authentication Settings
# ============================================

# Session timeout in minutes (player stays logged in after disconnect)
# Set to 0 to require login every time
session-timeout: 30

# Maximum login attempts before kick
max-login-attempts: 5

# Time in seconds to login after joining (0 = unlimited)
login-timeout: 60

# Password length limits
min-password-length: 6
max-password-length: 32

# Require uppercase + lowercase + number in password
require-strong-password: false

# Maximum accounts per IP (0 = unlimited)
max-accounts-per-ip: 3

# ============================================
# Player Restrictions (before login)
# ============================================

# Hide player from others until logged in (invisible + blindness)
hide-unlogged-players: true

# Prevent movement before login (allows head rotation)
block-movement: true

# Prevent all interactions before login
block-interactions: true

# Hide quit message if player disconnects without authenticating
hide-quit-if-not-logged: true

# Commands allowed before login (always includes /login, /register, /l, /reg, /2fa)
allowed-commands:
  - "/help"

# Teleport player to a fixed location on join (before login)
teleport-to-spawn: false

# Spawn location (set with /xlogin setspawn or manually here)
spawn-location:
  world: "world"
  x: 0.0
  y: 64.0
  z: 0.0
  yaw: 0.0
  pitch: 0.0

# ============================================
# Name Validation
# ============================================

name-validation:
  enabled: true
  pattern: "^[a-zA-Z0-9_]{3,16}$"
  blocked-words:
    - "admin"
    - "moderator"
    - "server"
    - "console"

# ============================================
# Registration & Login Behavior
# ============================================

# Allow new registrations
allow-registration: true

# Kick unregistered players who don't register within timeout
force-registration: true

# Show last login date and IP after successful authentication
show-last-login: true

# ============================================
# Security
# ============================================

# Notify admins on failed login (permission: xlogin.notify)
notify-on-failed-login: true

# Log all login/register events
log-events: true

# Only allow login from the last known IP
ip-lock: false

# Temporary IP ban and rate limiting
security:
  # Ban duration in minutes after max failed attempts
  ip-ban-duration: 30
  # Max failed attempts from same IP before temp-ban
  ip-rate-limit-max: 10
  # Time window for rate limiting (in minutes)
  ip-rate-limit-window: 10

# ============================================
# Premium Auto-Login
# ============================================

# Per-player online-mode via Netty pipeline injection.
# Premium players are verified with Mojang's session server.
# They connect seamlessly with their real skin, no /register or /login needed.
# Cracked players using a premium name are automatically disconnected.
#
# New premium players: auto-detected via Mojang API, auto-registered.
# Existing players: use /premium to enable, /unpremium to disable.
premium-auto-login: false

# ============================================
# Messages Display
# ============================================

# Title/subtitle shown on join
login-title:
  enabled: true
  login-title: "<gold><bold>Welcome back!"
  login-subtitle: "<gray>Use <white>/login <password>"
  register-title: "<gold><bold>Welcome!"
  register-subtitle: "<gray>Use <white>/register <password> <password>"
  fade-in: 10
  stay: 100
  fade-out: 10

# Boss bar countdown
login-bossbar:
  enabled: true
  text: "<red>Please authenticate - {time}s remaining"
  color: RED

# Action bar reminder (separate messages for login vs register)
login-actionbar:
  enabled: true
  login-text: "<yellow>... Use /login <password> ..."
  register-text: "<yellow>... Use /register <password> <password> ..."

# ============================================
# Proxy (BungeeCord / Velocity)
# ============================================

proxy:
  enabled: false
  # Server role: AUTH, LOBBY, or GAME
  #   AUTH  - Login/register server. Redirects to lobby after auth.
  #   LOBBY - Main hub. Session auto-login. Invalid session -> sent to auth.
  #   GAME  - Sub-server. Session auto-login. Invalid session -> sent to auth.
  role: AUTH
  # Server to redirect to after authentication (AUTH role)
  redirect-server: "lobby"
  # Server to send unauthenticated players to (LOBBY/GAME roles)
  auth-server: "auth"
  # Delay before redirect in ticks (20 = 1 second)
  redirect-delay: 20

# ============================================
# Two-Factor Authentication (2FA / TOTP)
# ============================================

two-factor:
  enabled: false

# ============================================
# Bedrock
# ============================================

bedrock:
  # Auto-login Bedrock players (authenticated via Xbox Live)
  auto-login: true

# ============================================
# Update Checker
# ============================================

update:
  check: true
  notifications: true
```

---

## Proxy Setup Guide

XLogin supports BungeeCord and Velocity networks with a role-based configuration. Install XLogin on every backend server and configure the `proxy` section accordingly.

### Architecture

```
                    +----------------------+
                    |   BungeeCord/Velocity |
                    |                      |
                    |  Default server: auth |
                    +----------+-----------+
                               |
              +----------------+----------------+
              |                |                |
        +-----v-----+   +-----v----+    +------v------+
        |   AUTH     |   |  LOBBY   |    |    GAME     |
        |            |   |          |    |  (skyblock,  |
        |  /login    |   |  Session |    |  minigames)  |
        |  /register |   |  auto-   |    |  Session     |
        |  Premium   |   |  login   |    |  auto-login  |
        |  handshake |   |          |    |              |
        +------------+   +----------+    +-------------+
```

### Server Configurations

**Auth Server** (`plugins/XCore/addons/XLogin/config.yml`):
```yaml
proxy:
  enabled: true
  role: AUTH
  redirect-server: "lobby"
  redirect-delay: 20
```

**Lobby Server**:
```yaml
proxy:
  enabled: true
  role: LOBBY
  auth-server: "auth"
```

**Game Servers** (skyblock, minigames, etc.):
```yaml
proxy:
  enabled: true
  role: GAME
  auth-server: "auth"
```

### Flow

1. Player connects to the network. The proxy sends them to the **AUTH** server.
2. On the AUTH server:
   - **Premium player**: Netty handshake verifies with Mojang. Auto-login. Instant redirect to lobby (player sees nothing on the auth server).
   - **Cracked player**: `/register` or `/login`. Redirect to lobby after auth.
3. On the **LOBBY** server: session found in database. Auto-login, no prompt.
4. Player switches to a **GAME** server: session found. Auto-login.
5. If a player somehow reaches LOBBY/GAME without a valid session, they are sent back to the AUTH server.

### Cross-Server Sessions

Sessions are stored in the shared XCore database (`xlogin_sessions` table). All servers connected to the same database see the same sessions. No Redis required (but if XCore's sync manager is enabled with Redis, session creation is also broadcast for instant propagation).

---

## Premium Auto-Login

### How It Works

XLogin implements **per-player online-mode** by injecting a custom handler into the server's Netty pipeline. This is the same technique used by plugins like JPremium.

1. Player connects to the server (offline-mode).
2. XLogin's Netty handler intercepts the `LoginStart` packet.
3. Checks if the player's name is a premium Mojang account (via API or database flag).
4. If premium: sends an `EncryptionRequest` to the client.
5. The client authenticates with Mojang and responds with `EncryptionResponse`.
6. XLogin decrypts the shared secret, enables AES/CFB8 encryption on the channel.
7. Verifies with Mojang's `hasJoined` session server endpoint.
8. If verified: stores the player's Mojang UUID and skin textures, lets the login proceed.
9. On join: auto-registers (if new), applies premium skin, authenticates instantly.

### Security

- **Name stealing protection** (strict mode): A cracked player using a premium username cannot complete the Mojang handshake and is disconnected automatically.
- **Encryption**: The connection is encrypted with AES/CFB8 after the handshake, same as vanilla online-mode.
- **Existing accounts**: Players who already registered as cracked are not forced into premium mode. They can opt-in with `/premium`.

### Strict Mode vs Opt-In Mode

The `premium-strict-mode` setting controls how premium detection works for new players:

| Mode | Config | Behavior |
|------|--------|----------|
| **STRICT** | `premium-strict-mode: true` | Any username that exists as a Mojang account is forced through encryption verification. Cracked players **cannot** use a premium username. New premium players are auto-registered on first join. Best for: maximum security, anti name-stealing. |
| **OPT-IN** | `premium-strict-mode: false` | Only players who have explicitly used `/premium` are verified. New players always go through `/register`, even with a premium username. After registering, they can `/premium` to enable auto-login. Best for: mixed servers where some premium players prefer `/login`. |

### Setup

1. Set `premium-auto-login: true` in `config.yml`.
2. Choose your mode: `premium-strict-mode: true` (strict) or `false` (opt-in).
3. Restart the server.
4. **Strict mode**: Premium players are auto-detected on first connection. Cracked players cannot use premium names.
5. **Opt-in mode**: All new players `/register` normally. Premium players can then `/premium` to enable auto-login.

---

## Two-Factor Authentication (2FA)

XLogin supports TOTP (Time-based One-Time Password), compatible with Google Authenticator, Authy, Microsoft Authenticator, and similar apps.

### Setup for Players

1. Admin enables it in config: `two-factor.enabled: true`
2. Player runs `/2fa setup`. They receive a secret key.
3. Player adds the key to their authenticator app.
4. Player runs `/2fa <code>` with the 6-digit code from the app to confirm.
5. On future logins, after entering the password with `/login`, the player must also enter `/2fa <code>`.

### Disable

- Player: `/2fa disable` (must be authenticated)
- Admin: `/unregister <player>` removes the account including 2FA.

---

## Player Restrictions

While unauthenticated, players are restricted from:

| Action | Behavior |
|--------|----------|
| Movement (walking) | Position locked, head rotation allowed |
| Chat | Blocked |
| Commands | Only `/login`, `/register`, `/l`, `/reg`, `/2fa`, and configured allowed commands |
| Block break/place | Blocked |
| Interactions | Blocked |
| Inventory | Blocked |
| Item drop/pickup | Blocked |
| Entity damage (give/receive) | Blocked |
| Vehicle enter | Blocked |
| Portal use | Blocked |
| Teleportation (non-plugin) | Blocked |
| Hand swap | Blocked |

---

## Database Tables

XLogin creates the following tables in XCore's shared database:

| Table | Purpose |
|-------|---------|
| `xlogin_accounts` | Player accounts (UUID, name, password hash, IPs, login count, premium flag, Mojang UUID) |
| `xlogin_sessions` | Active sessions for cross-server auto-login (UUID, IP, timestamp) |
| `xlogin_ip_bans` | Temporarily banned IPs (IP, reason, expiry) |
| `xlogin_2fa` | 2FA TOTP secrets (UUID, secret key, enabled date) |

All tables are created automatically on first startup. Column migrations (premium, mojang_uuid) are applied automatically.

---

## AuthMe Migration

To import accounts from an AuthMe database:

```
/xlogin import authme <table_name>
```

**Important notes:**
- The AuthMe table must be in the **same database** as XCore.
- Passwords are **not** migrated (AuthMe uses `$SHA$salt$hash`, XLogin uses a different format). Imported players will need to reset their password via `/changepassword` or an admin can use `/xlogin resetpassword <player> <newpassword>`.
- Only players who already exist in XCore's `xcore_players` table are imported.
- Duplicate accounts (already registered in XLogin) are skipped.

---

## Language File

All messages support [MiniMessage](https://docs.advntr.dev/minimessage/format.html) formatting with placeholders. Edit `lang.yml` to customize every message.

### Available Placeholders

| Placeholder | Used In |
|-------------|---------|
| `{password}` | Display only (not a real placeholder) |
| `{attempts}` / `{max}` | Login failed message |
| `{min}` / `{max}` | Password length messages |
| `{player}` | Admin notifications, info, accounts |
| `{ip}` | Admin notifications, last login info, accounts |
| `{date}` | Last login info, account info |
| `{time}` | Boss bar countdown |
| `{count}` | Account info (login count), import result |
| `{uuid}` | Account info |
| `{pattern}` | Name validation |
| `{server}` | Proxy redirect message |
| `{secret}` | 2FA setup |
| `{last_login}` | Accounts list |

---

## License

Part of the XCore ecosystem by [Xyness](https://github.com/Xyness).
