# 10 – Lessons Learned

[← Zurück zur Übersicht](../README.md)

Die wertvollsten Erkenntnisse kamen aus Dingen, die **nicht** auf Anhieb funktioniert haben.

## Fehlersuche

**Schicht für Schicht eingrenzen.** Beim Pi-hole-Problem (8 % geblockt) gab es nicht eine Ursache, sondern vier übereinander: zweiter DNS, DNS-Profil, kleine Blockliste, WLAN-Assist. Erst durch systematisches Prüfen („Kommt die Anfrage überhaupt an?" → „Wird sie geblockt?") wurde jede einzelne sichtbar. → [Kapitel 05](05-pihole.md#6-fehlersuche-warum-blockt-es-so-wenig)

**Logs sind die Wahrheit.** Ob ein Gerät Pi-hole wirklich nutzt, zeigt das Query Log, nicht die Einstellungen am Gerät. Ob Claude in der VM arbeitet, zeigt `cat` im Terminal, nicht Claudes eigene Aussage darüber.

**IP vs. Name testen.** `ping 1.1.1.1` geht, `ping deb.debian.org` nicht → DNS-Problem, nicht Netzwerk.

**Werkzeuge zeigen, was schiefging.** `pihole -g` meldete „Not found" bei der Blockliste. Die Ausgabe genau lesen spart Raten.

**Eigene Annahmen prüfen.** Mehrmals war meine erste Vermutung falsch (z. B. „der Browser umgeht Pi-hole"). Ein zweiter, unabhängiger Test ist wichtiger als die erste Theorie.

## Sicherheit

- **Downloads verifizieren:** Hash bei ISOs, Signaturschlüssel bei Paketquellen, Skripte vor dem Ausführen lesen und URLs prüfen
- **Least Privilege:** unprivileged Container, kein root-Passwort, Installer ohne `sudo`, VM ohne Zugangsdaten zu anderen Systemen
- **Bitten sind keine Grenzen:** Regeln in einer Konfigurationsdatei ersetzen keine Firewall
- **IPv6 nicht vergessen:** Eine IPv4-Sperre allein lässt eine Hintertür offen
- **Regeln in beide Richtungen testen:** blockiert sie, was sie soll, und lässt sie durch, was nötig ist?
- **Geheimnisse nie fotografieren oder teilen:** 2FA-QR-Codes, Recovery-Codes, WLAN-Karten. Passiert es doch: sofort neu erzeugen
- **Bequemlichkeit vs. Sicherheit bewusst abwägen:** z. B. Tailscale-DNS aus, damit nachts nicht das Internet wegfällt

## Linux & Terminal

- `Ctrl + C` = Abbrechen, nicht Kopieren. Im Browser öffnet `Ctrl + Shift + C` die Entwicklerkonsole
- `>` überschreibt, `>>` hängt an
- In nano gehts mit den Pfeiltasten, nicht mit der Maus
- Lange URLs und IP-Bereiche: kopieren, aber auf unsichtbare Zeichen achten
- Die Proxmox-Konsole kann nicht einfügen → für längere Arbeiten SSH nutzen
- Nach einer Installation in den Benutzerordner neu einloggen, damit `PATH` stimmt

## Planung

- **Vorher prüfen, was der Router kann**, bevor man ein Konzept baut (kein DNS-Feld → Plan geändert)
- Betriebsmodell (Server nachts aus) beeinflusst fast jede Entscheidung: DNS, VPN, Autostart
- Dokumentation **während** der Arbeit festhalten ist viel einfacher als im Nachhinein

---

[← 09 – Betrieb](09-betrieb.md) · [Zurück zur Übersicht](../README.md)
