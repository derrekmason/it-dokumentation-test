# Windows Server – Administration

## Zweck und Geltungsbereich

Diese Dokumentation beschreibt die typischen Arbeitsabläute bei der
Administration von Windows Server: Installation, Netzwerkgrundlagen,
Benutzerverwaltung, Updates, Protokollierung und Datensicherung.

**Behandelt wird:**

- Einsatzgebiete und typische Aufgaben
- Installation und Erstkonfiguration
- Hostname, IP-Konfiguration, DNS und Netzwerkanbindung
- Benutzer, Gruppen und Berechtigungen
- Windows-Updates und grundlegende Absicherung
- Firewall, Ereignisanzeige und Protokollierung
- Backup und Wiederherstellung
- typische Fehlerquellen

**Nicht behandelt wird:** Deep-Dive in einzelne Rollenprodukte, Clustering,
Hyper-V im Detail und Windows-Client-Konfiguration. Die verfügbaren
Funktions- und Editionenunterschiede ändern sich je nach Version –
Einzelheiten dazu stehen in der offiziellen Microsoft-Dokumentation (siehe
[Quellen](#weiterführende-quellen)).

Alle Beispieldaten (Hostnamen, Adressen, Konten) sind fiktiv.

---

## Grundlagen

### Typische Aufgaben und Einsatzgebiete

| Einsatzgebiet | Typische Aufgaben |
| --- | --- |
| Domänencontroller | Verzeichnisdienst bereitstellen, Konten und Gruppen verwalten, Group Policy verteilen, GPO- und DNS-Zonenverwaltung |
| Datei- und Anwendungsserver | Dateifreigaben (SMB), NTFS-Berechtigungen, Bereitstellung von Anwendungen |
| Infrastrukturrollen | DNS, DHCP, Printserver, Webserver (IIS), Remotedesktop (RDP) |
| Virtualisierung | Hyper-V: VMs, virtuelle Switches, Checkpoints |
| Überwachung | Ereignisprotokolle, Leistungsdaten, Protokollierung und Weiterleitung an ein zentrales System |

### Rollen, Funktionen und Abhängigkeiten

Windows Server wird über **Rollen** und **Funktionen** erweitert. Beispiele:

| Rolle / Funktion | Zweck | Typische Abhängigkeit |
| --- | --- | --- |
| DNS-Server | Namenauflösung, Zonenverwaltung | keine zwingende; wird bei AD DS automatisch mitinstalliert |
| DHCP-Server | automatische IP-Vergabe | keine zwingende; Autorisierung setzt AD DS voraus |
| Active Directory Domain Services (AD DS) | Verzeichnis, Konten, Gruppen, Richtlinien | AD DS selbst nutzt DNS zur DC-Suche |
| Datei- und Speicherdienste | SMB-Freigaben | keine zwingende |
| Remotedesktop-Dienste | RDP-Sitzungen | keine zwingende |
| Hyper-V | Virtualisierung | je nach Szenario Hardwarevirtualisierung |
| Windows-Backup | Datensicherung des Servers | Volume Shadow Copy Service (VSS) |

> **Wichtig:** *Nicht jede* Windows-Server-Installation ist ein
> Domänencontroller. Ein eigenständiger (Workgroup-)Server, ein
> Member-Server in einer bestehenden Domäne und ein Domänencontroller sind
> drei verschiedene Rollen. Ob eine Rolle benötigt wird, ergibt sich aus der
> gewünschten Funktion – nicht aus dem Betriebssystem.

Beim AD-DS-Aufbau gilt die Reihenfolge **Forest → Domain → Domain Controller**.
Für einen neuen Forest mit `Contoso` als fiktivem Beispiel:

```powershell
# Verfügbare AD-DS-Unterfunktionen prüfen (Forests / Domains / DC)
Get-WindowsFeature AD-Domain-Services

# Installation mit dem Server-Manager-Cmdlet (Beispiel für einen Testforest)
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
Install-ADDSForest -DomainName "contoso.example" -DomainNetbiosName "CONTOSO"
```

> `Install-ADDSForest` startet nach dem Neustart den Konfigurationsassistenten.
> Der Befehl ist ein Beispiel für die Größenordnung des Ablaufs; Produkt- und
> Umgebungsdetails gehören in die aktuelle Microsoft-Dokumentation, nicht in
> dieses Musterdokument.

### Hostname und Identität

```powershell
# Aktuellen Hostnamen und die zugewiesenen Domänen anzeigen
hostname
Get-CimInstance Win32_ComputerSystem | Select-Object Name, Domain, PartOfDomain
```

Umbenennen:

```powershell
Rename-Computer -NewName "srv-files01" -Restart
```

> **Nachsatz:** Die Änderung des Computernamens muss in den DNS-Einträgen
> nachgezogen werden. Bei Domänenmitgliedschaft aktualisiert der
> Computername sich in der Regel über die dynamische DNS-Aktualisierung.

---

## Vorgehen und Beispiele

### 1. Installation und Erstkonfiguration

Vorbereitung:

1. Installationsmedium und Edition passend zur Lizenz wählen
   (Microsoft vergibt unterschiedliche Editions für Server und Virtualisierung;
   siehe „Editionenvergleich“ in den Quellen).
2. **Server Core** oder **Desktop Experience** wählen. Desktop Experience ist
   komfortabler, benötigt aber mehr Ressourcen; Core ist für stark
   automatisierte Umgebungen üblich.
3. Installieren, Partition festlegen, Edition wählen.
4. Nach der Installation: Computername setzen, Updates einspielen und den
   Domänen- bzw. Arbeitsgruppen-Modus wählen (bei einem Domänencontroller
   zwingend die Domäne).

Nach der Installation:

```powershell
# Rollen und Features anzeigen bzw. installieren
Get-WindowsFeature
Install-WindowsFeature -Name Web-Server -IncludeManagementTools

# Installierte Komponenten abfragen
Get-WindowsFeature | Where-Object InstallState -eq "Installed"
```

Prüfen der Netzwerkkonfiguration:

```powershell
ipconfig /all
Get-NetIPConfiguration
Get-NetAdapter
```

### 2. Netzwerkanbindung

Statische IPv4-Adresse (Beispiel, fiktiv):

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet" `
  -IPAddress 192.168.10.20 -PrefixLength 24 `
  -DefaultGateway 192.168.10.1
```

DNS-Server eintragen:

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" `
  -ServerAddresses 192.168.10.53,192.168.10.54
```

Namensauflösung prüfen:

```powershell
Resolve-DnsName files01.contoso.example
Test-NetConnection -ComputerName files01.contoso.example -Port 445
```

| Netzwerkkonzept | Merksatz |
| --- | --- |
| DHCP | zentral verwaltete Adressvergabe, siehe [DHCP-Server](https://learn.microsoft.com/en-us/windows-server/networking/technologies/dhcp/dhcp-top) |
| DNS | Standard-Namensdienst in Windows-Netzwerken; Grundausstattung jeder Windows-Installation, siehe [DNS in Windows Server](https://learn.microsoft.com/en-us/windows-server/networking/dns/dns-overview) |
| IPv4-Adressierung | Grundlagen siehe [Private IPv4-Adressbereiche](../networking/private-ipv4.md) |

### 3. Benutzer, Gruppen und Berechtigungen

Ohne Domäne (Arbeitsgruppe) werden lokale Konten und lokale Gruppen verwaltet:

```powershell
# Lokalen Benutzer anlegen
$pw = Read-Host -AsSecureString "Neues Kennwort"
New-LocalUser -Name "svc-backup" -Password $pw -PasswordNeverExpires:$false

# Lokale Gruppe anlegen und Benutzer hinzufügen
New-LocalGroup -Name "Backup-Operatoren"
Add-LocalGroupMember -Group "Backup-Operatoren" -Member "svc-backup"
```

In einer Domäne kommen zentrale Konten und Gruppen hinzu:

```powershell
New-ADUser -Name "Anna Beispiel" -SamAccountName "a.beispiel" `
  -Path "OU=Benutzer,DC=contoso,DC=example"
New-ADGroup -Name "Ablage-Redaktion" -GroupScope "Global" `
  -Path "OU=Gruppen,DC=contoso,DC=example"
Add-ADGroupMember -Identity "Ablage-Redaktion" -Members "a.beispiel"
```

Grundsätze der Berechtigungen:

- **NTFS-Berechtigungen** (ACL) und **Freigabeberechtigungen** sind getrennt
  wirksam. Der effektive Zugriff ist die **engere** der beiden Regeln.
- Das Prinzip der geringsten Rechte gilt; das Konto
  `Domain Admins`/`BUILTIN\Administrators` ist nur fürAdministrationsaufgaben.
- Sonderkonten (`krbtgt`, `Guest`, `DefaultAccount`) nicht eigenmächtig ändern.
- Gruppen statt Einzelkonten verwenden – dann ändert sich die Berechtigung an
  einer Stelle.

### 4. Updates

Windows Update ist auf Servern in der Regel **nach wenigen Nutzern, nicht
automatisch** – ein unbedacht eingespielter Neustart kann laufende Dienste
unterbrechen. Empfohlene Vorgehensweise:

1. Die verfügbare Konfiguration prüfen:
   ```powershell
   Get-WindowsUpdateLog   # erzeugt eine lesbare Protokolldatei
   ```
2. In Test- oder Pilotumgebungen ausrollen, nicht zuerst in Produktion.
3. In größeren Umgebungen eine zentrale Verwaltung nutzen, z. B. Windows
   Server Update Services (WSUS) oder eine Update-Management-Lösung.
4. Für langlebige Installationen zusätzlich einen Servicing-Stack-Update bzw.
   ein aktuelles Feature-Update einplanen.
5. Nach dem Patch-Zyklus Rollen und Dienste prüfen (siehe
   [Fehlerbehebung bei Windows-Update-Fehlern](https://learn.microsoft.com/en-us/troubleshoot/windows-server/installing-updates-features-roles/fix-windows-update-errors)).

Grundlegende Absicherung:

```powershell
# BitLocker für Systemlaufwerk aktivieren (Beispiel, Schloss erforderlich)
Enable-BitLocker -MountPoint "C:" -EncryptionMethod XtsAes256 -UsedSpaceOnly
```

Zusätzlich sinnvoll: ein Wiederherstellungsschlüssel sicher verwahlen, Secure Boot
aktiv, UEFI-Passwort setzen, lokale Konten mit langen Kennwörtern versehen und
die Administration möglichst über die-rollenbasierte Zugriffskontrolle (RBAC)
steuern.

### 5. Firewall, Ereignisanzeige und Protokollierung

**Firewall mit erweiterter Sicherheit** steuert ein- und ausgehenden Netzwerkverkehr
und wird über `netfirewall` verwaltet:

```powershell
# Windows-Firewall-Status je Profil anzeigen
Get-NetFirewallProfile | Select-Object Name, Enabled

# Eingehende Regel für einen fiktiven Dienst (Beispiel)
New-NetFirewallRule -DisplayName "Intranet Dateidienst" `
  -Direction Inbound -Protocol TCP -LocalPort 445 -Action Allow -Profile Domain
```

> Regeln möglichst **pro Profil** und **pro Profilbereich** (Domain, Private,
> Öffentlich) bewusst setzen. Für reine Verwaltungsziele empfiehlt sich eine
> Beschränkung auf bestimmte Quellnetze über `-RemoteAddress`.

**Ereignisanzeige** (`eventvwr.msc`) zeigt die Windows-Ereignisprotokolle.
Die wichtigsten Kanäle für die Serveradministration:

| Kanal | Inhalt |
| --- | --- |
| System | Hardware-, Treiber-, Netzwerk- und Dienstereignisse |
| Application | Fehler und Warnungen installierter Anwendungen |
| Security | Anmeldungen, Rechteverwaltung, Richtlinienänderungen |
| PowerShell | PowerShell-Protokoll (Standard: nur Fehler) |
| AD DS-Diagnose | Active-Directory-relevante Ereignisse |

PowerShell zur Ereignissuche:

```powershell
# Systemereignisse der letzten 24 Stunden, Warnungen und Fehler
Get-WinEvent -FilterHashtable @{LogName='System'; StartTime=(Get-Date).AddHours(-24); Level=2,3} |
  Select-Object TimeCreated, Id, ProviderName, Message | Format-List

# Ereignisse einer bestimmten ID (Beispiel: 4625 = fehlgeschlagene Anmeldung)
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625} -MaxEvents 20
```

Protokollierung für den Wiederherstellungsfall: Ereignisprotokolle werden
regelmäßig gesichert (z. B. in eine zentrale Protokolldatenbank) und
Fehlerberichte der Übermittlung an Microsoft bewusst entschieden – je nach
Datenschutzanforderung des Betreibers.

### 6. Backup und Wiederherstellung

Wichtig ist die Unterscheidung:

- **Sicherungskopie** (Backup) ist eine gesicherte Kopie von Daten.
- **Wiederherstellung** (Restore) ist das Zurückspielen dieser Kopie.
- **Systemabbild / VSS**: Volume Shadow Copy Service stellt Momentaufnahmen
  eines Volumes bereit, etwa für die Anwendungssicherung. VSS ist eine
  Voraussetzung für viele Backup-Produkte, aber selbst noch keine Sicherung
  auf externem Medium.

Vorgehen in der Praxis:

1. Sicherungsziel festlegen: externes Medium, anderes System, andere
   Sicherungsschiene – niemals nur dieselbe Platte.
2. Bei AD-DS **System State** und bei Daten die verantwortlichen Freigaben
   sichern; die Datenbank von AD DS wird bei korrekter Sicherung konsistent
   mitgesichert.
3. Aufbewahrung nach Generationen oder Zeitplan (GFS) festlegen.
4. **Wiederherstellung testen** – eine ungetestete Sicherung ist keine
   Sicherung. Nutzerrechte und die Systempartition gehören zum Sicherungsumfang.
5. Sicherungsergebnisse protokollieren und Fehler nachverfolgen.

```powershell
# Volumeshadow Copy Service – Ausgangspunkt vieler Sicherungsprodukte
vssadmin list shadows
vssadmin list writers
```

Windows Server bringt mit „Windows Server Backup“ eine eigene Sicherungsfunktion
mit (`wbadmin`), die je nach Version als optionale Komponente verfügbar ist.

---

## Sicherheitshinweise und typische Fehler

> **Grundsatz:** Ein Server ohne funktionierende, getestete Sicherung ist kein
> Server. Ein Update ohne geprüften Backup-Punkt ebenfalls nicht.

Typische Fehlerquellen bei der Administration:

| Fehler | Auswirkung | Gegenmaßnahme |
| --- | --- | --- |
| Serverseitige Passwortrichtlinie nicht angepasst (Standard: oft zu kurz/zu komplex) | geplante Installation schlägt fehl | lokale Richtlinie *Kontenrichtlinie* vorab anpassen |
| Windows Firewall blockiert die Administrationsdienste | kein Zugriff (RDP/WinRM/Management) | gezielte Inbound-Regeln je Profil setzen |
| Falsche Subnetzmaske oder falsches Gateway | Host sieht sein eigenes Netz nicht | Konfiguration mit `Get-NetIPConfiguration` kontrollieren |
| Verwechslung von Arbeitsgruppe und Domäne | Domänendienste wie Group Policy fehlen | `Get-CimInstance Win32_ComputerSystem \| Select-Object PartOfDomain` prüfen |
| Namensauflösung über die falsche DNS-Zone | Dienste finden ihre Ziele nicht | DNS-Zonen und Forwarder prüfen |
| DHCP ohne Autorisierung in einer Domäne | Fremde Geräte erhalten Adressen | DHCP-Server autorisieren |
| Ausgelassener `wbadmin` nach einem Patch | keine Wiederherstellungsbasis | Backup-Nachweis im Patch-Prozess verankern |
| Standardkonten (Administrator) im Alltag benutzt | Angriffsfläche und unklare Zurechnung | eigener Administrationsweg mit protokollierten Konten |
| Standardkonto `krbtgt` wird verändert | Kerberos-Authentifizierung im Forest kann ausfallen | Konto unverändert lassen, Passwort nur in einem Wartungsfenster ändern |

---

## Weiterführende Quellen

### Microsoft Learn

- Windows Server – Übersicht: <https://learn.microsoft.com/en-us/windows-server/>
- Erste Schritte (Übersicht): <https://learn.microsoft.com/en-us/windows-server/get-started/overview>
- Installieren, Aufrüsten und Migrieren: <https://learn.microsoft.com/en-us/windows-server/get-started/install-upgrade-migrate>
- Editionsvergleich: <https://learn.microsoft.com/en-us/windows-server/get-started/editions-comparison>
- Active Directory Domain Services – Übersicht:
  <https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/virtual-dc/active-directory-domain-services-overview>
- Active Directory installieren:
  <https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/deploy/install-active-directory-domain-services--level-100->
- Active Directory Sicherheitsgruppen:
  <https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-security-groups>
- Active Directory Funktionsebenen:
  <https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/active-directory-functional-levels>
- DNS in Windows und Windows Server:
  <https://learn.microsoft.com/en-us/windows-server/networking/dns/dns-overview>
- DHCP-Server: <https://learn.microsoft.com/en-us/windows-server/networking/technologies/dhcp/dhcp-top>
- Netzwerk im Überblick: <https://learn.microsoft.com/en-us/windows-server/networking/>
- Hyper-V – Übersicht: <https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/overview>
- Windows Server Administration (Übersicht): <https://learn.microsoft.com/en-us/windows-server/administration/>
- Windows PowerShell: <https://learn.microsoft.com/en-us/powershell/scripting/>
- OpenSSH in Windows: <https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-overview>
- Windows-Firewall konfigurieren:
  <https://learn.microsoft.com/en-us/windows/security/operating-system-security/network-security/windows-firewall/configure>
- Windows-Update-Fehler beheben:
  <https://learn.microsoft.com/en-us/troubleshoot/windows-server/installing-updates-features-roles/fix-windows-update-errors>
- Windows Server Update Services (WSUS):
  <https://learn.microsoft.com/en-us/windows-server/administration/windows-server-update-services/get-started/windows-server-update-services-wsus>
- Volume Shadow Copy Service:
  <https://learn.microsoft.com/en-us/windows-server/storage/file-server/volume-shadow-copy-service>
- Audit-Richtlinien für AD DS:
  <https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/audit-policy-recommendations>
- Dienste und benötigte Netzwerkports:
  <https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/service-overview-and-network-port-requirements>

### Weitere Verweise in diesem Repository

- [Private IPv4-Adressbereiche](../networking/private-ipv4.md)
- [VLSM – Variable Length Subnet Masking](../networking/vlsm.md)
