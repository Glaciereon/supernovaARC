# v01_first_version

```cpp
//v1: first version. reads the bmp, gets velocity from a rolling list of heights, button moves the servo, and timeToFly is still broken.

//last years apollo stuff

//talk about making a velocity/height based liftoff trigger
// make the integer/float casting thing for recording data please
//implement the calculating/comunicating of the apogee
//BIG!!!: make timeToFly with the new york method

//libraries

#include <EEPROM.h>
#include <Wire.h>
#include <SPI.h>
#include <Adafruit_Sensor.h>
#include "Adafruit_BMP3XX.h"
#include <Servo.h>

//object creation

Servo myServo;

//constants

#define SEALEVELPRESSURE_HPA (1013.25)
#define PASTDATA_COLS 2
const byte emergencyTime = 6;
const int servoButton = 12;
const int servoPin = 4;

//vars
int listIndex = 0;
float velocity = 0;
float pastData[PASTDATA_ROWS][PASTDATA_COLS];	//number of rows depends on run time, needs to be even, test for this. Newest values at the end
int servoPos = 90;
float height_0 = 0;
float apogee = 0; //can be cut if we want
int lastSample = 0;
void setup() {

//attachment and begining
pinMode(13, OUTPUT);  //this is the onboard LED, the second red light that you see
pinMode(servoButton, INPUT);
myServo.attach(servoPin);
Serial.begin(115200);

//sensor
while (!Serial);
 Serial.println("Adafruit BMP388 / BMP390 test");
 bmp.setTemperatureOversampling(BMP3_OVERSAMPLING_4X);
 bmp.setPressureOversampling(BMP3_OVERSAMPLING_4X);
 bmp.setIIRFilterCoeff(BMP3_IIR_FILTER_COEFF_3);
 bmp.setOutputDataRate(BMP3_ODR_25_HZ);  				//the rest of the program is based off of this; don't edit

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

digitalWrite(13, HIGH);  // end of setup; two red lights
}

void loop() {

static unsigned long nextSample = millis(); //if a variable is static the next time the code will try to initialize it it will have the same value. kinda like a global variable but for a function
  if (millis() >= nextSample) {
    double t = getTime();
    float h = getHeight();
    updatePastData(h, t);
    velocity = getVelocity();
    Serial.println(String(pastData[PASTDATA_ROWS-1][1] - pastData[0][1]) + " " + String(getTime()) + " " + velocity);
    nextSample += 30;
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

if (false) { // <-- FIX: Replace with launch condition when ready
Serial.println("Flying");
timeToFly();
}

}

//functions

/*
checks if the button is down for .12 seconds
returns true if it is
*/
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

/*
Returns time since program started with two decimal places of precision
*/
double getTime() {
 return ((double)(millis())) / (double)(1000);
}

/*
Returns height relative to startup position.
*/
float getHeight() {
  return bmp.readAltitude(SEALEVELPRESSURE_HPA) - height_0;
}

/*
Puts next height and time into past data
*/
void updatePastData(float h, double t) {
  for (int i = 0; i < PASTDATA_ROWS - 1; i++) {
    pastData[i][0] = pastData[i + 1][0];
    pastData[i][1] = pastData[i + 1][1];
  }
  pastData[PASTDATA_ROWS - 1][0] = h;
  pastData[PASTDATA_ROWS - 1][1] = t;
}

/*
Returns velocity based on the sums of the top and bottom of the list
*/
float getVelocity() {
  float botHeightSum = 0;
  float topHeightSum = 0;
  float botTimeSum = 0;
  float topTimeSum = 0;
  for (int i = 0; i<PASTDATA_ROWS/2; i++) {
    botHeightSum += pastData[i][0];
    botTimeSum += pastData[i][1];
    topHeightSum += pastData[i+PASTDATA_ROWS/2][0];
    topTimeSum += pastData[i+PASTDATA_ROWS/2][1];
  }

  float deltaHeight = (topHeightSum - botHeightSum);
  float deltaTime = (topTimeSum - botTimeSum);
  return deltaHeight/deltaTime;
}

//you already know (broken at the moment)
void timeToFly() {
  float velocity;
  height_0 = bmp.readAltitude(SEALEVELPRESSURE_HPA);
  float time_0 = getTime();
  do {
    if (millis() >= nextSample) {
    double t = getTime();
    float h = getHeight();time
    updatePastData(h, t);
    velocity = getVelocity();
    Serial.println(String(pastData[PASTDATA_ROWS-1][1] - pastData[0][1]) + " " + String(getTime()) + " " + velocity);
    nextSample += 30;
  }
    record(pastData[PASTDATA_ROWS-1][0]);
    Serial.print("Velocity is: ");
    Serial.println(velocity);
    bool a = false; // <-- FIX: Replace with real apogee calculation
    bool b = (pastData[PASTDATA_ROWS-1][0] < 200/*precalculated height*/);
    bool c = pastData[PASTDATA_ROWS-1][1] - time_0 < emergencyTime; //control
  } while (false /*a && b && c*/);

  myServo.write(90);//the soul purpose of this program

  while (listIndex < 1024) {
    record(getHeight());
  }

  //sum with apogee
}

/*
records the data
makes important assumption that the change in height per interval doesnt exceed 100!!
*/
void record(float height) {
  Serial.print("height comparative to liftoff is: ");
  Serial.println(height, 2); // --- FIX: Print float with 2 decimal places
  if (listIndex < EEPROM.length()) {
    EEPROM.put(listIndex, height);
    listIndex++;
  }                                                  
}

/*
prints all data
*/
void printData() {
  for (int i = 0; i<PASTDATA_ROWS; i++) {
     Serial.print("Time: ");
     Serial.print(pastData[i][1], 4);
     Serial.print(" Height: ");
     Serial.println(pastData[i][0], 2);
  }
}


//data trial functions, don't put these in the acuall code

/*
mane function for these
@param sample size
*/
float dataTrials(int n) {
  float hSample[n];

  float t_0 = getTime();
  for (int i = 0; i<n; i++) {
    hSample[i] = bmp.readAltitude(SEALEVELPRESSURE_HPA) - height_0;
  }
  Serial.print("Putting the height into a variable takes ");
  Serial.print((getTime()-t_0)/n, 4);
  Serial.println(" seconds.");
  Serial.println("Average deviation is " + averageDeviation(hSample, n));
  
}

//chat gpt 🤑
float averageDeviation(float data[], int n) {
  if (n <= 0) return 0.0;

  // Step 1: Calculate the mean
  float sum = 0.0;
  for (int i = 0; i < n; i++) {
    sum += data[i];
  }
  float mean = sum / n;

  // Step 2: Calculate the average of absolute differences
  float totalDeviation = 0.0;
  for (int i = 0; i < n; i++) {
    totalDeviation += abs(data[i] - mean);
  }

  return totalDeviation / n;
}


//only in ohio 💀💀💀💀💀💀💀💀💀
// 👹
```
