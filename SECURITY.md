# Security notes (hardened build)

This branch is RoALT 1.0.2 with the fixes from an independent audit. The full findings, with evidence, are in [`docs/SECURITY-AUDIT.md`](docs/SECURITY-AUDIT.md) (paths there are relative to this repository's root).

## Changes from upstream 1.0.2

| Change | Where | Why |
|---|---|---|
| Online counter off by default | `src-tauri/src/model.rs` (`online_count: false`) | Fresh installs no longer ping a third-party Cloudflare Worker every 2 minutes. The switch in Settings, About still turns it on. |
| Discord Rich Presence off by default | `model.rs` (`discord_presence: false`) | Fresh installs no longer publish the game and alt count on the Discord profile. Opt-in in Settings, Discord. |
| Self-update check off by default | `model.rs` (`self_update: false`) | An upstream release, signed by upstream's key, would replace this build. Settings, About still offers it. |
| `.ROBLOSECURITY` only to `https://(*.)roblox.com` | `src-tauri/src/roblox_api/mod.rs` (`cookie_host`, checked in `request`) | Defense in depth: no future endpoint can send the cookie anywhere else. Unit test `the_cookie_goes_only_to_roblox`. |
| Data-folder ACL granted by SID | `src-tauri/src/main.rs` (`user_sid`, `restrict_to_user`) | Previously relied on `%USERNAME%`. Unit test `reads_this_users_sid`. |
| Release builds log at Info | `main.rs` (log plugin) | Debug lines (account ids, local paths) only in debug builds. |
| Every GitHub Action pinned to a commit SHA; `permissions: {}` for winget | `.github/workflows/*.yml` | A moved tag or branch can't run new code with the signing key or tokens. |
| `hardened-build.yml` | `.github/workflows/` | Tests, clippy, npm audit and cargo audit run before the installer is built; SHA-256 sums are published with it. |

Settings an existing install has already saved keep their stored values. The new defaults apply to fresh installs and to settings never saved.

## Every external host RoALT contacts

| Host | When | Sends |
|---|---|---|
| `*.roblox.com` (users, auth, apis, games, friends, presence, thumbnails, economy, premiumfeatures, accountsettings, twostepverification, clientsettingscdn) | Account checks, launching, server lists | `.ROBLOSECURITY` for authenticated calls only (the only place the cookie goes) |
| `setup.rbxcdn.com` | Roblox install or update (click, or auto-update if turned on) | Nothing; downloads Roblox's installer, then checks its Authenticode signer "Roblox Corporation" |
| `*.rbxcdn.com` | Avatar and game images in the UI | Nothing |
| `api.github.com`, `github.com`, `*.githubusercontent.com` | News tab; self-update (opt-in) | Nothing; an update installer must pass ECDSA P-256 verification against the pinned key |
| `devforum.roblox.com`, `about.roblox.com` | News tab | Nothing |
| `discord.com` | Only if you paste a webhook | Account display names and events; never cookies, tickets or logs |
| `roalt-online.roalt-counterlive.workers.dev` | Only if "online count" is turned on | A random per-launch id and the number of running clients |
| `\\.\pipe\discord-ipc-*` (local) | Only if Rich Presence is turned on | Activity text to the local Discord app |

## Known remaining risks

- Cookies are only as safe as your Windows account. In the default (DPAPI) vault mode, any program running as you can decrypt the vault. Set a master password for protection while RoALT is locked.
- "Reveal / copy / export cookies" hand out working session tokens on purpose. The export file is plain text and is deleted after 10 minutes.
- Roblox's launch ticket goes on the Roblox client's command line, which programs running as you can read. That is how Roblox's launcher protocol works.
- The installer is not Authenticode-signed, so SmartScreen will warn. Check the SHA-256 against the build's `SHA256SUMS.txt`.
- A global low-level mouse hook (free resizing) and `SendInput` (anti-AFK) are used. There is no keyboard hook.
