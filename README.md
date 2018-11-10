# LCD-Smartie 

## zeigt informationen z.B. : 
- Uhrzeit / Datum
- Aktuellen Liedtitel, Dauer uvm. (Winamp)
- CPU Auslastung
- GPU Auslastung
- RAM Verbrauch
- Temperaturen (HWINFO)
- RSS Feeds
- Festplatten
- derzeitiger Upload und Download (Netzwerk)
- Gesammte Up/Downloadmenge (seit letzen neustart)
- PC uptime

Zeigt diese Werte des Computers (Bei mir Windows 10 mit Ardunio nano) auf einen kleinen LCD Bildschirm an

## kann mit den Tastern steuern:

- Anzuzeigende Seite
- Nächstes Lied / Letztes Lied
- Lautstärke
- Theoretisch kann man jedes script oder Anwedung starten

## Hardware:

 - Arduino Nano
 - Pin 0,1 USB DATEN
 - Pin 2-12: Eingaenge // 10kOhm nach GND
 - Pin A0 : HintergrundLED - einstellbar
 - Pin A1 :
 - Pin A2 :
 - Pin A3 :
 - Pin A4 : SDA
 - Pin A5 : SCL
 - Pin A6 :
 - Pin A7 :

## Software:

 - LED Smartie 5.4
 - matrix.dll COM8,9600
 - Refresh intervall 50 ms
 - scroll intervall 600

## Optional:

 -  Bildschirmhelligkeit über 5kOhm Poti
 -  Externer reset Taster ohne Pulldownwiderstand
   
## Fehler:
 
 - beim Ausgeben des "þ" Zeichen
 
## Bilder: 

