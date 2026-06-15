Homelab Secured Self-Hosted Infrastructure

Mehrschichtig segmentierte Homelab-Umgebung mit Fokus auf Containment,
Zugriff ausschließlich per VPN und durchgängigem Monitoring / Threat
Detection. Alle Dienste laufen privat öffentlich erreichbare Workloads
werden bewusst ausgelagert (gemietete Hosts), nicht im Heimnetz betrieben.

- [Überblick](#überblick)
  
- [Architektur](#architektur)
  
- [Netzwerk-Segmentierung](#netzwerk-segmentierung)
  
- [Komponenten](#komponenten)
  
- [Zugriff & Authentifizierung](#zugriff--authentifizierung)
  
- [Härtung](#härtung)
  
- [Monitoring & Detection](#monitoring--detection)
  
- [Backups](#backups)
  
- [Design-Entscheidungen](#design-entscheidungen)
  
- [Nächste Schritte](#Nächste-schritte)



## Überblick
Dieses Repo dokumentiert Architektur und Absicherung meines privaten Homelabs.
Leitgedanke ist Containment: Die Serverlandschaft ist so vom Heimnetz
getrennt, dass eine Kompromittierung der Server nicht automatisch das private
Netz gefährdet und umgekehrt.

Wichtig:


Aktuell ist kein interner Server öffentlich erreichbar.

Die internen Server bilden kein klassisches DMZ. Die doppelte
Firewall-Schicht dient ausschließlich dazu, das Heimnetz gegen einen Befall
der Serverlandschaft abzuschirmen.

Öffentlich erreichbare Dienste werden bei Bedarf auf gemieteten Hosts
betrieben, nicht hier.


Stack: Proxmox VE · Windows Server · Debian 12 · WireGuard · Wazuh ·
Vaultwarden · Termix · Netdata · restic · fail2ban · ufw

## Architektur


<img width="500" height="500" alt="Netwark-Topology drawio" src="https://github.com/user-attachments/assets/9129cdc9-e47d-4a59-8251-f20cb2d0acd7" />

Legende: durchgezogene Pfeile = Netzwerk-Topologie / Trust-Grenzen ·
gestrichelte Pfeile = beispielhaft erlaubte Verbindungen


## Zusammenfassung: 

Externer Zugriff ausschließlich über WireGuard (UDP 10439) auf den Bastion-Host.

Firewall 1 trennt den Internet-Rand (Bastion) vom Server-Netz (VLAN 10).

Firewall 2 trennt das Server-Netz vom Heimnetz (VLAN 2) und lässt nur

explizit benötigte Verbindungen zu (Privater PC → Jumphost, SMB-Freigaben).

## Netzwerk-Segmentierung

| Segment | VLAN | Inhalt | Erreichbarkeit |
|---|---|---|---|
| **Heimnetz** | VLAN 2 | Privater PC, Endgeräte | Verbindungen ins Server-Netz nur über Firewall 2 und nur zu erlaubten Zielen |
| **Server-Netz** | VLAN 10 | Proxmox-Host mit allen VMs | Von außen ausschließlich über Bastion + WireGuard; vom Heimnetz nur gezielt freigegeben |

Beide Firewalls fahren Default-Deny und lassen ausschließlich benötigte
Ports/Verbindungen durch (umgesetzt mit ufw).

## Komponenten

| Host / VM | OS | Rolle |
|---|---|---|
| Bastion | Debian 12 | WireGuard-Endpoint, einziger Internet-Rand, stark gehärtet |
| Proxmox-Host | Proxmox VE | Virtualisierungs-Layer für alle VMs |
| File-Server | Windows Server | SMB-Dateifreigaben + Laufwerk für Restic-Backups |
| Jumphost | Windows Server | Ausschließlich Wartungszugriff auf Server (statt privatem PC) |
| Vaultwarden | Debian 12 | Self-hosted Passwort-Manager |
| Wazuh Manager | Debian 12 | SIEM / XDR — zentrale Log-Auswertung & Detection |
| Termix | Debian 12 | Zentrale Verwaltung der SSH-Zugänge / Keys |
| SAMBA AD/ Netdata Manger (Standalone Server) | Debian 12/Docker | Zentrale Verwaltung Benutzer / Rechte |

## Zugriff & Authentifizierung

VPN-only: Alle Dienste sind ausschließlich über WireGuard nutzbar — einzige
Ausnahme sind die SMB-Dateifreigaben des Windows-Servers, die intern aus dem
Heimnetz ohne VPN erreichbar sind (aber nicht aus dem Internet).

WireGuard auf nicht-standardmäßigem Port UDP 10439.

SSH ausschließlich per Key — keine Passwort-Authentifizierung auf irgendeinem Server.
Pro Key eine eigene Passphrase — wird ein Key kompromittiert, sind die
übrigen Zugänge nicht betroffen.

Termix als zentrale Stelle zur Verwaltung der SSH-Zugänge.

Jumphost-Prinzip: Server-Wartung läuft über eine dedizierte Jumphost-VM,
nicht vom privaten PC aus das Endgerät bleibt aus der Server-Administration heraus.
Verschlüsselung durchgängig (Transport über WireGuard/SSH, Daten verschlüsselt).

## Härtung

ufw Default-Deny: Auf jedem Host nur die tatsächlich benötigten Ports offen.
fail2ban überall, am Bastion-Host besonders aggressiv konfiguriert.
Hintergrund: Brute-Force- und Scan-Traffic wird bereits am Rand abgefangen,
damit die internen Server gar nicht erst mit der entsprechenden Log-Masse
belastet werden.

Keine öffentliche erreichbarkeit interner Dienste.

Netzwerk-Isolation über VLANs + zwei Firewalls (siehe Segmentierung).

## Monitoring & Detection

Wazuh-Agent auf jedem Server → zentrale Auswertung am Wazuh-Manager
(Log-Analyse, File Integrity Monitoring, Intrusion Detection).

Netdata auf jedem Server → Echtzeit-Performance-Monitoring pro Node.

## Backups

restic erstellt automatisch Full-Backups jedes Servers.
Ziel ist ein separates Laufwerk am Windows-File-Server, getrennt von den
produktiven Daten.

## Design-Entscheidungen

Doppelte Firewall statt DMZ: Ziel ist nicht, Dienste nach außen anzubieten,
sondern das Heimnetz hinter Firewall 2 vor einem kompromittierten Server-Segment
zu schützen. Bastion + Firewall 1 schützen umgekehrt die Server vor dem Internet.

Bastion als einziger Eintrittspunkt: Eine einzige, stark gehärtete
Angriffsfläche nach außen (WireGuard) statt mehrerer exponierter Dienste.

fail2ban am Rand verschärft: Filterung dort, wo der Müll ankommt reduziert
Last und Log-Rauschen auf den internen Systemen.

Pro-Key-Passphrasen: Begrenzt den Schaden eines einzelnen kompromittierten Keys.

Jumphost statt Direktzugriff: Trennt das Endgerät von der Server-Administration.
Öffentliche Workloads ausgelagert: Was öffentlich erreichbar sein muss, läuft
auf gemieteten Hosts das Heimnetz bleibt grundsätzlich von außen nicht erreichbar.

## Nächste-Schritte:

Wazuh-Alerting-Regeln erweitern und Dokumentieren

Test und Dokumentation Restic Backup Restore

Hinweis:

Der geänderte WireGuard-Port reduziert nur das Scan-Rauschen und ersetzt **keine** der
anderen Maßnahmen.




