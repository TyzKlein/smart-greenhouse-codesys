# 🍅 Automatisiertes IoT-Hydroponik System (Smart Greenhouse)

Projektbericht und SPS-Steuerungslogik für ein automatisiertes Hydroponik-Gewächshaus zur Kultivierung von Cherry-Tomaten.

---

## 1. Systemübersicht & Hardware
* **SPS (PLC):** Turck Compact-PLC `TBEN-L5-PLC-10` (IP67)
* **Entwicklungsumgebung:** CODESYS V3 (Programmiersprache: Structured Text / ST)
* **Feldbus & Kommunikation:** IO-Link (über Turck Master), Modbus-RTU, Ethernet
* **Sensorik & Aktorik:**
  * Modbus-Klimasensor (Temperatur & Feuchtigkeit)
  * Induktiver Näherungsschalter `Turck Ni50U` (Füllstandskontrolle)
  * Radar-Distanzsensor (Wachstumskontrolle / Fruchterkennung)
  * IO-Link LED-Signalsäule / RGB-Meldeleuchte

---

## 2. Steuerungslogik & Ablaufkonzept (Ablaufsteuerung)

### Logik und Grenzwerte:
1. **Klimaregelung:**
   * Temperaturprüfung: Sollwertbereich $20{,}0\,^\circ\text{C} \le T \le 35{,}5\,^\circ\text{C}$
   * Feuchtigkeitsprüfung: Sollwertbereich $20{,}0\,^\circ\text{C} \le T \le 60{,}0\,^\circ\text{C}$
2. **Nährstofffüllstand:**
   * Auswertung des induktiven Sensors (`SensorRawValue_Dist`). Bei Abfall des Nährstoffflusses wird Alarm ausgelöst.
3. **Wachstum / Fruchtreife (Radar):**
   * Erkennt der Radarsensor einen Abstand im Bereich `10000 bis 15000` (entspricht $1{,}0\text{ m} - 1{,}5\text{ m}$), wird Fruchtreife/Beschnitt gemeldet (Rote Warnlampe).
4. **Fehler- und Alarmpriorisierung (LED-Farbcodierung):**
   * Die RGB-Signalleuchte visualisiert kombinierte Zustände (z. B. Farbcode für Feuchtefehler, Temperaturfehler oder Trockenlaufschutz).
   * Manueller Taster (`Digi_Button`): Flankenauswertung zum Ein- und Ausschalten des Meldesignals.

---

## 3. SPS-Quellcode (CODESYS Structured Text)

Die vollständige Datei ist im Repository hinterlegt: [`Hydroponik System.txt`](Hydroponik%20System.txt)

