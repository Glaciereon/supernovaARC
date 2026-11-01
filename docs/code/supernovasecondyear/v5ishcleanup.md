# v05_setup_cleanup

```cpp
//v5: small cleanup of v4: pin and servo setup moved to the start of setup, not big change

//get onboard LED functioning for after liftoff

#include <Wire.h>
#include <SPI.h>
#include <Adafruit_Sensor.h>
#include "Adafruit_BMP3XX.h"
#include <Servo.h>
#include <SD.h> //blocking 🐭

Adafruit_BMP3XX bmp;
Servo myServo;
File myFile;

struct DataPoint {
  float height_m;
  float time_s;

  void printTo(Print &out) const {
    out.print("Height: ");
    out.print(height_m);
    out.print(", Time: ");
    out.println(time_s);
  }
};

#define SD_CS 10  
#define SEALEVELPRESSURE_HPA (1013.25)
#define LAUNCH_HEIGHT_M (10)
#define SEPERATION_HEIGHT_M (200)
#define DEFAULT_LOCKED_POSITION_DEG (90)
#define DEFAULT_UNLOCKED_POSITION_DEG (0)
const byte servoButtonPin  = 12;
const byte servoPin = 4;
const bool debug = true;
const byte maxSeperationTime_s = 6;
const int flushCount = 10;


int LOCKED_POSITION_DEG = 90;
int UNLOCKED_POSITION_DEG = 0;
float time_0_s = .0067;
float height0_m = 0;
float prevHeight_m = .001;
int dataPointIndex = 0;
byte state = 0;
const float filterCoeficcient = .7; //range from 0 to 1. Closer to 1 = reacts faster but less smooth, Closer to 0 = smoother but lags behind real 
String fileName = "gemini_guided_3.txt";
bool error = true;
bool locked = false;

DataPoint dataPoints[10];

enum States {preLaunch = 0, preSeparation = 1, postSeparation = 2,  communication = 3};

void setup() {
  
  Serial.begin(115200);
  pinMode(LED_BUILTIN, OUTPUT);
  pinMode(servoButtonPin, INPUT_PULLUP); //https://chatgpt.com/c/6965cdf3-5564-832c-a037-b57b931782c6
  myServo.attach(servoPin);
  Serial.println("Program start");
  Serial.print("Use defaults? y/n (locked position: ");
  Serial.print(DEFAULT_LOCKED_POSITION_DEG);
  Serial.print(" unlocked: ");
  Serial.print(DEFAULT_UNLOCKED_POSITION_DEG);
  Serial.println(" filename: gemini_guided_2.txt");
  while (Serial.available()  == 0) {
    delay(5);
  }

  String response = Serial.readStringUntil('\n');
  response.trim();

  if (response == "y") {
    LOCKED_POSITION_DEG = DEFAULT_LOCKED_POSITION_DEG;
    UNLOCKED_POSITION_DEG = DEFAULT_UNLOCKED_POSITION_DEG;
    int lastNum = 2;
    SD.begin(SD_CS);
      lastNum++;
      fileName[fileName.length() - 5] = char('0' + lastNum);
      
    myFile = SD.open(fileName.c_str(), FILE_WRITE);
    Serial.println(myFile ? "File initialized succesfully" : "Failed to create File");
    Serial.println("File name is ");
    Serial.println(fileName);
    if (myFile) {
      myFile.print("\nFile named ");
      myFile.println(myFile.name());
    }
    myFile.close();
    } else { // ***** EDIT 2: this was previously causing "expected unqualified-id" *****
    calibrateServo();
    setupSD_Card();
  }

  setupSensor();

  for (int i = 0; i<5; i++) {
    getHeight();
    delay(30);
  }
  
Serial.print("CONFIG | Separation height (m): ");
Serial.print(SEPERATION_HEIGHT_M);
Serial.print(", Launch height (m): ");
Serial.print(LAUNCH_HEIGHT_M);
Serial.print(", Max separation time (s): ");
Serial.print(maxSeperationTime_s);
Serial.print(", Filter coefficient: ");
Serial.print(filterCoeficcient);
Serial.println(", Status: READY");

myFile = SD.open(fileName.c_str(), FILE_WRITE);
if (myFile) {
  myFile.print("CONFIG | Separation height (m): ");
  myFile.print(SEPERATION_HEIGHT_M);
  myFile.print(", Launch height (m): ");
  myFile.print(LAUNCH_HEIGHT_M);
  myFile.print(", Max separation time (s): ");
  myFile.print(maxSeperationTime_s);
  myFile.print(", Filter coefficient: ");
  myFile.print(filterCoeficcient);
  myFile.println(", Status: READY");
} else {
  Serial.println("Couldn't write config data");
}

  myFile.close();

  digitalWrite(LED_BUILTIN, HIGH);
  
}

void loop() {
 static unsigned long nextSample_us = micros(); //static means global variable within a function.
  if (micros() >= nextSample_us) {
    nextSample_us += 30000; //30 ms
   switch (state) {
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
}

int preLaunchCycle() {
    float h = getHeight();
  
    if (h > LAUNCH_HEIGHT_M) {
      myFile = SD.open(fileName.c_str(), FILE_WRITE); //reopens the file
      state = preSeparation;
      myFile.println("Launch began at this many seconds");
      time_0_s = millis() / 1000.0f;
      myFile.println(time_0_s);
      myFile.close();
      return 0;
    }

    Serial.println(h);

    if (buttonPressed()) {  // moves it 90 degrees if button was pressed for .12 seconds
         if (locked) {
             myServo.write(UNLOCKED_POSITION_DEG);
             Serial.println("Servo unlocked, degree position: ");
             locked = false;
         } else {
             myServo.write(LOCKED_POSITION_DEG);
             Serial.println("Servo locked, degree position: ");
             locked = true;
         }
         Serial.print(myServo.read());
        delay(200);
    }

  return 0;
}

int preSeparationCycle() {
    DataPoint currentPoint;
    currentPoint.height_m = getHeight();
    currentPoint.time_s = millis()/1000.0f - time_0_s;
    dataPoints[dataPointIndex] = currentPoint;
  
    dataPointIndex++;
    if (dataPointIndex == flushCount) {
      myFile = SD.open(fileName.c_str(), FILE_WRITE); //reopens the file
      dataPointIndex = 0;
      if (myFile) {
        for (int i = 0; i<flushCount; i++) {
        dataPoints[i].printTo(myFile);
        }
      }
      myFile.flush();
      myFile.close();
    }
  
    bool belowCutoff = currentPoint.height_m > SEPERATION_HEIGHT_M;
    bool withinTimeWindow = (millis() / 1000.0f - time_0_s) > maxSeperationTime_s;
    if (belowCutoff || withinTimeWindow) {
      myFile = SD.open(fileName.c_str(), FILE_WRITE); //reopens the file
      myServo.write(UNLOCKED_POSITION_DEG);//the soul purpose of this program
      if (myFile) {
        myFile.print("Servo started turning at this many seconds: ");
        myFile.println(millis()/1000.0f - time_0_s); // this gon come out rong since i haven't flushed the data yet
      }
      myFile.close();
      state = postSeparation;
    }
  return 0;

}

void setupSD_Card() {

  bool cardStatus = (SD.begin(SD_CS));
  Serial.println(cardStatus ? "Card initialized" : "Initialization failed");

  
  Serial.println("Input file name: (lowercase letters, numbers, and _'s allowed only)");
  while (Serial.available() <= 0) {
    delay(5);
  }
  fileName = Serial.readStringUntil('\n');
  fileName.trim(); // strips \n, \r, spaces

  if (SD.exists(fileName.c_str())) {Serial.println("Ts name already exists. Be cautious");}
  myFile = SD.open(fileName.c_str(), FILE_WRITE);
  Serial.println(myFile ? "File initialized succesfully" : "Failed to create File");
  if (myFile) {
    Serial.print("File name: ");
    Serial.println(myFile.name());
    
    myFile.print("\nFile named ");
    myFile.println(myFile.name());
  }
  
  myFile = SD.open(fileName.c_str(), FILE_WRITE);
  myFile.print("Locked position in degrees is: ");
  myFile.println(LOCKED_POSITION_DEG);
  myFile.print("Unlocked position in degrees is: ");
  myFile.println(UNLOCKED_POSITION_DEG);

  myFile.close(); // this is for the file's safety
  
}

void setupSensor() {
  Serial.println("Adafruit BMP390 test");

  bool bmp_I2C_init = bmp.begin_I2C();
  Serial.print("BMP390 I2C initialization: ");
  Serial.println(bmp_I2C_init ? "Doing the most" : "FAIL: Could not find a valid BMP3 sensor, check wiring!");

  // Configure sensor
  bmp.setTemperatureOversampling(BMP3_OVERSAMPLING_4X);
  bmp.setPressureOversampling(BMP3_OVERSAMPLING_4X);
  bmp.setIIRFilterCoeff(BMP3_IIR_FILTER_COEFF_3);
  bmp.setOutputDataRate(BMP3_ODR_50_HZ);
/*
  if (debug) {
    myFile = SD.open(fileName, FILE_WRITE);
    
    Serial.println("BMP3XX configuration:");
    Serial.print("Temperature oversampling: ");
    Serial.println(bmp.getTemperatureOversampling());
    Serial.print("Pressure oversampling: ");
    Serial.println(bmp.getPressureOversampling());
    Serial.print("IIR filter coefficient: ");
    Serial.println(bmp.getIIRFilterCoeff());
    Serial.print("Output data rate: ");
    Serial.println(bmp.getOutputDataRate());

    if (myFile) {
      myFile.println("\nBMP3XX configuration:");
      myFile.print("Temperature oversampling: ");
      myFile.println(bmp.getTemperatureOversampling());
      myFile.print("Pressure oversampling: ");
      myFile.println(bmp.getPressureOversampling());
      myFile.print("IIR filter coefficient: ");
      myFile.println(bmp.getIIRFilterCoeff());
      myFile.print("Output data rate: ");
      myFile.println(bmp.getOutputDataRate());
    }

    myFile.close();
  }
*/
  bool bmp_read = bmp.performReading();
  Serial.print("Sensor read: ");
  Serial.println(bmp_read ? "SUCCESS" : "FAIL: BMP390 not producing data");

  int avgSampleCnt = 20;
  Serial.print("Warming and clearing sensor for ");
  Serial.print(2*avgSampleCnt*30);
  Serial.println(" ms");
  
  for (int i = 0; i<avgSampleCnt; i++) { //warms up the sensor's oversampling
    bmp.performReading();
    height0_m = bmp.readAltitude(SEALEVELPRESSURE_HPA);
    delay(30);
  } 
  height0_m = 0.0f; //lmao
    
  for (int i = 0; i<avgSampleCnt; i++) { // gets average of og height
    bmp.performReading();
    height0_m += (1.0f/avgSampleCnt) * bmp.readAltitude(SEALEVELPRESSURE_HPA);
    delay(30);
  }
  Serial.println(height0_m);
  
  Serial.println("Sample reading:");
  Serial.println("-------------------------- Initial Sample");
  Serial.print("Temperature: ");
  Serial.print(bmp.temperature, 2);
  Serial.println(" C");
  Serial.print("Pressure: ");
  Serial.print(bmp.pressure / 100.0, 2);
  Serial.println(" hPa");
  Serial.print("Altitude (std atm): ");
  Serial.print(bmp.readAltitude(SEALEVELPRESSURE_HPA), 2);
  Serial.println(" m");
  Serial.println("-------------------------------");

}

float getHeight() {
  
    if (!bmp.performReading()) {
      
    myFile = SD.open(fileName.c_str(), FILE_WRITE);
    Serial.println("BMP read failed");
      
    if (error) {
      myFile.println("BMP read failed at ");
      myFile.println(millis());
      myFile.println(" after launch");
      myFile.close();
      error = false;
    }
    
    return prevHeight_m; // fallback: return last valid height
   }
  
  float h = (bmp.readAltitude(SEALEVELPRESSURE_HPA) - height0_m)*filterCoeficcient + prevHeight_m*(1-filterCoeficcient);// h = current*c + previous*(1-c)
  prevHeight_m = h;
  return h;
}

bool buttonPressed() {
  /*
  bool isReallyPressed = false;
  if (digitalRead(servoButtonPin) != HIGH) { //i'm assuming this is false when pressed
    isReallyPressed = true;
    unsigned long time_01_ms = millis();
    byte buttonDebounceTime_ms = 150;
    while ((millis() < time_01_ms + buttonDebounceTime_ms)) {
      if (digitalRead(servoButtonPin) == HIGH) {
        isReallyPressed = false;
      }
    }
  }
  */
  if (Serial.available() > 0) {
    Serial.readStringUntil('\n');
    return true;
  } else {
    return false;
  }
 // return isReallyPressed;
}

int postSeparationCycle() {
  
  DataPoint currentPoint;
  currentPoint.height_m = getHeight();
  currentPoint.time_s = millis()/1000.0f - time_0_s;

  if (currentPoint.height_m > LAUNCH_HEIGHT_M) {
    dataPoints[dataPointIndex] = currentPoint;
    dataPointIndex++;
    if (dataPointIndex == flushCount) {
      myFile = SD.open(fileName.c_str(), FILE_WRITE);
      if (myFile) {
        for (int i = 0; i < flushCount; i++) {
          dataPoints[i].printTo(myFile);
        }
        myFile.flush();
        myFile.close();
      }
      dataPointIndex = 0;
    }
  } else {
    if (dataPointIndex > 0) {
      myFile = SD.open(fileName.c_str(), FILE_WRITE);
      if (myFile) {
        for (int i = 0; i < dataPointIndex; i++) {
          dataPoints[i].printTo(myFile);
        }
        
        myFile.flush();
        myFile.close();
      }
      dataPointIndex = 0;
    }

    digitalWrite(LED_BUILTIN, HIGH);
    delay(100);
    digitalWrite(LED_BUILTIN, LOW);
    delay(100);
  }

  return 0;
}

void calibrateServo() {
  Serial.println("Experiment with locking and unlocking the servo. remember the degree values of locked and unlocked stage\nsend done when done");
  String response = "0";
  while (!response.equals("done")) {
    if (Serial.available() > 0) {
      response = Serial.readStringUntil('\n');
      response.trim();
      myServo.write((response.toInt() % 180 + 180) % 180);
    }
  }
  
  Serial.println("input locked position: ");
  while (Serial.available() == 0){delay(1);}
  LOCKED_POSITION_DEG = Serial.readStringUntil('\n').toInt();

  Serial.println("input unlocked position: ");  
  while (Serial.available() == 0){delay(1);}
  UNLOCKED_POSITION_DEG = Serial.readStringUntil('\n').toInt();

  Serial.println("Done!");
}
```
