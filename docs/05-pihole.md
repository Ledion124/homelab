# 05 – Pi-hole (DNS-Adblocker)

[← Zurück zur Übersicht](../README.md)

**Pi-hole** ist ein DNS-Server, der Anfragen an bekannte Werbe-, Tracking- und Malware-Domains blockiert. Das wirkt für **alle Apps** eines Geräts, nicht nur im Browser.

## Netzweit oder pro Gerät?

Normalerweise trägt man Pi-hole im Router als DNS ein. **Mein Router lässt das nicht zu.** Optionen:

| Variante | Vorteil | Nachteil |
|---|---|---|
| A: Pi-hole übernimmt DHCP | wirkt automatisch auf alle Geräte | Server nachts aus → keine IPs/kein DNS für alle; Router verteilt per IPv6 weiter eigenen DNS |
| **B: pro Gerät eintragen** | einfach, sicher, nichts kann fürs ganze Haus ausfallen | wirkt nur auf eingestellten Geräten |
| C: Server 24/7 + A | sauberste Lösung | Server läuft nachts (Geräusch) |

**Entscheidung: Variante B.** Sie passt zum Betrieb (Server nachts aus) und betrifft niemanden sonst im Haushalt.

## 1. Container erstellen

**Warum ein LXC-Container statt einer VM?** Pi-hole ist klein. Ein Container teilt sich den Kernel mit Proxmox, startet in Sekunden und braucht nur einen Bruchteil des RAMs.

| Einstellung | Wert | Warum |
|---|---|---|
| ID / Hostname | 101 / `pihole` | |
| **Unprivileged** | ja | root im Container ist auf dem Host ein rechtloser Benutzer → **Least Privilege** |
| Template | `debian-13-standard` | Gleiches System wie überall |
| CPU / RAM / Swap / Disk | 1 / 512 MB / 512 MB / 8 GB | Pi-hole braucht sehr wenig |
| IPv4 | statisch `<PIHOLE-IP>/24`, Gateway `<ROUTER-IP>` | Ein DNS-Server **muss** eine feste IP haben |
| IPv6 | SLAAC | |

## 2. Installer prüfen, dann ausführen

Der offizielle Weg ist `curl -sSL https://install.pi-hole.net | bash`. Das führt ein Skript aus dem Internet **ungesehen als root** aus. Sauberer:

```bash
apt update && apt upgrade -y && apt install -y curl
curl -sSL https://install.pi-hole.net -o pihole-install.sh
less pihole-install.sh                                   # Kopf, Lizenz, set -e prüfen
grep -oE 'https?://[^ "]+' pihole-install.sh | sort -u   # alle URLs im Skript
bash pihole-install.sh
```

Die URL-Liste enthielt nur Pi-hole, GitHub, die StevenBlack-Blockliste, Doku-Links und Quellenangaben in Kommentaren. Das Skript lädt ausserdem **SHA1-Prüfsummen** für seine eigenen Downloads.

`curl`-Optionen: `-s` still, `-S` Fehler trotzdem zeigen, `-L` Weiterleitungen folgen, `-o` in Datei speichern.

## 3. Installer-Einstellungen

| Frage | Wahl | Warum |
|---|---|---|
| Upstream-DNS | **Quad9 (filtered, DNSSEC)**, ohne ECS | Non-Profit mit Sitz in der Schweiz, blockt zusätzlich Malware. **DNSSEC** = signierte Antworten. **Ohne ECS**, damit kein Teil meiner IP an fremde Server geht |
| Blockliste | StevenBlack: ja | Standardliste |
| Query Logging | an | Fehlersuche: welche Domain wurde geblockt? |
| Privacy Mode | 0 (alles anzeigen) | Nur eigene Geräte. Würden andere Personen Pi-hole nutzen, wäre eine höhere Stufe angebracht |

Danach eigenes Admin-Passwort setzen:
```bash
pihole setpassword
```

Weboberfläche über **HTTPS**: `https://<PIHOLE-IP>/admin` (selbst ausgestelltes Zertifikat). Über HTTP ginge das Passwort im Klartext durchs WLAN.

## 4. Testen

```powershell
nslookup google-analytics.com <PIHOLE-IP>    # 0.0.0.0 = geblockt
nslookup google.com <PIHOLE-IP>              # normale IP = erlaubt
```

## 5. Auf dem iPhone eintragen

*Einstellungen → WLAN → ⓘ → DNS konfigurieren → Manuell*: alle Einträge löschen, nur `<PIHOLE-IP>` eintragen. Gilt nur im Heim-WLAN.

## 6. Fehlersuche: Warum blockt es so wenig?

Ein Adblock-Test zeigte nur **8 %**. Die Ursache lag nicht an einer Stelle, sondern an **vier**:

| # | Ursache | Erkennung | Lösung |
|---|---|---|---|
| 1 | **Router als zweiter DNS** eingetragen | iOS fragt nicht strikt den ersten Server, sondern auch den anderen | Nur Pi-hole eintragen |
| 2 | **NextDNS-Profil** auf dem iPhone | Ein DNS-Profil hat **Vorrang** vor den WLAN-Einstellungen | Profil unter *VPN und Geräteverwaltung* entfernen |
| 3 | **Zu kleine Blockliste** | Anfragen kamen an, wurden aber nicht geblockt | HaGeZi Pro hinzugefügt |
| 4 | **WLAN-Assist** | Bei kurz schwachem WLAN gehen Anfragen über Mobilfunk an den DNS des Anbieters | WLAN-Assist ausgeschaltet |

Ergebnis danach: **93 %** im Test, rund **50 %** aller Anfragen im Alltag geblockt.

**Wie ich die Ursachen gefunden habe:** im **Query Log** nach der IP des Handys bzw. nach einer Test-Domain suchen.
- Keine Einträge → das Gerät fragt Pi-hole gar nicht (Ursachen 1, 2, 4)
- Einträge, aber nicht rot → Blockliste kennt die Domain nicht (Ursache 3)

### Blockliste HaGeZi Pro

*Lists → Add blocklist*, dann *Tools → Update Gravity* bzw. `pihole -g`:

```text
https://cdn.jsdelivr.net/gh/hagezi/dns-blocklists@latest/adblock/pro.txt
```

Zwei Fehler auf dem Weg dorthin:
- Tippfehler in der URL (`http` statt `https`, fehlendes `s`) → rotes ✖ bei der Liste. **Lange URLs immer kopieren.**
- Falscher Pfad → `pihole -g` meldete **„Status: Not found"**. Die Ausgabe von `pihole -g` zeigt pro Liste genau, ob der Download klappte.

Ergebnis: **~280'000 Domains** auf den Listen statt ~71'000.

## 7. Grenzen von DNS-Blocking

Pi-hole blockt **ganze Domains**. Werbung, die von **derselben Domain wie der Inhalt** kommt, lässt sich so nicht trennen:
- **YouTube, Instagram, TikTok-Feed:** Werbung bleibt → dafür einen Browser-Blocker nutzen
- **Werbung in Spielen** über fremde Werbenetzwerke: wird meist geblockt (auch Belohnungs-Werbung)

Bricht eine App, zeigt das Query Log die geblockte Domain. Mit **Allow** lässt sie sich freigeben.

## 8. Nachts

Ist der Server aus und nur Pi-hole als DNS eingetragen, hat das Gerät im WLAN **kein Internet**. Darum abends den DNS am Handy auf *Automatisch* stellen. Ein automatisches Ein- und Ausschalten des Servers steht auf der Roadmap.

---

[← 04 – Immich](04-immich.md) · [Weiter: 06 – Tailscale →](06-tailscale.md)
