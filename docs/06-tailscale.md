# 06 – Tailscale (VPN)

[← Zurück zur Übersicht](../README.md)

## Warum Tailscale?

- Mein Anschluss nutzt **CGNAT**, Port-Weiterleitungen gehen also nicht (siehe [Netzwerk](02-netzwerk.md#cgnat)). Ein klassischer WireGuard- oder OpenVPN-Server zuhause wäre von aussen nicht erreichbar.
- **Tailscale** baut auf WireGuard auf, verbindet die Geräte aber **von innen nach aussen** über eine Vermittlung. Darum braucht es **keinen offenen Port**.
- Ergebnis: Handy und Laptop verhalten sich unterwegs, als wären sie im Heimnetz.

## Architektur: Subnet Router

Statt Tailscale auf jedem Server einzeln zu installieren, läuft es **einmal im Pi-hole-Container** als **Subnet Router**. Er gibt das ganze Heimnetz `<LAN>/24` für meine Tailscale-Geräte frei.

```text
Handy (unterwegs) ──Tailscale──▶ LXC 101 (Subnet Router) ──▶ Proxmox / Immich / ...
```

## 1. Konto

Anmeldung über einen bestehenden Identity-Provider. **Wichtig:** Dieses Konto ist der Schlüssel zum ganzen Heimnetz, deshalb **2FA** am Login-Konto.

## 2. TUN-Gerät für den unprivileged Container

Ein VPN braucht `/dev/net/tun`. Ein unprivileged Container darf das standardmässig nicht. Freigabe auf dem **Proxmox-Host**:

```bash
cat >> /etc/pve/lxc/101.conf <<EOF
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
EOF
pct reboot 101
```

- Zeile 1 **erlaubt** das Gerät (`c` = character device, `10:200` = TUN, `rwm` = lesen/schreiben/anlegen)
- Zeile 2 **reicht es in den Container hinein**
- `>>` hängt an. Ein einfaches `>` würde die Datei **überschreiben**.

Prüfen im Container (`pct enter 101`):
```bash
ls -l /dev/net/tun
# crw-rw-rw- 1 nobody nogroup 10, 200 ... /dev/net/tun
```
`nobody nogroup` zeigt den unprivileged-Effekt: Das Gerät gehört auf dem Host root, im Container erscheint es als rechtloser Benutzer.

## 3. Installation

```bash
curl -fsSL https://tailscale.com/install.sh -o tailscale-install.sh
less tailscale-install.sh
sh tailscale-install.sh          # Shebang ist sh, nicht bash
```

## 4. IP-Forwarding

Ein Linux-System verwirft normalerweise Pakete, die nicht für es selbst bestimmt sind. Ein Subnet Router muss sie **weiterreichen**:

```bash
echo 'net.ipv4.ip_forward = 1' > /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' >> /etc/sysctl.d/99-tailscale.conf
sysctl -p /etc/sysctl.d/99-tailscale.conf
```

Dateien in `/etc/sysctl.d/` werden bei jedem Start geladen, die Einstellung ist also dauerhaft.

## 5. Starten

```bash
tailscale up --advertise-routes=<LAN>/24 --accept-dns=false
```

- `--advertise-routes`: Heimnetz anbieten
- `--accept-dns=false`: Der Container **ist** der DNS-Server (Pi-hole) und soll nicht Tailscales DNS übernehmen

Der angezeigte Login-Link wird im Browser bestätigt.

> **Lektion:** `Ctrl + C` im Terminal heisst **Abbrechen**, nicht Kopieren. Den Befehl einfach erneut starten, dann gibt es einen neuen Link.

Die Warnung zu *UDP GRO forwarding* betrifft nur den Durchsatz bei sehr viel Verkehr und kann im Container ignoriert werden.

## 6. Admin-Konsole

- **Route freigeben:** Gerät → *Edit route settings* → `<LAN>/24` anhaken. Tailscale verlangt diese ausdrückliche Bestätigung, damit kein Gerät sich selbst zur Tür in fremde Netze erklären kann.
- **Key expiry deaktivieren** nur für den Server, damit der Fernzugriff nicht nach 180 Tagen unbemerkt ausfällt. Bei Handy und Laptop bleibt es an.

## 7. Clients

- Tailscale-App auf iPhone und Laptop, gleiches Konto
- Laptop: *Use Tailscale subnets* an
- **Test:** WLAN aus, nur Mobilfunk → `http://<IMMICH-IP>:2283` und `https://<PVE-IP>:8006` laden ✅

Tailscale ist nur das **Netz**. Ein Terminal auf dem Handy braucht zusätzlich eine SSH-App.

## 8. Tailscale-DNS (bewusst aus)

Man könnte Pi-hole als DNS für das ganze Tailnet eintragen, dann blockt es auch unterwegs. Bewusst noch **nicht** gemacht:
- Server nachts aus + Tailscale-DNS an = **kein Internet** unterwegs
- Pi-hole beantwortet standardmässig nur Anfragen aus dem Heimnetz. Das müsste man vorsichtig freigeben, damit kein **offener DNS-Resolver** entsteht

Die Einstellung ist **pro Gerät** schaltbar („Use Tailscale DNS").

## 9. Koexistenz mit anderen VPNs

Ein weiterer WireGuard-Tunnel mit eigenen Netzen läuft parallel ohne Konflikt. Jedes VPN beansprucht nur seine Netze (**Split Tunneling**), und die Routing-Tabelle entscheidet pro Zieladresse. Konflikte gibt es nur, wenn ein VPN `0.0.0.0/0` beansprucht oder Netze sich überschneiden. Prüfen mit:

```powershell
route print
```

## Roadmap

- [ ] **ACLs:** festlegen, welches Gerät was erreichen darf
- [ ] **Tailnet Lock:** neue Geräte nur mit Signatur eigener Geräte
- [ ] Device approval aktivieren

---

[← 05 – Pi-hole](05-pihole.md) · [Weiter: 07 – Claude-Code-VM →](07-claude-code-vm.md)
