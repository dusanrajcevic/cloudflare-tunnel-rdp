# 🔐 Secure RDP Access via Cloudflare Tunnel (Windows Server)
This guide describes how to securely expose Remote Desktop (RDP) on a Windows Server using **Cloudflare Tunnel** and **Cloudflare Zero Trust**, without opening inbound ports.

---

## Overview
This setup provides:
- ✅ No public exposure of TCP port 3389  
- ✅ Identity-based access control via Cloudflare Access  
- ✅ Encrypted tunnel between client and server  
- ✅ Reduced attack surface compared to traditional RDP exposure  

---

## Architecture
- RDP runs locally on the Windows Server (`localhost:3389`)
- Cloudflare Tunnel exposes it securely via a hostname
- Cloudflare Access enforces authentication and authorization
- Client connects through `cloudflared` to a local forwarded port

---

## Server Setup (Windows Server)

### 1. Install `cloudflared`
Download the latest version from:
https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/downloads/

Place the binary here:
```text
C:\Cloudflared\bin\cloudflared.exe
```

---

### 2. Open elevated shell
Open **PowerShell** as an administrator and run:
```powershell
cd C:\Cloudflared\bin
```

---

### 3. Authenticate with Cloudflare
Run:
```powershell
cloudflared.exe tunnel login
```

This opens a browser window for authentication and domain selection.
A certificate will be stored locally after successful login.

---

### 4. Create the tunnel
Run:
```powershell
cloudflared.exe tunnel create name-for-the-server
```
This will:
- create the tunnel in Cloudflare
- create a local credentials JSON file

The command output will show the tunnel UUID. You will need this value for `config.yml`.

### 5. Create `config.yml`

Create the following file:
```text
C:\Cloudflared\config.yml
```

Add this content to the file:
```yml
tunnel: YOUR-TUNNEL-UUID
credentials-file: C:\Cloudflared\YOUR-TUNNEL-UUID.json

ingress:
  - hostname: your-domain.example
    service: rdp://localhost:3389
  - service: http_status:404
```
Replace:
- `YOUR-TUNNEL-UUID` with the actual tunnel UUID
- `your-domain.example` with the RDP hostname you want to use

---

### 6. Create DNS route
Run:
```powershell
cloudflared.exe tunnel route dns YOUR-TUNNEL-UUID your-domain.example
```

This maps your hostname to the tunnel.

---

### 7. Test tunnel (manually)
Run:
```powershell
cloudflared.exe tunnel --config C:\Cloudflared\config.yml run YOUR-TUNNEL-UUID
```

If it starts successfully, configuration is valid.

--

### 8. Install cloudflared as a Windows service
Run:
```powershell
cloudflared.exe service install
```
Then make sure the Windows service uses the correct config file:
```powershell
sc config cloudflared binPath= "\"C:\Cloudflared\bin\cloudflared.exe\" --config \"C:\Cloudflared\config.yml\" tunnel run"
```

Start the service:
```powershell
sc start cloudflared
```

---

### 9. Verify tunnel status
Run:
```powershell
cloudflared.exe tunnel list
cloudflared.exe tunnel info YOUR-TUNNEL-UUID
```
You should see an active connector

---

## Cloudflare Zero Trust Configuration

### 10. Create a Self-hosted application in Cloudflare Access

In Cloudflare Zero Trust dashboard:

- Go to **Access controls** -> **Applications**
- Click **Add an application**
- Choose **Self-hosted**

Configuration:
- **Application name**: e.g. `COMPANYNAME RDP`
- **Session Duration**: <= `6 hours`
- Click on App public hostname, and choose **Input method** -> **Custom** and enter the hostname you used in `config.yml`, for example:
  ```text
  your-domain.example
  ```
Save the application.

This creates the protected Access application for your RDP hostname.

---

### 11. Create access policy
After creating the application, add a policy by selecting:
- **Access controls** -> **Policies**
- **Add a policy**.

**Recommended settings:**
- **Policy name**: `allow-admin`
- **Action**: `Allow`
- **Session duration**: `6 hours`
In the **Add rules** section, under **Include**
- **Selector**: `Emails`
- Add only the email addresses that should be allowed to connect. e.g.:
    ```text
    name.surname@domain.name
    name2.surname2@domain.name
    ```
  - Avoid:
    - `Everyone`
    - `Everyone in the zone`
Save the policy.

---

### 12. Verify the Access policy

Ensure:
- Only the required email addresses are allowed
- No overly broad policies exist

This ensures that only authorized users can reach the RDP application.

---

## Client Connection
### 13. Connect from client computer

1. On the client computer, download the latest version from:
https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/downloads/

2. Place it in `C:\Windows\cloudflared.exe` or add it to `PATH`
3. Open powershell and start local tunnel:
  ```powershell
  cloudflared access rdp --hostname your-domain.example --url rdp://localhost:13389
  ```
4. Open Remote Desktop and connect to:
```text
localhost:13389
```
Keep the command window open while the RDP session is active.

---

