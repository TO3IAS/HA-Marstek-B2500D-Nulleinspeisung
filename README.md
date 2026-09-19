# Marstek B2500 – Dynamische Nulleinspeisung

Home-Assistant-Blueprint zur dynamischen Regelung eines Marstek B2500 Balkonkraftwerk-Speichers. Der Speicher gibt nur so viel Leistung ab, wie das Haus gerade aus dem Netz bezieht – Ziel ist ein Netzbezug nahe null, ohne dabei ins Netz einzuspeisen.

Die Regelung arbeitet **inkrementell**: Basis ist der zuletzt gesetzte Sollwert, auf den pro Schritt nur ein Teil der gemessenen Abweichung addiert wird. Das hält die träge Rückmeldung von Sensoren und Speicher aus der Regelschleife heraus – der häufigste Grund dafür, dass selbstgebaute Nulleinspeisungen zwischen Einspeisung und Netzbezug hin- und herschwingen.

Dazu kommt ein Punkt, den die meisten Regelungen dieser Art übersehen: Der B2500 kann seine Zellen nur balancieren, wenn er bei vollem Akku Energie ungehindert durchleiten darf. Eine gut funktionierende Nulleinspeisung verhindert genau das. Dieses Blueprint zieht sich deshalb bei 100 % Ladezustand bewusst zurück.

---

## Funktionen

- Nulleinspeisungs-Regelung auf Basis eines Netzleistungssensors
- Inkrementeller Regler mit einstellbarer Verstärkung (Kp) und Totband
- Asymmetrische Wartezeiten: schnell reduzieren, langsam erhöhen
- **Balancing-Automatik**: Überschusseinspeisung tagsüber ein, nachts aus
- **Verriegelung** von Überschusseinspeisung und Zeitplan, die sich gegenseitig ausschließen
- **Anlaufsperre**, weil der Speicher nach dem Einschalten ein bis zwei Minuten braucht
- **Watchdog** gegen eingefrorene Messwerte, unabhängig von eintreffenden Sensordaten
- Tiefentladeschutz mit SoC-Hysterese (Abschalten / Wiedereinschalten)
- Sparmodus: Drosselung auf einen festen Wert bei niedrigem Akkustand
- Lastspitzen-Drosselung, statt sinnlos gegen den Wasserkocher anzuregeln
- Drosselung wird aufgehoben, wenn die PV-Ladeleistung die Ausgabe ohnehin trägt
- Hardware-Mindestleistung: unterhalb davon wird sauber abgeschaltet

---

## Voraussetzungen

