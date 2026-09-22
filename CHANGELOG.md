# Changelog

Alle nennenswerten Änderungen am Blueprint. Die Version steht jeweils am Anfang der Blueprint-Beschreibung.

## v2.4 – 2026-09-22

### Behoben

- **Verriegelung beim Einschalten der Überschusseinspeisung:** War der Zeitplan aus und lag Netzbezug an, schaltete derselbe Durchlauf nach der Überschusseinspeisung auch den Zeitplan wieder ein, weil der Zielwert noch mit dem alten Zustand berechnet war. Beides gleichzeitig aktiv legt den Speicher lahm. Jetzt:
  - Vor dem Einschalten der Überschusseinspeisung wird auf die Bestätigung „Zeitplan aus“ gewartet. Ohne Bestätigung bricht der Durchlauf ab.
  - Nach jedem Schalten der Überschusseinspeisung wird auf die Bestätigung gewartet und der Durchlauf beendet.
  - Der Zeitplan wird nur eingeschaltet, wenn die Überschusseinspeisung live eindeutig „aus“ meldet.
- Balancing-Fenster (Eintritt, Dauer, Ausstieg, Nachlaufzeit) unverändert

### Hinzugefügt

- Optionale **PV-Durchleitung trotz SoC-Sperre** (Standard aus): Gibt während der Sperre Leistung ab, solange der Akku mindestens um die Reserve weiterlädt. Neue Optionen: *PV-Durchleitung trotz SoC-Sperre* und *Reserve der PV-Durchleitung* (50 W). Setzt den Merker für die SoC-Sperre voraus und gilt nie unterhalb der Abschaltschwelle.

### Dokumentation

- README mit dem Blueprint abgeglichen: Entitätenliste mit Pflicht- und optionalen Entitäten, Kaskade mit Stufe 2b, neue Abschnitte zur Anlaufsperre und zur PV-Durchleitung, Ablauf der Verriegelung
- Korrigiert: Ohne hinterlegten Schalter muss die Überschusseinspeisung dauerhaft **aus** bleiben (stand fälschlich als „eingeschaltet lassen“ in der Tabelle)
- Troubleshooting: Übrig bleiben bis zu ±Totband/Kp am Netzanschluss, als Bezug oder Einspeisung
- Changelog aus dem README in diese Datei verschoben

## v2.3 – 2026-09-22

- **Keine Drosselung mehr bei Bedarf über dem Maximum.** Bisher sprang die Ausgabe bei Dauerlasten knapp über der maximalen Ausgabeleistung (mit Standardwerten ca. 820–1200 W) ständig zwischen Drosselwert und Vollwert. Bei noch höherer Last blieb sie trotz vollem Akku auf dem Drosselwert. Jetzt bleibt sie am Maximum; gedrosselt wird nur noch bei niedrigem Akkustand
- Neuer optionaler **Merker für die SoC-Sperre** (`input_boolean`): Die Wiedereinschaltschwelle gilt damit nur noch nach einer Abschaltung wegen Tiefentladung, nicht nach jeder Abschaltung
- Nachlaufzeit der Überschusseinspeisung: Schwankungen der Ladeleistung unter 10 W setzen sie nicht mehr zurück
- Watchdog prüft `last_reported` statt `last_updated`: Ein gleichbleibender, aber weiterhin gemeldeter Netzwert löst keine Abschaltung mehr aus
- Nach dem Schreiben des Sollwerts wartet die Automation bis zu 15 s auf die Rückmeldung der Number-Entity (`mode: single` statt `restart`)

## v2.2 – 2026-09-19

- **Verriegelung:** Überschusseinspeisung und Zeitplan werden nie gleichzeitig aktiviert. Beides zusammen legt den Speicher lahm
- Das Balancing-Fenster gibt nicht mehr die volle Ausgabe frei, sondern schaltet den Zeitplan ab und überlässt dem Gerät das Feld
- Beim Einschalten der Überschusseinspeisung wird zwingend zuerst der Zeitplan deaktiviert

## v2.1 – 2026-09-19

- Optionale Automatik für die **Überschusseinspeisung**: wird der Schalter hinterlegt, schaltet die Automation ihn bei 100 % Ladezustand tagsüber ein und nach Sonnenuntergang wieder aus
- Damit steht der volle Speicher nachts wieder für die normale Regelung zur Verfügung, statt seine Ausgabe an die Eingangsleistung zu koppeln
- Neue Option: Nachlaufzeit vor dem Abschalten

## v2.0 – 2026-09-19

- **Balancing-Fenster:** Bei 100 % SoC und anliegender Ladeleistung wird die volle Ausgabe freigegeben, damit das BMS die Zellen ausgleichen kann
- **Anlaufsperre:** Nach dem Einschalten wird für eine einstellbare Zeit nicht nachgeregelt
- **Watchdog:** Ein zusätzlicher Minuten-Trigger sorgt dafür, dass die Altersprüfung des Netzwerts auch dann greift, wenn gar keine Sensorwerte mehr eintreffen. Bisher konnte sie das nicht, weil die Automation ohne Sensoränderung nie lief
- Bei ungültigem oder veraltetem Netzwert wird jetzt **abgeschaltet** statt nur abgebrochen
- Regelbasis ist der zuletzt gesetzte Sollwert statt der gemessenen Ausgangsleistung – die Sensorverzögerung fällt damit aus der Regelschleife
- Der Sensor für die Ausgangsleistung wird nicht mehr benötigt
- Erkennung „Ausgabe ist aus" über den Schalterzustand statt über eine Ausgangsleistung von 0 W
- Totband-Vorgabe von 25 W auf 15 W gesenkt

## v1.0 – 2026-09-08

- Erstveröffentlichung
