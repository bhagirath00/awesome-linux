# Chapter 20: SSH - SSH Tunneling and Port Forwarding

## 1. Introduction to SSH Tunneling

**SSH Tunneling** (or **SSH Port Forwarding**) is a method of encapsulating unencrypted or firewalled network traffic inside an encrypted SSH connection. 

It is widely utilized in DevOps to:
- Access remote database instances (e.g., MySQL, Redis, PostgreSQL) listening strictly on `127.0.0.1` of a remote server.
- Expose a local development web service to a public cloud server for client testing.
- Bypass restrictive corporate firewalls by establishing dynamic SOCKS proxy tunnels.

---

## 2. The Three Types of SSH Port Forwarding

```
+-------------------------------------------------------------------------+
|                  SSH Port Forwarding Classification                     |
+-------------------------------------------------------------------------+
|  1. Local Port Forwarding (-L):                                         |
|     Client opens local port -> Tunnel -> Remote Server forwards to target|
|     [Local Browser: 9000] ===== SSH Tunnel =====> [Remote DB: 5432]     |
+-------------------------------------------------------------------------+
|  2. Remote Port Forwarding (-R):                                        |
|     Remote opens port -> Tunnel -> Client forwards to local service     |
|     [Remote Web: 8080] <===== SSH Tunnel <===== [Local Dev App: 3000]   |
+-------------------------------------------------------------------------+
|  3. Dynamic Port Forwarding (-D):                                       |
|     Creates a local SOCKS5 Proxy that routes all browser/tool traffic    |
|     [Browser (SOCKS5)] ===== SSH Tunnel =====> [Entire Remote Network]  |
+-------------------------------------------------------------------------+
```

---

## 3. Local Port Forwarding (`-L`)

**Syntax:**
```bash
ssh -L [local_interface:][local_port]:[target_destination_host]:[target_destination_port] user@ssh_server
```

### Scenario:
A production PostgreSQL database is running on a private remote server (`db.internal`) listening on port `5432`. It is not accessible from the public internet, but you have SSH access to an intermediate server `app.company.com`.

```bash
# Bind local port 9999 to db.internal:5432 through app.company.com
ssh -N -f -L 9999:db.internal:5432 devops@app.company.com
```

Flag breakdown:
- `-N`: Do not open an interactive shell (just forward ports).
- `-f`: Fork into the background immediately.
- `-L`: Forward local port 9999 to remote `db.internal:5432`.

Now, point your local database GUI (e.g., DBeaver or `psql`) to `localhost:9999` to interact directly with the private database!

---

## 4. Remote Port Forwarding (`-R`)

**Syntax:**
```bash
ssh -R [remote_interface:][remote_port]:[target_destination_host]:[target_destination_port] user@ssh_server
```

### Scenario:
You are developing a webhook receiver on your local machine at `http://localhost:3000`. You want an external service (like Stripe or GitHub webhooks) to send requests to a public server (`public.company.com`) that forwards traffic down to your laptop.

```bash
ssh -N -R 8080:localhost:3000 user@public.company.com
```

Any traffic arriving at `public.company.com:8080` will be piped directly into your laptop's port `3000` through the encrypted SSH tunnel.

> [!NOTE]
> For remote port forwarding to bind to external network interfaces on the remote server, `GatewayPorts yes` must be set in the remote server's `/etc/ssh/sshd_config`.

---

## 5. Dynamic Port Forwarding (`-D`) - SOCKS5 Proxy

Instead of forwarding a single static port, **Dynamic Forwarding** turns the SSH client into a full **SOCKS5 proxy server**.

```bash
ssh -D 1080 -N -C devops@jump.company.com
```

Flag breakdown:
- `-D 1080`: Listen as a SOCKS5 proxy on local port `1080`.
- `-C`: Enable data compression for faster throughput.
- `-N`: Background tunnel mode without interactive shell.

### Browser or CLI Usage:
Configure Firefox, Chrome, or `curl` to use `socks5h://127.0.0.1:1080`:
```bash
curl --socks5-hostname 127.0.0.1:1080 https://internal-dashboard.company.local
```
All DNS resolution and web traffic flow through the remote SSH host!

---

## 6. Port Forwarding Comparison Summary

| Parameter | Type | Direction of Connection | Primary Use Case |
| :--- | :--- | :--- | :--- |
| `-L [port]:[host]:[port]` | **Local** | Outward from client machine | Connect to remote databases or private services. |
| `-R [port]:[host]:[port]` | **Remote** | Inward to client machine | Expose local dev server to the outside internet. |
| `-D [port]` | **Dynamic** | SOCKS5 proxy routing | Browse private cloud intranets and internal web consoles. |

---

## 7. Summary Takeaways

- SSH port forwarding provides instant, ad-hoc encrypted tunnels without needing a dedicated VPN client.
- Local (`-L`) pulls remote resources to your machine.
- Remote (`-R`) pushes local resources to a remote server.
- Dynamic (`-D`) transforms your connection into a full SOCKS5 proxy.
- Use `-N -f` flags to run tunnels quietly in the background.
