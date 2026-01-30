#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <SoftwareSerial.h>
#include <TinyGPS++.h>

void readGPS();
void sendLocationSMS();

LiquidCrystal_I2C lcd(0x27, 16, 2);

SoftwareSerial gsm(2, 3);       
SoftwareSerial gpsSerial(4, 5); 
TinyGPSPlus gps;


#define MQ3 A2
#define LED 7

int alcoholThreshold = 450;   
bool alertSent = false;

float latitude = 0.0;
float longitude = 0.0;

void setup() {
  pinMode(LED, OUTPUT);
  digitalWrite(LED, LOW);

  Serial.begin(9600);
  gsm.begin(9600);
  gpsSerial.begin(9600);

  lcd.init();
  lcd.backlight();

  lcd.setCursor(0, 0);
  lcd.print("Alcohol Detect");
  lcd.setCursor(0, 1);
  lcd.print("System Init...");
  delay(3000);
  lcd.clear();
}

void loop() {
  readGPS();  

  int alcoholValue = analogRead(MQ3);

  Serial.print("Alcohol: ");
  Serial.println(alcoholValue);

  lcd.setCursor(0, 0);
  lcd.print("Alcohol: ");
  lcd.print(alcoholValue);
  lcd.print("   ");

  if (alcoholValue > alcoholThreshold) {
    digitalWrite(LED, HIGH);
    lcd.setCursor(0, 1);
    lcd.print("ALCOHOL ALERT ");

    if (!alertSent) {
      sendLocationSMS();
      alertSent = true;
    }
  } 
  else {
    digitalWrite(LED, LOW);
    lcd.setCursor(0, 1);
    lcd.print("Safe to Drive ");
    alertSent = false;
  }

  delay(1000);
}


void readGPS() {
  gpsSerial.listen();

  while (gpsSerial.available()) {
    gps.encode(gpsSerial.read());
  }

  if (gps.location.isValid()) {
    latitude = gps.location.lat();
    longitude = gps.location.lng();

    Serial.print("Lat: ");
    Serial.print(latitude, 6);
    Serial.print(" Lon: ");
    Serial.println(longitude, 6);
  }
}


void sendLocationSMS() {
  if (!gps.location.isValid()) {
    lcd.clear();
    lcd.print("GPS Not Ready");
    delay(2000);
    return;
  }

  gsm.listen();

  lcd.clear();
  lcd.print("Sending SMS");

  gsm.println("AT");
  delay(1500);
  gsm.println("AT+CMGF=1");
  delay(1500);

  gsm.println("AT+CMGS=\"+919042716186\""); 
  delay(1500);

  gsm.print("ALERT!\nAlcohol detected\nLocation:\n");
  gsm.print("https://maps.google.com/?q=");
  gsm.print(latitude, 6);
  gsm.print(",");
  gsm.print(longitude, 6);

  gsm.write(26); 
  delay(4000);

  lcd.clear();
  lcd.print("SMS Sent");
}
