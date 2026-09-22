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
- Optional: **PV-Durchleitung** während der SoC-Sperre, solange der Akku trotzdem lädt
- Sparmodus: Drosselung auf einen festen Wert bei niedrigem Akkustand
- Drosselung wird aufgehoben, wenn die PV-Ladeleistung die Ausgabe ohnehin trägt
- Hardware-Mindestleistung: unterhalb davon wird sauber abgeschaltet

---

## Voraussetzungen

- Home Assistant **2024.10** oder neuer
- Ein Marstek B2500, eingebunden z. B. über [hm2mqtt](https://github.com/tomquist/hm2mqtt)
- Ein Netzleistungssensor (Powerfox, Shelly 3EM, Tibber Pulse, …)

Pflicht sind fünf Entitäten, vier davon aus der Speicher-Integration:

| Typ | Beispiel (hm2mqtt) |
| --- | --- |
| Netzleistung | Powerfox, Shelly 3EM, …, am besten geglättet (siehe unten) |
| Ladeleistung | `sensor.…_total_input_power` |
| Ladezustand | `sensor.…_battery_percentage` |
| Ausgabe-Schalter | `switch.…_time_period_1_enabled` |
| Ausgabe-Sollwert | `number.…_time_period_1_output_value` |

Optional, aber empfohlen:

| Typ | Beispiel |
| --- | --- |
| Schalter Überschusseinspeisung | `switch.…_surplus_feed_in` (hm2mqtt) |
| Merker für die SoC-Sperre | selbst angelegter `input_boolean`-Helfer, siehe [unten](#merker-für-die-soc-sperre-empfohlen) |

Für die Tag/Nacht-Erkennung nutzt die Automation außerdem `sun.sun`. Die gemessene Ausgangsleistung wird seit v2.0 **nicht** mehr benötigt.

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

So hält die Automation die Verriegelung ein:

1. Vor dem Einschalten der Überschusseinspeisung schaltet sie den Zeitplan aus und wartet bis zu 15 s, bis er als aus gemeldet wird. Ohne diese Bestätigung bricht sie ab und versucht es beim nächsten Trigger erneut.
2. Nach jedem Schalten der Überschusseinspeisung wartet sie auf die Bestätigung und beendet dann den Durchlauf. Erst der nächste Trigger regelt wieder, mit dem neuen Zustand.
3. Direkt vor dem Einschalten des Zeitplans liest sie den aktuellen Zustand der Überschusseinspeisung. Meldet der Schalter nicht eindeutig „aus“, auch bei `unavailable`, bleibt der Zeitplan aus.

> Zu prüfen: Ob hm2mqtt Schalterzustände erst nach Bestätigung durch das Gerät meldet oder sofort optimistisch setzt. Im zweiten Fall bestätigen die Wartezeiten nur den Zustand in Home Assistant.

### Merker für die SoC-Sperre (empfohlen)

Nach einer Abschaltung wegen Tiefentladung gibt die Automation die Ausgabe erst ab der Wiedereinschaltschwelle wieder frei. Damit sie diese Sperre von anderen Abschaltungen unterscheiden kann (geringer Bedarf, Watchdog, Überschusseinspeisung), braucht sie einen Merker:

*Einstellungen → Geräte & Dienste → Helfer → Helfer erstellen → Schalter (`input_boolean`)*, z. B. „B2500 SoC-Sperre“, und im Blueprint unter *Akku-Schutz* hinterlegen. Den Helfer nicht von Hand schalten.

Ohne Merker gilt das bisherige Verhalten: Jede ausgeschaltete Ausgabe bleibt unterhalb der Wiedereinschaltschwelle aus. Schaltet der Speicher abends bei 20 % wegen kurz geringem Bedarf ab, bleibt er dann bis zum nächsten Tag aus. Die [PV-Durchleitung](#pv-durchleitung-trotz-soc-sperre-optional) setzt den Merker ebenfalls voraus.

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

Die fünf Pflicht-Entitäten auswählen. Als Netzleistungssensor den geglätteten Helfer angeben, nicht den rohen Sensor. Der Schalter der Überschusseinspeisung und der Merker für die SoC-Sperre werden unter *Akku-Schutz* hinterlegt.

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
| Merker SoC-Sperre | leer | Optional, empfohlen; siehe [Merker für die SoC-Sperre](#merker-für-die-soc-sperre-empfohlen) |
| PV-Durchleitung trotz SoC-Sperre | aus | Optional, braucht den Merker; siehe [PV-Durchleitung](#pv-durchleitung-trotz-soc-sperre-optional) |
| Reserve der PV-Durchleitung | `50 W` | Ladeleistung, die während der Durchleitung mindestens im Akku ankommt |
| Balancing-Fenster | an | Bei 100 % SoC tagsüber die Überschusseinspeisung einschalten, nach Sonnenuntergang aus |
| Schalter Überschusseinspeisung | leer | Optional, dringend empfohlen; leer = Überschusseinspeisung muss dauerhaft **aus** bleiben |
| Nachlaufzeit der Überschusseinspeisung | `10 min` | Wartezeit bei ruhender Ladeleistung vor dem Abschalten |

### Leistungsgrenzen

| Option | Standard | Bedeutung |
| --- | --- | --- |
| Maximale Ausgabeleistung | `800 W` | Obergrenze der geregelten Ausgabe (während der Überschusseinspeisung regelt der Speicher selbst) |
| Gedrosselte Ausgabeleistung | `200 W` | Fester Wert bei Drosselung |
| Hardware-Mindestleistung | `80 W` | Darunter wird abgeschaltet, auch bei der PV-Durchleitung |
| Hysterese der Drosselung | `100 W` | Zusätzliche Ladeleistung, um die Drosselung zu verlassen oder die PV-Durchleitung einzuschalten |

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
| 0 | Netzwert ungültig oder zu alt (Watchdog) | 0 W, Switch aus |
| 1 | Überschusseinspeisung ist eingeschaltet | 0 W, Zeitplan aus (Balancing) |
| 2 | SoC ≤ Abschaltschwelle | 0 W, Switch aus |
| 2b | SoC < Wiedereinschaltschwelle **und** SoC-Sperre aktiv | 0 W; mit [PV-Durchleitung](#pv-durchleitung-trotz-soc-sperre-optional) höchstens Ladeleistung − Reserve |
| 3 | Sollwert < Hardware-Minimum (inkl. Einspeisung) | 0 W, Switch aus |
| 4 | SoC < Drosselschwelle **und** Ladeleistung trägt die Ausgabe nicht | Drosselwert |
| 5 | alles andere | Sollwert, gedeckelt auf das Maximum |

Die SoC-Sperre ist mit Merker nur nach einer Abschaltung wegen Tiefentladung aktiv, ohne Merker immer dann, wenn die Ausgabe aus ist.

Ein Bedarf über der maximalen Ausgabeleistung führt nicht zur Drosselung, die Ausgabe bleibt einfach am Maximum. Bei voller Ausgabe liegt der Sollwert bei jedem Netzbezug über dem Maximum. Eine Drosselung darauf würde die Ausgabe bei Dauerlasten knapp über dem Maximum ständig zwischen Drosselwert und Vollwert springen lassen.

Danach greifen Totband, Wartezeit und Anlaufsperre. Der Switch wird nur bei echtem Zustandswechsel geschaltet, und der Watt-Wert wird **vor** dem Einschalten gesetzt, damit der Speicher beim Anlaufen nie kurz einen veralteten Wert ausgibt. Nach dem Schreiben wartet die Automation bis zu 15 Sekunden, bis die Number-Entity den neuen Wert zurückmeldet. Trigger in dieser Zeit werden verworfen, damit kein Durchlauf mit dem alten Wert als Basis rechnet.

### Anlaufsperre

Nach dem Einschalten braucht der Speicher ein bis zwei Minuten, bis er tatsächlich Leistung abgibt. In dieser Zeit zeigt der Netzsensor weiter den vollen Bezug. Ohne Sperre würde die Regelung das als „noch zu wenig“ deuten und den Sollwert Schritt für Schritt erhöhen. Sobald der Speicher dann liefert, gibt er viel zu viel ab und speist ein.

Deshalb ändert die Automation den Sollwert nach dem Einschalten für die eingestellte Zeit nicht (Standard 120 s, gemessen ab dem Einschalten des Ausgabe-Schalters). Abschalten auf 0 W ist davon ausgenommen und erfolgt immer sofort. Das gilt auch für die PV-Durchleitung: Fällt die PV in dieser Zeit ab, wird erst danach nachgeregelt.

Speist der Speicher nach dem Anlaufen trotzdem kurz ein, ist die Sperre zu kurz (siehe [Feintuning](#feintuning-in-der-praxis)).

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
| Nach Sonnenuntergang, Ladeleistung seit der Nachlaufzeit unter 10 W | Überschusseinspeisung aus |
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

### PV-Durchleitung trotz SoC-Sperre (optional)

Nach einer Abschaltung wegen Tiefentladung bleibt die Ausgabe bis zur Wiedereinschaltschwelle gesperrt. Morgens heißt das: Die PV lädt den Akku mit mehreren hundert Watt, und das Haus bezieht trotzdem aus dem Netz. Die Durchleitung gibt in dieser Phase einen Teil der Ladeleistung direkt ans Haus weiter. Der Akku lädt dabei immer mindestens mit der eingestellten Reserve weiter.

- **Einschalten:** wenn die Ladeleistung den aktuellen Wunschwert um Reserve + Hysterese der Drosselung übersteigt (mit Standardwerten 150 W)
- **Während sie läuft:** Die Ausgabe folgt dem Hausbedarf, höchstens aber Ladeleistung − Reserve
- **Ausschalten:** wenn diese Grenze unter die Hardware-Mindestleistung fällt
- **Die Sperre bleibt bestehen:** Der Merker wird weiterhin erst ab der Wiedereinschaltschwelle gelöst. Unterhalb der Abschaltschwelle bleibt die Ausgabe immer aus, auch mit Durchleitung.

Beispiel bei 20 % SoC (gesperrt), Hausbedarf 300 W, Reserve 50 W:

| Ladeleistung | Ausgabe | Akku netto |
| --- | --- | --- |
| 700 W | ≈ 300 W (Hausbedarf) | +400 W |
| 350 W | ≈ 300 W | ≈ +50 W |
| 250 W | 200 W (gedeckelt) | +50 W |
| 120 W | 0 W (Grenze 70 W < Hardware-Minimum) | +120 W |

Voraussetzung ist der [Merker für die SoC-Sperre](#merker-für-die-soc-sperre-empfohlen). Ohne Merker würde die eingeschaltete Ausgabe die Sperre aufheben, deshalb hat die Option dann keine Wirkung.

> **Zu prüfen:** Die Durchleitung nimmt an, dass `total_input_power` auch bei aktiver Ausgabe die PV-Eingangsleistung meldet und nicht die Netto-Ladeleistung (PV minus Ausgabe). Andernfalls würde sich die Durchleitung selbst wieder abschalten. Das lässt sich in den Traces prüfen: Bei laufender Durchleitung sollte die Ladeleistung ungefähr der PV-Leistung entsprechen.

---

## Feintuning in der Praxis

**Die Ausgabe schwingt, der Netzbezug pendelt in beide Richtungen**
Kp verringern, zuerst auf `0.4`. Hilft das nicht, die Wartezeit beim Erhöhen auf 30 s anheben.

**Nach dem Einschalten springt die Ausgabe auf Maximum und speist kurz ein**
Die Anlaufsperre ist zu kurz. Beobachten, wie lange der Speicher nach dem Einschalten tatsächlich braucht, und den Wert entsprechend erhöhen.

**Es bleibt ein kleiner konstanter Netzbezug oder eine kleine Einspeisung stehen**
Das kommt vom Totband. Weil jeder Schritt auf dem zuletzt gesetzten Sollwert aufbaut, wirkt die Regelung integrierend und baut die Abweichung bis auf einen Rest ab. Übrig bleibt nur, was unter dem Totband liegt: bis zu ±Totband/Kp am Netzanschluss, mit Standardwerten rund ±25 W. Je nach Richtung der letzten Änderung ist das Bezug oder Einspeisung. Mit 10 W Totband sind es rund ±17 W, dafür werden häufiger Sollwerte geschrieben. Bleibt deutlich mehr stehen, liegt es nicht am Totband. Dann in den Traces prüfen, ob Wartezeit, Anlaufsperre, Drosselung oder die maximale Ausgabeleistung greifen.

**Die Ausgabe springt zwischen Drosselwert und Vollwert**
Passiert bei schwankender PV nahe der Umschaltschwelle. Hysterese der Drosselung auf 150–200 W erhöhen.

**Der Speicher gibt gar nichts mehr ab**
Zuerst prüfen, ob Überschusseinspeisung und Zeitplan gleichzeitig eingeschaltet sind – das verträgt die Firmware nicht. Ist der Schalter im Blueprint hinterlegt, kann das nicht passieren.

**Der Speicher gibt bei 100 % SoC nichts ab, obwohl er voll ist**
Bei eingeschalteter Überschusseinspeisung hängt die Ausgabe an der Eingangsleistung. Ohne PV bleibt sie damit bei null. Den Schalter im Blueprint hinterlegen, dann schaltet die Automation abends um.

**Die Überschusseinspeisung wird nie eingeschaltet**
Die Automation schaltet sie erst bei 100 % Ladezustand ein. Erreicht der Speicher die 100 % nie, passiert auch nichts. In dem Fall die Entladetiefe vorübergehend begrenzen, damit er mittags wirklich voll wird.

**Die Automation regelt gar nicht mehr**
Die Regelung liest ihre eigene Basis aus der Number-Entity und misst die Wartezeit über deren `last_changed`. Meldet die Integration diese Entity nicht zuverlässig zurück, steht die Regelung, und nach jedem Schreiben läuft die 15-Sekunden-Wartezeit auf die Rückmeldung voll ab. Zum Test die Wartezeiten auf `0` setzen – regelt es dann wieder, liegt es daran.

**Die Automation läuft, tut aber nichts**
In den Traces prüfen, an welcher Bedingung sie abbricht. Häufigste Ursachen: Totband nicht überschritten, Wartezeit oder Anlaufsperre noch nicht abgelaufen, oder ein Sensor liefert `unavailable`.

---

## Grenzen

Perfekte Nulleinspeisung ist mit diesem Aufbau nicht erreichbar. Aus Sensortakt, Glättung und Reaktionszeit des Speichers ergibt sich eine Gesamtverzögerung von typischerweise 15–20 Sekunden, bis eine Laständerung vollständig ausgeregelt ist. Bei jedem Ein- und Ausschalten größerer Verbraucher entsteht in dieser Zeit unvermeidlich Netzbezug oder Einspeisung. Das liegt an der Kette aus Messung und Hardware, nicht an der Regelung.

---

## Changelog

### v2.3

- **Keine Drosselung mehr bei Bedarf über dem Maximum.** Bisher sprang die Ausgabe bei Dauerlasten knapp über der maximalen Ausgabeleistung (mit Standardwerten ca. 820–1200 W) ständig zwischen Drosselwert und Vollwert. Bei noch höherer Last blieb sie trotz vollem Akku auf dem Drosselwert. Jetzt bleibt sie am Maximum; gedrosselt wird nur noch bei niedrigem Akkustand
- Neuer optionaler **Merker für die SoC-Sperre** (`input_boolean`): Die Wiedereinschaltschwelle gilt damit nur noch nach einer Abschaltung wegen Tiefentladung, nicht nach jeder Abschaltung
- Nachlaufzeit der Überschusseinspeisung: Schwankungen der Ladeleistung unter 10 W setzen sie nicht mehr zurück
- Watchdog prüft `last_reported` statt `last_updated`: Ein gleichbleibender, aber weiterhin gemeldeter Netzwert löst keine Abschaltung mehr aus
- Nach dem Schreiben des Sollwerts wartet die Automation bis zu 15 s auf die Rückmeldung der Number-Entity (`mode: single` statt `restart`)

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
