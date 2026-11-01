# v03_teensy_sd_states

```cpp
//v3: moved to the teensy with sd card logging, a state machine (preLaunch/preSeparation/postSeparation), a serial menu, and a smoothing filter on height

/*
general info
frequency is 33hz ( 30ms between cycles)
rollover at 71 mins, don't worry about it
//implement storing the name in the all roots folder
// explore the int pin on the bmp
*/

//libraries

#include <Wire.h>
#include <SPI.h>
#include <Adafruit_Sensor.h>
#include "Adafruit_BMP3XX.h"
#include <Servo.h>
#include <SD.h>

//object creation

Adafruit_BMP3XX bmp;
Servo myServo;
File myFile;


//constants

#define SEALEVELPRESSURE_HPA (1013.25)
#define PASTDATA_ROWS 16
//PASTDATA_ROWS has to be even, don't touch if you don't know what you're doing.
const byte servoButton  = 12;
const byte servoPin = 4;
const byte emergencyTime = 6;
const int chipSelect = BUILTIN_SDCARD;

//vars

float height_0 = 0;
float velocity = 0;
float prevH = 1;
int servoPos = 90;
byte state = 0;
byte flushCount = 0;
float height[PASTDATA_ROWS];
unsigned long time[PASTDATA_ROWS];

enum States {preLaunch = 0, preSeparation = 1, postSeparation = 2,  communication = 3};
String fileName = "";
//setup

void setup() {

  //attachment and begining
pinMode(13, OUTPUT);  //this is the onboard LED, the second red light that you see
pinMode(servoButton, INPUT);
myServo.attach(servoPin);
Serial.begin(115200);

  //sensor setup
  
  while (!Serial); //may be iffy on the teensy
 Serial.println("Adafruit BMP388 / BMP390 test");
 bmp.setTemperatureOversampling(BMP3_OVERSAMPLING_4X);
 bmp.setPressureOversampling(BMP3_OVERSAMPLING_4X);
 bmp.setIIRFilterCoeff(BMP3_IIR_FILTER_COEFF_3);
 bmp.setOutputDataRate(BMP3_ODR_50_HZ);  				//the rest of the program is based off of these; don't edit

if (!bmp.begin_I2C()) {
   Serial.println("Could not find a valid BMP3 sensor, check wiring!");
   while (1) ;
 }
 if (!bmp.performReading()) {
   Serial.println("Failed to perform reading aw hell naw :(");
   return;
 }
Serial.println("sensor nominal");
myServo.write(90);  // Set servo to initial position
Serial.println("servo unlocked");
height_0 = bmp.readAltitude(SEALEVELPRESSURE_HPA); //set initial height for reference of when to blast off.

for (int i = 0; i<PASTDATA_ROWS; i++) {
  height[i] = i+1;
  time[i] = i+1; //making the lists 0 cause yk
}
  getHeight();
  getHeight();  //initialization proclamation
  
digitalWrite(13, HIGH);  // end of setup; two red lights
}

void loop() {
 static unsigned long nextSample = micros(); //static means global variable within a function.
  if (micros() >= nextSample) {
    nextSample += 30000; //30 ms
   switch (state) {
    case communication:
    communicationProg();
    break;
    case preLaunch:
    preLaunchCycle();
    break;
    case preSeparation:
    preSeparationCycle();
    break;
    case postSeparation:
    postSeparationCycle();
    break;
  }
  }
  delay(5); //waits a bit at the end of everyloop to prevent board usage, overheating, etc
}
/*
the board stops all processes and communicated with the usb
*/
int communicationProg() {
  Serial.println("Talking sequence initiated. What would you like to do?\n\n 0 - quit, go back to detecting apogee\n1 - lock/unlock servo\n3 - open a new height data file (no spaces please or like anything wild, just letters and _'s\n4 - goon idk");
  while (Serial.available() <= 0) {
    delay(5);
  }
  int a = (int)Serial.read()-48;
  while (Serial.available() > 0) {
    Serial.read();
  }
  switch (a) {
    case 0:
    state = preLaunch;
    return 0;
    break;
    case 1:
    if (servoPos == 90) {
           servoPos = 0;
    } else {
           servoPos = 90;
    }
    myServo.write(servoPos);
    break;
    case 3:
    newHeightFile();
    break;
  }
  
}

int preLaunchCycle() {
   height_0Update();
    unsigned long t = micros();  // returns the time since boot in micro seconds (1,000,000 micro seconds = 1 second)
    float h = getHeight();
    updatePastData(t,h);
    velocity = getVelocity();
  
    if (h > 10) { //liftoff is greater than 10 meters
      state = preSeparation;
      return 0;
    }

    if (Serial.available() > 0) { //We sent it a message
      state = communication;
      return 0;
    }

    if (buttonPressed()) {  // moves it 90 degrees if button was pressed for .12 seconds
         if (servoPos == 90) {
           servoPos = 0;
         } else {
           servoPos = 90;
         }
    myServo.write(servoPos);
    delay(200);
    Serial.println("Servo moved 90 dgrees");
 }

  return 0;
}
   


int preSeparationCycle() {
    myFile = SD.open(fileName, FILE_WRITE); //reopens the file
  static float time_0 = millis()/1000; //time_0 is in seconds
    unsigned long t = micros();  // returns the time since boot in micro seconds (1,000,000 micro seconds = 1 second)
    float h = getHeight();
    if (myFile) {
      myFile.println("Height in meters off the ground: " + (String)h + " time in seconds since liftoff: " + (String)(millis()/1000 - time_0));
    }
    updatePastData(t,h);
    velocity = getVelocity();
    bool a = false; // <-- FIX: Replace with real apogee calculation
    bool b = h < 200;   //cutoff height
    bool c = t/1000 - time_0 < emergencyTime; //control
    if (a && b && c) {
      myServo.write(90);//the soul purpose of this program
      state = postSeparation;
    }
  return 0;
}

int postSeparationCycle() {
    float h = getHeight();
  float time_0 = millis()/1000; // second
  if (millis()/1000 - time_0 < 40) {
    if (myFile) {
      myFile.println("Height in meters off the ground: " + (String)h + " time in seconds since liftoff: " + (String)(millis()/1000 - time_0));
    }
  } else {
    myFile.close(); //STOP
    while (true) {
      delay(5);
    }
  }
  return 0; //i can make the button do sum
}
//returns true if the button is pressed, false otherwise
bool buttonPressed() {
 if (digitalRead(servoButton) != HIGH) {
   delay(25);
   if (digitalRead(servoButton) != HIGH) {
     delay(25);  // this is for accidental presses or accidental contact
     if (digitalRead(servoButton) != HIGH) {
       delay(100);
       if (digitalRead(servoButton) != HIGH) {
         Serial.println("Button pressed");
         delay(20);
         return true;
       }
     }
   }
 }
 return false;
}
//updates initial height every 5 mins
void height_0Update() {
  static unsigned long nextSample1 = millis();
  if (millis() >= nextSample1) {
    nextSample1 += 300000;// 5 mins
    if(bmp.performReading()){
    height_0 = bmp.readAltitude(SEALEVELPRESSURE_HPA);  //reset initial height to account for slight weather changes
} else {
    Serial.println("BMP read failed");
}
  }
}
//Returns height relative to startup position.
float getHeight() {
  // experimental features
  double filterCoeficciant = .7; //range from 0 to 1. Closer to 1 = reacts faster but less smooth, Closer to 0 = smoother but lags behind real altitude
  
   if (!bmp.performReading()) {
    myFile = SD.open(FileName, FILE_WRITE);
    Serial.println("BMP read failed");
    
    return prevH; // fallback: return last valid height
  } //the sensor makes a reading
  
  float h = (bmp.readAltitude(SEALEVELPRESSURE_HPA) - height_0)*filterCoeficciant + prevH*(1-filterCoeficciant); //h_n(1-coeficciant) + h_n-1*coeficciant
  prevH = h;
  return h;
}
//🧽
void updatePastData (unsigned long t, float h){
    for (int i = 0; i < PASTDATA_ROWS - 1; i++) {
    time[i] = time[i + 1];
    height[i] = height[i + 1];
  }
  time[PASTDATA_ROWS - 1] = t;
  height[PASTDATA_ROWS - 1] = h;
}
/*
what it's all about
returns instantanious velocity in m/s
*/
float getVelocity() {
  long unsigned d_T = time[PASTDATA_ROWS-1] -time[0]; //difference in newest to oldest data in micros
  float botHeightSum = 0;
  float topHeightSum = 0;

  for (int i = 0; i<PASTDATA_ROWS/2; i++) {
    botHeightSum += height[i];
    topHeightSum += height[i+PASTDATA_ROWS/2];
  }
float avgBot = botHeightSum / (PASTDATA_ROWS / 2); //average here for sake of avoiding rounding errors
float avgTop = topHeightSum / (PASTDATA_ROWS / 2);
float deltaHeight = avgTop - avgBot;
float dt = d_T / 1000000.0;  // convert microseconds to seconds 
return deltaHeight / dt;     // velocity in m/s
}

void newHeightFile() {
  
  if (!SD.begin(chipSelect)) {
    Serial.println("Initialization failed!");
    return;
  }
  Serial.println("Card initialized\ninput name of new file: ");
  
  
  while (Serial.available() <= 0) {
    delay(5);
  }
  fileName = "";
  while (Serial.available() > 0) {
      fileName = Serial.readStringUntil('\n');
      fileName.trim(); // strips \n, \r, spaces
  }

  myFile = SD.open(fileName, FILE_WRITE);
  if (myFile) {
    Serial.println("File created successfully.");
  } else {
    Serial.println("Error creating file.");
  }
  myFile.close(); // this is for the file's safety
  
}
```