## Windows Server Hardening
### 14. Disable inbound RDP rules in Windows Firewall
On the server, open:
- **Windows Defender Firewall with Advanced Security**
- Go to **Inbound Rules** and disable these rules:
  - Remote Desktop - User Mode (TCP-In)
  - Remote Desktop - User Mode (UDP-In)
  - Remote Desktop - Shadow (TCP-In) if not needed
This is recommended when using Cloudflare Tunnel, since public inbound RDP is not required.

---

### 15. Verity port 3389 is not exposed
- From a device outside the local network, test the server's public IP on port `3389`.
  - Example:
    ```powershell
    Test-NetConnection YOUR_PUBLIC_IP -Port 3389
    ```
- Expected result:
  ```text
  TcpTestSucceeded : False
  ```
  This confirms that public RDP is not exposed.

---

### 16. Review local RDP and Administrator groups
Run:
```powershell
net localgroup "Remote Desktop Users"
net localgroup Administrators
```

Recommended result:
- Remote Desktop Users is empty
- Administrators contains only the accounts that truly need administrator access

---

### 17. Use a named admin account for RDP
- Do not use built-in Administrator
- Create and use a named administrator account for normal remote access.
- Add that named account to the local Administrators group.

---

### 18. Block the built-in Administrator account for RDP
Open:
```powershell
secpol.msc
```

Navigate to:
**Security Settings** -> **Local Policies** -> **User Rights Assignment**

Edit:
```text
Deny log on through Remote Desktop Services
```
Add:
```text
Administrator
```

This keeps the built-in Administrator account available for local/emergency use, but blocks it from remote desktop logon.

Make sure your named admin account is working before doing this.

---

### 19. Confirm Remote Desktop logon rights

In the same section, check:
- **Allow log on through Remote Desktop Services**

This should normally include:
- **Administrators**
- **Remote Desktop Users**

---

## Security Notes
- ❌ Do NOT expose TCP port 3389 on the router or firewall publicly.
- ❌ Do NOT use broad Cloudflare Access policies
- ✅ Use identity-based access control
- ✅ Use named admin accounts
- ✅ Keep Windows Firewall hardened
- ✅ Monitor Cloudflare Access logs

---

## Notes
- This setup assumes Cloudflare Zero Trust is properly configured
- Works best in environments with identity-based access control requirements
- Suitable for production use with proper hardening

---

## 🛠️ Troubleshooting

### Tunnel does not start

**Symptoms:**
- `cloudflared` exits immediately
- No active connector in `cloudflared tunnel list`

**Checks:**
- Verify `config.yml` path is correct
- Ensure `tunnel UUID` matches the credentials file
- Confirm the credentials file exists:
  ```
  C:\Cloudflared\YOUR-TUNNEL-UUID.json
  ```
- Run manually to see errors:
  ```powershell
  cloudflared.exe tunnel --config C:\Cloudflared\config.yml run YOUR-TUNNEL-UUID
  ```

---

### DNS hostname not working
**Symptoms:**
- Cannot resolve `your-domain.example`
- Cloudflare Access page not loading

**Checks:**
- Verify DNS route:
  ```powershell
  cloudflared.exe tunnel route dns YOUR-TUNNEL-UUID your-domain.example
  ```
- Confirm DNS record exists in Cloudflare dashboard
- Wait a few minutes for DNS propagation

---

### Access denied after login
**Symptoms:**
- Authentication succeeds, but access is blocked

**Checks:**
- Verify Access policy includes your email
- Ensure no conflicting policies exist
- Confirm application hostname matches `config.yml`

---

### RDP connection fails (client side)

**Symptoms:**
- Cannot connect to `localhost:13389`
- RDP client shows connection error

**Checks:**
- Ensure `cloudflared access rdp` is running
- Confirm correct command:

  ```powershell
  cloudflared access rdp --hostname your-domain.example --url rdp://localhost:13389
  ```
- Verify RDP service is running on server:

  ```powershell
  Get-Service TermService
  ```

---

### Port 3389 still reachable from the internet

**Symptoms:**
- `Test-NetConnection` returns `True`

**Checks:**
- Disable inbound RDP rules in Windows Firewall
- Check router/firewall port forwarding settings
- Ensure no external NAT rule exposes port 3389

---

### Windows service does not start

**Symptoms:**
- `cloudflared` service fails to start

**Checks:**
- Verify service configuration:
  ```powershell
  sc qc cloudflared
  ```
- Confirm correct `--config` path
- Check Windows Event Viewer for errors

---

### Tunnel works manually but not as a service

**Symptoms:**
- Works when run manually, fails as service

**Checks:**
- Ensure service uses correct working directory
- Verify file permissions for:

  - `config.yml`
  - credentials JSON
- Restart service:

  ```powershell
  sc stop cloudflared
  sc start cloudflared
  ```

---

## Debug Tips
- Run `cloudflared` manually before using it as a service
- Use verbose logs:
  ```powershell
  cloudflared.exe tunnel run --loglevel debug
  ```
- Check Cloudflare dashboard → **Zero Trust → Logs**

---

For questions or improvements, feel free to open an issue.