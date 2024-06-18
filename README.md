Video:
[Youtube](https://www.youtube.com/watch?v=-DAD3s97CBk)


Code:
``` c++
#include <SPI.h>
#include <MFRC522.h>

// Definition der Pins
#define SS_PIN 10
#define RST_PIN 9
MFRC522 mfrc522(SS_PIN, RST_PIN);   // Erstellen einer MFRC522-Instanz

// Definition der Pin-Nummern für LEDs, Lautsprecher, Ultraschallsensor und Button
int LEDblau = 3; // Blaue LED an Pin 3
int LEDrot = 6; // Rote LED an Pin 6
int LEDgruen = 5; // Grüne LED an Pin 5
int speaker = 8; // Lautsprecher an Pin 8
int trigPin = 4; // Trig-Pin des Ultraschallsensors
int echoPin = 2; // Echo-Pin des Ultraschallsensors
int buttonPin = 7; // Button an Pin 7
int p = 1000; // Pause von 1000ms (1 Sekunde)
int brightness = 255; // Leuchtstärke der LEDs (Wert zwischen 0 und 255)
int dunkel = 0; // Wert 0 bedeutet LED aus

// Definition von Variablen zur Steuerung der Sensoren und LED-Status
bool objectDetected = false;
unsigned long lastSoundTime = 0;
unsigned long lastActivityTime = 0;
unsigned long rfidInactiveStart = 0;
bool ultrasonicEnabled = false;
bool rfidEnabled = false;
bool rfidInactiveCountdown = false;
int buttonState = 0;
int lastButtonState = 0;
int buttonPressCount = 0;

void setup() 
{
  Serial.begin(9600);   // Serielle Kommunikation starten
  SPI.begin();          // SPI-Bus initialisieren
  mfrc522.PCD_Init();   // MFRC522 initialisieren
  Serial.println();

  // Pins als Ausgang definieren
  pinMode(LEDblau, OUTPUT);
  pinMode(LEDgruen, OUTPUT);
  pinMode(LEDrot, OUTPUT);
  pinMode(trigPin, OUTPUT);
  pinMode(echoPin, INPUT);
  pinMode(buttonPin, INPUT_PULLUP); // Pullup-Widerstand für den Button aktivieren

  // LEDs ausschalten
  analogWrite(LEDblau, dunkel); 
  analogWrite(LEDgruen, dunkel);
  analogWrite(LEDrot, dunkel); 
}

void loop() 
{
  buttonState = digitalRead(buttonPin); // Button-Zustand lesen
  
  // Prüfen, ob der Button-Zustand sich geändert hat
  if (buttonState != lastButtonState) 
  {
    // Wenn der Button gedrückt wurde
    if (buttonState == LOW) 
    {
      buttonPressCount++; // Button-Drück-Zähler erhöhen
      // Je nach Anzahl der Drücker verschiedene Sensoren aktivieren/deaktivieren
      if (buttonPressCount == 1) 
      {
        ultrasonicEnabled = true;
        Serial.println("Ultraschallsensor aktiviert");
      } 
      else if (buttonPressCount == 2) 
      {
        rfidEnabled = true;
        lastActivityTime = millis(); // Aktivitäts-Timer zurücksetzen
        Serial.println("RFID Leser aktiviert");
      } 
      else if (buttonPressCount == 3) 
      {
        ultrasonicEnabled = false;
        rfidEnabled = false;
        buttonPressCount = 0;
        rfidInactiveCountdown = false;
        Serial.println("Alle Sensoren deaktiviert");
      }
    }
    delay(50); // Entprellzeit für den Button
  }
  lastButtonState = buttonState; // Button-Zustand speichern

  // RFID-Scanner nach 20 Sekunden Inaktivität deaktivieren
  if (rfidEnabled && millis() - lastActivityTime > 20000) 
  {
    rfidEnabled = false;
    Serial.println("RFID Leser deaktiviert aufgrund von Inaktivität");
    rfidInactiveStart = millis();
    rfidInactiveCountdown = true;
    buttonPressCount = 1; // Button-Drück-Zähler zurücksetzen, um Reaktivierung zu ermöglichen
  }

  // Alle Sensoren nach 20 Sekunden Inaktivität deaktivieren
  if (rfidInactiveCountdown && millis() - rfidInactiveStart > 20000) 
  {
    if (ultrasonicEnabled) 
    {
      ultrasonicEnabled = false;
      Serial.println("Ultraschallsensor deaktiviert nach 20s Inaktivität");
    }
    rfidInactiveCountdown = false;
    buttonPressCount = 0; // Button-Drück-Zähler zurücksetzen, um Reaktivierung zu ermöglichen
  }

  // Wenn der Ultraschallsensor aktiviert ist und ein Objekt in der Nähe erkannt wird
  if (ultrasonicEnabled && isObjectInProximity()) 
  {
    if (!objectDetected) 
    {
      objectDetected = true;
      unsigned long currentMillis = millis();
      if (currentMillis - lastSoundTime > 2000) 
      {
        playOpenSound(); // Öffnungssound abspielen
        lastSoundTime = currentMillis;
        Serial.println("Karte jetzt dran halten...");
      }
    }
    
    // Blaue LED einschalten
    analogWrite(LEDrot, dunkel);
    analogWrite(LEDgruen, dunkel);
    analogWrite(LEDblau, brightness); // blau einschalten

    if (rfidEnabled) 
    {
      // Nach neuen Karten suchen
      if (!mfrc522.PICC_IsNewCardPresent()) 
      {
        return;
      }
      // Eine der Karten auswählen
      if (!mfrc522.PICC_ReadCardSerial()) 
      {
        return;
      }
      lastActivityTime = millis(); // Aktivitäts-Timer zurücksetzen, wenn eine Karte gelesen wird
      // UID der Karte auf dem seriellen Monitor anzeigen
      Serial.print("Karte (UID):");
      String content = "";
      byte letter;
      for (byte i = 0; i < mfrc522.uid.size; i++) 
      {
        Serial.print(mfrc522.uid.uidByte[i] < 0x10 ? " 0" : " ");
        Serial.print(mfrc522.uid.uidByte[i], HEX);
        content.concat(String(mfrc522.uid.uidByte[i] < 0x10 ? " 0" : " "));
        content.concat(String(mfrc522.uid.uidByte[i], HEX));
      }
      Serial.println();
      Serial.print("Nachricht : ");
      content.toUpperCase();
      // Prüfen, ob die UID der Karte berechtigt ist
      if (content.substring(1) == "FE D3 31 17") // Hier die UID der Karten ändern, die Zugriff haben sollen
      {
        Serial.println("Zugriff gewährt!");
        Serial.println();
        
        analogWrite(LEDgruen, brightness); // grüne LED einschalten
        analogWrite(LEDblau, dunkel);
        analogWrite(LEDrot, dunkel);
        playStarWars(); // Star Wars Sound abspielen
        delay(p); // Pause
        analogWrite(LEDgruen, dunkel); // grüne LED ausschalten
      }
      else 
      {
        Serial.println("Zugriff verweigert!");

        analogWrite(LEDrot, brightness); // rote LED einschalten
        analogWrite(LEDgruen, dunkel);
        analogWrite(LEDblau, dunkel);
        playErrorTone(); // Error-Ton abspielen
        delay(p); // Pause
        analogWrite(LEDrot, dunkel); // rote LED ausschalten
      }
    }
  } 
  else 
  {
    objectDetected = false;
    analogWrite(LEDrot, dunkel); 
    analogWrite(LEDgruen, dunkel);
    analogWrite(LEDblau, dunkel);
  }
}

// Funktion zur Erkennung eines Objekts in der Nähe mittels Ultraschallsensor
bool isObjectInProximity() 
{
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);

  long duration = pulseIn(echoPin, HIGH); // Dauer des Echosignals messen
  float distance = (duration * 0.0343) / 2; // Abstand berechnen (Schallgeschwindigkeit: 0.0343 cm/us)
  
  if (distance <= 5.0) // Wenn der Abstand kleiner oder gleich 5 cm ist
  {
    return true;
  } 
  else 
  {
    return false;
  }
}

// Funktion zum Abspielen eines Fehler-Tons
void playErrorTone() 
{
  tone(speaker, 100, 500); // 100Hz Ton für 500ms abspielen
  delay(500);
  tone(speaker, 200, 500); // 200Hz Ton für 500ms abspielen
  delay(500);
  noTone(speaker);
}

// Funktion zum Abspielen eines Öffnungssounds
void playOpenSound() 
{
  // Eine einfache angenehme Melodie
  int melody[] = { 262, 294, 330 };
  int noteDurations[] = { 150, 150, 150 };

  for (int thisNote = 0; thisNote < 3; thisNote++) 
  {
    tone(speaker, melody[thisNote], noteDurations[thisNote] * 0.8);
    delay(noteDurations[thisNote] * 1.30);
    noTone(speaker);
  }
}

// Funktion zum Abspielen der Star Wars Melodie (vereinfacht)
void playStarWars() 
{
  int melody[] = {
    440, 440, 440, 349, 523, 440, 349, 523, 440
  };
  int noteDurations[] = {
    500, 500, 500, 350, 150, 500, 350, 150, 650
  };
  for (int thisNote = 0; thisNote < 9; thisNote++) 
  {
    int noteDuration = noteDurations[thisNote];
    if (melody[thisNote] != 0) {
      tone(speaker, melody[thisNote], noteDuration);
    }
    int pauseBetweenNotes = noteDuration * 1.30;
    delay(pauseBetweenNotes);
    noTone(speaker);
  }
}
```
