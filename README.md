# Talk - HackMyVM (Easy)
 
![Talk.png](Talk.png)

## Übersicht

*   **VM:** Talk
*   **Plattform:** HackMyVM (https://hackmyvm.eu/machines/machine.php?vm=Talk)
*   **Schwierigkeit:** Easy
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 11. Oktober 2022
*   **Original-Writeup:** https://alientec1908.github.io/Talk_HackMyVM_Easy/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel der "Talk"-Challenge war die Erlangung von User- und Root-Rechten. Der Weg begann mit der Enumeration eines Webservers (Port 80), auf dem eine PHP-Anwendung mit Login- und Registrierungsfunktion lief. Eine SQL-Injection-Schwachstelle wurde im `username`-Parameter der `login.php` gefunden. Mittels `sqlmap` wurden Klartext-Passwörter aus der Datenbank `chat` (Tabelle `user`) extrahiert, darunter `nona:thatsmynonapass`. Dieses Passwort funktionierte für den SSH-Login (Port 22) als Benutzer `nona`. Die User-Flag wurde in dessen Home-Verzeichnis gefunden. Die Privilegieneskalation zu Root erfolgte durch Ausnutzung einer unsicheren `sudo`-Regel: `nona` durfte `/usr/bin/lynx` als jeder Benutzer ohne Passwort ausführen. Durch Starten von `sudo /usr/bin/lynx` und anschließendes Ausführen eines Shell-Escapes (`!`) innerhalb von `lynx` wurde eine Root-Shell erlangt.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `gobuster`
*   `nmap`
*   `wfuzz`
*   `sqlmap`
*   `hydra`
*   `ssh`
*   `cat`
*   `sudo`
*   `lynx`
*   Standard Linux-Befehle (`id`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Talk" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Web Enumeration:**
    *   IP-Findung mit `arp-scan` (`192.168.2.138`).
    *   `nmap`-Scan identifizierte offene Ports: 22 (SSH - OpenSSH 7.9p1), 80 (HTTP - Nginx 1.14.2 "Talk - Home Page") und 6660 (unbekannter Dienst mit Hinweis auf Benutzer `paul`).
    *   `gobuster` auf Port 80 fand diverse PHP-Dateien (`index.php`, `login.php`, `register.php`, `profile.php`, `home.php`, `logout.php`, `conn.php`) und Verzeichnisse (`/img/`, `/css/`, `/js/`, `/db/`).

2.  **Vulnerability Analysis (SQL Injection):**
    *   `wfuzz` mit SQLi-Payloads gegen `login.php` (Parameter `username`) identifizierte mehrere erfolgreiche Injections (HTTP 302). Ein manueller Test mit `' or 1=1--` bestätigte die SQLi und listete Benutzernamen.
    *   `sqlmap -r talk.sql ...` (Request-Datei für `login.php`) wurde verwendet, um die Datenbank zu untersuchen.
    *   Identifizierung der Datenbank `chat` und der Tabelle `user`.
    *   Extraktion von Klartext-Passwörtern aus `user`, darunter: `pao:pao`, `nona:myfriendtom`, `tina:davidwhatpass`, `jerry:thatsmynonapass`, `david:adrianthebest`.

3.  **Initial Access (SSH Login als `nona`):**
    *   `hydra -t64 ssh://talk.vm -l nona -P [Passwortliste_mit_SQLMap_Funden]` fand das Passwort `thatsmynonapass` für `nona`. (Später wurde im Bericht das Passwort `myfriendtom` für nona im sqlmap output gezeigt, aber `thatsmynonapass` für den SSH-Login von `jerry`s Account verwendet. Hier wird angenommen, dass `thatsmynonapass` für `nona` korrekt war, basierend auf dem Hydra-Erfolg.)
    *   Erfolgreicher SSH-Login als `nona` mit dem Passwort `thatsmynonapass` (`ssh nona@talk.vm`).
    *   User-Flag `wordsarelies` in `/home/nona/user.txt` gelesen.

4.  **Privilege Escalation (Sudo Lynx zu `root`):**
    *   `sudo -l` als `nona` zeigte: `(ALL : ALL) NOPASSWD: /usr/bin/lynx`.
    *   Ausführung von `sudo /usr/bin/lynx`.
    *   Innerhalb von `lynx` wurde durch Drücken der `!`-Taste eine Shell gestartet.
    *   Diese Shell lief mit Root-Rechten (`uid=0(root)`).
    *   Root-Flag `talktomeroot` in `/root/root.txt` gelesen.

## Wichtige Schwachstellen und Konzepte

*   **SQL Injection (SQLi):** Eine klassische SQL-Injection-Schwachstelle im `username`-Parameter eines Login-Formulars ermöglichte die Umgehung der Authentifizierung und das Auslesen von Daten.
*   **Speicherung von Passwörtern im Klartext:** Die Datenbank speicherte Benutzerpasswörter im Klartext, was nach erfolgreicher SQLi deren direkte Kompromittierung ermöglichte.
*   **Passwort-Wiederverwendung (impliziert):** Das Datenbankpasswort eines Benutzers war auch dessen SSH-Passwort.
*   **Unsichere `sudo`-Konfiguration (Lynx):** Die Erlaubnis, `lynx` (einen interaktiven Textbrowser) als `root` ohne Passwort auszuführen, ermöglichte einen einfachen Shell-Escape und somit volle Root-Rechte.
*   **Informationsleck durch Dienst-Banner:** Der Dienst auf Port 6660 gab einen Hinweis auf einen Benutzernamen (`paul`).

## Flags

*   **User Flag (`/home/nona/user.txt`):** `wordsarelies`
*   **Root Flag (`/root/root.txt`):** `talktomeroot`

## Tags

`HackMyVM`, `Talk`, `Easy`, `SQL Injection`, `sqlmap`, `Password Cracking`, `Klartext Passwörter`, `sudo Exploitation`, `Lynx`, `SSH`, `Linux`, `Web`, `Privilege Escalation`
