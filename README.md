# Votech Edge Agent: Installation Guide

The Votech Edge Agent runs on a computer at your site. It reads your field
devices (IEC 60870-5-104, Modbus), stores every reading locally, and sends the
data securely to Votech. If the internet connection drops, data is kept on the
computer and sent automatically when the connection comes back.

The agent runs as a Docker container. **You don't need a GitHub or Docker
account:** the agent and these files are public and free to download from
<https://github.com/europower-enerji/edge-agent-release>:

| File                 | Purpose                                                                                     |
| -------------------- | ------------------------------------------------------------------------------------------- |
| `docker-compose.yml` | Describes the agent. You don't edit it, apart from the version when upgrading.              |
| `env.example`        | Template for your settings. You copy it to `.env` and fill in the values Votech gave you. |
| `README.md`          | This guide.                                                                                 |
| `source/`            | The complete source code of each version (the agent is free software, GPL-3.0-only). Not needed to install. |

Every step has a **Windows** and a **Linux** part. Follow the one for your computer.

---

## 1. Before you start

**Computer**

| Item   | Windows                                                                                             | Linux                                                                                         |
| ------ | --------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| System | Windows 10 or 11, 64-bit. **Not Windows Server**: Docker Desktop doesn't run there, use Linux (a virtual machine is fine). | x86-64 (industrial PC, server, virtual machine) or 64-bit ARM (Raspberry Pi 4/5 with a **64-bit** OS) |
| Memory | 4 GB RAM or more                                                                                    | 1 GB RAM or more                                                                              |
| Disk   | 10 GB free or more                                                                                  | 6 GB free or more (the data buffer can grow to 2 GB, see [Data buffer](#data-buffer))         |
| Clock  | Set automatically (Settings → Time & language → Date & time → **Set time automatically**)          | Synchronized with NTP                                                                         |

The clock must be right: every reading carries its time.

**Good to know on Windows**

- Docker Desktop starts when someone **signs in** to Windows. After a restart or a power cut, the agent runs again once a user has signed in. Use a computer that stays on, and sign in after every restart. For a computer that must run unattended, Linux is the better choice.
- Docker Desktop is free for personal use and for companies with fewer than 250 employees **and** less than USD 10 million annual revenue. Larger companies need a paid Docker subscription (<https://www.docker.com/pricing/>). On Linux, Docker is free for everyone.
- Modbus RTU over a serial (COM) port and [host networking](#host-networking) work on Linux only.

**Checks on Linux.** `uname -m` must print `x86_64` or `aarch64` (`armv7l` means a 32-bit OS, which isn't supported: install the 64-bit OS). `timedatectl` should show `System clock synchronized: yes`.

**Network.** The agent only makes outgoing connections. No incoming ports need to be opened.

| From the agent computer to                                          | Port                                               | Why                                                                         |
| ------------------------------------------------------------------- | -------------------------------------------------- | --------------------------------------------------------------------------- |
| `ghcr.io`, `pkg-containers.githubusercontent.com`                   | 443                                                | Download the agent software                                                 |
| `github.com`, `codeload.github.com`, `raw.githubusercontent.com`    | 443                                                | Download the installation files (only on the computer you download them on) |
| Your `API_BASE_URL` host                                            | 443                                                | Site configuration                                                          |
| Your `MQTT_HOST`                                                    | 8883 (or your `MQTT_PORT`)                         | Sending the readings                                                        |
| Your field devices (local network)                                  | 2404 (IEC 104), 502 (Modbus TCP), or as configured | Reading device data                                                         |

If your company firewall limits outgoing connections, ask your IT team to allow these. If your RTUs or PLCs only accept connections from known IP addresses, allow the **agent computer's IP address**.

**Values from Votech.** Have these ready: `AGENT_KEY` (shown once when the agent is registered on the Votech **Site connectors** page, where its sites are chosen), `API_BASE_URL`, `MQTT_HOST` (and `MQTT_PORT` if it isn't 8883). One agent can serve one site or several; you never list the sites here.

---

## 2. Install Docker (once per computer)

### Windows

1. Download **Docker Desktop for Windows** from <https://www.docker.com/products/docker-desktop/> and run the installer. Keep **Use WSL 2** ticked. Restart Windows when it asks.
2. Open **Docker Desktop** from the Start menu and accept the terms. Signing in is not needed: choose **Skip** or **Continue without signing in**. Wait until the bottom-left corner shows **Engine running**.
3. In Docker Desktop, open **Settings** (gear icon) → **General**, tick **Start Docker Desktop when you sign in to your computer**, and click **Apply**.

If the installer says virtualization is disabled, turn on virtualization (Intel VT-x or AMD-V) in the computer's BIOS/UEFI settings, or ask your IT team.

### Linux

Skip this if `docker compose version` already prints a version.

```bash
curl -fsSL https://get.docker.com | sudo sh
```

```bash
sudo systemctl enable --now docker
```

```bash
docker compose version
```

> The Linux commands in this guide use `sudo`. To run Docker without `sudo`, run
> `sudo usermod -aG docker $USER`, then log out and back in.

---

## 3. Get the files

### Windows

1. Open <https://github.com/europower-enerji/edge-agent-release> in your browser.
2. Click the green **Code** button, then **Download ZIP**.
3. Right-click the downloaded ZIP → **Extract All…**, type `C:\` as the destination, and click **Extract**.
4. In File Explorer, rename the folder `C:\edge-agent-release-main` to `C:\VotechAgent`. You now have `C:\VotechAgent\docker-compose.yml`.
5. **Open PowerShell in that folder:** open `C:\VotechAgent` in File Explorer, click the address bar, type `powershell` and press **Enter**.

Run every Windows command of this guide in that PowerShell window. To open it again later, repeat step 5.

### Linux

```bash
sudo mkdir -p /opt/edge-agent
```

All remaining commands are run **inside this folder**:

```bash
cd /opt/edge-agent
```

**If the agent computer has internet access,** download the two files directly:

```bash
sudo curl -fLO https://raw.githubusercontent.com/europower-enerji/edge-agent-release/main/docker-compose.yml
```

```bash
sudo curl -fLO https://raw.githubusercontent.com/europower-enerji/edge-agent-release/main/env.example
```

**Otherwise,** download them on your laptop (open each file on the page above and click **Download raw file**) and copy them over, for example with a USB stick or `scp`:

```bash
scp docker-compose.yml env.example user@agent-computer:/tmp/
```

Then, on the agent computer:

```bash
sudo mv /tmp/docker-compose.yml /tmp/env.example /opt/edge-agent/
```

---

## 4. Fill in your settings (`.env`)

Create your settings file from the template and open it in an editor.

### Windows

```powershell
Copy-Item env.example .env
```

```powershell
notepad .env
```

Replace the example values, press **Ctrl+S** to save, and close Notepad. Always open the file with this command: Notepad started from the Start menu may save it as `.env.txt`, which the agent doesn't read.

### Linux

```bash
sudo cp env.example .env
```

```bash
sudo nano .env
```

(In nano: edit, then **Ctrl+O**, **Enter** to save, and **Ctrl+X** to exit.)

### Required settings

| Variable       | What to enter                                                                                  | Example                                |
| -------------- | ---------------------------------------------------------------------------------------------- | -------------------------------------- |
| `AGENT_KEY`    | The secret key of **this** agent. Treat it like a password.                                    | `eak_live_8f3k2l9m4n5p6q7r_...`        |
| `API_BASE_URL` | Votech backend address. Must start with `https://`; the agent does not start with `http://`. | `https://api.example.com`              |
| `MQTT_HOST`    | Votech MQTT broker. Host name only, **no** `https://` and **no** `:port`.                    | `mqtt.example.com`                     |
| `MQTT_PORT`    | Broker port. Keep `8883` unless told otherwise.                                                | `8883`                                 |
| `MQTT_USE_TLS` | Encrypted connection. Must be `true`; the agent does not start without it.                     | `true`                                 |

Use exactly the values Votech gave you. Never use `localhost` or `127.0.0.1`: inside the agent's container, they mean the container itself.

### Optional settings (leave commented out unless Votech asks)

| Variable                         | When to use it                                                                                                                                                       |
| -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `TENANT_ID`                      | Your organisation id. The agent then refuses a key of another organisation. Leave it out for a Votech platform agent.                                              |
| `MQTT_USERNAME`, `MQTT_PASSWORD` | Only if Votech gave you separate MQTT credentials. Default: the key id of AGENT_KEY and AGENT_KEY                                                                  |
| `MQTT_CA_CERT`, `API_CA_CERT`    | Only if Votech gave you a CA certificate file (see [Private CA certificate](#private-ca-certificate))                                                              |
| `DB_PASSWORD`                    | Encrypts the local data buffer on disk. Decide **before the first start** and never change it afterwards; the agent can't open its buffer with a different password. |
| `LOG_LEVEL`                      | `Information` (default). Use `Debug` only when Votech support asks.                                                                                                |

To use an optional setting, remove the `# ` at the start of its line and enter a value.

### Example of a completed `.env`

```dotenv
AGENT_KEY=eak_live_8f3k2l9m4n5p6q7r_4d1c0b2a9f8e7d6c5b4a39281706f5e4d3c2b1a0f9e8d7c6b5a4938271605f4e
API_BASE_URL=https://api.example.com
MQTT_HOST=mqtt.example.com
MQTT_PORT=8883
MQTT_USE_TLS=true
```

### Format rules

- One `NAME=value` per line, with **no spaces** around `=`.
- **No comments after a value**. `DB_PASSWORD=   # secret` would set the password to `# secret`.
- No quotes are needed. If a value contains `$`, wrap it in single quotes: `AGENT_KEY='abc$123'`.
- The file must be named exactly `.env` and sit next to `docker-compose.yml`.

### Check the file

Windows:

```powershell
docker compose config --quiet
```

Linux (the first command also protects the file from other users):

```bash
sudo chmod 600 .env
```

```bash
sudo docker compose config --quiet
```

No output means the settings are complete. An error such as
`required variable AGENT_KEY is missing a value` names the setting to fix.

---

## 5. Start the agent

Windows:

```powershell
docker compose up -d
```

Linux:

```bash
sudo docker compose up -d
```

The first start downloads the agent (about 100 MB). From then on it starts by
itself: on Linux whenever the computer boots or recovers from a power cut, on
Windows whenever Docker Desktop starts (when someone signs in).

## 6. Check that it works

```bash
docker compose ps
```

(Linux: put `sudo` in front, as everywhere in this guide.) The `STATUS` column should say `Up`. Now watch the log:

```bash
docker compose logs -f
```

Within about a minute you should see these lines:

| Log line                                                                                                       | Meaning                                                |
| -------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| `SQLite WAL mode enabled. Database ready.`                                                                     | Local storage is ready                                 |
| `[MQTT] Connected successfully`                                                                                | Connected to Votech                                  |
| `[SYNC-CONFIG] Running configuration revision ... from the backend: N connection(s), N station(s), N point(s)` | Site configuration downloaded                          |
| `[IEC104-STARTDT] Connection ... confirmed STARTDT` or `[MODBUS-CONN] ... is open.`                            | Connected to your RTU or meter                         |
| `[STATUS] Online: N/N endpoints connected.`                                                                    | Every endpoint is connected (logged when this changes) |
| `[MQTT-V2] Sent N batches, N samples in 60 s; N waiting.`                                                      | Data is flowing (summarised once a minute)             |

Press **Ctrl+C** to stop watching. The agent keeps running.

**No `[MQTT] Connected successfully`?** Test whether this computer can reach the Votech broker. Put your `MQTT_HOST` in place of `mqtt.example.com`:

Windows (look for `TcpTestSucceeded : True`):

```powershell
Test-NetConnection mqtt.example.com -Port 8883
```

Linux (look for `succeeded` or `open`):

```bash
nc -zv mqtt.example.com 8883
```

If the test fails, a firewall blocks outgoing port 8883: ask your IT team to allow it to your `MQTT_HOST`. The agent keeps the readings meanwhile and sends them once the connection works.

---

## 7. Upgrading to a new version

Votech announces each new version, for example `5.0.0`.

1. Change the version at the end of the `image:` line in `docker-compose.yml`:
   - Windows: `notepad docker-compose.yml`
   - Linux: `sudo nano docker-compose.yml` (inside `/opt/edge-agent`)
   ```yaml
   image: ghcr.io/europower-enerji/edge-agent:5.0.0
   ```
   (Or download the new `docker-compose.yml` as in [step 3](#3-get-the-files): it already carries the newest version. Replace only that file and **keep your `.env`**.)
2. Apply the upgrade (Linux: with `sudo`):
   ```bash
   docker compose up -d
   ```
3. Check the log as in [step 6](#6-check-that-it-works).
4. Optionally, free disk space by removing the old version:
   ```bash
   docker image prune -f
   ```

Your buffered data and settings are kept during upgrades. When a new version
changes the local database, it first copies it to `edge.db.bak-<version>` in
the data volume; the copy holds everything that was not sent yet.

**Upgrading from 1.8.x.** Version 2 reads the new site configuration
(communication endpoints and signals) and needs Votech backend 4.63 or later;
Votech tells you when your site is ready. Go through version 2.0.2 first: it
sends the data that 1.8.x had not sent yet, and its log shows
`[LEGACY] Sent N legacy telemetry point(s); the legacy buffer is empty.` when it
is done (no `[LEGACY]` line at all means nothing was left). Then upgrade to the
latest version. Version 2.1 and later no longer
send 1.8.x data: if you upgrade straight to them, that data stays only in the
`edge.db.bak-<version>` copy.

**Rolling back:** put the previous version back on the `image:` line and run
`docker compose up -d`. If the release notes mention a database change,
contact Votech before rolling back: the previous version needs the
`edge.db.bak-<version>` copy put back in place of `edge.db`.

---

## Everyday commands

Run these in the agent folder: on Windows in PowerShell in `C:\VotechAgent`, on Linux in `/opt/edge-agent` with `sudo` in front.

| Task                             | Command                                       |
| -------------------------------- | --------------------------------------------- |
| Status                           | `docker compose ps`                           |
| Live log                         | `docker compose logs -f`                      |
| Save last hour of logs to a file | `docker compose logs --since 1h > edge-agent.log` |
| Restart                          | `docker compose restart`                      |
| Stop / start                     | `docker compose stop` / `docker compose start` |
| Apply changes made to `.env`     | `docker compose up -d`                        |

> `restart` does **not** pick up changes to `.env`. Use `up -d` after editing it.

---

## Troubleshooting

To test a connection named below: on Windows `Test-NetConnection <host or IP> -Port <port>`, on Linux `nc -zv <host or IP> <port>`.

| What you see                                                                                                       | What it means                                                                                                                                                                                       | What to do                                                                                                                                                                                                                                                                                                           |
| ------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Windows: `The term 'docker' is not recognized`                                                                    | Docker Desktop is not installed, or PowerShell was opened before it was installed                                                                                                                    | Install Docker Desktop ([step 2](#2-install-docker-once-per-computer)), then open a new PowerShell window.                                                                                                                                                                                                             |
| Windows: `error during connect` or `dockerDesktopLinuxEngine: The system cannot find the file specified`         | Docker Desktop isn't running                                                                                                                                                                        | Start Docker Desktop from the Start menu and wait for **Engine running**.                                                                                                                                                                                                                                              |
| `required variable AGENT_KEY is missing a value`                                                                   | `.env` not found, or the value is empty                                                                                                                                                             | Run the commands inside the agent folder. List the files (`dir` on Windows, `ls -la` on Linux): the file must be `.env`. On Windows, if you see `.env.txt`, run `Rename-Item .env.txt .env`.                                                                                                                        |
| `Error while connecting with host 'Unspecified/127.0.0.1:...'` or `'Unspecified/localhost:...'`                     | `MQTT_HOST` points at the agent's own computer, usually a value copied from another installation                                                                                                    | Set `MQTT_HOST`, `MQTT_PORT` and `MQTT_USE_TLS` to the values Votech gave you, then `docker compose up -d`.                                                                                                                                                                                                           |
| `[AUTH] The backend refused agent key ...` | Votech refused the agent; the line says why (`EDGE_KEY_REVOKED`, `TENANT_SUSPENDED`, ...). The sites keep running for now | If the line names a key code, enter the current key from the Site connectors page (and the right `TENANT_ID`, or remove it), then `docker compose up -d`. Without a code, check the proxy or firewall in front of Votech. |
| `[AUTH-FATAL] The backend refused agent key ...` | The key was refused for 10 minutes: it was revoked or rotated on the Site connectors page, or `TENANT_ID` names another organisation. Every site stopped; the agent keeps asking | Enter the current key (and the right `TENANT_ID`, or remove it), then `docker compose up -d`. |
| `[AGENT] Agent '...' serves 0 site(s)` or `[AGENT] Site ... is no longer assigned` | No site, or not this site, is assigned to the agent | Assign the sites on the Votech Site connectors page; the agent starts them within a minute, without a restart. |
| `EdgeAgent:SiteId (SITE_ID) is no longer used since agent 3.0` | An `.env` of agent 2.x | Remove `SITE_ID`, put the agent's new `AGENT_KEY` in `.env`, then `docker compose up -d`. |
| `MQTT_CA_CERT` or `API_CA_CERT` ... `does not exist` or `holds no certificate`, and the agent stops | The CA file is not where the setting points, or is not a certificate | Check the `./certs` volume line and the path in `.env`, then `docker compose up -d`. Clear the setting if Votech did not give you a CA file. |
| `[MqttTls] Refused the certificate of broker ...` or `[API-TLS] Refused the certificate of ...` | The server's certificate was not issued by your CA file, or does not name the address you connect to | Check that `MQTT_HOST` / `API_BASE_URL` use the name on the certificate, and that `ca.pem` is the one Votech gave you. Ask Votech if both look right. |
| `[MQTT-AUTH] MQTT broker rejected credentials`                                                                     | Wrong `AGENT_KEY`, or wrong `MQTT_USERNAME`/`MQTT_PASSWORD`                                                                                                                                         | Check those values.                                                                                                                                                                                                                                                                                                  |
| `[MQTT] Reconnect attempt #...` repeating                                                                          | The broker can't be reached                                                                                                                                                                         | Test the connection to your `MQTT_HOST` on port 8883 (see [step 6](#6-check-that-it-works)). Check that `MQTT_HOST` has no `https://` or `:port`. Data is buffered and sent once the connection returns.                                                                                                          |
| `[SYNC-CONFIG] The backend at ... cannot be reached. Running on the cached configuration.`                         | The backend can't be reached                                                                                                                                                                        | Check internet, DNS and firewall: test the connection to the host of `API_BASE_URL` on port 443. The agent keeps collecting with its last known configuration.                                                                                                                                                       |
| `[SYNC-CONFIG] No cached configuration for site ...`                                                               | First start, and the backend can't be reached                                                                                                                                                       | Nothing is read until the backend answers. Fix the connection to `API_BASE_URL`.                                                                                                                                                                                                                                     |
| `EdgeAgent:ApiBaseUrl must be an https address` or `EdgeAgent:Mqtt:UseTls must be true`                            | The agent refuses an unencrypted connection                                                                                                                                                         | Set `API_BASE_URL` to `https://...` and `MQTT_USE_TLS=true`, then `docker compose up -d`.                                                                                                                                                                                                                            |
| `[SYNC-CONFIG] Endpoint ... is not acquired: ...`                                                                  | A device in your site configuration uses a protocol this agent doesn't read                                                                                                                         | Nothing to do on the agent. Votech shows these devices as not acquired.                                                                                                                                                                                                                                            |
| `[SYNC-CONFIG] The server cannot be reached and the cached configuration was fetched at ...`                       | No contact with Votech for more than 72 hours, or the computer clock went back                                                                                                                    | Restore internet access and check the clock. The agent restarts itself and resumes.                                                                                                                                                                                                                                  |
| `[IEC104-CONN] Connection ... is down ... next attempt in ... s` or `[MODBUS-CONN] ... is down` repeating          | A field device can't be reached                                                                                                                                                                     | From the agent computer, test the connection to the device's IP on port 2404 (Modbus: `502`). Check the RTU's allowed-IP list. If your device network uses `172.16.x.x`–`172.31.x.x`, see [Host networking](#host-networking). Live values of that device show as Offline in Votech until it answers again.         |
| `... accepted the TCP connection but did not confirm STARTDT within t1`                                            | The RTU answers on the network but does not start a data session for this agent: another master (SCADA, an old agent) holds its session, or it only serves the master IP addresses configured in it | Stop the other master, or have the RTU's allowed-master list include the agent computer's IP address. The agent retries by itself, waiting 5 s, then up to about a minute between attempts, so a held RTU is not flooded.                                                                                            |
| `Another agent is running on ...: ...-lock is locked` at startup                                                   | A second agent was started on the same database (two installs, or a copy still running)                                                                                                             | Stop the other agent; its process id is in the `-lock` file. Only one agent may run per database, because both would connect to every RTU.                                                                                                                                                                           |
| `[IEC104-CONFIG] Connection ... receives common address N, which is no configured station`                         | The RTU sends a common address (ASDU address) that the site configuration doesn't have                                                                                                              | Tell Votech the address; the station address in the configuration must match the RTU.                                                                                                                                                                                                                              |
| `[IEC104-CA] Connection ...: the RTU refused the general interrogation of common address N` | The common address in Votech is not the RTU's; the line names the address the RTU sends | Set the RTU's **Common address** in Votech to the address named in the line. |
| `[IEC104-CONFIG] Connection ... has no point with an IEC 104 address`, or the RTU shows **Degraded** in Votech with "Connected, but none of its signals has an address" | The signals of that RTU have no IOA in Votech, so nothing is read | Give the signals their addresses (IO list import). |
| `[IEC104-CLOCK] Connection ...: the RTU clock says ...` | The RTU clock is wrong (a past year, ahead, or local time instead of UTC); the line says whether values get the time they arrive | Set the RTU clock (NTP or GPS on the RTU). The agent sets it only when clock synchronisation is configured for the RTU in Votech, because other masters may share the clock. |
| `[MODBUS-RX] ...: Unit N does not answer` or `[MODBUS-RX] ... unit N: HoldingRegister ...` with a Modbus exception | A Modbus unit id or register address in the configuration doesn't match the device                                                                                                                  | Tell Votech the unit id and register; the agent keeps reading the other units.                                                                                                                                                                                                                                     |
| `[CRITICAL-STORAGE-ALARM] The disk is at its free space floor`                                                     | The disk is almost full and the buffer holds values not yet sent; the oldest unsent values are deleted                                                                                             | Free disk space and restore the connection. See [Data buffer](#data-buffer).                                                                                                                                                                                                                                         |
| `exec format error` or `no matching manifest for linux/arm/v7`                                                     | 32-bit operating system                                                                                                                                                                             | Install a 64-bit OS (see [Before you start](#1-before-you-start)).                                                                                                                                                                                                                                                   |
| Linux: `permission denied ... docker.sock`                                                                         | Docker needs administrator rights                                                                                                                                                                   | Use `sudo`, or add your user to the `docker` group.                                                                                                                                                                                                                                                                  |
| Timestamps in Votech are wrong                                                                                   | The computer clock isn't synchronized                                                                                                                                                               | Windows: turn on **Set time automatically** (Settings → Time & language → Date & time). Linux: `sudo timedatectl set-ntp true`.                                                                                                                                                                                      |

---

## Optional configuration

### Private CA certificate

Only needed if Votech gives you a `ca.pem` file, because the Votech server uses its own certificate authority.

1. Create a folder `certs` in the agent folder (`C:\VotechAgent\certs` on Windows, `sudo mkdir -p /opt/edge-agent/certs` on Linux) and copy `ca.pem` into it.
2. In `docker-compose.yml`, uncomment the line `- ./certs:/etc/edge-agent/certs:ro`.
3. In `.env`, set `MQTT_CA_CERT=/etc/edge-agent/certs/ca.pem`, and `API_CA_CERT=/etc/edge-agent/certs/ca.pem` when the API uses the same CA (Votech tells you). Keep these paths as they are on Windows too: they are paths inside the container.
4. `docker compose up -d`

With a CA file, the agent trusts only that CA, not the certificates of the operating system. The server's certificate must also name the address in `MQTT_HOST` or `API_BASE_URL`. If the agent cannot read the file, it does not start, so a wrong path never falls back to weaker checks.

### Modbus RTU (serial / RS-485 adapter)

Linux only.

1. Find the adapter: `ls /dev/ttyUSB* /dev/ttyACM*`
2. In `docker-compose.yml`, uncomment the `devices:` and `group_add:` sections and
   set the correct device path (for example `/dev/ttyUSB0:/dev/ttyUSB0`).
3. `sudo docker compose up -d`

### Data buffer

Every value the agent reads is stored on disk until Votech confirms it. The
buffer holds up to 25 million values in at most 2 GB. How long that covers an
outage depends on how many values your site reads per second (Votech can tell
you):

| Values per second | Outage covered by the buffer |
| ----------------: | ---------------------------- |
|                25 | about 11 days                |
|                50 | about 6 days                 |
|               100 | about 3 days                 |
|               200 | about 1.5 days               |

The agent stops after 72 hours without Votech anyway (see
`[SYNC-CONFIG] The server cannot be reached ...` above). Values already sent are kept for a day; to stay
within its size, the buffer deletes only those. Unsent values are deleted only
when the disk itself is about to fill up (`[CRITICAL-STORAGE-ALARM]`): the
agent keeps 20 % of the disk (at least 2 GB) free. Votech sees how many
values were dropped or deleted in the agent's status.

### Host networking

Linux only. If your field-device network overlaps Docker's internal range (`172.16.0.0/12`),
uncomment `network_mode: host` in `docker-compose.yml` and run
`sudo docker compose up -d`.

---

## Uninstalling

```bash
docker compose down
```

(Linux: with `sudo`.) This removes the container but keeps the buffered data. To also delete all
local data (only when decommissioning the site, after Votech confirms all
data has arrived):

```bash
docker compose down -v
```

## Getting help

Send Votech support:

- the version (the `image:` line in `docker-compose.yml`)
- the output of `docker compose ps`
- the log file from `docker compose logs --since 1h > edge-agent.log`

**Never send your `.env` file or your `AGENT_KEY`.**
