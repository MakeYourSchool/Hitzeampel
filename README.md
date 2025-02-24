# Hitzeampel
----
Im Folgenden findet ihr den Beispiel-Code für die Make your School Hitzeampel. Die Hitzeampel hilft dabei, die Temperatur in einem Raum sichtbar zu machen, indem sie je nach Temperatur in verschiedenen Farben leuchtet und ihre Form verändern kann. Die Hitzeampel ist ideal für den Einsatz in einem Klassenzimmer oder auch für zu Hause. Sie kann auch als dekoratives Licht genutzt werden.

Viel Spaß beim Nachbauen!

## Aufbau

<img src=https://github.com/MakeYourSchool/Hitzeampel/blob/main/Abbildungen/Hitzeampel.jpg width=1000px>

Bildquelle: *Wissenschaft im Dialog gGmbH*

## Code

```
//Bibliotheken einbinden
#include "Adafruit_NeoPixel.h"                          //"Adafruit NeoPixel" für die LED-Bar
#include <Wire.h>                                       
#include "rgb_lcd.h"                                    //"Grove - LCD RGB Backlight"
#include "Grove_Temperature_And_Humidity_Sensor.h"      //"Grove Temperature And Humidity Sensor"
#include <Servo.h>

//Vorbereitungen für LED-Bar
#define PIN 4
#define NUMPIXELS 10
Adafruit_NeoPixel pixels = Adafruit_NeoPixel(NUMPIXELS, PIN, NEO_GRB + NEO_KHZ800);

//Vorbereitungen für den Temperatur-Sensor 
#define DHTTYPE DHT22                                   // wird ein anderer Sensor eingesetzt, muss DHT22 durch den jeweiligen Namen ersetzt werden
#define DHTPIN 2                                        // bei einem DHT10 oder DHT20 entfällt diese Zeile
DHT dht(DHTPIN, DHTTYPE);                               // und der erste Parameter "DHTPIN" in der Klammer

//Vorbereitungen für das LCD-Display
rgb_lcd lcd;

//Vorbereitungen für den Servomotor
Servo ServoMotor;


//Globale Variablen deklarieren
int colorR, colorG, colorB  = 0;                        //Variablen für die Lichtfarben | R:Rot, G:Gruen, B:Blau

int MotorPosition = 0;                                  //Variable für die Motor-Position
int MinMotorPos = 40;                                   //Hier können die Grenzwerte des Motor-Bewegungsraums definiert werden
int MaxMotorPos = 130;

int Buzzer = 5;                                         //Pin-Anschluss vom passiven Buzzer

int Taster1Pin = 7;                                     //Pins der Taster
int Taster2Pin = 8;
bool Taster1, Taster2, Eingabe;                         //Variablen zum Erkennen, ob die Taster gedrückt wurden

float WunschTemp = 22;                                  //Variable der anpassbaren Wunsch-Temperatur
int MotorTempBereich = 10;                              //Temperatur-Bereich in dem der Motor seine maximale Bewegung durchführt
float aktTemp, aktHum = 0;                              //Variablen zum Speichern der aktuellen Temperatur und Luftfeuchtigkeit
int TempVergleich = 0;                                  // -1 zu kalt, 0 genau richtig, 1 zu warm

unsigned long Zeitstempel = 0;                          //Zeitstempel, der genutzt wird, um alle Aktionen zu timen
int AktionsIntervall = 2000;                            //in ms, die Dauer, bis eine erneute Aktion (z.B. Tonwiedergabe, Motorpositionierung, Temperaturmessung) durchgeführt wird


//******************************************************
//                  void setup()
//******************************************************

void setup() {
  pixels.setBrightness(255);
  pixels.begin();

  lcd.begin(16, 2);

  Wire.begin();
  Serial.begin(9600);

  dht.begin();

  pinMode(Buzzer, OUTPUT);
  pinMode(Taster1Pin, INPUT);
  pinMode(Taster2Pin, INPUT);

  ServoMotor.attach(3);
}


//******************************************************
//                  void loop()
//******************************************************

void loop() {

  check_Taster();                                       
  if (Eingabe) {                                        //wurde ein Taster gedrückt, wird die Wunsch-Temperatur entsprechend angepasst in 0.5er Schritten
    WunschTemp = WunschTemp + 0.5 * (Taster1 - Taster2);
    Zeitstempel = millis();
    lcd.clear();                                        //Am Display wird die Wunsch-Temperatur-Änderung angezeigt
    lcd.print(" * WunschTemp * ");
    lcd.setCursor(5, 1);
    lcd.print(WunschTemp);
  }

  if (millis() - Zeitstempel >= AktionsIntervall) {     //der folgende Abschnitt wird immer exakt nach der Zeit "Aktionsintervall" ausgeführt
    Zeitstempel = millis();
    check_Sensor();

    lcd.setCursor(0, 0);                                //Allgemeine Temperatur-Anzeige
    lcd.print(" | Temperatur | ");
    lcd.setCursor(0, 1);
    lcd.print(" |            | ");
    lcd.setCursor(5, 1);
    lcd.print(aktTemp);
    lcd.print(" C");

    check_MotorPosition();
    ServoMotor.write(MotorPosition);

    check_TempDifferenz();

    switch (TempVergleich) {                            //je nachdem ob die Temperatur höher oder niedriger als die Wunsch-Temperatur ist, werden unterschiedliche "case" durchgeführt
      case -1:                                          //die aktuelle Temperatur ist zu kalt
        colorR = 0;
        colorG = 0;
        colorB = 255;
        BuzzerAktion(1);
        break;

      case 0:                                           //die aktuelle Temperatur ist genau richtig
        colorR = 0;
        colorG = 255;
        colorB = 0;
        BuzzerAktion(0);
        break;

      case 1:                                           //die aktuelle Temperatur ist zu warm
        colorR = 255;
        colorG = 0;
        colorB = 0;
        BuzzerAktion(2);
        break;
    }

    lcd.setRGB(colorR / 5, colorG / 5, colorB / 5);     //Display-Hintergrundfarbe anpassen, allerdings nur mit gedimmter Farbe
    LEDchange(colorR, colorG, colorB);                  //LED-Farben anpassen
  }
  delay(10);
}


//******************************************************
//                  weitere Funktionen
//******************************************************


void BuzzerAktion(int Melodie) {                        //Hier werden Buzzer-Melodien oder Sounds zusammengestellt. Diese können oben mit der jeweiligen Nummer abgespielt werden
  switch (Melodie) {
    case 0:
      noTone(Buzzer);
      break;

    case 1:                                             //kurzes Knacken, zeigt kalte Temperatur an
      tone(Buzzer, 2000);
      delay(10);
      noTone(Buzzer);
      delay(100);
      tone(Buzzer, 2000);
      delay(10);
      noTone(Buzzer);
      delay(500);
      tone(Buzzer, 2000);
      delay(10);
      noTone(Buzzer);
      delay(100);
      tone(Buzzer, 2000);
      delay(10);
      noTone(Buzzer);
      break;

    case 2:                                             //tiefes Brummen, zeigt warme, schwere Temperaturen an
      tone(Buzzer, 100);
      delay(100);
      noTone(Buzzer);
      delay(300);
      tone(Buzzer, 100);
      delay(100);
      noTone(Buzzer);
      delay(300);
      break;

    default:
      break;
  }
}

void check_MotorPosition() {                            //errechnet die nächste Position des Motors anhand der aktuellen Temperatur und der Differenz zur Wunsch-Temperatur
  int MinTempbereich = WunschTemp - MotorTempBereich / 2;
  int MaxTempBereich = WunschTemp + MotorTempBereich / 2;
  MotorPosition = map(aktTemp, MinTempbereich, MaxTempBereich, MinMotorPos, MaxMotorPos);
  if (MotorPosition < MinMotorPos) {
    MotorPosition = MinMotorPos;
  } else if (MotorPosition > MaxMotorPos) {
    MotorPosition = MaxMotorPos;
  }
}

void check_Sensor() {                                   //liest den Temperatur- und Luftfeuchtigkeitssensor aus und speichert die Variablen ab
  float temp_hum_val[2] = { 0 };
  if (!dht.readTempAndHumidity(temp_hum_val)) {
    aktHum = temp_hum_val[0];
    aktTemp = temp_hum_val[1];
  } else {
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("sensor-error");
    delay(5000);
  }
}

void LEDchange(int r, int g, int b) {                   //ein Makro, um immer alle LEDs gleichzeitig mit einer aktuellen Farbe umzuschalten
  for (int i = 0; i < NUMPIXELS; i++) {
    pixels.setPixelColor(i, r, g, b);
  }
  pixels.show();
}

void check_Taster() {                                   //überprüft, ob ein Taster gedrückt wird
  Taster1 = digitalRead(Taster1Pin);
  Taster2 = digitalRead(Taster2Pin);
  if (Taster1 || Taster2) {
    Eingabe = 1;
    delay(300);
  } else {
    Eingabe = 0;
  }
}

void check_TempDifferenz() {                            //überprüft ob die aktuelle Temperatur über oder unter der Wunsch-Temperatur liegt
  if (aktTemp > WunschTemp + 1) {
    TempVergleich = 1;
  } else if (aktTemp < WunschTemp - 1) {
    TempVergleich = -1;
  } else {
    TempVergleich = 0;
  }
}

```

