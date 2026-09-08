# Marstek B2500 – Dynamische Nulleinspeisung

Home-Assistant-Blueprint zur dynamischen Regelung eines Marstek B2500 Balkonkraftwerk-Speichers. Der Speicher gibt nur so viel Leistung ab, wie das Haus gerade aus dem Netz bezieht – Ziel ist ein Netzbezug nahe null, ohne dabei ins Netz einzuspeisen.

Die Regelung ist bewusst **gedämpft** ausgelegt. Zwischen einer Sollwertänderung und dem Zeitpunkt, an dem die Sensoren deren Wirkung zurückmelden, vergehen je nach Aufbau 10–20 Sekunden. Wer in dieser Zeit mehrfach den vollen Fehler nachregelt, erzeugt Schwingungen. Dieses Blueprint korrigiert deshalb pro Schritt nur einen Teil der Abweichung und hält Mindestwartezeiten zwischen Sollwertänderungen ein.

---

## Funktionen

- Nulleinspeisungs-Regelung auf Basis eines Netzleistungssensors
- Gedämpfter P-Regler mit einstellbarer Verstärkung (Kp) und Totband
- Asymmetrische Wartezeiten: schnell reduzieren, langsam erhöhen
- Tiefentladeschutz mit SoC-Hysterese (Abschalten / Wiedereinschalten)
- Sparmodus: Drosselung auf einen festen Wert bei niedrigem Akkustand
- Lastspitzen-Drosselung, statt sinnlos gegen den Wasserkocher anzuregeln
- Drosselung wird aufgehoben, wenn die PV-Ladeleistung die Ausgabe ohnehin trägt (mit eigener Hysterese)
- Hardware-Mindestleistung: unterhalb davon wird sauber abgeschaltet statt unrealistische Werte zu senden
- Schutz vor unverfügbaren **und** eingefrorenen Sensorwerten

---

## Voraussetzungen