- Home Assistant **2024.10** oder neuer
- Ein Marstek B2500, eingebunden z. B. über [hm2mqtt](https://github.com/tomquist/hm2mqtt)
- Ein Netzleistungssensor (Powerfox, Shelly 3EM, Tibber Pulse, …)

Aus der Speicher-Integration werden vier Entitäten benötigt:

| Typ | Beispiel (hm2mqtt) |
| --- | --- |
| Ladeleistung | `sensor.…_total_input_power` |
| Ladezustand | `sensor.…_battery_percentage` |
| Ausgabe-Schalter | `switch.…_time_period_1_enabled` |
| Ausgabe-Sollwert | `number.…_time_period_1_output_value` |
| Überschusseinspeisung (optional) | `switch.…_surplus_feed_in` |

Zusammen mit dem Netzleistungssensor sind das fünf Entitäten, mit dem optionalen Schalter für die Überschusseinspeisung sechs. Die gemessene Ausgangsleistung wird seit v2.0 **nicht** mehr benötigt.

### Vorzeichen des Netzleistungssensors

> **Wichtig:** Der Sensor muss **positiv bei Netzbezug** und **negativ bei Einspeisung** sein.
>
> Ist es umgekehrt, regelt die Automation in die falsche Richtung: Bei PV-Überschuss würde die Ausgabe hochgefahren statt gedrosselt. Vor dem Scharfschalten einmal prüfen, während PV-Überschuss anliegt.

### Überschusseinspeisung am Speicher

Überschusseinspeisung (surplus feed-in) und Zeitplan **dürfen nie gleichzeitig aktiv sein**. Sind sie es doch, stellt der Speicher den Betrieb ein und gibt gar nichts mehr ab.

> **Deshalb gehört der Schalter der Überschusseinspeisung ins Blueprint.** Ist er hinterlegt, hält die Automation beide auseinander: Solange die Überschusseinspeisung läuft, bleibt der Zeitplan aus, und für das Balancing wird sie tagsüber ein- und nach Sonnenuntergang wieder ausgeschaltet.
>
> Bleibt das Feld leer, muss die Überschusseinspeisung **dauerhaft ausgeschaltet** sein. Die Regelung würde sonst den Zeitplan dazuschalten und den Speicher lahmlegen. Ohne Überschusseinspeisung findet allerdings auch kein Balancing statt – siehe [Balancing](#balancing-warum-die-regelung-sich-zurückzieht).

Der zweite Grund für die Automatik: Bei vollem Akku und eingeschalteter Überschusseinspeisung koppelt der Speicher seine Ausgabe an die Eingangsleistung. Nachts ist die null – der Speicher gäbe also nichts ab, obwohl er voll ist. Das Abschalten nach Sonnenuntergang gibt ihn für die normale Regelung wieder frei.

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

> Bei aktivem *Keep last sample* wird der Sensor bei einer Störung der Datenquelle **nicht** `unavailable`, sondern friert auf dem letzten Wert ein. Genau dafür gibt es den Watchdog.

---

## Installation

### Variante A – Import per Link (empfohlen)

[![Blueprint importieren](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FTO3IAS%2FHA-Marstek-B2500D-Nulleinspeisung%2Fblob%2Fmain%2Fmarstek_b2500_nulleinspeisung.yaml)

Alternativ in Home Assistant unter *Einstellungen → Automatisierungen & Szenen → Blueprints → Blueprint importieren* diese URL einfügen:

```
https://github.com/TO3IAS/HA-Marstek-B2500D-Nulleinspeisung/blob/main/marstek_b2500_nulleinspeisung.yaml
```

### Variante B – Manuell

Die Datei `marstek_b2500_nulleinspeisung.yaml` nach `config/blueprints/automation/marstek/` kopieren und die Automatisierungen neu laden (*Entwicklerwerkzeuge → YAML → Automatisierungen*).

### Danach

*Einstellungen → Automatisierungen & Szenen → Automatisierung erstellen → Blueprint verwenden*

> Läuft bereits eine eigene Nulleinspeisungs-Automation, diese vorher **deaktivieren**. Zwei Regler, die auf dieselbe Number-Entity schreiben, bekämpfen sich gegenseitig.

---

## Konfiguration

### Entitäten

Die fünf oben genannten Entitäten auswählen. Als Netzleistungssensor den geglätteten Helfer angeben, nicht den rohen Sensor.

### Regelverhalten

| Option | Standard | Bedeutung |
| --- | --- | --- |
| Regelverstärkung (Kp) | `0.6` | Anteil der Abweichung, der pro Schritt auf den Sollwert addiert wird |
| Totband | `15 W` | Kleinere Sollwertänderungen werden nicht geschrieben |
| Wartezeit beim Erhöhen | `20 s` | Mindestabstand, bevor der Sollwert steigt |
| Wartezeit beim Reduzieren | `8 s` | Mindestabstand, bevor der Sollwert sinkt |
| Anlaufsperre | `120 s` | Nach dem Einschalten nicht nachregeln |

Abschalten (0 W) ist von Wartezeit und Anlaufsperre ausgenommen und erfolgt immer sofort.

### Akku-Schutz

| Option | Standard | Bedeutung |
| --- | --- | --- |
| SoC Abschaltschwelle | `15 %` | Darunter wird die Ausgabe gesperrt |
| SoC Wiedereinschaltschwelle | `25 %` | Erst hier wird wieder freigegeben |
| SoC Drosselschwelle (Eco) | `40 %` | Darunter greift die Drosselung |
| Balancing-Fenster | an | Bei 100 % SoC volle Ausgabe freigeben |
| Schalter Überschusseinspeisung | leer | Optional; leer = dauerhaft von Hand eingeschaltet lassen |
| Nachlaufzeit der Überschusseinspeisung | `10 min` | Wartezeit bei ruhender Ladeleistung vor dem Abschalten |

### Leistungsgrenzen

| Option | Standard | Bedeutung |
| --- | --- | --- |
| Maximale Ausgabeleistung | `800 W` | Obergrenze im Normalbetrieb und beim Balancing |
| Gedrosselte Ausgabeleistung | `200 W` | Fester Wert bei Drosselung |
| Hardware-Mindestleistung | `80 W` | Darunter wird abgeschaltet |
| Hysterese der Drosselung | `100 W` | Zusatzbedarf, um die Drosselung zu verlassen |

> Die gedrosselte Ausgabeleistung muss **mindestens so hoch** wie die Hardware-Mindestleistung sein. Sonst entsteht ein Zielwert, den keiner der beiden Ausführungszweige annimmt – die Automation täte in dem Fall schlicht nichts.

### Feintuning

| Option | Standard | Bedeutung |
| --- | --- | --- |
| Maximales Alter des Netzwerts | `120 s` | Watchdog, `0` deaktiviert die Prüfung |
| Zusätzlicher SoC-Trigger | an | Reagiert sofort auf Schwellwerte statt erst beim nächsten Netz-Update |

---

## Funktionsweise

Der neue Sollwert entsteht aus dem **zuletzt gesetzten** Sollwert plus einem Anteil des gemessenen Netzbezugs:

```
neuer_sollwert = alter_sollwert + (netzleistung × Kp)
```

Weil jeder Schritt auf dem vorherigen aufbaut, baut die Regelung den Restfehler über mehrere Durchläufe vollständig ab. Es bleibt keine dauerhafte Abweichung übrig – was stehen bleibt, kommt allein vom Totband. Deshalb wirkt das Totband geteilt durch Kp auf den Netzbezug: 15 W bei Kp 0,6 entsprechen rund 25 W Toleranz am Netzanschluss.

Ist der Ausgabe-Schalter aus, gilt als Basis 0 W, weil dann real nichts abgegeben wird.

Anschließend läuft eine Kaskade von oben nach unten – die erste zutreffende Regel gewinnt:

| Stufe | Bedingung | Ergebnis |
| --- | --- | --- |
| 0 | Netzwert ungültig oder zu alt | 0 W, Switch aus |
| 1 | Überschusseinspeisung ist eingeschaltet | 0 W, Zeitplan aus (Balancing) |
| 2 | SoC ≤ Abschaltschwelle | 0 W, Switch aus |
| 2 | SoC < Wiedereinschaltschwelle **und** Ausgabe ist aus | bleibt aus |
| 3 | Sollwert < Hardware-Minimum (inkl. Einspeisung) | 0 W, Switch aus |
| 4 | (SoC < Drosselschwelle **oder** Sollwert > Maximum) **und** Ladeleistung trägt die Ausgabe nicht | Drosselwert |
| 5 | alles andere | Sollwert, gedeckelt auf das Maximum |

Danach greifen Totband, Wartezeit und Anlaufsperre. Der Switch wird nur bei echtem Zustandswechsel geschaltet, und der Watt-Wert wird **vor** dem Einschalten gesetzt, damit der Speicher beim Anlaufen nie kurz einen veralteten Wert ausgibt.

### Balancing: warum die Regelung sich zurückzieht

Der B2500 gleicht seine Zellspannungen ausschließlich während der Überschusseinspeisung aus, und zwar passiv, indem er die vollsten Zellen entlädt. Das braucht Zeit und setzt voraus, dass der Speicher bei 100 % Ladezustand die ankommende Energie ungehindert durchleiten darf.

Genau hier liegt der Konflikt: Eine Nulleinspeisungs-Regelung deckelt die Ausgabe auf den Hausverbrauch. Liefert die PV 700 W, das Haus braucht aber nur 200 W, wird nichts durchgeleitet – und das Balancing kommt nicht in Gang. Je besser die Regelung funktioniert, desto weniger balanciert der Speicher.

Driften die Zellen auseinander, diktiert beim Laden die vollste Zelle das Ende des Ladevorgangs und beim Entladen die schwächste. Der Kapazitätsverlust summiert sich über Monate und lässt sich nicht mehr zurückholen, ohne dass wieder balanciert wird.

Die beiden Betriebsarten schließen sich außerdem technisch aus: Läuft die Überschusseinspeisung, muss der Zeitplan abgeschaltet sein. Die Automation regelt also nicht mit, sondern tritt ganz zur Seite.

Daraus ergibt sich der Tagesablauf:

| Zeitpunkt | Was passiert |
| --- | --- |
| Speicher erreicht tagsüber 100 % | Zeitplan wird abgeschaltet, dann die Überschusseinspeisung eingeschaltet |
| Nachmittag | Der Speicher leitet durch und balanciert; die Regelung ruht |
| Nach Sonnenuntergang, Ladeleistung seit der Nachlaufzeit bei null | Überschusseinspeisung aus |
| Abend und Nacht | Normale Nulleinspeisungs-Regelung über den Zeitplan |

> **Der Preis:** Am Nachmittag wird ins Netz eingespeist, sobald die PV-Leistung über dem Hausverbrauch liegt. Wer das partout nicht will, schaltet die Balancing-Automatik ab – sollte dem Speicher dann aber auf anderem Weg regelmäßig Zeit bei 100 % Ladezustand mit eingeschalteter Überschusseinspeisung verschaffen.

Die Tag/Nacht-Unterscheidung läuft über `sun.sun`. Sie verhindert, dass die Automation nachts zwischen Ein und Aus pendelt, wenn der Speicher noch bei 100 % steht. Fehlt die Entity, wird durchgehend Tag angenommen und nur noch eingeschaltet.

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

**Die Ausgabe schwingt, der Netzbezug pendelt in beide Richtungen**
Kp verringern, zuerst auf `0.4`. Hilft das nicht, die Wartezeit beim Erhöhen auf 30 s anheben.

**Nach dem Einschalten springt die Ausgabe auf Maximum und speist kurz ein**
Die Anlaufsperre ist zu kurz. Beobachten, wie lange der Speicher nach dem Einschalten tatsächlich braucht, und den Wert entsprechend erhöhen.

**Es bleibt ein konstanter Netzbezug stehen**
Das kommt vom Totband. Auf 10 W verringern, wenn es enger sein soll – dafür werden häufiger Sollwerte geschrieben.

**Die Ausgabe springt zwischen Drosselwert und Vollwert**
Passiert bei schwankender PV nahe der Umschaltschwelle. Hysterese der Drosselung auf 150–200 W erhöhen.

**Der Speicher gibt gar nichts mehr ab**
Zuerst prüfen, ob Überschusseinspeisung und Zeitplan gleichzeitig eingeschaltet sind – das verträgt die Firmware nicht. Ist der Schalter im Blueprint hinterlegt, kann das nicht passieren.

**Der Speicher gibt bei 100 % SoC nichts ab, obwohl er voll ist**
Bei eingeschalteter Überschusseinspeisung hängt die Ausgabe an der Eingangsleistung. Ohne PV bleibt sie damit bei null. Den Schalter im Blueprint hinterlegen, dann schaltet die Automation abends um.

**Die Überschusseinspeisung wird nie eingeschaltet**
Die Automation schaltet sie erst bei 100 % Ladezustand ein. Erreicht der Speicher die 100 % nie, passiert auch nichts. In dem Fall die Entladetiefe vorübergehend begrenzen, damit er mittags wirklich voll wird.

**Die Automation regelt gar nicht mehr**
Die Regelung liest ihre eigene Basis aus der Number-Entity und misst die Wartezeit über deren `last_changed`. Meldet die Integration diese Entity nicht zuverlässig zurück, steht die Regelung. Zum Test die Wartezeiten auf `0` setzen – regelt es dann wieder, liegt es daran.

**Die Automation läuft, tut aber nichts**
In den Traces prüfen, an welcher Bedingung sie abbricht. Häufigste Ursachen: Totband nicht überschritten, Wartezeit oder Anlaufsperre noch nicht abgelaufen, oder ein Sensor liefert `unavailable`.

---

## Grenzen

Perfekte Nulleinspeisung ist mit diesem Aufbau nicht erreichbar. Aus Sensortakt, Glättung und Reaktionszeit des Speichers ergibt sich eine Gesamtverzögerung von typischerweise 15–20 Sekunden, bis eine Laständerung vollständig ausgeregelt ist. Bei jedem Ein- und Ausschalten größerer Verbraucher entsteht in dieser Zeit unvermeidlich Netzbezug oder Einspeisung. Das liegt an der Kette aus Messung und Hardware, nicht an der Regelung.

---

## Changelog

### v2.2

- **Verriegelung:** Überschusseinspeisung und Zeitplan werden nie gleichzeitig aktiviert. Beides zusammen legt den Speicher lahm
- Das Balancing-Fenster gibt nicht mehr die volle Ausgabe frei, sondern schaltet den Zeitplan ab und überlässt dem Gerät das Feld
- Beim Einschalten der Überschusseinspeisung wird zwingend zuerst der Zeitplan deaktiviert

### v2.1

- Optionale Automatik für die **Überschusseinspeisung**: wird der Schalter hinterlegt, schaltet die Automation ihn bei 100 % Ladezustand tagsüber ein und nach Sonnenuntergang wieder aus
- Damit steht der volle Speicher nachts wieder für die normale Regelung zur Verfügung, statt seine Ausgabe an die Eingangsleistung zu koppeln
- Neue Option: Nachlaufzeit vor dem Abschalten

### v2.0

- **Balancing-Fenster:** Bei 100 % SoC und anliegender Ladeleistung wird die volle Ausgabe freigegeben, damit das BMS die Zellen ausgleichen kann
- **Anlaufsperre:** Nach dem Einschalten wird für eine einstellbare Zeit nicht nachgeregelt
- **Watchdog:** Ein zusätzlicher Minuten-Trigger sorgt dafür, dass die Altersprüfung des Netzwerts auch dann greift, wenn gar keine Sensorwerte mehr eintreffen. Bisher konnte sie das nicht, weil die Automation ohne Sensoränderung nie lief
- Bei ungültigem oder veraltetem Netzwert wird jetzt **abgeschaltet** statt nur abgebrochen
- Regelbasis ist der zuletzt gesetzte Sollwert statt der gemessenen Ausgangsleistung – die Sensorverzögerung fällt damit aus der Regelschleife
- Der Sensor für die Ausgangsleistung wird nicht mehr benötigt
- Erkennung „Ausgabe ist aus" über den Schalterzustand statt über eine Ausgangsleistung von 0 W
- Totband-Vorgabe von 25 W auf 15 W gesenkt

### v1.0

- Erstveröffentlichung

---

## Haftungsausschluss

Dieses Blueprint wird ohne jede Gewähr bereitgestellt. Die Nutzung erfolgt auf eigene Verantwortung. Für Schäden an Geräten, entgangene Erträge oder Probleme mit dem Netzbetreiber wird keine Haftung übernommen. Die Vorgaben des Netzbetreibers und die Anmeldepflichten für Balkonkraftwerke und Speicher sind vom Betreiber selbst einzuhalten.

Dieses Projekt steht in keiner Verbindung zu Marstek, Hame, Powerfox oder Home Assistant.

---

## Lizenz

MIT – siehe [LICENSE](LICENSE).
