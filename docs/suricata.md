# 🛡️ Suricata IDS/IPS mit Telegram Alerting

## Was ist Suricata?

Suricata ist ein **Intrusion Detection/Prevention System (IDS/IPS)** das jeden
Netzwerkpaket analysiert und bei verdächtigen Mustern Alarm schlägt.

```
Netzwerkverkehr
      ↓
 OPNsense Firewall
      ↓
 Suricata analysiert jeden Paket
      ↓
 Verdächtig? → Telegram Nachricht! 🚨
```

**Unterschied zur Firewall:**
- Firewall = prüft Quelle/Ziel/Port
- Suricata = schaut **hinein** in den Inhalt der Pakete

---

## 📦 Installation

> System → Firmware → Pakete → `suricata` → installieren

In OPNsense eingebaut unter:
> **Dienste → Intrusion Detection (IDS)**

---

## ⚙️ Konfiguration

### Allgemeine Einstellungen

| Einstellung | Wert |
|---|---|
| Aktiviert | ✅ |
| Capture mode | PCAP live mode (IDS) |
| Promiscuous-Modus | ✅ |
| Schnittstellen | LAN, WAN, TAILSCALE, WG_NORDVPN |
| Syslog-Warnungen | ✅ |
| Eve-Syslog-Ausgabe | ✅ |

---

## 📋 Aktivierte Regelsets

| Regelset | Beschreibung |
|---|---|
| ET open/emerging-dos | DDoS Erkennung |
| ET open/emerging-exploit | Exploit Versuche |
| ET open/emerging-malware | Malware Erkennung |
| ET open/emerging-scan | Port Scanning |
| ET open/emerging-attack_response | Angriffs-Antworten |
| ET open/emerging-dns | DNS Angriffe |
| ET open/botcc | Bekannte Botnet C2 Server |
| ET open/drop | Bekannte schlechte IPs |
| abuse.ch/ThreatFox | Aktuelle Malware/Botnet IPs |
| abuse.ch/URLhaus | Bekannte Malware URLs |

**Total: 202'765 Regeln** ✅

---

## 🤖 Telegram Alerting

### Telegram Bot erstellen

1. In Telegram: `@BotFather` suchen
2. `/newbot` — Name und Username vergeben
3. **Token** kopieren und sicher speichern
4. Chat ID herausfinden via `@myidbot` → `/getid`
5. Bot starten: eigenen Bot suchen → **Start**

### Script

Gespeichert unter `/usr/local/bin/suricata-telegram.sh`:

```sh
#!/bin/sh
LOG="/var/log/suricata/eve.json"
LAST_ALERT="/tmp/suricata_last_alert"

# Letzten gesendeten Alert holen
if [ -f "$LAST_ALERT" ]; then
    LAST=$(cat "$LAST_ALERT")
else
    LAST=""
fi

# Nur externe IPs — keine lokalen 192.168.x.x / 10.x.x.x / 172.16.x.x
CURRENT=$(grep "event_type.*alert" "$LOG" | grep -v '"src_ip":"192\.168\.' | grep -v '"src_ip":"10\.' | grep -v '"src_ip":"172\.16\.' | tail -1)

# Nur senden wenn neuer externer Alert
if [ -n "$CURRENT" ] && [ "$CURRENT" != "$LAST" ]; then
    SIG=$(echo "$CURRENT" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['alert']['signature'].replace(' ','+'))")
    SRC=$(echo "$CURRENT" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('src_ip','?'))")
    DST=$(echo "$CURRENT" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('dest_ip','?'))")
    curl "https://api.telegram.org/bot${BOT_TOKEN}/sendMessage?chat_id=${CHAT_ID}&text=Suricata+Alert%0A${SIG}%0ASrc:+${SRC}%0ADst:+${DST}"
    echo "$CURRENT" > "$LAST_ALERT"
fi
```

> ⚠️ Token und Chat ID direkt im Script speichern — keine Variablen in OPNsense Shell (tcsh)!

### Cron einrichten

```sh
echo "*/5 * * * * root sh /usr/local/bin/suricata-telegram.sh" >> /etc/crontab
```

Script läuft alle 5 Minuten — sendet aber **nur bei neuem externen Alert**!

---

## 📱 Beispiel Telegram Nachricht

```
🚨 Suricata Alert
ET EXPLOIT MS-SQL SQL Injection attempt
Src: 45.33.32.156
Dst: 192.168.1.10
```

---

## 🔄 Automatische Regelaktualisierung

> System → Einstellungen → Geplante Aufgaben (Cron)

```
Beschreibung: IDS Regeln aktualisieren
Befehl:       Update and reload intrusion detection rules
Minuten:      0
Stunden:      3
Tag:          *
Monat:        *
Wochentag:    *
```

Jeden Tag um 03:00 Uhr werden die Regelsets automatisch aktualisiert.

---

## 💡 Lessons Learned

- OPNsense nutzt **tcsh** als Standard-Shell — `$()` Variablen funktionieren nicht!
- Lösung: `sh` Script mit `#!/bin/sh` und direkt in `/bin/sh` ausführen
- Token/Chat ID direkt im Script eintippen — nicht kopieren wegen Sonderzeichen
- Lokale IPs (`192.168.x.x`) erzeugen viele False Positives — filtern!
- `eve.json` ist das Hauptlog — alle Alerts als JSON gespeichert

---

*Teil von [andro-lab](https://github.com/sudoAndro/andro-lab) — Homelab & IT-Tools von Andrija Tadic*