![Bild 1](https://github.com/ToWipf/LCD-Smartie/blob/master/Bild1.jpg)
***
![Bild 2](https://github.com/ToWipf/LCD-Smartie/blob/master/Bild2.jpg)
***
![Bild 3](https://github.com/ToWipf/LCD-Smartie/blob/master/Bild3.jpg)

Programm dazu: http://lcdsmartie.sourceforge.net/

Erweitert um 11 Taster und viele kleine Fehlerbehebungen
 

## Code: (als datei [hier](https://github.com/ToWipf/LCD-Smartie/blob/master/smartie.ino))
``` c
/*
    VERSION 5.5
   Wipf           12.06.2016
                  14.07.2016
                  07.11.2016
                  20.01.2017 V4 Laufzeitverbesserungeng
                  23.01.2017 int zu byte
                  27.01.2017 Tasterverhalten optimiert
                  09.02.2017 
               15/18.02.2017 V5 Funktionsaufteilung - Taster gehen nun nicht nur bei der Bildschirmaktualisierung
                  07.06.2017 Kleine Aenderungen - weniger Speicher noetig

   Hardware:
    Arduino Nano
      Pin 0,1 USB DATEN
      Pin 2-12: Eingaenge // 10kOhm nach GND
      Pin 13 LED leuchtet beim Tastendruck
      Pin A0 : HintergrundLED - einstellbar
      Pin A1 : Frei
      Pin A2 : Frei
      Pin A3 : Frei
      Pin A4 : SDA - LCD ueber i2c
      Pin A5 : SCL - LCD ueber i2c
      Pin A6 : Frei
      Pin A7 : Frei

   Software:
    LED Smartie 5.4
       matrix.dll COM8,9600
      Refresh intervall 50 ms
      Scroll intervall 600
      $MObutton(1)  -  $MObutton(:) und $MObutton(;) mit >= then XXXX

   Optional:
     Bildschirmhelligkeit über 5kOhm Poti
     Externer RST Taster ohne Pull down widerstand
*/
#include <LiquidCrystal_I2C.h>

#define VERSION "V 5.5"
#define TASTERANZAHL 11 //Anzahl der externen Taster
#define LED A0 // PIN mit der Led -Hintergrund beleuchtung + Start / Gelbe LED
#define PAUSE 50000 //Wartezeit zwischen den eingaben
unsigned int warten = 0;
byte i = 0;
byte pin;
byte IN;
byte incoming, rxbyte, col, row;

LiquidCrystal_I2C lcd(0x3F, 20, 4);

void setup()
{
  pinMode(LED, OUTPUT);
  digitalWrite(LED, HIGH);
  Serial.begin(9600);
  for (pin = 2; pin < TASTERANZAHL+2; pin++)
  {
//    pinMode(pin , INPUT_PULLUP);
    pinMode(pin , INPUT);   
  }
  pinMode(13, OUTPUT);
  lcd.begin();
  lcd.clear();
  lcd.setCursor(2, 0);
  lcd.print("Smartie by Wipf");
  lcd.setCursor(0, 3);
  lcd.print(TASTERANZAHL);
  lcd.print(" Tasten ");
  lcd.setCursor(15, 3);
  lcd.print(VERSION);
  digitalWrite(LED, LOW);
}

byte serial_getch()
{
  while (Serial.available() == 0)
  {
    taster();
  }
  incoming = Serial.read(); // read the incoming byte:
  return (byte) (incoming & 0xff);
}

void loop()
{
  //Auswertung
  {
    rxbyte = serial_getch();
    if (rxbyte == 254) //Matrix Orbital uses 254 prefix for commands
    {
      switch (serial_getch())
      {
        case 66: //backlight on (at previously set brightness)
          digitalWrite(LED, HIGH);
          break;
        case 70:
          digitalWrite(LED, LOW);
          break;
        case 71:  //set cursor position
          col = (serial_getch() - 1);  //get column byte
          row = (serial_getch());
          lcd.setCursor(col, row - 1);
          break;
        case 72:  //cursor home (reset display position)
          lcd.setCursor(0, 0);
          break;
        //      case 86: //BEENDEN Des Programmes
        //        lcd.clear();
        //        lcd.setCursor(0, 0);
        //        lcd.print("Smartie wurde Beendet");
        //        delay(5000);
        case 88:  //clear display, cursor home
          lcd.clear();
          lcd.setCursor(0, 0);
          break;

      }
      return;
    }

    switch (rxbyte)    //Zeichen anpassen
    {
      case 0x02:
        rxbyte = 0xC6; //  1/3  Block Altanativ: 0xA4
        break;
      case 0x03:
        rxbyte = 0xDB;  // 2/3 Block
        break;
      case 0x01:
        rxbyte = 0xFF;  // 3/3 block
        break;
      /////////////LEERZEICHEN das beim Beeenden nicht setsamrme zeichen kommen;
      case 0x04:
      case 0x05:
      case 0x06:
      case 0x07:
      case 0x08:
        //   rxbyte = NULL;
        rxbyte = 0x20;
        break;
      //////////Ende Leerzeichen


      case 0xE4: //ASCII "a" umlaut
        rxbyte = 0xE1;
        break;
      case 0xF1: //ASCII "n" tilde
        rxbyte = 0xEE;
        break;
      case 0xF6: //ASCII "o" umlaut
        rxbyte = 0xEF;
        break;
      case 0xFC: //ASCII "u" umlaut
        rxbyte = 0xF5;
        break;

      case 0xA3: //sterling (pounds)--liere -etv eigenes zeichen
        rxbyte = 0xED;
        break;
      case 0xB0: //degrees symbol
        rxbyte = 0xDF;
        break;
      case 0xB5: //mu
        rxbyte = 0xE4;
        break;

      case 0xC4: //A Umlaut-----gros------------------------------
        rxbyte = 0xE1;
        break;
      case 0xC0: //"A" variants
      case 0xC1:
      case 0xC2:
      case 0xC3:
      case 0xC5:
        rxbyte = 0x41;
        break;
      case 0xC8: //"E" variants
      case 0xC9:
      case 0xCA:
      case 0xCB:
        rxbyte = 0x45;
        break;
      case 0xCC: //"I" variants
      case 0xCD:
      case 0xCE:
      case 0xCF:
        rxbyte = 0x49;
        break;
      case 0xD1: //"N" tilde -> plain "N"
        rxbyte = 0x43;
        break;
      case 0xD6:// O Umlaut-----gros-------------------------------------
        rxbyte = 0xEF;
        break;
      case 0xD2: //"O" variants
      case 0xD3:
      case 0xD4:
      case 0xD5:
      case 0xD8:
        rxbyte = 0x4F;
        break;
      case 0xDC:// U Umlaut----gros----------------------------------------
        rxbyte = 0xF5;
        break;
      case 0xD9: //"U" variants
      case 0xDA:
      case 0xDB:
        rxbyte = 0x55;
        break;
      case 0xDD: //"Y" acute -> "Y"
        rxbyte = 0x59;
        break;
      /*    case 0xDF: //beta  //mucks up LCDSmartie's degree symbol??
            rxbyte = 0xE2;
            break;
      */
      case 0xE0: //"a" variants except umlaut
      case 0xE1:
      case 0xE2:
      case 0xE3:
      case 0xE5:
        rxbyte = 0x61;
        break;
      case 0xE7: //"c" cedilla -> "c"
        rxbyte = 0x63;
        break;
      case 0xE8: //"e" variants
      case 0xE9:
      case 0xEA:
      case 0xEB:
        rxbyte = 0x65;
        break;
      case 0xEC: //"i" variants
      case 0xED:
      case 0xEE:
      case 0xEF:
        rxbyte = 0x69;
        break;
      case 0xF2: //"o" variants except umlaut
      case 0xF3:
      case 0xF4:
      case 0xF5:
      case 0xF8:
        rxbyte = 0x6F;
        break;
      case 0xF7: //division symbol
        rxbyte = 0xFD;
        break;
      case 0xF9: //"u" variants except umlaut
      case 0xFA:
      case 0xFB:
        rxbyte = 0x75;
        break;
      default:
        // FEHLERHAFTE AUSGABEN !
        break;
    }
    lcd.write(rxbyte);  //otherwise a plain char so we print it to lcd
    //   lcd.print(rxbyte);
  }
}

void taster(void)
{
  if (warten > PAUSE)  // Damit nicht der tastendruck zu oft angenommen wird
  {
    for (i = 2; i < TASTERANZAHL + 2; i++)
    {
      IN = digitalRead(i);
   //   if (IN == LOW)
       if (IN == HIGH)     
      {
        digitalWrite(13, HIGH);
        Serial.write(i + 47); //pin 2 == 50 == ASCII "2"
        //       delay(1);
        Serial.flush(); 
        warten = 0;
        break;
      }
    }
  //  Serial.flush(); 
  //  warten = 0;

   if (warten > PAUSE+10000)
   {
    warten = PAUSE;
   }
  }
  if (warten == PAUSE)
  {
    digitalWrite(13, LOW);
  }
  warten++;
}

```
***

### Code mit externen Tastern: [hier](https://github.com/ToWipf/LCD-Smartie/blob/master/smartieExterneTasten.ino)

Tobias Fritsch  August 2016 - April 2018
