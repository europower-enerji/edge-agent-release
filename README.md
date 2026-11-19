# Votech Edge Agent: Installation Guide

The Votech Edge Agent is a small program that runs on a computer at your site. It reads your field devices (RTUs and
meters, over IEC 60870-5-104 or Modbus), writes every reading to the computer's disk, and sends the readings securely
to Votech over the internet. When the internet connection drops, the readings wait on the disk and are sent
automatically when the connection comes back. A connection problem delays readings; it does not lose them.

This guide takes you from an empty computer to a working agent. Follow the steps in order. Every step says who does
it, what to run, what you should see, and what to do when you see something else.

**Downloads.** You don't need a GitHub or Docker account. Everything is public and free at
<https://github.com/europower-enerji/edge-agent-release>. Every release there has these files:

| File                                   | What it is                                                                                           |
| -------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `votech-edge-agent-windows-x64.zip`    | The agent for Windows, installed as a Windows service. Needs no Docker and no other software.        |
| `docker-compose.yml`                   | The agent for Linux (and Docker Desktop). You don't edit it, apart from the version when upgrading. |
| `env.example`                          | The template of the settings file. Both installations use the same settings.                         |
| `README.md`                            | This guide.                                                                                          |
| `edge-agent-<version>-source.tar.gz`   | The complete source code (the agent is free software, GPL-3.0-only). Not needed to install.          |

---

## Contents