- Home Assistant **2024.10** oder neuer
- Ein Marstek B2500, eingebunden z. B. über [hm2mqtt](https://github.com/tomquist/hm2mqtt)
- Ein Netzleistungssensor (Powerfox, Shelly 3EM, Tibber Pulse, …)

Aus der Speicher-Integration werden vier Entitäten benötigt:

| Typ | Beispiel (hm2mqtt) |
| --- | --- |
| Ausgangsleistung | `sensor.…_total_output_power` |
| Ladeleistung | `sensor.…_total_input_power` |
| Ladezustand | `sensor.…_battery_percentage` |
| Ausgabe-Schalter | `switch.…_time_period_1_enabled` |
| Ausgabe-Sollwert | `number.…_time_period_1_output_value` |

### Vorzeichen des Netzleistungssensors

> **Wichtig:** Der Sensor muss **positiv bei Netzbezug** und **negativ bei Einspeisung** sein.
>
> Ist es umgekehrt, regelt die Automation in die falsche Richtung: Bei PV-Überschuss würde die Ausgabe hochgefahren statt gedrosselt. Vor dem Scharfschalten einmal prüfen, während PV-Überschuss anliegt.

### Geglätteter Sensor (empfohlen)

Der rohe Netzsensor ist für die Regelung meist zu unruhig. Ein Statistik-Helfer glättet ihn:

*Einstellungen → Geräte & Dienste → Helfer → Helfer erstellen → Statistik*

| Option | Wert |
| --- | --- |
| Statistische Kenngröße | `Median` |
| Sampling size | `20` |
| Max age | `00:00:15` |
| Keep last sample | an |
| Precision | `0` |

Der Median ignoriert einzelne Ausreißer vollständig – anders als der Mittelwert, den ein kurzer 2000-W-Peak sofort mitzieht. Damit der Median wirkt, müssen mindestens drei Werte im Fenster liegen: Bei einem Sensor, der alle 5 Sekunden aktualisiert, sind 15 Sekunden Fenster das Minimum. Die Sampling size wird nur so groß gesetzt, dass sie nie limitiert – das Fenster definiert allein die Zeit.

Größere Fenster glätten besser, kosten aber Reaktionszeit: Der Median hinkt der Realität etwa ein halbes Fenster hinterher.

> Bei aktivem *Keep last sample* wird der Sensor bei einer Störung der Datenquelle **nicht** `unavailable`, sondern friert auf dem letzten Wert ein. Das Blueprint fängt das über die Option *Maximales Alter des Netzwerts* ab.

---

## Installation

### Variante A – Import per Link (empfohlen)

[![Blueprint importieren](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Ftobiasscheiderer-prog%2FHA-Marstek-B2500D-Nulleinspeisung%2Fblob%2Fmain%2Fmarstek_b2500_nulleinspeisung.yaml)

Alternativ in Home Assistant unter *Einstellungen → Automatisierungen & Szenen → Blueprints → Blueprint importieren* diese URL einfügen:

```
https://github.com/tobiasscheiderer-prog/HA-Marstek-B2500D-Nulleinspeisung/blob/main/marstek_b2500_nulleinspeisung.yaml
```

### Variante B – Manuell

Die Datei `marstek_b2500_nulleinspeisung.yaml` nach `config/blueprints/automation/marstek/` kopieren und die Automatisierungen neu laden (*Entwicklerwerkzeuge → YAML → Automatisierungen*).

### Danach

*Einstellungen → Automatisierungen & Szenen → Automatisierung erstellen → Blueprint verwenden*

> Läuft bereits eine eigene Nulleinspeisungs-Automation, diese vorher **deaktivieren**. Zwei Regler, die auf dieselbe Number-Entity schreiben, bekämpfen sich gegenseitig.

---

## Konfiguration

### Entitäten

Die sechs oben genannten Entitäten auswählen. Als Netzleistungssensor den geglätteten Helfer angeben, nicht den rohen Sensor.

### Regelverhalten

| Option | Standard | Bedeutung |
| --- | --- | --- |
| Regelverstärkung (Kp) | `0.6` | Anteil der Abweichung, der pro Schritt ausgeregelt wird |
| Totband | `25 W` | Kleinere Abweichungen werden ignoriert |
| Wartezeit beim Erhöhen | `20 s` | Mindestabstand, bevor der Sollwert steigt |
| Wartezeit beim Reduzieren | `8 s` | Mindestabstand, bevor der Sollwert sinkt |

Abschalten (0 W) ist von der Wartezeit ausgenommen und erfolgt immer sofort.

### Akku-Schutz

| Option | Standard | Bedeutung |
| --- | --- | --- |
| SoC Abschaltschwelle | `15 %` | Darunter wird die Ausgabe gesperrt |
| SoC Wiedereinschaltschwelle | `25 %` | Erst hier wird wieder freigegeben |
| SoC Drosselschwelle (Eco) | `40 %` | Darunter greift die Drosselung |

### Leistungsgrenzen

| Option | Standard | Bedeutung |
| --- | --- | --- |
| Maximale Ausgabeleistung | `800 W` | Obergrenze im Normalbetrieb |
| Gedrosselte Ausgabeleistung | `200 W` | Fester Wert bei Drosselung |
| Hardware-Mindestleistung | `80 W` | Darunter wird abgeschaltet |
| Hysterese der Drosselung | `100 W` | Zusatzbedarf, um die Drosselung zu verlassen |

> Die gedrosselte Ausgabeleistung muss **mindestens so hoch** wie die Hardware-Mindestleistung sein. Sonst entsteht ein Zielwert, den keiner der beiden Ausführungszweige annimmt – die Automation täte in dem Fall schlicht nichts.

### Feintuning

| Option | Standard | Bedeutung |
| --- | --- | --- |
| Maximales Alter des Netzwerts | `120 s` | Schutz vor eingefrorenen Werten, `0` deaktiviert die Prüfung |
| Zusätzlicher SoC-Trigger | an | Reagiert sofort auf Schwellwerte statt erst beim nächsten Netz-Update |

---

## Funktionsweise

Der Zielwert entsteht aus der aktuellen Ausgabe plus dem gedämpften Netzbezug:

```
sollwert = ausgangsleistung + (netzleistung × Kp)
```

Anschließend läuft eine Kaskade von oben nach unten – die erste zutreffende Regel gewinnt:

| Stufe | Bedingung | Ergebnis |
| --- | --- | --- |
| 1 | SoC ≤ Abschaltschwelle | 0 W, Switch aus |
| 1 | SoC < Wiedereinschaltschwelle **und** Ausgabe steht auf 0 | bleibt 0 W |
| 2 | Sollwert < Hardware-Minimum (inkl. Einspeisung) | 0 W, Switch aus |
| 3 | (SoC < Drosselschwelle **oder** Sollwert > Maximum) **und** Ladeleistung trägt die Ausgabe nicht | Drosselwert |
| 4 | alles andere | Sollwert, gedeckelt auf das Maximum |

Danach greifen noch Totband und Wartezeit. Der Switch wird nur bei echtem Zustandswechsel geschaltet, und der Watt-Wert wird **vor** dem Einschalten gesetzt, damit der Speicher beim Anlaufen nie kurz einen veralteten Wert ausgibt.

### Warum die Drosselung bei genug Ladeleistung entfällt

Der Sparmodus soll den Akku vor dem Leerlaufen schützen. Fließt gerade mehr Ladeleistung hinein, als abgegeben werden soll, wird der Akku netto trotzdem voller – dann gibt es keinen Grund zu drosseln. Verglichen wird deshalb die Ladeleistung gegen die **tatsächlich geplante Ausgabe**, nicht gegen den Bedarf.

Beispiel bei 35 % SoC, Bedarf 1500 W:

| Ladeleistung | Ausgabe | Warum |
| --- | --- | --- |
| 0 W | 200 W | Akku würde sich leeren |
| 500 W | 200 W | trägt die 800 W nicht |
| 1000 W | 800 W | volle Ausgabe, netto weiterhin +200 W Ladung |

---

## Feintuning in der Praxis

**Die Ausgabe schwingt, der Netzbezug pendelt zwischen −200 und +200 W**
Kp verringern, zuerst auf `0.4`. Hilft das nicht, die Wartezeit beim Erhöhen auf 30 s anheben. Erst danach am Totband drehen.

**Es bleibt dauerhaft ein Netzbezug von 50–80 W stehen**
Das ist das normale Verhalten eines P-Reglers und die günstigere Seite des Fehlers. Wer es enger will: Kp auf `0.8` und Totband auf `15 W`.

**Die Ausgabe springt zwischen Drosselwert und Vollwert**
Passiert bei schwankender PV nahe der Umschaltschwelle. Hysterese der Drosselung auf 150–200 W erhöhen.

**Die Automation regelt gar nicht mehr**
Die Wartezeiten messen über `last_changed` der Number-Entity. Meldet die Integration diese Entity nicht zuverlässig zurück, blockiert die Wartezeit dauerhaft. Zum Test beide Wartezeiten auf `0` setzen – regelt es dann wieder, liegt es daran.

**Die Automation läuft, tut aber nichts**
In den Traces prüfen, an welcher Bedingung sie abbricht. Häufigste Ursachen: Totband nicht überschritten, Wartezeit noch nicht abgelaufen, oder ein Sensor liefert `unavailable`.

---

## Grenzen

Perfekte Nulleinspeisung ist mit diesem Aufbau nicht erreichbar. Aus Sensortakt, Glättung und Reaktionszeit des Speichers ergibt sich eine Gesamtverzögerung von typischerweise 15–20 Sekunden, bis eine Laständerung vollständig ausgeregelt ist. Bei jedem Ein- und Ausschalten größerer Verbraucher entsteht in dieser Zeit unvermeidlich Netzbezug oder Einspeisung. Das liegt an der Kette aus Messung und Hardware, nicht an der Regelung.

---

## Haftungsausschluss

Dieses Blueprint wird ohne jede Gewähr bereitgestellt. Die Nutzung erfolgt auf eigene Verantwortung. Für Schäden an Geräten, entgangene Erträge oder Probleme mit dem Netzbetreiber wird keine Haftung übernommen. Die Vorgaben des Netzbetreibers und die Anmeldepflichten für Balkonkraftwerke und Speicher sind vom Betreiber selbst einzuhalten.

Dieses Projekt steht in keiner Verbindung zu Marstek, Hame, Powerfox oder Home Assistant.

---

## Lizenz

MIT – siehe [LICENSE](LICENSE).
