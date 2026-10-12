Home Assistant custom integration for Daze EV charging products (Dazebox)
=========================================================================

[![hacs_badge](https://img.shields.io/badge/HACS-Custom-orange.svg)](https://github.com/hacs/integration)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

Home Assistant custom integration that exposes sensor data from a [Daze](https://www.dazeservice.com/) EV wallbox charger,
read from the same backend used by the official [webportal.dazeservice.com](https://webportal.dazeservice.com/) web app.

> **Fork of [rdndnl/ha-daze](https://github.com/rdndnl/ha-daze)** that adds sign-in for **accounts that log in with
> Google** (and therefore have no password). Email/password accounts keep working exactly as before.
> Based on upstream v0.3.1.

> **Unofficial project.** This integration is not affiliated with, endorsed by, or supported by Daze.
> It was built by reverse-engineering the public web portal's network traffic, since Daze does not publish
> a public API. It may break without notice if Daze changes their backend.

## What this fork adds

The upstream integration signs in with an email and a password. If you registered with the
"Sign in with Google" button there is no password, so that flow cannot work. This fork adds a second way
to set up the integration: you paste the **Cognito refresh token** that the web portal stores in your
browser after a Google login, and the integration exchanges it for fresh access tokens on its own
(`REFRESH_TOKEN_AUTH`, the same mechanism it already used to renew sessions).

- New setup menu: *Email and password* or *Google account (refresh token)*.
- The account email is read from the ID token, so you do not have to type it.
- If the stored token expires or is revoked, Home Assistant starts the normal re-authentication flow and
  asks for a new one.
- Everything else (sensors, buttons, polling) is unchanged from upstream.

> **Status: experimental.** The Google sign-in path is new and has had much less real-world testing than
> the email/password path. Please open an issue if something does not work.

## Features

Current release contains **read-only sensors**, no charge control (start/stop, eco mode) - that may come in a future release.

Per charging socket:

- Power (W) and current session energy (kWh)
- Charging current and AC voltage per phase (L1/L2/L3, when the installation is three-phase)
- Board and case temperature
- Fan status
- Status, EVSE state, and system error, decoded to human-readable values

Per wallbox (diagnostic):

- Wi-Fi SSID, software/firmware version
- Grid current per phase

Per site ("network" in Daze's terminology):

- Energy tariff and currency

## Requirements

- Home Assistant 2024.1.0 or newer
- A Daze account, either:
  - the email/password you use with the mobile App or at [webportal.dazeservice.com](https://webportal.dazeservice.com/), or
  - an account that signs in with Google (see [Google accounts](#google-accounts))

## Installation

### Via HACS (custom repository)

This integration is not in the default HACS store, so it has to be added as a custom repository.

The quickest way is this button, which opens the repository directly in your own Home Assistant's HACS:

<a href="https://my.home-assistant.io/redirect/hacs_repository/?owner=depgiuseppe&amp;repository=ha-daze&amp;category=Integration" target="_blank" rel="noreferrer noopener"><img src="https://my.home-assistant.io/badges/hacs_repository.svg" alt="Open your Home Assistant instance and open a repository inside the Home Assistant Community Store."></a>

Then click **Download**, and restart Home Assistant.

Manually, instead:

1. HACS → the `⋮` menu (top right) → **Custom repositories**
2. Repository: `https://github.com/depgiuseppe/ha-daze`, Category: **Integration**
3. Find "Daze" in HACS and install it
4. Restart Home Assistant

> This fork uses the same integration domain (`daze`) as upstream. If you previously installed
> `rdndnl/ha-daze`, remove it from HACS (and remove it from the custom repositories list) before installing
> this one, then restart Home Assistant.

### Manual

Copy `custom_components/daze` into your Home Assistant `config/custom_components/` directory,
then restart Home Assistant.

## Configuration

Settings → Devices & Services → **Add Integration** → search for **Daze**, then choose how you sign in.

### Email and password

Choose **Email and password** and enter your Daze account email and password.

### Google accounts

Choose **Google account (refresh token)** and paste your refresh token. To get it:

1. Open a **private/incognito window** and go to [webportal.dazeservice.com](https://webportal.dazeservice.com/).
2. Sign in with Google.
3. Open the browser developer tools (F12) → **Application** → **Local storage** →
   `https://webportal.dazeservice.com`.
4. Copy the value of the key that ends with `.refreshToken`
   (it looks like `CognitoIdentityServiceProvider.<client id>.Google_<number>.refreshToken`).
5. **Close the window without clicking "Log out".** Logging out revokes the token and the integration
   would stop working.

Notes:

- Treat the refresh token like a password. Do not paste it in issues, chats or screenshots.
- How long a refresh token stays valid is decided by Daze. When it expires Home Assistant will ask you to
  re-authenticate; repeat the steps above and paste a new one.
- Using a private window gives the integration its own session, separate from the one in your everyday browser.

### Options

Polling interval (default 60s) can be adjusted afterwards from the integration's **Configure** options.

## Limitations

- No charge start/stop or eco mode control yet (planned for a later release).
- `status`/`evse_state`/`system_error` labels were cross-referenced against the official web app's
  source and cover every value defined there, but only a handful have actually been observed live
  (mostly `standby`/`none`). If you see a sensor fall back to `unknown`, please open an issue with
  the raw value.
- Email/password accounts authenticate against the same AWS Cognito app client the web portal uses, via direct
  `USER_PASSWORD_AUTH` (no browser/webview involved). If Daze disables that auth flow this integration will need rework.
- Google accounts cannot be signed in automatically: Cognito only accepts the redirect addresses registered for the
  web portal, so the refresh token has to be copied from the browser by hand and renewed when it expires.

## Companion projects

Independent projects built on top of this integration, maintained by other people:

- **[DAZE Dashboard](https://github.com/fabiovit/daze-dashboard)** by [@fabiovit](https://github.com/fabiovit) -
  a Home Assistant sidebar panel for Daze wallboxes (live charging view, diagnostics, session statistics).

> These are not maintained by, affiliated with, or supported by this project - please report issues with them
in their own repositories.

## Development

### Setup

Requires Python 3.14+ and [`uv`](https://docs.astral.sh/uv/).

```bash
uv sync
```

This installs the dev/test tooling declared in `pyproject.toml`'s `dev` dependency group
(Home Assistant core, pytest, and the libraries used by `scripts/auth_spike.py`). The
integration itself has no runtime dependencies beyond what Home Assistant already
provides - see `custom_components/daze/manifest.json`.

### Running the tests

```bash
uv run python -m pytest tests/ -q
```

Use `python -m pytest`, not bare `uv run pytest` - the latter doesn't add the repo
root to `sys.path`, so the `custom_components` package won't be importable.

Tests run against fixture payloads captured from real HAR sessions
(`tests/fixtures/*.json`) plus Home Assistant's own test harness
(`pytest-homeassistant-custom-component`) - no live Daze account or network access is
needed to run them.

### Testing against a real Home Assistant instance

Symlink the integration into your HA config and restart HA:

```bash
ln -s "$(pwd)/custom_components/daze" /path/to/homeassistant/config/custom_components/daze
```

Then add it from Settings → Devices & Services as usual.

### `scripts/auth_spike.py`

A standalone script (not part of the shipped integration) used to verify which Cognito
auth flow the backend accepts, without going through Home Assistant. Useful if Daze
ever changes their auth setup and `custom_components/daze/auth.py` needs revisiting:

```bash
DAZE_EMAIL="you@example.com" DAZE_PASSWORD="..." uv run python scripts/auth_spike.py
```

## Contributing

Issues and PRs for this fork are welcome at [github.com/depgiuseppe/ha-daze](https://github.com/depgiuseppe/ha-daze/issues).
For anything that is not specific to Google sign-in, consider also reporting it to the original project at
[github.com/rdndnl/ha-daze](https://github.com/rdndnl/ha-daze/issues).


## Acknowledgements

This project is a fork of [rdndnl/ha-daze](https://github.com/rdndnl/ha-daze); all the reverse-engineering
and the integration itself are the work of its author and contributors.

Thanks to [MasterJKapp](https://github.com/jkuran) for the contribution in
charging control (start/stop/resume) + session sensors [PR #15](https://github.com/rdndnl/ha-daze/pull/15).

## License

[MIT](LICENSE), same as the original project.