- [The installation at a glance](#the-installation-at-a-glance)
- [How the agent connects](#how-the-agent-connects)
- [Step 1: Get your values from Votech](#step-1-get-your-values-from-votech)
- [Step 2: Prepare the computer](#step-2-prepare-the-computer)
- [Step 3: Check the network from the agent computer](#step-3-check-the-network-from-the-agent-computer)
- [Step 4: Install the agent](#step-4-install-the-agent)
  - [On Windows: install the agent as a Windows service](#on-windows-install-the-agent-as-a-windows-service)
  - [On Linux: install the agent with Docker](#on-linux-install-the-agent-with-docker)
  - [On Windows 10 or 11 with Docker Desktop](#on-windows-10-or-11-with-docker-desktop)
  - [The settings file](#the-settings-file)
- [Step 5: Check that the agent works](#step-5-check-that-the-agent-works)
- [Step 6: Confirm with Votech and test an outage](#step-6-confirm-with-votech-and-test-an-outage)
- [Everyday operation](#everyday-operation)
- [Upgrading to a new version](#upgrading-to-a-new-version)
- [Changing a setting or the agent key](#changing-a-setting-or-the-agent-key)
- [Moving the agent to another computer](#moving-the-agent-to-another-computer)
- [Uninstalling](#uninstalling)
- [How readings are kept safe](#how-readings-are-kept-safe)
- [Problems and solutions](#problems-and-solutions)
- [Optional configuration](#optional-configuration)
- [Getting help](#getting-help)

---

## The installation at a glance

| Step                                                                                         | Who                                          | When                                                                                   | Takes      |
| -------------------------------------------------------------------------------------------- | -------------------------------------------- | -------------------------------------------------------------------------------------- | ---------- |
| [Step 1: Get your values from Votech](#step-1-get-your-values-from-votech)                   | Votech sends them to your IT team            | Before the installation day                                                            | -          |
| [Step 2: Prepare the computer](#step-2-prepare-the-computer)                                 | Your IT team                                 | Before the installation day                                                            | 30 minutes |
| [Step 3: Check the network from the agent computer](#step-3-check-the-network-from-the-agent-computer) | Your IT team, on the agent computer | At least one working day before the installation, so firewall changes can still be made | 15 minutes |
| [Step 4: Install the agent](#step-4-install-the-agent)                                       | An administrator of the agent computer       | Installation day                                                                       | 15 minutes |
| [Step 5: Check that the agent works](#step-5-check-that-the-agent-works)                     | The same person                              | Right after the installation                                                           | 10 minutes |
| [Step 6: Confirm with Votech and test an outage](#step-6-confirm-with-votech-and-test-an-outage) | The same person, with Votech on the phone | Installation day, and a short check the next day                                       | 20 minutes |

**Which installation?**

| The agent computer runs                         | Install                                                                                                   |
| ----------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| Windows Server 2016 or later, or Windows 10/11  | [As a Windows service](#on-windows-install-the-agent-as-a-windows-service). Recommended for every Windows computer: nothing else to install, and the agent starts after every restart without anyone signing in. |
| Linux                                           | [With Docker](#on-linux-install-the-agent-with-docker).                                                   |
| Windows 10/11 that already uses Docker Desktop  | [With Docker Desktop](#on-windows-10-or-11-with-docker-desktop) is possible, but the Windows service is simpler: Docker Desktop runs only while someone is signed in. |

---

## How the agent connects

Read this before you talk to your firewall team.

- **Only outgoing connections.** The agent connects out to Votech. Nothing connects in to your network: no inbound
  ports, no port forwarding, no VPN.
- **Two addresses at Votech, both on port 443 (HTTPS):**
  - the API (`API_BASE_URL`): the agent downloads its site configuration there;
  - the broker (`MQTT_HOST`): the agent sends the readings there, as MQTT inside a secure WebSocket
    (`wss://<MQTT_HOST>/mqtt`).

  Both can be the same name.
- **Encrypted and checked.** The connections use TLS 1.2 or 1.3, and the agent checks Votech's certificate. Your
  network must pass the certificate through unchanged: no TLS (SSL) inspection for the two Votech names.
- **No proxy server.** The agent connects directly. If this computer reaches the internet only through a proxy server,
  your IT team must allow a direct outgoing connection on port 443 to the two Votech names.
- **One long connection.** The connection to the broker stays open for days. The agent sends a small keep-alive every
  15 seconds. The firewall must allow WebSocket connections and must not close them after a fixed time.
- **Port 8883.** Only when Votech gives you that port: the agent then connects to the broker on port 8883 (MQTT over
  TLS, no web path). Everything else stays the same.
- **The devices.** The agent connects to each RTU or meter on your local network: IEC 104 on port 2404, Modbus TCP on
  port 502 (or the ports in your device list). Many RTUs accept only the IP addresses entered in them.

---

## Step 1: Get your values from Votech

**Who:** Votech prepares the values; your IT team receives them. **When:** before the installation day.

Votech registers this agent on its **Site connectors** page and chooses the site or sites it serves. You never enter
a site name or a site number: the agent asks Votech which sites it serves. You receive:

| Value                 | What it is                                                                                                                     | Looks like                                       |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------ |
| `AGENT_KEY`           | The secret key of this one installation. It is shown only once when Votech creates it, so it reaches you separately from the other values. Keep it in your password manager. | `eak_live_`, then 16 characters, `_`, and 64 characters (digits and the letters a to f) |
| `API_BASE_URL`        | The address of the Votech API                                                                                                  | `https://votech.example.com`                     |
| `MQTT_HOST`           | The name of the Votech broker: the name only, without `https://` and without a port                                           | `mqtt.votech.example.com`                        |
| `MQTT_PORT`           | The broker's port                                                                                                              | `443` (or `8883` only when Votech says so)       |
| `MQTT_USE_TLS`        | Encryption, always on                                                                                                          | `true`                                           |
| `MQTT_WEBSOCKET_PATH` | The broker's web path on port 443 (not used with port 8883)                                                                    | `/mqtt`                                          |
| The device list       | Every RTU or meter the agent reads: its IP address and port, or its COM port for Modbus RTU                                    | `192.168.10.21` port `2404`                      |

**Check:**

- You have all six values and the device list. If anything is missing, ask Votech now, not on the installation day.
- One key is for one installation. Never install two agents with the same key: they push each other off the broker
  every few seconds and neither works properly. For a second computer, ask Votech for a second key.
- Treat the key like a password: don't send it by email together with the other values, and never put it in a support
  ticket or a screenshot.

---

## Step 2: Prepare the computer

**Who:** your IT team. **When:** before the installation day.

### What the computer needs

|                     | Windows (installed as a Windows service)                                                  | Linux (installed with Docker)                                                       | Windows 10/11 with Docker Desktop                       |
| ------------------- | ------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------- | ------------------------------------------------------- |
| System              | Windows Server 2016, 2019, 2022 or 2025, or Windows 10/11; 64-bit; with or without desktop (Server Core works) | 64-bit Linux on x86-64, or on 64-bit ARM (Raspberry Pi 4 or 5 with a **64-bit** system) | Windows 10/11, 64-bit, with virtualization turned on    |
| Processor           | 2 cores                                                                                    | 2 cores (1 is enough for a small site)                                              | 2 cores                                                 |
| Memory              | 2 GB                                                                                       | 1 GB                                                                                | 4 GB                                                    |
| Free disk           | 6 GB on drive `C:`                                                                         | 6 GB under `/var/lib/docker`                                                        | 10 GB                                                   |
| Account             | A local administrator, for the installation                                                | `root`, or a user who may use `sudo`                                                | An administrator                                        |
| Physical or virtual | Both work                                                                                  | Both work                                                                           | Physical (in a virtual machine Docker Desktop needs nested virtualization) |

### Rules for the computer

1. **It stays on.** Turn off sleep and hibernation. Servers normally never sleep; on a Windows 10/11 computer run
   these two commands in an administrator PowerShell:

   ```powershell
   powercfg /change standby-timeout-ac 0
   ```

   ```powershell
   powercfg /change hibernate-timeout-ac 0
   ```

2. **It starts by itself after a power cut.** On a physical computer, set **Restore on AC power loss** (or **After
   power loss**) to **Power on** in the BIOS or UEFI settings. On a virtual machine, set it to start automatically
   with its host. A UPS is recommended.
3. **Its clock is right and set automatically.** Every reading carries its time; a wrong clock puts readings at the
   wrong time in Votech.
4. **The antivirus leaves the agent alone.** Allow the agent's folders, or the antivirus may slow down or remove the
   agent's files. On Windows these are `C:\Program Files\EdgeAgent` and `C:\ProgramData\EdgeAgent`; for Microsoft
   Defender, run in an administrator PowerShell:

   ```powershell
   Add-MpPreference -ExclusionPath 'C:\Program Files\EdgeAgent', 'C:\ProgramData\EdgeAgent'
   ```

   For any other antivirus, your IT team adds the same two folders in its console.
5. **Only one copy of the agent runs.** Don't clone a virtual machine that has the agent installed, and don't keep an
   old agent computer running next to a new one. Two agents with the same key push each other off the broker.
6. **Restarts are planned.** While the computer restarts (for example for Windows updates), nobody reads the devices,
   so plan restarts outside important hours. The agent itself starts again by itself about two minutes after Windows
   has started.
7. **Snapshots are not backups for the agent.** If the virtual machine is put back to an old snapshot, readings that
   were taken after the snapshot and not yet sent are lost. Readings already sent are safe at Votech.

### Check the computer on Windows

Open PowerShell (Start menu, type **PowerShell**, press **Enter**). Run each command and compare with **Expected**.

```powershell
Get-CimInstance Win32_OperatingSystem | Select-Object Caption, OSArchitecture, Version
```

**Expected:** `Caption` is Windows Server 2016 or later, or Windows 10/11; `OSArchitecture` is `64-bit`; `Version`
is `10.0.14393` or higher. **If not:** this computer can't run the agent; use another computer.

```powershell
"Memory: {0:N1} GB" -f ((Get-CimInstance Win32_ComputerSystem).TotalPhysicalMemory / 1GB); "Free on C: {0:N1} GB" -f ((Get-PSDrive C).Free / 1GB)
```

**Expected:** memory 2 GB or more; free on `C:` 6 GB or more. **If not:** add memory, or free space on `C:`. The
agent always keeps its data on `C:`; another drive can't be chosen.

```powershell
w32tm /query /status
```

**Expected:** the `Source:` line names a time server or a domain controller. **If it says `Local CMOS Clock` or
`Free-running System Clock`:** the clock is not synchronized. On a computer in a domain, your IT team fixes the
domain time. On a computer outside a domain, run these two commands in an administrator PowerShell (they need
outgoing UDP port 123 to `time.windows.com`), then run `w32tm /query /status` again:

```powershell
w32tm /config /manualpeerlist:"time.windows.com,0x8" /syncfromflags:manual /update
```

```powershell
Restart-Service w32time; w32tm /resync
```

**Only for Modbus RTU over a serial port:**

```powershell
[System.IO.Ports.SerialPort]::GetPortNames()
```

**Expected:** the COM port your RS-485 adapter uses, for example `COM3`. Give that name to Votech; it is part of the
site configuration. **If it is missing:** install the adapter's driver, then check again.

### Check the computer on Linux

```bash
uname -m
```

**Expected:** `x86_64` or `aarch64`. **If it says `armv7l` or `armhf`:** the system is 32-bit, which isn't
supported. Install a 64-bit system (on a Raspberry Pi: Raspberry Pi OS **64-bit**).

```bash
free -h; df -h /var
```

**Expected:** `Mem:` total 1 GB or more; `Avail` 6 GB or more.

```bash
timedatectl
```

**Expected:** `System clock synchronized: yes`. **If it says `no`:** run `sudo timedatectl set-ntp true`, wait a
minute, and check again. The computer needs outgoing UDP port 123 to a time server.

---

## Step 3: Check the network from the agent computer

**Who:** your IT team. **Where:** on the agent computer itself, not on a laptop: firewalls often treat every computer
differently. **When:** at least one working day before the installation, so firewall changes can still be made.

### What the agent computer must reach

| To                                                                                  | Port                                         | Why                                         | When                                    |
| ----------------------------------------------------------------------------------- | -------------------------------------------- | ------------------------------------------- | --------------------------------------- |
| The host of your `API_BASE_URL`                                                     | 443 (TCP)                                    | The site configuration                      | Always                                  |
| Your `MQTT_HOST`                                                                    | 443 (TCP), or 8883 when Votech gives you 8883 | The readings                                | Always                                  |
| Your devices (from the device list)                                                 | 2404 (IEC 104), 502 (Modbus TCP), or as listed | Reading the devices                       | Always                                  |
| A time server, unless the computer takes its time from a domain controller          | 123 (UDP)                                    | The clock                                   | Always                                  |
| `github.com`, `release-assets.githubusercontent.com`                                | 443 (TCP)                                    | Downloading the Windows package             | Installation and upgrades only          |
| `ghcr.io`, `pkg-containers.githubusercontent.com`                                   | 443 (TCP)                                    | Downloading the agent for Docker            | Installation and upgrades only          |
| `github.com`, `raw.githubusercontent.com`                                           | 443 (TCP)                                    | Downloading the Docker files                | Only on the computer you download them on |

Without internet access for downloads, bring the files on a USB stick or a network share: each installation below
says how.

Allow the Votech names in the firewall by name when it can. If your firewall accepts only IP addresses, ask Votech
for its IP addresses; Votech tells you before they change.

If your devices accept only known IP addresses, add the agent computer's IP address to each device.

### Check the network on Windows

Open PowerShell on the agent computer (administrator rights are not needed). First enter your values. Copy this
block into a text editor, put in your values from [Step 1: Get your values from Votech](#step-1-get-your-values-from-votech), then paste it into
PowerShell and press **Enter**:

```powershell
[Net.ServicePointManager]::SecurityProtocol = [Net.ServicePointManager]::SecurityProtocol -bor [Net.SecurityProtocolType]::Tls12
[Net.WebRequest]::DefaultWebProxy = $null
$ApiHost  = 'votech.example.com'        # the name in API_BASE_URL, without https://
$MqttHost = 'mqtt.votech.example.com'   # MQTT_HOST
$MqttPort = 443                         # MQTT_PORT
$MqttPath = '/mqtt'                     # MQTT_WEBSOCKET_PATH
```

The first line turns on TLS 1.2 for this window; older Windows versions don't use it by default. The second line
makes the checks ignore any proxy server, because the agent doesn't use one either. If you close the window, paste
the block again in the new one.

Then run the checks one after the other.

**Check 1: the names are found.**

```powershell
Resolve-DnsName $ApiHost; Resolve-DnsName $MqttHost
```

**Expected:** each name shows at least one `IPAddress`. **If you see `DNS name does not exist`:** check the spelling
first. If it is right, this computer's DNS server doesn't resolve internet names; this is common on networks that
reach the internet only through a proxy server. Your IT team must let the computer resolve the two Votech names.

**Check 2: the ports are open.**

```powershell
Test-NetConnection $ApiHost -Port 443 -WarningAction SilentlyContinue | Select-Object ComputerName, RemoteAddress, TcpTestSucceeded
```

```powershell
Test-NetConnection $MqttHost -Port $MqttPort -WarningAction SilentlyContinue | Select-Object ComputerName, RemoteAddress, TcpTestSucceeded
```

**Expected:** `TcpTestSucceeded` is `True` both times. **If it is `False`:** a firewall blocks the outgoing
connection. Your IT team allows outgoing TCP to that name on that port. If a browser on this computer still opens
websites, the computer goes out through a proxy server: the agent needs the direct connection anyway.

**Check 3: the Votech API answers, and the clock is right.**

```powershell
$r = Invoke-WebRequest -UseBasicParsing "https://$ApiHost/health" -TimeoutSec 15; "Votech API: HTTP $($r.StatusCode)"; "Clock difference to Votech: {0:N0} s" -f ([DateTime]::UtcNow - [DateTime]::Parse($r.Headers['Date']).ToUniversalTime()).TotalSeconds
```

**Expected:** `Votech API: HTTP 200`, and a clock difference between `-5` and `5` seconds.

| What you see instead                                                                                                     | What it means                                                         | What to do                                                                                                   |
| ------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| `The underlying connection was closed: An unexpected error occurred on a send` or `Could not create SSL/TLS secure channel` | TLS 1.2 is off in this window                                       | Paste the block with your values again, then repeat the check.                                               |
| `The remote certificate is invalid according to the validation procedure` or `Could not establish trust relationship`     | Something in your network replaces Votech's certificate (TLS inspection) | See the certificate check below.                                                                                             |
| `Unable to connect to the remote server` or `The operation has timed out`                                                | The firewall blocks or drops the connection                           | As in the port check above.                                                                                               |
| `(407) Proxy Authentication Required`                                                                                    | A proxy server sits in the way                                        | Your IT team allows a direct connection to the two Votech names.                                             |
| `(503) Server Unavailable` or another number                                                                             | Your network is fine; Votech's side has a problem                     | Tell Votech. Your installation can go ahead.                                                                 |
| A clock difference of more than 5 seconds                                                                                | This computer's clock is wrong                                        | Fix the clock as in [Check the computer on Windows](#check-the-computer-on-windows), then repeat the check.                         |

**Check 4: the certificates are Votech's own (no TLS inspection).**

```powershell
foreach ($t in @(@($ApiHost, 443), @($MqttHost, $MqttPort))) { try { $c = New-Object Net.Sockets.TcpClient($t[0], $t[1]); $s = New-Object Net.Security.SslStream($c.GetStream()); $s.AuthenticateAsClient($t[0]); "$($t[0]): issued by $($s.RemoteCertificate.Issuer)"; $c.Close() } catch { "$($t[0]): $($_.Exception.GetBaseException().Message)" } }
```

**Expected:** two lines, each `issued by` a public certificate authority (for example `Let's Encrypt`).

**If you see the name of your company, your firewall, your antivirus or another security product** (for example
`ESET SSL Filter CA`, Kaspersky, Bitdefender, Fortinet, Palo Alto, Check Point, Sophos, Zscaler), or
`The remote certificate is invalid`: something opens the encrypted connections on the way (TLS inspection). Your IT
team excludes the two Votech names from it: in the firewall's TLS inspection, or in the antivirus' SSL/TLS
filtering (often called "SSL/TLS protocol filtering" or "Scan encrypted connections"). Don't skip this: with Docker
the agent refuses such a certificate and can't connect at all; as a Windows service it may connect, because Windows
trusts the inspecting certificate, but inspection often cuts long connections, and the agent then reconnects again
and again.

**Check 5: the broker accepts a secure WebSocket.** Only with port 443 and a web path; with port 8883 the port check
and the certificate check are enough.

```powershell
$ws = New-Object Net.WebSockets.ClientWebSocket; $ws.Options.AddSubProtocol('mqtt'); try { if ($ws.ConnectAsync([Uri]"wss://${MqttHost}:$MqttPort$MqttPath", [Threading.CancellationToken]::None).Wait(15000)) { "Votech broker: $($ws.State)" } else { 'Votech broker: no answer within 15 s' } } catch { "Votech broker failed: $($_.Exception.GetBaseException().Message)" } finally { $ws.Dispose() }
```

**Expected:** `Votech broker: Open`.

| What you see instead                                         | What it means                                                       | What to do                                                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `The remote server returned an error: (404) Not Found`       | The name or the web path is not Votech's broker                     | Check `$MqttHost` and `$MqttPath` against your values. If they are right, tell Votech.        |
| `(400) Bad Request`, `(403) Forbidden` or `(405)`            | A web filter or proxy in your network blocks WebSocket connections  | Your IT team allows WebSocket connections to the Votech broker name.                           |
| `The remote name could not be resolved`                      | The name is not found                                               | As in the name check above.                                                                                 |
| `no answer within 15 s`                                      | A firewall drops the connection                                     | As in the port check above.                                                                                 |
| `The remote certificate is invalid`                          | TLS inspection                                                      | As in the certificate check above.                                                                                 |

**Check 6: the devices answer.** Run this once for every device in your device list, with its IP address and port:

```powershell
Test-NetConnection 192.168.10.21 -Port 2404 -WarningAction SilentlyContinue | Select-Object RemoteAddress, RemotePort, TcpTestSucceeded
```

**Expected:** `TcpTestSucceeded` is `True`. **If it is `False`:** check the IP address and port; check that the agent
computer has a route to the device network (on a computer with two network cards, the device network must be on one
of them); and check that the device accepts the agent computer's IP address. Some IEC 104 RTUs accept only one or two
connections at a time: if your current SCADA system already uses them all, the RTU refuses the agent too. Ask the
RTU's supplier to allow one more connection for the agent's IP address.

**Check 7: the download works.** Only if you will download the package on this computer:

```powershell
Test-NetConnection github.com -Port 443 -WarningAction SilentlyContinue | Select-Object ComputerName, TcpTestSucceeded
```

**Expected:** `True`. **If it is `False`:** bring the package on a USB stick or a network share instead; the
installation explains how.

### Check the network on Linux

Open a terminal on the agent computer. Enter your values from [Step 1: Get your values from Votech](#step-1-get-your-values-from-votech) (edit
the four lines, then paste them):

```bash
API_HOST=votech.example.com; MQTT_HOST=mqtt.votech.example.com; MQTT_PORT=443; MQTT_PATH=/mqtt
```

**Check 1: the names are found.**

```bash
getent hosts "$API_HOST" "$MQTT_HOST"
```

**Expected:** an IP address for each name. **If nothing is printed:** check the spelling; otherwise the DNS server
doesn't resolve internet names (see the same check [on Windows](#check-the-network-on-windows)).

**Check 2: the ports are open.**

```bash
timeout 5 bash -c "</dev/tcp/$API_HOST/443" && echo "Votech API port 443: open" || echo "Votech API port 443: BLOCKED"
```

```bash
timeout 5 bash -c "</dev/tcp/$MQTT_HOST/$MQTT_PORT" && echo "Votech broker port $MQTT_PORT: open" || echo "Votech broker port $MQTT_PORT: BLOCKED"
```

**Expected:** `open` twice. **If `BLOCKED`:** a firewall blocks the outgoing connection; your IT team allows it.

**Check 3: the Votech API answers, and the clock is right.**

```bash
curl -sS --noproxy '*' -o /dev/null -w 'Votech API: HTTP %{http_code}\n' --max-time 15 "https://$API_HOST/health"
```

```bash
echo "Clock difference to Votech: $(( $(date -u +%s) - $(date -u -d "$(curl -sI --noproxy '*' --max-time 15 "https://$API_HOST/health" | sed -n 's/^[Dd]ate: //p' | tr -d '\r')" +%s) )) s"
```

**Expected:** `Votech API: HTTP 200`, and a clock difference between `-5` and `5` seconds. **If curl prints
`SSL certificate problem`:** TLS inspection, see the certificate check below. **If it prints `Could not resolve host`,
`Connection timed out` or `Connection refused`:** as in the name and port checks above. **If it prints `HTTP 503` or another
number:** your network is fine; tell Votech.

**Check 4: the certificates are Votech's own (no TLS inspection).**

```bash
for h in "$API_HOST:443" "$MQTT_HOST:$MQTT_PORT"; do echo | timeout 10 openssl s_client -connect "$h" -servername "${h%:*}" 2>/dev/null | grep -E '^issuer=|Verify return code'; done
```

**Expected:** twice an `issuer=` of a public certificate authority (for example `Let's Encrypt`) and
`Verify return code: 0 (ok)`. **If the issuer names your company, your firewall or a security product:** your IT
team excludes the two Votech names from TLS inspection (see the same check [on Windows](#check-the-network-on-windows)).

**Check 5: the broker accepts a secure WebSocket.** Only with port 443 and a web path.

```bash
curl -s --noproxy '*' -o /dev/null -w 'Votech broker: HTTP %{http_code} (101 means OK)\n' --max-time 5 --http1.1 -H 'Connection: Upgrade' -H 'Upgrade: websocket' -H 'Sec-WebSocket-Version: 13' -H 'Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==' -H 'Sec-WebSocket-Protocol: mqtt' "https://$MQTT_HOST:$MQTT_PORT$MQTT_PATH"
```

**Expected:** `Votech broker: HTTP 101`. The command waits 5 seconds before it prints; that is normal. **If it says
`404`:** check the name and the path; if they are right, tell Votech. **`400`, `403` or `405`:** a web filter blocks
WebSocket connections. **`000`:** no connection at all, as in the port check above.

**Check 6: the devices answer.** Once for every device, with its IP address and port:

```bash
timeout 5 bash -c "</dev/tcp/192.168.10.21/2404" && echo "device: open" || echo "device: NOT REACHABLE"
```

**Expected:** `open`. **If not:** see the same check [on Windows](#check-the-network-on-windows).

**Check 7: the agent download works** (only if you download on this computer):

```bash
curl -s -o /dev/null -w 'Agent download: HTTP %{http_code} (401 means reachable)\n' --max-time 15 https://ghcr.io/v2/
```

**Expected:** `HTTP 401`. That number is right: it means the download server answers. **`000`:** blocked; your IT
team allows `ghcr.io` and `pkg-containers.githubusercontent.com` on port 443.

---

## Step 4: Install the agent

**Who:** an administrator of the agent computer. **When:** the installation day, after every check in
[Step 3: Check the network from the agent computer](#step-3-check-the-network-from-the-agent-computer) passed.

Choose one of the three ways below, following [Which installation?](#the-installation-at-a-glance). Never use two
ways on the same computer, and never install the agent on two computers with the same key.

### On Windows: install the agent as a Windows service

The agent runs as the Windows service **Votech Edge Agent**. No Docker, no WSL, no .NET installation is needed. It
starts after every restart without anyone signing in, and COM ports (Modbus RTU) work.

**Before you start:** sign in on the agent computer itself, at its screen or through Remote Desktop. Don't use
PowerShell remoting (`Enter-PSSession` or `Invoke-Command`): the installer opens Notepad for your settings, which
can't appear there, and the installer would wait for it forever.

#### 1. Open PowerShell as administrator

Start menu, type **PowerShell**, right-click **Windows PowerShell**, choose **Run as administrator**, and answer
**Yes**. (Or right-click the Start button and choose **Terminal (Admin)**.) Run every command of this installation
in this window, one after the other.

**Check:** the window's title starts with `Administrator:`. **If not:** close it and open it again as above.

#### 2. Turn on TLS 1.2 for this window

```powershell
[Net.ServicePointManager]::SecurityProtocol = [Net.ServicePointManager]::SecurityProtocol -bor [Net.SecurityProtocolType]::Tls12
```

Older Windows Server versions don't use TLS 1.2 by default, and GitHub needs it. The command prints nothing.

#### 3. Make an empty download folder

```powershell
Remove-Item C:\VotechSetup -Recurse -Force -ErrorAction SilentlyContinue; New-Item -ItemType Directory C:\VotechSetup | Out-Null
```

This folder holds only the download. Your settings and data are kept elsewhere, so emptying it is always safe.

#### 4. Download the package

The newest package is always at the same address. Download it (about 40 MB) and its checksum:

```powershell
Invoke-WebRequest -UseBasicParsing https://github.com/europower-enerji/edge-agent-release/releases/latest/download/votech-edge-agent-windows-x64.zip -OutFile C:\VotechSetup\votech-edge-agent-windows-x64.zip
```

```powershell
Invoke-WebRequest -UseBasicParsing https://github.com/europower-enerji/edge-agent-release/releases/latest/download/votech-edge-agent-windows-x64.zip.sha256 -OutFile C:\VotechSetup\votech-edge-agent-windows-x64.zip.sha256
```

Check that the download is complete and unchanged:

```powershell
if ((Get-Content C:\VotechSetup\votech-edge-agent-windows-x64.zip.sha256).Split(' ')[0] -eq (Get-FileHash C:\VotechSetup\votech-edge-agent-windows-x64.zip -Algorithm SHA256).Hash) { 'Package checksum: OK' } else { 'Package checksum: DIFFERENT' }
```

**Expected:** `Package checksum: OK`.

| What you see instead                                                         | What to do                                                                                                                         |
| ---------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `Could not create SSL/TLS secure channel` or `An unexpected error occurred on a send` | Run the command of "Turn on TLS 1.2 for this window" again, then the download.                                             |
| `Unable to connect to the remote server` or `The operation has timed out`    | This computer can't reach GitHub. Use the download without internet below.                                                          |
| `(404) Not Found`                                                            | A typing error in the address: copy the command again.                                                                             |
| `Package checksum: DIFFERENT`                                                | The download is incomplete or changed, or a new version came out between the two downloads. Do [Make an empty download folder](#3-make-an-empty-download-folder) and [Download the package](#4-download-the-package) again. If it stays different, don't install it: tell Votech. |

**Without internet on the agent computer:** on another computer, open
<https://github.com/europower-enerji/edge-agent-release/releases/latest> and download
`votech-edge-agent-windows-x64.zip` and `votech-edge-agent-windows-x64.zip.sha256`. Copy both files to
`C:\VotechSetup` on the agent computer (USB stick, a network share, or copy and paste over Remote Desktop). Then run
the checksum check above.

#### 5. Unpack the package and go to its folder

```powershell
Expand-Archive C:\VotechSetup\votech-edge-agent-windows-x64.zip -DestinationPath C:\VotechSetup
```

```powershell
Get-ChildItem C:\VotechSetup -Recurse | Unblock-File; Set-Location (Get-ChildItem C:\VotechSetup -Directory).FullName; Get-ChildItem
```

`Unblock-File` removes the "downloaded from the internet" mark that Windows puts on copied files, so PowerShell
doesn't ask about each one.

**Expected:** the prompt now ends in a folder such as `C:\VotechSetup\votech-edge-agent-4.6.0-windows-x64`, and the
list shows `agent`, `env.example`, `install.ps1`, `LICENSE`, `README.md` and `THIRD-PARTY-NOTICES.md`.

**If `agent` is missing,** or the list is empty: the unpacking failed or an antivirus removed files. Check the
antivirus' quarantine, allow the folders as in [Rules for the computer](#rules-for-the-computer), and start again from [Make an empty download folder](#3-make-an-empty-download-folder).

#### 6. Run the installer

```powershell
powershell -ExecutionPolicy Bypass -File .\install.ps1
```

`-ExecutionPolicy Bypass` allows this one script to run without changing the computer's script policy.

What happens:

1. The installer checks the package and creates the settings file `C:\ProgramData\EdgeAgent\.env` from the template.
2. **Notepad opens the settings file.** Replace the example values with your values from
   [Step 1: Get your values from Votech](#step-1-get-your-values-from-votech), keeping the names and following
   [the settings file rules](#the-settings-file). Press **Ctrl+S** to save, and close Notepad. The installer waits
   until you close Notepad.
3. The installer checks every value and prints `PASS` or `FAIL` for each. On a `FAIL` it opens Notepad again so you
   can correct it (three times at most).
4. It copies the program to `C:\Program Files\EdgeAgent`, creates the service **Votech Edge Agent**, starts it, waits
   15 seconds and shows the last log lines.

**Expected** (your values and version instead of these):

```text
== Package
PASS  package: Votech Edge Agent 4.6.0
PASS  settings file created from env.example: C:\ProgramData\EdgeAgent\.env
INFO  Notepad opens C:\ProgramData\EdgeAgent\.env. Enter the values Votech gave you, save (Ctrl+S) and close Notepad to continue.

== Settings
PASS  AGENT_KEY: eak_live_8f3k2l9m4n5p6q7r_... (the rest is not shown)
PASS  API_BASE_URL: https://votech.example.com
PASS  MQTT_HOST: mqtt.votech.example.com
PASS  MQTT_PORT: 443
PASS  MQTT_USE_TLS: true
PASS  MQTT_WEBSOCKET_PATH: /mqtt (the agent sends over HTTPS on port 443)

== Install the program
PASS  program in C:\Program Files\EdgeAgent (with install.ps1 for Status, Config and Uninstall)

== Service
PASS  service VotechEdgeAgent created
PASS  runs as NT SERVICE\VotechEdgeAgent, starts after boot, restarts after an error, reads C:\ProgramData\EdgeAgent\.env
PASS  permissions: Administrators and SYSTEM full; the service reads the program and .env and writes data and logs only

== Start
PASS  the agent is running

== Last 25 lines of C:\ProgramData\EdgeAgent\logs\agent-<date>.log
...
[MQTT] Connected successfully (SessionPresent=False).
...
RESULT: PASS
```

Then go on with [Step 5: Check that the agent works](#step-5-check-that-the-agent-works).

**A `FAIL` line in the settings check:**

| Line                                                                    | What to do                                                                                                                  |
| ----------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `FAIL  AGENT_KEY: enter the key of this agent from the Site connectors page` | The example key is still there. Paste your key.                                                                        |
| `FAIL  AGENT_KEY: not a whole agent key ...`                            | Part of the key is missing, or a space or line break got in. Copy the whole key again, on one line.                         |
| `FAIL  API_BASE_URL must start with https:// ...`                       | Write the address with `https://` in front.                                                                                 |
| `FAIL  API_BASE_URL is still the example ...`, `FAIL  MQTT_HOST: enter the host name Votech gave you` | Replace the example value with yours.                                                         |
| `FAIL  MQTT_HOST: the host name only, no https:// and no :port ...`     | Remove `https://`, `wss://`, `/mqtt` and any `:443` from `MQTT_HOST`.                                                       |
| `FAIL  MQTT_HOST: localhost or 127.x is this computer, not Votech`      | Enter Votech's broker name.                                                                                                 |
| `FAIL  MQTT_PORT must be a port number ...`                             | Enter only the number, for example `443`.                                                                                   |
| `FAIL  MQTT_USE_TLS must be true ...`                                   | Write `MQTT_USE_TLS=true`.                                                                                                  |
| `FAIL  MQTT_WEBSOCKET_PATH must be a path such as /mqtt ...`            | Enter only the path, for example `/mqtt`, not a whole address.                                                              |
| `WARN  MQTT_WEBSOCKET_PATH is set and MQTT_PORT is ...`                 | A web path is set, but the port is not 443. Use the port Votech gave you: with 443 keep the path; with 8883 put `#` in front of `MQTT_WEBSOCKET_PATH`. |
| `the settings still have FAIL lines ...` and `RESULT: FAIL`             | Three tries were used up. Run `powershell -ExecutionPolicy Bypass -File .\install.ps1` again: it keeps what you entered and continues. |

**The installer stops with `RESULT: FAIL` elsewhere:**

| Line                                                                    | What it means and what to do                                                                                                |
| ----------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `this needs an elevated PowerShell`                                     | PowerShell is not running as administrator. Do "Open PowerShell as administrator" again.                                    |
| `this agent needs 64-bit Windows`                                       | The computer can't run the agent; use another one.                                                                          |
| `no agent\Votech.Edge.Service.exe next to this script`                  | The installer is not running from the unpacked folder, or files are missing. Start again from [Make an empty download folder](#3-make-an-empty-download-folder); check the antivirus' quarantine. |
| `running scripts is disabled on this system`                            | Start the installer exactly as shown, with `-ExecutionPolicy Bypass`. If it still appears, a company policy (Group Policy) blocks scripts and overrides that switch: your IT team allows scripts for this installation. `Get-ExecutionPolicy -List` shows the policy; `MachinePolicy` or `UserPolicy` set means Group Policy. |
| `copying the program to C:\Program Files\EdgeAgent failed (robocopy exit ...)` | A program file is open: close File Explorer windows on that folder, wait a minute (antivirus scans), and run the installer again. |
| `icacls could not set the permissions ...` or `sc.exe ... failed`       | Windows refused the change. Sign in with a local administrator account, or ask your IT team whether a policy blocks new services. |
| `the service did not start: ... logon failure`                          | A company policy controls **Log on as a service**. Your IT team adds `NT SERVICE\ALL SERVICES` to that policy, then run the installer again. |
| `FAIL  the agent is 'Stopped': see the log lines below`                 | The agent started and stopped again. The log lines below it say why; find them in [Problems and solutions](#problems-and-solutions). |

#### Where everything is

| What                                     | Where                                                                    |
| ---------------------------------------- | ------------------------------------------------------------------------ |
| The service                              | `VotechEdgeAgent`, shown as **Votech Edge Agent** in **Services** (`services.msc`) |
| The program, with `install.ps1`          | `C:\Program Files\EdgeAgent`                                             |
| The settings file                        | `C:\ProgramData\EdgeAgent\.env` (only administrators can open it)        |
| The data buffer                          | `C:\ProgramData\EdgeAgent\data`                                          |
| The logs, one file per day, kept 7 days  | `C:\ProgramData\EdgeAgent\logs` (only administrators can open them)      |

The service runs as its own account, `NT SERVICE\VotechEdgeAgent`. It has no password and no administrator rights; it
can only read the program and the settings, and write its data and logs. It starts automatically about two minutes
after Windows starts, and Windows restarts it after an error (after 10, 30 and 60 seconds).

You can delete `C:\VotechSetup` after the installation.

### On Linux: install the agent with Docker

Every command in this part is run in a terminal on the agent computer.

#### 1. Install Docker

Skip this if `docker compose version` already prints a version.

```bash
curl -fsSL https://get.docker.com | sudo sh
```

```bash
sudo systemctl enable --now docker
```

**Check:**

```bash
docker compose version
```

**Expected:** `Docker Compose version v2...`. **If `docker: command not found`:** the installation failed; read its
messages. Without internet on this computer, install Docker from your Linux distribution's packages instead
(`docker.io` and `docker-compose-v2` on Ubuntu or Debian).

The commands below use `sudo`. To use Docker without `sudo`, run `sudo usermod -aG docker $USER`, then sign out and
in again.

#### 2. Make the agent folder and get the files

```bash
sudo mkdir -p /opt/edge-agent && cd /opt/edge-agent
```

All remaining commands are run **inside this folder**. In a new terminal, run `cd /opt/edge-agent` first.

**With internet on this computer**, download the two files:

```bash
sudo curl -fLO https://raw.githubusercontent.com/europower-enerji/edge-agent-release/main/docker-compose.yml
```

```bash
sudo curl -fLO https://raw.githubusercontent.com/europower-enerji/edge-agent-release/main/env.example
```

**Without internet,** download them on your laptop (open each file on
<https://github.com/europower-enerji/edge-agent-release> and click **Download raw file**), copy them over, for
example with `scp docker-compose.yml env.example user@agent-computer:/tmp/`, and then on the agent computer:

```bash
sudo mv /tmp/docker-compose.yml /tmp/env.example /opt/edge-agent/
```

The agent itself is downloaded from `ghcr.io` when it starts. Without any internet for downloads, ask Votech for the
agent as a file (`docker load`).

**Check:**

```bash
ls -l /opt/edge-agent
```

**Expected:** `docker-compose.yml` and `env.example`.

#### 3. Fill in the settings file

```bash
sudo cp env.example .env && sudo chmod 600 .env && sudo nano .env
```

Replace the example values with your values from [Step 1: Get your values from Votech](#step-1-get-your-values-from-votech), following
[the settings file rules](#the-settings-file). In nano: **Ctrl+O** and **Enter** save, **Ctrl+X** exits.
`chmod 600` lets only the administrator read the file, because it holds your key.

#### 4. Check the settings file

```bash
sudo docker compose config --quiet
```

**Expected:** no output. **If you see `required variable AGENT_KEY is missing a value`** (or another name): that
setting is empty, or the file is not named exactly `.env` in `/opt/edge-agent`.

#### 5. Start the agent

```bash
sudo docker compose up -d
```

The first start downloads the agent (about 100 MB). From then on it starts by itself whenever the computer starts.

**Check:**

```bash
sudo docker compose ps
```

**Expected:** the `edge-agent` line has `Up` in its `STATUS` column. **If it says `Restarting`:** the agent stops
right after starting; `sudo docker compose logs --tail 30` shows why (see
[Problems and solutions](#problems-and-solutions)). **If the download fails** (`pull access denied`,
`i/o timeout`, `no matching manifest for linux/arm/v7`): see [Problems and solutions](#problems-and-solutions).

Then go on with [Step 5: Check that the agent works](#step-5-check-that-the-agent-works).

### On Windows 10 or 11 with Docker Desktop

Only when your company already uses Docker Desktop on this computer; otherwise
[install the agent as a Windows service](#on-windows-install-the-agent-as-a-windows-service).

Good to know first:

- Docker Desktop starts only when someone **signs in** to Windows. After every restart or power cut, the agent runs
  again only once a user has signed in.
- Docker Desktop is free for personal use and for companies with fewer than 250 employees **and** less than USD 10
  million annual revenue; larger companies need a paid Docker subscription.
- Modbus RTU over a COM port doesn't work with Docker Desktop.

#### 1. Install Docker Desktop

1. Download **Docker Desktop for Windows** from <https://www.docker.com/products/docker-desktop/> and run the
   installer. Keep **Use WSL 2** ticked. Restart Windows when it asks.
2. Open **Docker Desktop** from the Start menu and accept the terms. Signing in is not needed: choose **Skip** or
   **Continue without signing in**. Wait until the bottom-left corner shows **Engine running**.
3. In Docker Desktop, open **Settings** (gear icon), then **General**, tick **Start Docker Desktop when you sign in
   to your computer**, and click **Apply**.

**If the installer says virtualization is disabled:** turn on virtualization (Intel VT-x or AMD-V) in the BIOS or
UEFI settings.

#### 2. Get the files

1. Open <https://github.com/europower-enerji/edge-agent-release> in your browser.
2. Click the green **Code** button, then **Download ZIP**.
3. Right-click the downloaded ZIP, choose **Extract All...**, type `C:\` as the destination, and click **Extract**.
4. In File Explorer, rename the folder `C:\edge-agent-release-main` to `C:\VotechAgent`.
5. Open PowerShell in that folder: open `C:\VotechAgent` in File Explorer, click the address bar, type `powershell`
   and press **Enter**. Run every command below in that window.

**Check:** `dir` lists `docker-compose.yml` and `env.example`.

#### 3. Fill in the settings file

```powershell
Copy-Item env.example .env; notepad .env
```

Replace the example values with yours, following [the settings file rules](#the-settings-file), press **Ctrl+S** and
close Notepad. Always open the file with this command: Notepad started from the Start menu may save it as
`.env.txt`, which the agent doesn't read.

#### 4. Check the settings file

```powershell
docker compose config --quiet
```

**Expected:** no output. **If you see `required variable AGENT_KEY is missing a value`:** that setting is empty, or
the file is not named `.env`; `dir` shows the real name (`Rename-Item .env.txt .env` fixes the usual mistake).

#### 5. Start the agent

```powershell
docker compose up -d
```

**Check:** `docker compose ps` shows `Up`. Then go on with
[Step 5: Check that the agent works](#step-5-check-that-the-agent-works).

### The settings file

All three installations use the same settings file, `.env`. A completed file looks like this:

```dotenv
AGENT_KEY=eak_live_8f3k2l9m4n5p6q7r_4d1c0b2a9f8e7d6c5b4a39281706f5e4d3c2b1a0f9e8d7c6b5a4938271605f4e
API_BASE_URL=https://votech.example.com
MQTT_HOST=mqtt.votech.example.com
MQTT_PORT=443
MQTT_USE_TLS=true
MQTT_WEBSOCKET_PATH=/mqtt
```

Only when Votech gives you port 8883 (a direct broker connection), use it and put `#` in front of the web path:

```dotenv
MQTT_PORT=8883
# MQTT_WEBSOCKET_PATH=/mqtt
```

**Rules:**

- One `NAME=value` per line, with **no spaces** around `=`.
- **No comments after a value.** `DB_PASSWORD=   # secret` would set the password to `# secret`. Comments go on
  their own line, starting with `#`.
- No quotes are needed. If a value contains `$`, wrap it in single quotes: `AGENT_KEY='abc$123'`.
- Use exactly the values Votech gave you. Never `localhost` or `127.0.0.1`: they mean the agent's own computer.
- The key is one line: no spaces or line breaks inside it.

**Optional settings.** Leave them with `#` in front unless Votech asks. To use one, remove the `# ` at the start of its
line and enter a value.

| Setting                          | When to use it                                                                                                                                               |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `TENANT_ID`                      | Your organisation's id. The agent then refuses a key of another organisation. Leave it out for a Votech platform agent.                                     |
| `MQTT_USERNAME`, `MQTT_PASSWORD` | Only if Votech gave you separate broker credentials. Without them the agent signs in with its key.                                                          |
| `MQTT_CA_CERT`, `API_CA_CERT`    | Only if Votech gave you a certificate file (see [Private CA certificate](#private-ca-certificate)).                                                         |
| `DB_PASSWORD`                    | Encrypts the data buffer on disk. Decide **before the first start** and never change it afterwards: the agent can't open its buffer with another password. |
| `LOG_LEVEL`                      | `Information` (the default). Use `Debug` only when Votech support asks.                                                                                     |

---

## Step 5: Check that the agent works

**Who:** the person who installed the agent. **When:** right after the installation.

### Open the live log

**Windows service** (in PowerShell as administrator; the logs are readable for administrators only):

```powershell
Get-Content (Get-ChildItem C:\ProgramData\EdgeAgent\logs\agent-*.log | Sort-Object LastWriteTime | Select-Object -Last 1).FullName -Tail 40 -Wait
```

**Docker** (Linux: in `/opt/edge-agent` with `sudo` in front; Docker Desktop: in `C:\VotechAgent`):

```bash
docker compose logs -f
```

Press **Ctrl+C** to stop watching. The agent keeps running.

### What you should see, in this order, within two minutes

| Log line                                                                                                        | What it means                                       | If it doesn't come                                                                                                         |
| --------------------------------------------------------------------------------------------------------------- | --------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| `[AGENT] Edge agent 4.6.0 with key eak_live_...: asking https://... which sites it serves.`                     | The agent started; the line shows its version       | The agent didn't start: see the lines before it, and [Problems and solutions](#problems-and-solutions).                     |
| `[AGENT] Agent '...' serves N site(s) of N organisation(s): ...`                                                | Votech accepted the key and named the sites         | `[AUTH] ...` lines: the key was refused, see [Problems and solutions](#problems-and-solutions). `serves 0 site(s)`: Votech hasn't assigned a site to this agent yet; tell Votech. |
| `[STORAGE] Database ... ready (WAL).`                                                                           | The data buffer is ready                            | See [Problems and solutions](#problems-and-solutions).                                                                     |
| `[SYNC-CONFIG] Running configuration revision N from the backend: N connection(s), N station(s), N point(s)`     | The site configuration arrived                      | `cannot be reached`: as in the API check of [Step 3: Check the network from the agent computer](#step-3-check-the-network-from-the-agent-computer).                           |
| `[MQTT] Connecting to broker wss://mqtt.votech.example.com:443/mqtt (TLS=True, ClientId=edge-...)...`           | The address the agent uses                          | Check that the address matches your values (with port 8883: `mqtt.votech.example.com:8883`).                               |
| `[MQTT] Connected successfully (SessionPresent=...).`                                                           | The broker accepted the agent                       | See "the agent doesn't connect to the broker" in [Problems and solutions](#problems-and-solutions).                        |
| `[IEC104-STARTDT] Connection ... confirmed STARTDT on attempt 1.` or `[MODBUS-CONN] ... is open.`, once per device | The device answers                               | `[IEC104-CONN] ... is down` or `[MODBUS-CONN] ... is down` repeating: see "devices" in [Problems and solutions](#problems-and-solutions). |
| `[STATUS] Online: N/N endpoints connected.`                                                                     | Every device is connected                           | `Degraded: 2/3 endpoints connected.` means one or more devices don't answer yet.                                           |
| `[MQTT-V2] Sent N batches, N samples in 60 s; 0 waiting.` (once a minute)                                       | Readings flow to Votech, nothing waits              | A `waiting` number that keeps growing means readings can't be sent fast enough: check the connection lines above it.       |

With several sites, the configuration and device lines appear once for each site.

### Check after 10 minutes: the connection stays up

A good connection stays up for days. Count the connection drops since the start:

**Windows service** (PowerShell as administrator):

```powershell
$log = (Get-ChildItem C:\ProgramData\EdgeAgent\logs\agent-*.log | Sort-Object LastWriteTime | Select-Object -Last 1).FullName; "Connection drops in today's log: " + @(Select-String -Path $log -Pattern '\[MQTT\] Disconnected').Count
```

**Docker** (Linux with `sudo`):

```bash
docker compose logs --since 10m | grep -c '\[MQTT\] Disconnected'
```

**Expected:** `0`. **If the number grows:** see "the connection drops again and again" in
[Problems and solutions](#problems-and-solutions). The readings are not lost meanwhile; they wait in the buffer.

---

## Step 6: Confirm with Votech and test an outage

**Who:** the person who installed the agent, with Votech on the phone. **When:** the installation day, and a short
check the next day.

### Confirm with Votech

Tell Votech the agent computer's name and IP address, the agent version (from the first `[AGENT]` line), and the time
of the installation. Votech checks on its side and tells you:

1. **Site connectors page:** the agent is **Online**, with its version and its sites, and every device is connected.
2. **The site page:** live values change, with the current time.

If Votech sees something different from your log, compare the times: the log uses UTC (world time), not local time.

### Test an outage (recommended, 10 minutes)

This proves on your own network that readings taken while Votech can't be reached are kept and sent afterwards.

**Windows service** (PowerShell as administrator). First block the connection to Votech for the agent computer; the
devices stay reachable. Enter your two Votech names in the first line:

```powershell
$ApiHost = 'votech.example.com'; $MqttHost = 'mqtt.votech.example.com'; $VotechIps = @($ApiHost, $MqttHost | ForEach-Object { (Resolve-DnsName $_ | Where-Object IPAddress).IPAddress }) | Sort-Object -Unique; $VotechIps
```

**Expected:** one or more IP addresses. Then:

```powershell
New-NetFirewallRule -DisplayName 'Votech outage test' -Direction Outbound -Action Block -RemoteAddress $VotechIps | Out-Null; Restart-Service VotechEdgeAgent
```

[Open the live log](#open-the-live-log). Within a minute you see `[MQTT] Disconnected` and
`[MQTT] Reconnect attempt #...` lines, and
`[SYNC-CONFIG] The backend at ... cannot be reached. Running on the cached configuration.`. The devices stay
connected and the agent keeps reading them into its buffer. Votech now sees the agent as **Offline**.

**If you see `[MQTT] Connected successfully` during the test:** the Windows firewall is turned off on this computer,
so the test can't block anything. Remove the rule (below) and skip the test.

After 5 minutes, remove the block:

```powershell
Remove-NetFirewallRule -DisplayName 'Votech outage test'
```

**Expected within about a minute:**

1. `[MQTT] Reconnected successfully on attempt #N.`
2. `[MQTT-V2] N samples waited since ...: live readings go first, and this backlog follows ...`
3. A few minutes later: `[MQTT-V2] The backlog is sent: N samples in N s, behind the live readings.`

Votech then sees the agent **Online** again, with the 5 minutes of readings filled in on the site's charts.

**Never forget the last command:** while the rule exists, the agent can't reach Votech. To check that it is gone:
`Get-NetFirewallRule -DisplayName 'Votech outage test'` must answer `No MSFT_NetFirewallRule objects found`.

**Linux or Docker Desktop:** if the internet connection and the device network are separate, unplug the internet
connection of the agent computer for 5 minutes and watch the same lines. Otherwise skip the test.

### Check the next day

1. The connection drops are still `0` ([Check after 10 minutes](#check-after-10-minutes-the-connection-stays-up)).
2. The newest `[MQTT-V2] Sent ...` line says `0 waiting`.
3. Votech confirms the agent stayed **Online** overnight.

Write down for your records: the agent computer's name and IP address, the installation way (Windows service or
Docker), the agent version, where the key is stored, and your contact at Votech.

---

## Everyday operation

### Windows service

Run these in PowerShell as administrator.

| Task                                            | Command                                                                                                    |
| ----------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| Status, settings check and the last log lines   | `powershell -ExecutionPolicy Bypass -File "C:\Program Files\EdgeAgent\install.ps1" -Action Status`        |
| Change settings (Notepad, check, restart)       | `powershell -ExecutionPolicy Bypass -File "C:\Program Files\EdgeAgent\install.ps1" -Action Config`        |
| Restart                                         | `Restart-Service VotechEdgeAgent`                                                                          |
| Stop / start                                    | `Stop-Service VotechEdgeAgent` / `Start-Service VotechEdgeAgent`                                           |
| Live log (Ctrl+C stops watching)                | `Get-Content (Get-ChildItem C:\ProgramData\EdgeAgent\logs\agent-*.log \| Sort-Object LastWriteTime \| Select-Object -Last 1).FullName -Tail 20 -Wait` |

### Docker

Run these in the agent folder: `/opt/edge-agent` with `sudo` in front on Linux, `C:\VotechAgent` in PowerShell with
Docker Desktop.

| Task                                       | Command                                              |
| ------------------------------------------ | ---------------------------------------------------- |
| Status                                     | `docker compose ps`                                  |
| Live log                                   | `docker compose logs -f`                             |
| Save the last hour of logs to a file       | `docker compose logs --since 1h > edge-agent.log`    |
| Restart                                    | `docker compose restart`                             |
| Stop / start                               | `docker compose stop` / `docker compose start`       |
| Apply changes made to `.env`               | `docker compose up -d`                               |

`docker compose restart` does **not** read changes to `.env`; use `docker compose up -d` after editing it.

A stopped agent reads nothing. Stop it only for maintenance, and start it again right after.

---

## Upgrading to a new version

Votech announces each new version and says whether it needs anything else. Settings and buffered readings are kept
during upgrades. When a new version changes the data buffer's format, it first copies the buffer to
`edge.db.bak-<version>` next to it; that copy holds everything that was not sent yet.

### Windows service

Do the installation steps again, in a new PowerShell window opened as administrator:
[Open PowerShell as administrator](#1-open-powershell-as-administrator),
[Turn on TLS 1.2 for this window](#2-turn-on-tls-12-for-this-window),
[Make an empty download folder](#3-make-an-empty-download-folder),
[Download the package](#4-download-the-package),
[Unpack the package and go to its folder](#5-unpack-the-package-and-go-to-its-folder) and
[Run the installer](#6-run-the-installer). The download always takes the newest version.

This time Notepad doesn't open. **Expected:** `PASS  settings kept: C:\ProgramData\EdgeAgent\.env`,
`== Update the program`, `PASS  agent stopped`, the new version in the `PASS  package:` line, and `RESULT: PASS`. Then
do [Step 5: Check that the agent works](#step-5-check-that-the-agent-works) again.

**Going back to an older version:** on <https://github.com/europower-enerji/edge-agent-release/releases>, open the
older release and download its `votech-edge-agent-<version>-windows-x64.zip` instead, then unpack it and run the
installer as above. If the newer version's release notes mention a change of the data buffer, ask Votech first: the
older version needs the `edge.db.bak-<version>` copy put back.

### Docker

1. Change the version at the end of the `image:` line in `docker-compose.yml` (Linux: `sudo nano docker-compose.yml`
   in `/opt/edge-agent`; Docker Desktop: `notepad docker-compose.yml` in `C:\VotechAgent`):
   ```yaml
   image: ghcr.io/europower-enerji/edge-agent:4.6.0
   ```
   Or download the new `docker-compose.yml` as in the installation: it carries the newest version. Replace only that
   file and **keep your `.env`**.
2. Apply the upgrade (Linux with `sudo`):
   ```bash
   docker compose up -d
   ```
3. Do [Step 5: Check that the agent works](#step-5-check-that-the-agent-works) again.
4. Optionally, free disk space by removing the old version: `docker image prune -f`.

**Going back to an older version:** put the older version back on the `image:` line and run `docker compose up -d`.
If the newer version's release notes mention a change of the data buffer, ask Votech first.

**Upgrading from 1.8.x (Docker only).** Version 2 reads the new site configuration and needs Votech backend 4.63 or
later; Votech tells you when your site is ready. Go through version 2.0.2 first: it sends the readings that 1.8.x had
not sent yet, and its log shows `[LEGACY] Sent N legacy telemetry point(s); the legacy buffer is empty.` when it is
done (no `[LEGACY]` line at all means nothing was left). Then upgrade to the newest version. Version 2.1 and later
no longer send 1.8.x readings: if you upgrade straight to them, those readings stay only in the
`edge.db.bak-<version>` copy.

---

## Changing a setting or the agent key

**Windows service:**

```powershell
powershell -ExecutionPolicy Bypass -File "C:\Program Files\EdgeAgent\install.ps1" -Action Config
```

Notepad opens the settings file. Change the value, save, and close Notepad. The installer checks the settings and
restarts the agent. **Expected:** `PASS` lines, `PASS  the agent is running` and `RESULT: PASS`.

**Docker:** edit `.env` (`sudo nano .env` in `/opt/edge-agent`, or `notepad .env` in `C:\VotechAgent`), then run
`docker compose up -d`.

**A new key.** When Votech replaces the agent's key (for example because the old one was exposed), the old key stops
working. The agent then logs `[AUTH] ...` lines and keeps reading for 10 minutes; after that it stops its sites until
it gets the new key. Enter the new key as above as soon as you receive it. Readings stay in the buffer and are sent
once the new key works.

**`DB_PASSWORD` can't be changed** once the agent has started: the agent couldn't open its buffer any more.

---

## Moving the agent to another computer

1. Prepare the new computer and check its network:
   [Step 2: Prepare the computer](#step-2-prepare-the-computer) and [Step 3: Check the network from the agent computer](#step-3-check-the-network-from-the-agent-computer).
2. If the devices accept only known IP addresses, add the new computer's IP address to them.
3. On the old computer, wait until the newest `[MQTT-V2] Sent ...` line says `0 waiting`: everything has been sent.
4. Stop and uninstall the agent on the old computer, keeping its data (see [Uninstalling](#uninstalling)).
5. Install the agent on the new computer with the **same** key, then do [Step 5: Check that the agent works](#step-5-check-that-the-agent-works) and
   [Step 6: Confirm with Votech and test an outage](#step-6-confirm-with-votech-and-test-an-outage).
6. When Votech confirms that the new computer works, delete the old computer's data.

Never run both computers at the same time with the same key.

---

## Uninstalling

**Windows service** (PowerShell as administrator):

```powershell
powershell -ExecutionPolicy Bypass -File "C:\Program Files\EdgeAgent\install.ps1" -Action Uninstall
```

This removes the service and the program and keeps the settings, the buffered readings and the logs in
`C:\ProgramData\EdgeAgent`. To delete those too, add `-RemoveData` at the end, but only after Votech confirms that
all readings have arrived.

**Docker** (Linux with `sudo`):

```bash
docker compose down
```

This removes the agent and keeps the buffered readings. To delete them too, only when the site is decommissioned and
Votech confirms that all readings have arrived:

```bash
docker compose down -v
```

---

## How readings are kept safe

Every reading is written to the agent's disk first, and stays there until Votech confirms that it arrived. A
connection drop, a restart of the agent or a power cut of the agent computer doesn't lose readings that were already
taken.

The buffer holds up to 25 million readings in at most 2 GB. How long that covers an outage depends on how many
readings your site takes per second (Votech can tell you):

| Readings per second | Outage the buffer covers |
| ------------------: | ------------------------ |
|                  25 | about 11 days            |
|                  50 | about 6 days             |
|                 100 | about 3 days             |
|                 200 | about 1.5 days           |

Limits:

- **After 72 hours without Votech** the agent stops, because its site configuration may no longer match the field
  (`[SYNC-CONFIG] The server cannot be reached and the cached configuration was fetched at ...`). It starts again by
  itself and resumes as soon as Votech can be reached.
- **The disk.** Readings already sent are kept for one day and are the only ones deleted to keep the buffer within
  its size. Unsent readings are deleted only when the disk itself is about to fill up: the agent keeps 20 % of the
  disk (at least 2 GB) free and then deletes the oldest unsent readings (`[CRITICAL-STORAGE-ALARM]`). Votech sees in
  the agent's status how many readings were deleted.
- **While the agent computer is off,** nobody reads the devices.

When the connection returns after an outage, new readings are sent first, so Votech shows your site live again
within seconds. The stored readings follow, oldest first, about ten times faster than real time: a one-day outage is
sent in about two and a half hours, without slowing down the live readings. When Votech is busy, it may ask the agent
for a while to send fewer, larger batches and to send stored readings more slowly
(`[FLOW] The backend is behind and asks for fewer batches ...`); nothing is lost, and the agent returns to its own
settings by itself (`[FLOW] No flow hint from the backend ...`).

---

## Problems and solutions

To test a connection named below: on Windows `Test-NetConnection <name or IP> -Port <port>`; on Linux
`timeout 5 bash -c "</dev/tcp/<name or IP>/<port>" && echo open`. Docker commands are shown without `sudo`; on Linux
put `sudo` in front.

"Restart the agent" means: Windows service `Restart-Service VotechEdgeAgent`; Docker `docker compose up -d`. After
changing a setting, use `install.ps1 -Action Config` on Windows (see
[Changing a setting or the agent key](#changing-a-setting-or-the-agent-key)).

### The agent doesn't start or stops right away

| What you see                                                                                       | What it means                                                                         | What to do                                                                                                                                       |
| -------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| Windows: the service is `Stopped` right after starting                                              | A setting is wrong, or a file is missing                                              | Run `install.ps1 -Action Status`: it checks the settings and shows the last log lines. Find the line in these tables.                            |
| Docker: `STATUS` says `Restarting`                                                                  | The same                                                                              | `docker compose logs --tail 30` shows the reason.                                                                                                |
| `EdgeAgent:ApiBaseUrl must be an https address` or `EdgeAgent:Mqtt:UseTls must be true`             | The agent refuses an unencrypted connection                                           | Set `API_BASE_URL` to `https://...` and `MQTT_USE_TLS=true`.                                                                                     |
| `EdgeAgent:Mqtt:WebSocketPath (MQTT_WEBSOCKET_PATH) must be a path such as /mqtt`                   | A whole address was put in `MQTT_WEBSOCKET_PATH`                                      | Only the path there (`/mqtt`), the name in `MQTT_HOST`, the port in `MQTT_PORT`.                                                                 |
| `EdgeAgent:SiteId (SITE_ID) is no longer used since agent 3.0`                                      | A settings file of agent 2.x                                                          | Remove the `SITE_ID` line and put the agent's new `AGENT_KEY` in.                                                                                |
| `The CA certificate file '...' does not exist`, or the file `holds no certificate`                  | `MQTT_CA_CERT` or `API_CA_CERT` points to a missing or wrong file                     | Fix the path (see [Private CA certificate](#private-ca-certificate)), or remove the setting if Votech didn't give you a certificate file.          |
| `Another agent is running on ...: ...-lock is locked`                                               | A second agent uses the same data buffer (two installations, or a copy still running) | Stop the other agent; its process number is in the `-lock` file. Only one agent may run on one buffer.                                           |
| Docker: `required variable AGENT_KEY is missing a value`                                            | The settings file is not found, or that value is empty                                | Run the command inside the agent folder. `ls -la` (Linux) or `dir` (Windows) must show `.env`; on Windows `Rename-Item .env.txt .env` fixes a `.env.txt`. |
| Docker Desktop: `The term 'docker' is not recognized`                                               | Docker Desktop is not installed, or PowerShell was opened before it was installed     | Install Docker Desktop, then open a new PowerShell window.                                                                                       |
| Docker Desktop: `error during connect` or `dockerDesktopLinuxEngine: The system cannot find the file specified` | Docker Desktop is not running                                                | Start Docker Desktop and wait for **Engine running**.                                                                                            |
| Docker: `exec format error` or `no matching manifest for linux/arm/v7`                              | A 32-bit system                                                                       | Install a 64-bit system ([Check the computer on Linux](#check-the-computer-on-linux)).                                                                                |
| Docker: `permission denied ... docker.sock`                                                         | Docker needs administrator rights                                                     | Put `sudo` in front, or add your user to the `docker` group.                                                                                     |
| Docker: `pull access denied`, `i/o timeout` or `TLS handshake timeout` while downloading            | `ghcr.io` can't be reached                                                            | The download check in [Check the network on Linux](#check-the-network-on-linux); your IT team allows `ghcr.io` and `pkg-containers.githubusercontent.com`.                      |

### The key or the site is refused

| What you see                                                                                       | What it means                                                                                                    | What to do                                                                                                                       |
| -------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `[AUTH] The backend refused agent key ...` with `EDGE_KEY_REVOKED`                                  | The key was replaced or withdrawn at Votech. The sites keep running for 10 minutes                               | Ask Votech for the current key and enter it.                                                                                     |
| `[AUTH] ...` with `TENANT_SUSPENDED`                                                               | Your organisation's Votech account is suspended                                                                  | Contact Votech; nothing to change on the computer.                                                                               |
| `[AUTH] ...` without a code                                                                        | Something between the agent and Votech answered instead of Votech                                                | Do the API check in [Step 3: Check the network from the agent computer](#step-3-check-the-network-from-the-agent-computer) again: usually a proxy or a firewall page.             |
| `[AUTH-FATAL] The backend refused agent key ...`                                                    | The key was refused for 10 minutes; every site stopped. The agent keeps asking                                    | Enter the current key (and the right `TENANT_ID`, or remove that line).                                                          |
| `[AGENT] Agent '...' serves 0 site(s)` or `[AGENT] Site ... is no longer assigned to this agent`    | No site, or no longer this site, is assigned to the agent at Votech                                              | Votech assigns the site on its Site connectors page; the agent starts it within a minute, without a restart.                     |
| `[SYNC-CONFIG] The backend refused the configuration of site ... (HTTP 403 ...)`                    | The key doesn't serve that site                                                                                  | Votech checks the site assignment.                                                                                               |
| `[MQTT-AUTH] MQTT broker rejected credentials`, or `not authorized`                                 | The broker refused the key: a typing error, a withdrawn key, or wrong `MQTT_USERNAME`/`MQTT_PASSWORD`            | Check the key; remove `MQTT_USERNAME` and `MQTT_PASSWORD` unless Votech gave you them.                                           |

### The agent doesn't connect to the broker

| What you see                                                                                       | What it means                                                                         | What to do                                                                                                                                                 |
| -------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `[MQTT] Reconnect attempt #...` repeating, never `Connected successfully`                           | The broker can't be reached                                                           | Do the name, port, certificate and WebSocket checks of [Step 3: Check the network from the agent computer](#step-3-check-the-network-from-the-agent-computer) again. `MQTT_HOST` must have no `https://` and no `:port`.          |
| `The server returned status code '404' when status code '101' was expected`                         | The agent reached a web server, but the name or the path is not Votech's broker      | Check `MQTT_HOST`, `MQTT_PORT=443` and `MQTT_WEBSOCKET_PATH` against your values. If they are right, tell Votech.                                          |
| Another status code instead of `404` (for example `400`, `403`, `405`)                              | A web filter or proxy in your network blocks WebSocket connections                     | Your IT team allows WebSocket connections to the Votech broker name.                                                                                       |
| `Error while connecting with host 'Unspecified/127.0.0.1:...'` or `'Unspecified/localhost:...'`      | `MQTT_HOST` points at the agent's own computer, usually copied from another installation | Enter the `MQTT_HOST`, `MQTT_PORT` and `MQTT_USE_TLS` Votech gave you.                                                                                   |
| `The remote certificate is invalid`, `UntrustedRoot`, or `[MqttTls] Refused the certificate of broker ...` | Your network replaces Votech's certificate (TLS inspection), or a certificate file doesn't fit | Your IT team excludes the Votech names from TLS inspection (the certificate check in [Step 3: Check the network from the agent computer](#step-3-check-the-network-from-the-agent-computer)). With a certificate file from Votech: see [Private CA certificate](#private-ca-certificate). |
| The network checks pass, but the agent still doesn't connect                                        | The checks ran with a user's network settings; the agent runs without them (for example, a firewall allows only signed-in users, or a proxy) | Your IT team allows the agent computer itself, not a user, to connect out on port 443 to the two Votech names.                                           |

### The connection drops again and again

| What you see                                                                                       | What it means                                                                                    | What to do                                                                                                                               |
| -------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `[MQTT] Disconnected` every few seconds, and Votech says another session took over                  | Two agents use the same key                                                                      | Find and stop the other one: an old Docker installation next to the Windows service, a second computer, or a copied virtual machine.     |
| `[MQTT] Disconnected` at regular times, for example every 30 or 60 seconds                          | Something on the way closes long connections after a fixed time                                  | Tell Votech the exact interval (the times of the lines). If Votech's side is fine, your firewall or proxy closes long connections: your IT team removes that limit for the Votech names. |
| `[MQTT] Disconnected` at irregular times, many times an hour, with only one agent running            | TLS inspection in the firewall or in the antivirus on this computer cuts the connection | Do the certificate check of [Step 3: Check the network from the agent computer](#step-3-check-the-network-from-the-agent-computer) again; your IT team excludes the Votech names from the inspection. |
| `[MQTT] Disconnected` now and then (a few times a day)                                              | Short internet interruptions                                                                     | Nothing to do: the agent reconnects within a minute and sends what waited.                                                               |
| Disconnects after the computer slept                                                                | Sleep is on                                                                                      | Turn sleep off ([Rules for the computer](#rules-for-the-computer)).                                                                                      |

### Site configuration

| What you see                                                                                       | What it means                                                                         | What to do                                                                                                                       |
| -------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `[SYNC-CONFIG] The backend at ... cannot be reached. Running on the cached configuration.`          | Votech's API can't be reached; the agent keeps reading with its last configuration    | Check the connection to the host of `API_BASE_URL` (the API check in [Step 3: Check the network from the agent computer](#step-3-check-the-network-from-the-agent-computer)).     |
| `[SYNC-CONFIG] No cached configuration for site ...`                                                | First start, and the API can't be reached                                             | Nothing is read until the API answers. Fix the connection to `API_BASE_URL`.                                                     |
| `[SYNC-CONFIG] The server cannot be reached and the cached configuration was fetched at ...`        | No contact with Votech for more than 72 hours, or the computer's clock jumped forward | Restore the internet connection and check the clock. The agent starts again by itself and resumes.                               |
| `[SYNC-CONFIG] Endpoint ... is not acquired: ...`                                                   | A device in the site configuration uses a protocol this agent doesn't read            | Nothing to do on the computer; Votech shows these devices as not acquired.                                                       |

### Devices

| What you see                                                                                       | What it means                                                                                                                       | What to do                                                                                                                                                                       |
| -------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `[IEC104-CONN] Connection ... is down ... next attempt in ... s` or `[MODBUS-CONN] ... is down`, repeating | The device can't be reached                                                                                                 | The device check in [Step 3: Check the network from the agent computer](#step-3-check-the-network-from-the-agent-computer). Check the device's list of allowed IP addresses. Docker on Linux with devices in `172.16.x.x` to `172.31.x.x`: see [Host networking](#host-networking). The device shows as Offline in Votech until it answers. |
| `... accepted the TCP connection but did not confirm STARTDT within t1`                             | The RTU answers on the network but doesn't start a data session with the agent: another master (SCADA, an old agent) holds it, or it serves only the master addresses configured in it | Stop the other master, or add the agent computer's IP address to the RTU's allowed masters. The agent retries by itself, at most about once a minute.  |
| `[IEC104-CONFIG] Connection ... receives common address N, which is no configured station`          | The RTU sends a common address (ASDU address) the site configuration doesn't have                                                   | Tell Votech the address.                                                                                                                                                         |
| `[IEC104-CA] Connection ...: the RTU refused the general interrogation of common address N`         | The common address at Votech is not the RTU's                                                                                       | Tell Votech the address the RTU sends.                                                                                                                                           |
| `[IEC104-CONFIG] Connection ... has no point with an IEC 104 address`, or the RTU shows **Degraded** at Votech with "Connected, but none of its signals has an address" | The signals of that RTU have no addresses at Votech                                    | Votech imports the signal list with addresses.                                                                                                                                   |
| `[IEC104-CLOCK] Connection ...: the RTU clock says ...`                                             | The RTU's clock is wrong (a past year, ahead, or local time instead of UTC)                                                         | Set the RTU's clock (NTP or GPS on the RTU). The agent sets it only when Votech configured clock synchronisation for the RTU, because other masters may share the clock.         |
| `[MODBUS-RX] ...: Unit N does not answer`, or a Modbus exception for a register                     | A unit number or register address in the configuration doesn't match the device                                                     | Tell Votech the unit and register; the agent keeps reading the other units.                                                                                                      |

### Clock, disk and buffer

| What you see                                                                                       | What it means                                                                         | What to do                                                                                                                       |
| -------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| Times in Votech are wrong, or Votech reports a clock difference                                     | The agent computer's clock is wrong                                                   | Fix the clock ([Step 2: Prepare the computer](#step-2-prepare-the-computer)). Linux: `sudo timedatectl set-ntp true`.                                  |
| `[MQTT-V2] Sent ...; N waiting` with N growing for a long time                                      | Readings can't be sent as fast as they are taken                                      | Check the connection lines first. If the connection is fine, tell Votech.                                                        |
| `[STORAGE] Buffer limit reached ...`                                                                | The buffer reached its size during a long outage                                      | Restore the connection. Only readings already sent are deleted at this point.                                                    |
| `[CRITICAL-STORAGE-ALARM] The disk is at its free space floor`                                      | The disk is almost full and holds unsent readings; the oldest unsent readings are deleted | Free disk space at once, and restore the connection. See [How readings are kept safe](#how-readings-are-kept-safe).          |

---

## Optional configuration

### Private CA certificate

Only needed if Votech gives you a `ca.pem` file, because the Votech server uses its own certificate authority.

**Windows service:**

1. Copy `ca.pem` to `C:\ProgramData\EdgeAgent\data\ca.pem` (the service can read files only in its own folders).
2. Run `install.ps1 -Action Config` and set `MQTT_CA_CERT=C:\ProgramData\EdgeAgent\data\ca.pem`, and
   `API_CA_CERT=C:\ProgramData\EdgeAgent\data\ca.pem` when Votech says the API uses the same certificate authority.

**Docker:**

1. Create a folder `certs` in the agent folder (`sudo mkdir -p /opt/edge-agent/certs` on Linux,
   `C:\VotechAgent\certs` with Docker Desktop) and copy `ca.pem` into it.
2. In `docker-compose.yml`, remove the `#` in front of the line `- ./certs:/etc/edge-agent/certs:ro`.
3. In `.env`, set `MQTT_CA_CERT=/etc/edge-agent/certs/ca.pem`, and `API_CA_CERT=/etc/edge-agent/certs/ca.pem` when
   Votech says so. Keep these paths as they are with Docker Desktop too: they are paths inside the agent's container.
4. `docker compose up -d`

With a certificate file, the agent trusts only that certificate authority, not the ones of the operating system. The
server's certificate must also name the address in `MQTT_HOST` or `API_BASE_URL`. If the agent can't read the file,
it doesn't start, so a wrong path never falls back to weaker checks.

### Modbus RTU (serial port or RS-485 adapter)

**Windows service:** nothing to set on the computer. The COM port (for example `COM3`, see
[Check the computer on Windows](#check-the-computer-on-windows)) is part of the site configuration at Votech.

**Linux with Docker:**

1. Find the adapter: `ls /dev/ttyUSB* /dev/ttyACM*`
2. In `docker-compose.yml`, remove the `#` in front of the `devices:` and `group_add:` sections and set the device
   path (for example `/dev/ttyUSB0:/dev/ttyUSB0`).
3. `sudo docker compose up -d`

**Docker Desktop:** not possible; use the Windows service.

### Host networking

Linux with Docker only. If your device network uses addresses in Docker's internal range (`172.16.0.0` to
`172.31.255.255`), remove the `#` in front of `network_mode: host` in `docker-compose.yml` and run
`sudo docker compose up -d`.

---

## Getting help

Send Votech support:

- **Windows service:** the output of `install.ps1 -Action Status`, and the newest file from
  `C:\ProgramData\EdgeAgent\logs`.
- **Docker:** the version (the `image:` line in `docker-compose.yml`), the output of `docker compose ps`, and the file
  from `docker compose logs --since 1h > edge-agent.log`.
- What you did, and when (with the time zone).

The logs show only the beginning of the key, so they are safe to send. **Never send your `.env` file or your
`AGENT_KEY`.**