```pascal
PROGRAM PLC_PRG
VAR
   SensorRawValue : WORD;            // Rohwert vom Sensorregister (Temperatur)
   SensorRawValue_Humi : WORD;       // Rohwert Luftfeuchtigkeit
   SensorRawValue_Dist : BOOL;       // Induktionssensor: ja/nein
   IsRedDetected : DINT;             // Radarsensor / Objektabstand

   Temp_Celsius : REAL;              // Eingangswert vom Sensor
   Humidity : REAL;                  // Luftfeuchtigkeit in %
   Anzeig_Farbe : USINT;
   LED : USINT;     
   Temp_Okay : BOOL := FALSE;
   Humidity_Okay : BOOL;
	
   // Farbe der Anzeigelampe anpassen
   Temp_Lampe : BOOL;
   Humidity_Lampe : BOOL;
   Wasser_Lampe : BOOL;
   Tomaten_Lampe : BOOL;
   Alles_Lampe : BOOL;
	
   // Button zum Ein/Ausschalten des Dauerlichts
   Digi_Button   : BOOL;             // externer Schließer
   LED_Enabled   : BOOL := TRUE;
   Digi_Button_Vorher : BOOL := FALSE;
END_VAR

// Temperatur berechnen: Registerwert ÷ 20
Temp_Celsius := WORD_TO_REAL(SensorRawValue) / 20.0;
// Feuchtigkeit berechnen: Registerwert ÷ 100
Humidity := WORD_TO_REAL(SensorRawValue_Humi) / 100.0;

// Temperatur & Feuchtigkeit prüfen
Temp_Okay := (Temp_Celsius >= 20.0) AND (Temp_Celsius <= 35.5);
Humidity_Okay := (Humidity >= 20.0) AND (Humidity <= 60.0);

// Flankenauswertung für Taster
IF Digi_Button AND NOT Digi_Button_Vorher THEN
   LED_Enabled := NOT LED_Enabled;
END_IF;
Digi_Button_Vorher := Digi_Button;

// Lampen zurücksetzen
Temp_Lampe := FALSE;
Humidity_Lampe := FALSE;
Wasser_Lampe := FALSE;
Tomaten_Lampe := FALSE;
Alles_Lampe := FALSE;
LED := 0; 

IF  LED_Enabled THEN	
	IF (NOT Humidity_Okay) AND SensorRawValue_Dist AND (NOT Temp_Okay)  THEN
		LED := 1;        // Blinken einschalten
		Anzeig_Farbe := 13; // Grundfarbe (z.B. "braun")
		Wasser_Lampe := TRUE; // Visuelle Lampe leuchtet
		Humidity_Lampe := TRUE; 
		Temp_Lampe := TRUE;
		Alles_Lampe := TRUE;
		
	ELSIF (NOT Humidity_Okay) AND SensorRawValue_Dist  THEN
		LED := 1;        // Blinken einschalten
		Anzeig_Farbe := 11; // Grundfarbe (z.B. "braun")
		Wasser_Lampe := TRUE; // Visuelle Lampe leuchtet
		Humidity_Lampe := TRUE;
		Alles_Lampe := TRUE;
	
		
	ELSIF (NOT Temp_Okay) AND SensorRawValue_Dist  THEN
		LED := 1;        // Blinken einschalten
		Anzeig_Farbe := 10; // Grundfarbe (z.B. "braun")
		Wasser_Lampe := TRUE; // Visuelle Lampe leuchtet
		Temp_Lampe := TRUE;
		Alles_Lampe := TRUE;
		
	ELSIF SensorRawValue_Dist THEN
		LED := 1;        // Blinken einschalten
		Anzeig_Farbe := 0; // Grundfarbe (z.B. "braun")
		Wasser_Lampe := TRUE; // Visuelle Lampe leuchtet
		Alles_Lampe := TRUE;
	
		
	ELSIF (NOT Temp_Okay) AND (NOT Humidity_Okay) THEN
		LED := 1;
		Anzeig_Farbe := 12;
		Temp_Lampe := TRUE;
		Humidity_Lampe := TRUE;
		Alles_Lampe := TRUE;
		
	ELSIF (NOT Humidity_Okay) THEN
		LED := 1;
		Anzeig_Farbe := 9; // Blau
		Humidity_Lampe := TRUE; // Visuelle Lampe leuchtet
		Alles_Lampe := TRUE;
	ELSIF (NOT Temp_Okay) THEN
		LED := 1;
		Anzeig_Farbe := 4; // Gelb
		Temp_Lampe := TRUE; // Visuelle Lampe leuchtet
		Alles_Lampe := TRUE; // Visuelle Lampe leuchtet
		
	ELSIF (IsRedDetected >= 10000) AND (IsRedDetected <= 15000) THEN 
			LED := 2;
			Anzeig_Farbe := 1; // Rot
			Tomaten_Lampe := TRUE;// Visuelle Lampe leuchtet
			Alles_Lampe := TRUE;
		
	ELSE
		//kein Alarm
		LED := 0;        // sonst LED aus
	END_IF;
	
ELSE
  // Sobald Button gedrückt: LED aus
  LED := 0;

END_IF
