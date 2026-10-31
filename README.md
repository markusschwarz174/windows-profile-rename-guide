# Einen Windows-Benutzerprofilordner umbenennen — ein Praxisleitfaden

**Deutsch** · [English](README.en.md)

Ein praxiserprobter Schritt-für-Schritt-Leitfaden zum Umbenennen eines bestehenden Windows-Benutzerprofilordners (<code>C:\Users\&lt;name&gt;</code>) — einschliesslich aller Probleme, auf die wir tatsächlich gestossen sind, ihrer Ursachen und der genauen Lösungen. Entstanden ist dieser Leitfaden, nachdem wir den Vorgang auf demselben Rechner mit Windows 11 Pro zweimal durchgeführt hatten.

> ⚠️ **Haftungsausschluss:** Das Umbenennen eines Benutzerprofilordners wird von **Microsoft offiziell nicht unterstützt**. Dieser Leitfaden funktionierte bei einer frischen Installation mit wenig installierter Software zuverlässig. Erstelle vor Beginn eine Sicherung oder einen Systemwiederherstellungspunkt, lies den gesamten Leitfaden einmal vollständig durch, bevor du Befehle ausführst, und beachte, dass die Ergebnisse auf einem älteren, intensiv genutzten Rechner abweichen können. Weitere Hinweise findest du am Ende unter [Warum das auf einem älteren System schwieriger ist](#warum-das-auf-einem-älteren-system-schwieriger-ist).
>
> **Empfehlung: Führe diese Änderung möglichst früh durch — idealerweise unmittelbar nach der Einrichtung von Windows und bevor du weitere Software installierst.** Bei einer sauberen Installation gibt es nur wenige, vollständig bekannte Stellen, an denen der alte Profilpfad zwischengespeichert wird. Jede später installierte Anwendung kann eine weitere Stelle erzeugen, die gefunden und korrigiert werden muss.

---

## Ausgangslage und Ziel

**Ausgangslage:** Ein lokales Windows-Konto wurde ursprünglich mit einem provisorischen Namen für den Profilordner angelegt, beispielsweise <code>C:\Users\jdoe</code>. Dieser Ordner sollte einen aussagekräftigen Namen erhalten, beispielsweise <code>C:\Users\jane.doe</code> — ohne das Konto neu anzulegen, Daten zu verlieren oder Software neu zu installieren.

**Ziel:** <code>C:\Users\jdoe</code> → <code>C:\Users\jane.doe</code>. Anschliessend soll das Konto vollständig funktionieren: Desktop, Dokumente und Downloads müssen auf die richtigen Speicherorte zeigen, Systemdialoge und Verknüpfungen dürfen nicht beschädigt sein und installierte Anwendungen sollen — abgesehen von einzelnen veralteten Pfadverweisen — nicht angepasst werden müssen.

In den folgenden Beispielen steht <code>jdoe</code> für den alten und <code>jane.doe</code> für den neuen Namen. Ersetze beide Werte durch deine eigenen Angaben.

---

## Warum das nicht „nur ein Umbenennen des Ordners“ ist

Das eigentliche Umbenennen benötigt nur einen Befehl. Der entscheidende Aufwand entsteht dadurch, dass Windows und zahlreiche Anwendungen den **absoluten Profilpfad** an mehreren Stellen zwischenspeichern. Keine dieser Stellen wird beim Umbenennen des Ordners automatisch aktualisiert:

1. <code>HKLM\...\ProfileList\&lt;SID&gt;\ProfileImagePath</code> — legt systemweit fest, welcher Ordner zu welchem Konto gehört; Änderungen sind nur mit Administratorrechten möglich.
2. <code>HKCU\...\Explorer\Shell Folders</code> und <code>User Shell Folders</code> — enthalten die tatsächlichen Speicherorte von Desktop, Dokumenten, Downloads und weiteren Ordnern; sie gelten pro Benutzer.
3. Mehrere ältere NTFS-**Junctions** im Profilordner dienen der Abwärtskompatibilität mit älterer Software.
4. <code>HKCU:\Environment</code> — enthält benutzerbezogene Umgebungsvariablen wie <code>PATH</code> oder anwendungsspezifische Variablen, die den alten absoluten Pfad enthalten können.
5. **Eigene Registry-Einträge oder Konfigurationsdateien von Drittanbieter-Anwendungen**, in denen der alte Pfad unabhängig gespeichert wurde, etwa Deinstallations-, Installations-, Shell-Erweiterungs- oder Autostart-Einträge.

Der restliche Leitfaden korrigiert diese Stellen der Reihe nach. Er nennt die genauen Befehle und beschreibt die Symptome, die auftreten, wenn ein Schritt fehlt.

---

## Teil 1 — Ordner umbenennen und <code>ProfileImagePath</code> korrigieren

Dieser Teil muss **aus einem anderen Konto heraus** ausgeführt werden. Das Zielkonto muss vollständig abgemeldet sein; ein blosser Benutzerwechsel über die schnelle Benutzerumschaltung genügt nicht, weil der Ordner sonst gesperrt bleibt. Ausserdem sind **Administratorrechte** erforderlich.

~~~powershell
# 1. Prüfen, ob der Ordner vorhanden und nicht gesperrt ist
Test-Path "C:\Users\jdoe"

# 2. Ordner umbenennen
Rename-Item -Path "C:\Users\jdoe" -NewName "jane.doe"

# 3. SID des Kontos ermitteln
Get-ChildItem "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProfileList" | ForEach-Object {
    [PSCustomObject]@{
        SID  = $_.PSChildName
        Path = (Get-ItemProperty $_.PSPath -Name ProfileImagePath -ErrorAction SilentlyContinue).ProfileImagePath
    }
}

# 4. ProfileImagePath auf den neuen Ordner setzen
$sid = "S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX-1001"   # aus Schritt 3
Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProfileList\$sid" `
    -Name "ProfileImagePath" -Value "C:\Users\jane.doe"

# 5. Ergebnis überprüfen
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProfileList\$sid" |
    Select-Object ProfileImagePath
~~~

### Problem bei uns: verwaistes Profil mit zufälligem Suffix (<code>name_xxxxxxx</code>)

Bei einem unserer beiden Konten war Schritt 2 durchgeführt worden, Schritt 4 jedoch noch nicht abgeschlossen, als sich der Benutzer erneut anmeldete. Windows erkannte den umbenannten Ordner deshalb nicht als das zu dieser SID gehörende Profil. Stattdessen legte es ein **neues Profil mit einem zufälligen Suffix** an — beispielsweise <code>jdoe_a1b2c3d4</code> — und liess die beabsichtigte Umbenennung unvollständig zurück.

**So erkennst du das Problem:** Ein Ordner unter <code>C:\Users</code> mit einem zufälligen alphanumerischen Suffix ist ein deutlicher Hinweis darauf, dass Windows ein Ausweichprofil erstellt hat, weil <code>ProfileImagePath</code> nicht zu einem vorhandenen und zugeordneten Ordner passte.

**So wird es behoben:** Prüfe zuerst, in welchem Ordner sich die tatsächlichen Benutzerdaten befinden. Aktuelle Zeitstempel sowie gefüllte Ordner wie <code>Desktop</code>, <code>Documents</code> und <code>AppData</code> zeigen normalerweise, welches Profil im Alltag verwendet wurde. Wiederhole danach die Schritte 1 bis 5, wobei der Ordner mit Suffix als „alter“ Name und der gewünschte bereinigte Name als „neuer“ Name verwendet wird.

~~~powershell
Rename-Item -Path "C:\Users\jdoe_a1b2c3d4" -NewName "jane.doe"
Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProfileList\$sid" `
    -Name "ProfileImagePath" -Value "C:\Users\jane.doe"
~~~

---

## Teil 2 — Alle Verweise neu verknüpfen (im umbenannten Konto ausführen)

Ab hier betreffen alle Änderungen <code>HKCU</code> und damit nur den aktuell angemeldeten Benutzer. Melde dich deshalb im umbenannten Konto selbst an.

### 2.1 Shell-Ordner (Desktop, Dokumente, Downloads, ...)

Es gibt **zwei** parallele Registry-Schlüssel:

- <code>HKCU:\...\Explorer\User Shell Folders</code> — der Schlüssel, aus dem Windows die Pfade tatsächlich auflöst.
- <code>HKCU:\...\Explorer\Shell Folders</code> — eine **zwischengespeicherte Kopie** derselben Pfade, die beim Umbenennen des Profils **nicht** automatisch aktualisiert wird.

**Symptom bei einem veralteten Eintrag:** Komponenten, die direkt auf diese Kopie zugreifen, funktionieren nicht mehr. In unserem Fall liess sich die Windows-App **Einstellungen nicht starten**, bis die Pfade korrigiert waren.

~~~powershell
$old = "C:\Users\jdoe"
$new = "C:\Users\jane.doe"

foreach ($keyPath in @(
    "HKCU:\Software\Microsoft\Windows\CurrentVersion\Explorer\Shell Folders",
    "HKCU:\Software\Microsoft\Windows\CurrentVersion\Explorer\User Shell Folders"
)) {
    (Get-ItemProperty $keyPath).PSObject.Properties |
      Where-Object { $_.Value -is [string] -and $_.Value -match [regex]::Escape($old) } |
      ForEach-Object {
        $newValue = $_.Value -replace [regex]::Escape($old), $new
        Set-ItemProperty -Path $keyPath -Name $_.Name -Value $newValue
        Write-Output "Fixed [$($_.Name)]: $($_.Value) -> $newValue"
      }
}
~~~

Auf unserem System wurden dadurch ungefähr 28 Werte angepasst: Desktop, Documents, Downloads, Music, Pictures, Videos, Favorites, AppData, Local AppData, Cache, Cookies, History, Start Menu, Programs, Startup, Templates, NetHood, PrintHood, Contacts, Links, Saved Games, Searches, RoamingTiles, Libraries, CD Burning, LocalLow und Administrative Tools.

### 2.2 Ältere Kompatibilitäts-Junctions

Windows bewahrt im Stammverzeichnis des Profils mehrere NTFS-**Junctions** für ältere Software auf. Diese erwartet historische Ordnernamen, die auf nicht englischen Systemen lokalisiert sein können, beispielsweise <code>Anwendungsdaten</code> für <code>AppData\Roaming</code>. Das Ziel eines solchen Reparse Points ist als **wörtlicher absoluter Pfad** gespeichert und wird beim gewöhnlichen Umbenennen eines Ordners nicht angepasst.

**Erkennen:** Verlasse dich dafür nicht auf <code>Get-Item.Target</code>, da diese Angabe unzuverlässig sein kann. Mit <code>fsutil</code> wird der tatsächlich gespeicherte Wert angezeigt.

~~~powershell
fsutil reparsepoint query "C:\Users\jane.doe\Application Data"
# Substitute Name kann weiterhin zeigen: \??\C:\Users\jdoe\AppData\Roaming
~~~

**Korrigieren:** Lösche jede Junction und erstelle sie mit dem richtigen Ziel erneut. Anders als symbolische Verknüpfungen erfordert dies keine Administratorrechte, sondern lediglich Schreibzugriff auf den Profilordner. <code>Remove-Item</code> entfernt bei einer Junction nur die Verknüpfung und niemals die dahinterliegenden echten Daten.

~~~powershell
$base = "C:\Users\jane.doe"
$junctions = @{
  "Application Data"  = "AppData\Roaming"
  "Cookies"            = "AppData\Local\Microsoft\Windows\INetCookies"
  "PrintHood"          = "AppData\Roaming\Microsoft\Windows\Printer Shortcuts"
  "My Documents"       = "Documents"
  "Local Settings"     = "AppData\Local"
  "NetHood"            = "AppData\Roaming\Microsoft\Windows\Network Shortcuts"
  "Recent"             = "AppData\Roaming\Microsoft\Windows\Recent"
  "SendTo"             = "AppData\Roaming\Microsoft\Windows\SendTo"
  "Start Menu"         = "AppData\Roaming\Microsoft\Windows\Start Menu"
  "Templates"          = "AppData\Roaming\Microsoft\Windows\Templates"
}
foreach ($kv in $junctions.GetEnumerator()) {
    $linkPath   = Join-Path $base $kv.Key
    $targetPath = Join-Path $base $kv.Value
    if (Test-Path $targetPath) {
        Remove-Item $linkPath -Force
        New-Item -ItemType Junction -Path $linkPath -Target $targetPath | Out-Null
        Write-Output "Fixed: $($kv.Key) -> $targetPath"
    } else {
        Write-Output "SKIP (target missing): $($kv.Key) -> $targetPath"
    }
}
~~~

Die Namen im Beispiel entsprechen den englischen Standardnamen. Verwende auf einem lokalisierten Windows-System die tatsächlich vorhandenen Ordnernamen. Diese lassen sich mit <code>Get-ChildItem $base -Force</code> anzeigen.

### 2.2b Ältere Junctions innerhalb von „Documents“

Dasselbe Problem tritt **eine Ebene tiefer** auf: Auf lokalisierten, nicht englischen Installationen legt Windows innerhalb von <code>Documents</code> weitere Junctions mit historischen Namen an, beispielsweise <code>Eigene Bilder</code> → <code>Pictures</code>, <code>Eigene Musik</code> → <code>Music</code> und <code>Eigene Videos</code> → <code>Videos</code>. Diese Stellen werden leicht übersehen, weil Abschnitt 2.2 nur das Stammverzeichnis des Profils kontrolliert und <code>fsutil reparsepoint query</code> für jede Junction einzeln ausgeführt werden muss.

**Symptom bei einem veralteten Ziel:** Die Junction verweist auf einen nicht mehr vorhandenen Pfad; <code>Test-Path</code> liefert für das Ziel <code>$false</code>. Dies ist kosmetisch harmlos, weil Windows selbst diese Junctions nicht benötigt. Wie im Abschnitt [Warum das nicht „nur ein Umbenennen des Ordners“ ist](#warum-das-nicht-nur-ein-umbenennen-des-ordners-ist) beschrieben, können solche fehlerhaften Verweise später dennoch verwirren und sollten deshalb im selben Durchgang korrigiert werden.

~~~powershell
$docBase = "C:\Users\jane.doe\Documents"
$docJunctions = @{
  "Eigene Bilder" = "..\Pictures"
  "Eigene Musik"  = "..\Music"
  "Eigene Videos" = "..\Videos"
}
foreach ($kv in $docJunctions.GetEnumerator()) {
    $linkPath = Join-Path $docBase $kv.Key
    if (Test-Path $linkPath) {
        $targetPath = Resolve-Path (Join-Path $docBase $kv.Value)
        Remove-Item $linkPath -Force
        New-Item -ItemType Junction -Path $linkPath -Target $targetPath | Out-Null
        Write-Output "Fixed: $($kv.Key) -> $targetPath"
    }
}
~~~

### 2.3 Umgebungsvariablen auf Benutzerebene

~~~powershell
$envKey = "HKCU:\Environment"
$old = "C:\Users\jdoe"
$new = "C:\Users\jane.doe"
(Get-ItemProperty $envKey).PSObject.Properties |
  Where-Object { $_.Value -is [string] -and $_.Value -match [regex]::Escape($old) } |
  ForEach-Object {
    $newValue = $_.Value -replace [regex]::Escape($old), $new
    Set-ItemProperty -Path $envKey -Name $_.Name -Value $newValue
    Write-Output "Fixed [$($_.Name)]: $($_.Value) -> $newValue"
  }
~~~

In unserem Fall wurden dabei zwei veraltete Einträge gefunden: ein von einer früher installierten Anwendung ergänzter Wert in <code>Path</code> und eine veraltete <code>OneDrive</code>-Variable.

### 2.4 Abschliessende Suche

~~~powershell
$leftover = @()
$leftover += (Get-ItemProperty "HKCU:\Software\Microsoft\Windows\CurrentVersion\Explorer\Shell Folders").PSObject.Properties |
  Where-Object { $_.Value -is [string] -and $_.Value -match "jdoe" }
$leftover += (Get-ItemProperty "HKCU:\Environment").PSObject.Properties |
  Where-Object { $_.Value -is [string] -and $_.Value -match "jdoe" }
if ($leftover.Count -eq 0) { "Clean." } else { $leftover }
~~~

### 2.5 Plausibilitätsprüfung

~~~powershell
Start-Process "ms-settings:"
Get-Process | Where-Object { $_.ProcessName -match "SystemSettings" }
~~~

Wenn die Einstellungen-App geöffnet wird und aktiv bleibt, sind die zentralen Verweise auf die Shell-Ordner richtig verknüpft.

---

## Fallstudie: Eine Drittanbieter-Anwendung mit eigenem zwischengespeichertem Pfad reparieren (OneDrive)

Dasselbe Muster — *jede Stelle mit dem alten zwischengespeicherten Pfad finden und korrigieren* — gilt auch für einzelne installierte Anwendungen und nicht nur für Windows selbst. Wir stiessen darauf, als wir OneDrive nach der Umbenennung deinstallieren wollten: **Die Deinstallation schlug fehl**, weil der Registry-Eintrag des Deinstallationsprogramms auf eine nicht mehr vorhandene Datei verwies.

**An diesen Stellen hatte OneDrive den alten Pfad gespeichert; alle liegen unter <code>HKCU</code>:**

| Schlüssel | Wert(e) | Auswirkung eines veralteten Eintrags |
|---|---|---|
| <code>HKCU:\Software\Microsoft\OneDrive\&lt;version&gt;</code> | <code>InstallPath</code>, <code>InstallPaths</code> | Kosmetik / interne Verwaltung |
| <code>HKCU:\Software\Classes\CLSID\{&lt;OneDrive-shell-extension-GUID&gt;}</code> | <code>(default)</code> | Fehlerhaftes Symbol in der Seitenleiste des Explorers |
| <code>HKCU:\...\Uninstall\OneDriveSetup.exe</code> | <code>UninstallString</code>, <code>DisplayIcon</code> | **Deinstallation schlägt vollständig fehl** — dies war die eigentliche Ursache |
| <code>HKCU:\...\Run</code> | <code>OneDrive</code> | Anwendung startet bei der Anmeldung nicht automatisch, ohne eine Meldung anzuzeigen |

**Allgemeines Vorgehen bei jeder Anwendung in dieser Situation:**

1. Durchsuche den bekannten Registry-Baum der Anwendung rekursiv nach dem alten Pfad.
2. Kontrolliere insbesondere <code>HKCU:\...\CurrentVersion\Uninstall\&lt;AppName&gt;</code>. Diesen Eintrag lesen das Deinstallationsprogramm und die Schaltfläche „Deinstallieren“ in den Windows-Einstellungen. Ein veralteter <code>UninstallString</code> ist eine typische Ursache dafür, dass sich eine Anwendung nicht deinstallieren lässt.
3. Kontrolliere den Schlüssel <code>Run</code> auf einen veralteten Autostartpfad.
4. Korrigiere alle Werte, prüfe, ob die Datei am neuen Zielpfad wirklich vorhanden ist, und führe danach das nun korrekt referenzierte Deinstallationsprogramm aus.

~~~powershell
$new = "C:\Users\jane.doe"

Set-ItemProperty "HKCU:\Software\Microsoft\OneDrive\<version>" -Name "InstallPath" `
    -Value "$new\AppData\Local\Microsoft\OneDrive\<version>"

$unKey = "HKCU:\Software\Microsoft\Windows\CurrentVersion\Uninstall\OneDriveSetup.exe"
Set-ItemProperty $unKey -Name "UninstallString" `
    -Value "`"$new\AppData\Local\Microsoft\OneDrive\<version>\OneDriveSetup.exe`"  /uninstall"

$uninstaller = "$new\AppData\Local\Microsoft\OneDrive\<version>\OneDriveSetup.exe"
Start-Process -FilePath $uninstaller -ArgumentList "/uninstall" -Wait
~~~

### Zusätzliche Hürde: verbliebener Prozess oder gesperrter Installationsordner

Nach der Deinstallation lief der Prozess noch weiter, weshalb sich der Installationsordner nicht sofort entfernen liess. Windows kann noch geöffnete Dateien zum Löschen vormerken. Dadurch kann der Ordner für <code>Test-Path</code> bereits verschwunden sein, bevor der Prozess tatsächlich beendet wurde. Warte kurz und kontrolliere danach erneut:

~~~powershell
Stop-Process -Name OneDrive -Force -ErrorAction SilentlyContinue
Start-Sleep -Seconds 2
Remove-Item "$new\AppData\Local\Microsoft\OneDrive" -Recurse -Force -ErrorAction SilentlyContinue
~~~

**Falls <code>Stop-Process -Force</code> weiterhin „Access denied“ meldet:** Prüfe, ob der Prozess zu einem anderen angemeldeten Konto gehört, beispielsweise zu einem weiteren Benutzer, der durch die schnelle Benutzerumschaltung noch im Hintergrund aktiv ist. Prozesse eines anderen Kontos können erwartungsgemäss nicht beendet werden; dies ist kein Fehler.

~~~powershell
$p = Get-CimInstance Win32_Process -Filter "ProcessId = <PID>"
Invoke-CimMethod -InputObject $p -MethodName GetOwner
~~~

### Zusätzliche Hürde: Änderungen am <code>HKLM</code>-Run-Schlüssel erfordern eine PowerShell mit erhöhten Rechten

Autostart-Einträge befinden sich entweder unter <code>HKCU:\...\Run</code> — pro Benutzer und ohne erhöhte Rechte veränderbar — oder unter <code>HKLM:\...\Run</code> — systemweit und nur mit **Administratorrechten** veränderbar. Wird ein <code>HKLM</code>-Eintrag in einer nicht erhöhten PowerShell-Sitzung mit <code>-ErrorAction SilentlyContinue</code> geändert oder entfernt, kann der Fehler unbemerkt bleiben: Der Befehl scheint erfolgreich zu sein, der Wert bleibt jedoch unverändert.

Prüfe deshalb, in welchem Hive sich ein Eintrag tatsächlich befindet, bevor du davon ausgehst, dass die Korrektur funktioniert hat:

~~~powershell
Get-CimInstance Win32_StartupCommand | Select-Object Name, Location
# Ein mit HKLM\... beginnender Speicherort benötigt eine erhöhte Sitzung
# („Als Administrator ausführen“).
~~~

Wenn für eine einmalige Korrektur keine Administratorrechte zur Verfügung stehen oder verwendet werden sollen, kann der Eintrag bestehen bleiben. Er ist nur kosmetisch, weil er lediglich beeinflusst, ob das Programm automatisch startet; die Profilumbenennung selbst wird dadurch nicht blockiert.

---

## Checkliste

- [ ] Zielkonto vollständig abgemeldet, nicht nur gewechselt
- [ ] Ordner aus einer Administratorsitzung heraus umbenannt
- [ ] <code>ProfileImagePath</code> unter <code>HKLM</code> aktualisiert
- [ ] Nach der nächsten Anmeldung ist kein verwaister Ordner mit Zufallssuffix entstanden
- [ ] <code>Shell Folders</code> **und** <code>User Shell Folders</code> unter <code>HKCU</code> korrigiert
- [ ] Alle älteren Junctions korrigiert und mit <code>fsutil reparsepoint query</code> überprüft
- [ ] <code>HKCU:\Environment</code> korrigiert, einschliesslich <code>Path</code> und anwendungsspezifischer Variablen
- [ ] Einstellungen-App und zentrale Windows-Dialoge öffnen ohne Fehler
- [ ] Die abschliessende Suche in Shell Folders und Environment findet keine alten Verweise
- [ ] Drittanbieter-Anwendungen mit eigenen Synchronisations- oder Cache-Ordnern einzeln geprüft, beispielsweise Cloud-Speicher-Clients, Entwicklungsumgebungen mit zuletzt geöffneten Projekten und Sicherungsprogramme

---

## Warum das auf einem älteren System schwieriger ist

Auf einem frisch installierten System ist die Zahl der Stellen, an denen der alte Profilpfad zwischengespeichert wird, **klein und vollständig erfassbar** — genau diese Stellen sind oben dokumentiert. Auf einem länger genutzten Rechner kommt eine nicht abschliessend bestimmbare Reihe weiterer Einträge hinzu: Listen zuletzt geöffneter Projekte in Entwicklungsumgebungen, Browsereinstellungen, andere Cloud-Synchronisationsprogramme, Verknüpfungen mit fest eingetragenen absoluten Zielpfaden, der Suchindex, geplante Aufgaben sowie jede weitere Anwendung, die irgendwo einen absoluten Pfad gespeichert hat.

Die **Methode** bleibt dieselbe: Den zwischengespeicherten Verweis finden und neu schreiben. Realistisch nachzuweisen, dass wirklich **alle** Stellen gefunden wurden, ist jedoch kaum möglich. Genau deshalb unterstützt Microsoft dieses Vorgehen nicht offiziell und die meisten Anleitungen empfehlen stattdessen, ein neues Konto anzulegen und die Daten zu migrieren.
