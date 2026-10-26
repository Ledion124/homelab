# Spickzettel

[← Zurück zur Übersicht](README.md)

Platzhalter siehe [Netzwerk](docs/02-netzwerk.md#platzhalter).

## Adressen

| Dienst | Adresse |
|---|---|
| Proxmox | `https://<PVE-IP>:8006` |
| Immich | `http://<IMMICH-IP>:2283` |
| Pi-hole | `https://<PIHOLE-IP>/admin` |

## Verbinden

```bash
ssh root@<PVE-IP>            # Proxmox-Host
ssh <user>@<IMMICH-IP>       # Immich-VM
ssh <user>@<CLAUDE-IP>       # Claude-VM
exit
```

## Proxmox-Host

```bash
pct list / qm list           # Container / VMs anzeigen
pct enter 101                # in den Pi-hole-Container springen
pct start|stop|reboot 101    # Container steuern
qm start|shutdown 100        # VM steuern
apt update && apt full-upgrade
shutdown now / reboot
```

## Debian (VMs / Container)

```bash
sudo apt update && sudo apt upgrade -y
ip a / hostname -I           # IP-Adressen
ping -c 3 1.1.1.1            # Internet ohne DNS
ping -c 3 deb.debian.org     # Internet mit DNS
df -h / free -h / top        # Speicher, RAM, Prozesse
sudo ss -tlnp                # offene Ports
```

## Docker / Immich

```bash
cd ~/immich-app
sudo docker compose ps
sudo docker compose up -d / down
sudo docker compose logs -f
sudo docker compose pull && sudo docker compose up -d   # Update
```

## Pi-hole

```bash
pihole status
pihole -g                    # Blocklisten aktualisieren
pihole -up                   # Pi-hole updaten
pihole setpassword
pihole disable 5m / pihole enable
pihole -t                    # Live-Log
```

## Tailscale

```bash
tailscale status
tailscale ip
tailscale up --advertise-routes=<LAN>/24 --accept-dns=false
```

## Claude-VM

```bash
tmux new -s claude           # neue Sitzung
tmux attach -t claude        # anhängen
# Ctrl+B, dann D             # abkoppeln
claude remote-control
claude --version / claude update
```

## Windows

```powershell
ipconfig
nslookup google-analytics.com <PIHOLE-IP>   # 0.0.0.0 = geblockt
route print
certutil -hashfile datei.iso SHA256
```

## Editoren

| nano | |
|---|---|
| `Ctrl+O`, Enter | speichern |
| `Ctrl+X` | beenden |
| `Ctrl+W` | suchen |

| less | |
|---|---|
| `/wort`, `n` | suchen, nächster Treffer |
| `q` | beenden |

## Wichtige Dateien

| Datei | Zweck |
|---|---|
| `/etc/network/interfaces` | feste IP (Debian) |
| `/etc/resolv.conf` | DNS-Server |
| `~/immich-app/.env` | Immich-Einstellungen |
| `/etc/pve/lxc/101.conf` | Pi-hole-Container (auf dem Host) |
| `/etc/sysctl.d/99-tailscale.conf` | IP-Forwarding |
| `/etc/sysctl.d/99-no-ipv6.conf` | IPv6 aus (Claude-VM) |
| `~/projects/CLAUDE.md` | Regeln für Claude Code |
