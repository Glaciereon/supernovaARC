# v07_kalman_per_cycle_release_sim

```cpp
//v7: kalman now updates every cycle off a saved previous point, and the release prediction steps through servo opening, spring boost, and parachute drag sim

//STILL need to do the tests with the sensor. Flags left in setup sensor
// I can also forcebly add gravity to the kalman but that assumes i'm adding acceleration so  . . . .

#include <Wire.h>
#include <SPI.h>
#include <Adafruit_Sensor.h>
#include "Adafruit_BMP3XX.h"
#include <Servo.h>
#include <SD.h>
#include <BasicLinearAlgebra.h>

using namespace BLA;

Adafruit_BMP3XX bmp;
Servo myServo;
File myFile;

struct DataPacket {
  float height_m = 0;
  float velocity_m = 0;
  float rawHeight_m = 0;
};

struct DataPoint {
  float time_s;
  DataPacket data;

  void printTo(Print &out) const {
    out.print(", Time: ");
    out.print(time_s, 4);
    out.print(" Raw height: ");
    out.print(data.rawHeight_m, 4);
    out.print(" Height: ");
    out.print(data.height_m, 4);
    out.print(", Velocity: ");
    out.println(data.velocity_m, 4);
  }
};

#define SD_CS 10  
#define SERVO_PIN 4
#define BLUETOOTH_ON 1
#define SEALEVELPRESSURE_HPA (1013.25)
#define TARGET_APOGEE (228.6)
#define LAUNCH_HEIGHT_M (10)
#define SEPERATION_HEIGHT_M (220) //from sim
#define MAX_SEPERATION_TIME_S (5)
#define FLUSH_COUNT (125) // this makes sure the the data writing period is not when the rocket will even try to seperate

#define MASS_KG (.61)
#define SERVO_UNLOCK_DURATION_S (.2)
#define VELOCITY_GAIN_FROM_SPRINGS_M (8)
#define PARACHUTE_FULL_UNFURL_DURATION_S (.6)
#define K_DURING_FULL_PARACHUTE (1.842) //guess lol
#define SIM_DT (.01)

#define BASE_VARIANCE (.28) //variance in the height data
#define ACCELERATION_UNCERTAINTY (10) //idk really 

byte state = 0;
bool locked = false;
bool cardOpen = false;
bool recordingData = false;
int lockedPositionDeg = 80;
int unlockedPositionDeg = 5;
int dataPointIndex = 0;
float dataRecordingTimeInit_s = .0067;
float height0M = 0;
float apogee = 0;
String fileName = "default.txt";
DataPoint prevDPoint;
DataPoint dataPoints[FLUSH_COUNT];

Matrix<2,1> State; /* state matrix; 
H_k kalmanized height
V_k kalmanized velocity
*/
Matrix<2,2> sensorCovarience; /* how the variances relate together eg. if height is underestimated what does that mean about velocity
stdH^2 covariance
covariance dtdV^2
*/
Matrix<2,2> Transition; /* transition matrix
1 dt (dt is time between current kalmanized sample and next data sample)
0 1 
*/
Matrix<2,2> ModelVarience; /* model varaince matrix how wrong is my model (not accounting for accleration 🤣)
stdH^2 0 
0 dtdV^2
*/
Matrix <1,2> Measurement;/* Measurement matrix
height_from_sensor 0 (not measuring velocity)
*/
Matrix <1,1> R; /* sensor noise measured on the ground (it's a matrix since it's esier to do operations that way)
R
*/
Matrix <2,1> KalmanGain; /* matrix storing coefficients for how off the model was vs the acuall sensor measurements
height_coefficient velocity_coefficient
*/
Matrix <2,2> I; //you're cooked if you don't know this

enum States {onGround = 0, preLaunch = 1, preSeparation = 2, postSeparation = 3};

void setup() {

  State = {0,
           0};

  sensorCovarience = {
           1, 0,
           0, 1
  };
  Transition = {
           1, .02, //the 02 is a placeholder here for the acuall dt
           0, 1
  };
  Measurement = {
            1, 0
  };
  ModelVarience = {
           1, 0,
           0, 1
  };
  R = {BASE_VARIANCE};
  I = {    1, 0,
           0, 1
  };
  
  Serial.begin(115200);
  Serial1.begin(9600); //don't change, this is the standard

  printBoth("Program start");
  printBoth("The rocket will seperate at ");
  printBoth(SEPERATION_HEIGHT_M);
  printBoth(" meters, start recording data at ");
  printBoth(LAUNCH_HEIGHT_M);
  printBoth(" meters, automatically unlock at ");
  printBoth(MAX_SEPERATION_TIME_S);
  printBoth(" seconds, and servo locked and unlocked positions are ");
  printBoth(lockedPositionDeg);
  printBoth(unlockedPositionDeg);
  printBoth(" degrees respectively.");
  printBoth("Seperation height, launch height, and max seperation time can't be changed. The rest can be using the functions provided");

  setupServo();
  setupSDCard();
  setupSensor();

  for (int i = 0; i<5; i++) {
    getHeight();
    delay(30);
  }

  printBoth("Setup complete, gl");
  printBoth(BLUETOOTH_ON ? "Using bluetooth" : "Using serial");
}

void loop() {
 static unsigned long nextSample_us = micros(); //static means global variable within a function.
  if (micros() >= nextSample_us) {
    nextSample_us += 30000; //30 ms
    switch (state) {
      case onGround:
        onGroundCycle();
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
}

void onGroundCycle() {
  printBoth("Select operation");
  int answer = scanner().toInt();
  switch (answer) {// 1 - rotate 2 - reinitialize height_0 3 - input locked and unlocked positions 4 - input card name 5 - prime rocket
    case 1:
      rotateServo();
      break;
    case 2:
      initializeHeight0M();
      break;
    case 3:
      adjustServoPos();
      break;
    case 4:
      inputCardName();
      break;
    case 5:
      primeRocket();
      // we don't break
  }
}

void preLaunchCycle() {
  float h = getHeight();

  if (h > LAUNCH_HEIGHT_M) {
    myFile = SD.open(fileName.c_str(), FILE_WRITE);
    cardOpen = true;
    recordingData = true;
    state = preSeparation;
    myFile.println("Launch began at this many seconds");
    dataRecordingTimeInit_s = millis() / 1000.0f;
    myFile.println(dataRecordingTimeInit_s);
    return;
  }
  
  updateMatrices(h, millis()/1000.0f - dataRecordingTimeInit_s);
}

void preSeparationCycle() { //where the magic happens i'm crine
  float h = getHeight();
  updateMatrices(h, millis()/1000.0f - dataRecordingTimeInit_s);

  bool validReleaseHeight = prevDPoint.data.height_m + deltaHToFullStop() >= TARGET_APOGEE;
  bool exceededMaxSeparationTime  = prevDPoint.time_s > MAX_SEPERATION_TIME_S;
  if (validReleaseHeight || exceededMaxSeparationTime) {
    myServo.write(unlockedPositionDeg);//the soule purpose of this program
    if (myFile) {
      myFile.print("Servo started turning at this many seconds: ");
      myFile.println(prevDPoint.time_s); // this gon come out rong since i haven't flushed the data yet
    }
    state = postSeparation;
  }
}

void postSeparationCycle() {
  if (prevDPoint.data.height_m > LAUNCH_HEIGHT_M) {
    float h = getHeight();
    updateMatrices(h, millis()/1000.0f - dataRecordingTimeInit_s);
  } else {
    if (recordingData) {
      for (int i = 0; i < dataPointIndex; i++) {
        dataPoints[i].printTo(myFile);
      }
      recordingData = false;
      myFile.close();
    }
    printBoth("The apogeee in meters: ");
    printBoth(apogee);
  }
}

String scanner() {

  Stream &port = BLUETOOTH_ON ? Serial1 : Serial; // polymorphism 🤑🤑 compsci acually coming in handy

  while (port.available() == 0) {
    delay(5);
  }

  String response = port.readStringUntil('\n');
  response.trim();
  return response;
}

void printBoth(const char *msg) {
  Serial.println(msg);
  if (BLUETOOTH_ON) {
    Serial1.println(msg);
  }
}

void printBoth(float value) {
  // OVERLOADED FUNCTIONS 😍😍😍 this is for efficiency
  Serial.println(value);
  if (BLUETOOTH_ON) {
    Serial1.println(value);
  }
}

float getHeight() {
  bmp.performReading();
  return bmp.readAltitude(SEALEVELPRESSURE_HPA) - height0M;
}

void setupServo() {
  myServo.attach(SERVO_PIN);
  printBoth("Servo attached. Position:");
  printBoth(myServo.read());
}

void setupSDCard() {
  bool cardStatus = (SD.begin(SD_CS));
  printBoth(cardStatus ? "Card initialized" : "Initialization failed");

  printBoth("Input file name! Send ('n') if using default (make sure to include the .txt :)");
  String s = scanner();
  if (s != "n") {
    fileName = s;
  }

  if (SD.exists(fileName.c_str())) {
    printBoth("Ts name already exists. Be cautious");
  }

}

void setupSensor() {
  printBoth("Setting up sensor: ");
  printBoth("Adafruit BMP390 test");

  bool bmp_I2C_init = bmp.begin_I2C();
  printBoth("BMP390 I2C initialization: ");
  printBoth(bmp_I2C_init ? "Doing the most" : "FAIL: Could not find a valid BMP3 sensor, check wiring!");

  bmp.setTemperatureOversampling(BMP3_OVERSAMPLING_4X);
  bmp.setPressureOversampling(BMP3_OVERSAMPLING_4X);
  bmp.setIIRFilterCoeff(BMP3_IIR_FILTER_COEFF_3);
  bmp.setOutputDataRate(BMP3_ODR_50_HZ);

  bool bmp_read = bmp.performReading();
  printBoth("Sensor read: ");
  printBoth(bmp_read ? "SUCCESS" : "FAIL: BMP390 not producing data");

  int avgSampleCnt = 20;
  printBoth("Warming and clearing sensor for ");
  printBoth(2*avgSampleCnt*30);
  printBoth(" ms");
  
  for (int i = 0; i<avgSampleCnt; i++) { //warms up the sensor's oversampling
    bmp.performReading();
    height0M = bmp.readAltitude(SEALEVELPRESSURE_HPA);
    delay(30); //⚠️⚠️⚠️⚠️⚠️⚠️⚠️⚠️⚠️⚠️⚠️⚠️⚠️ magic number
  } 
  height0M = 0.0f; //lmao
    
  for (int i = 0; i<avgSampleCnt; i++) { // gets average of og height
    bmp.performReading();
    height0M += (1.0f/avgSampleCnt) * bmp.readAltitude(SEALEVELPRESSURE_HPA);
    delay(30);
  }
  printBoth(height0M);
  
  printBoth("Sample reading:");
  printBoth("-------------------------- Initial Sample");
  printBoth("Temperature: ");
  printBoth((int)bmp.temperature);
  printBoth(" C");
  printBoth("Pressure: ");
  printBoth((int)(bmp.pressure / 100.0));
  printBoth(" hPa");
  printBoth("Altitude (std atm): ");
  printBoth((int)bmp.readAltitude(SEALEVELPRESSURE_HPA));
  printBoth(" m");
  printBoth("-------------------------------");
}

void rotateServo() {
  if (locked) {
    myServo.write(unlockedPositionDeg);
    printBoth("Servo unlocked, degree position: ");
    locked = false;
  } else {
    myServo.write(lockedPositionDeg);
    printBoth("Servo locked, degree position: ");
    locked = true;
  }
  delay(200);
  printBoth("Servo position: ");
  printBoth(myServo.read());
  delay(800);
}

void initializeHeight0M() {
  printBoth("Old height_0: ");
  printBoth(height0M);
  height0M = 0;

  int avgSampleCnt = 20;
  for (int i = 0; i<avgSampleCnt; i++) { // gets average of og height
    bmp.performReading();
    height0M += (1.0f/avgSampleCnt) * bmp.readAltitude(SEALEVELPRESSURE_HPA);
    delay(30);
  }

  printBoth("New height_0: ");
  printBoth(height0M);
}

void adjustServoPos() {
  printBoth("Experiment with locking and unlocking the servo. remember the degree values of locked and unlocked stage\nsend done when done");

  String response = scanner();
  while (response != "done") {
    myServo.write((response.toInt() % 180 + 180) % 180); //legacy code yesir
    response = scanner();
  }
  
  printBoth("input locked position: ");
  lockedPositionDeg = scanner().toInt();
  printBoth("input unlocked position: ");  
  unlockedPositionDeg = scanner().toInt();

  printBoth("Done!");
}

void inputCardName() {
  printBoth("Input file name: (lowercase letters, numbers, and _'s allowed only)");
  fileName = scanner();
  fileName.trim();
  if (SD.exists(fileName.c_str())) {
      printBoth("Ts name already exists. Be cautious");
  }
}

void primeRocket() {
  //I'm so excited; this is the coolest thing I've ever made
  if (SD.exists(fileName.c_str())) {
    printBoth("Ts name already exists. Be cautious");
  }

  myFile = SD.open(fileName.c_str(), FILE_WRITE);
  cardOpen = true;
  printBoth(myFile ? "File initialized succesfully" : "Failed to create File");
  
  if (myFile) {
    printBoth("\nFile named ");
    printBoth(myFile.name());
    printBoth("CONFIG | Target Apogee (m): ");
    printBoth(TARGET_APOGEE);
    printBoth(", Launch height (m): ");
    printBoth(LAUNCH_HEIGHT_M);
    printBoth(", Max separation time (s): ");
    printBoth(MAX_SEPERATION_TIME_S);
    printBoth("Locked position in degrees is: ");
    printBoth(lockedPositionDeg);
    printBoth("Unlocked position in degrees is: ");
    printBoth(unlockedPositionDeg);
    printBoth(", Status: READY");
    
    myFile.print("\nFile named ");
    myFile.println(myFile.name());
    myFile.print("CONFIG | Separation height (m): ");
    myFile.println(SEPERATION_HEIGHT_M);
    myFile.print(", Launch height (m): ");
    myFile.println(LAUNCH_HEIGHT_M);
    myFile.print(", Max separation time (s): ");
    myFile.println(MAX_SEPERATION_TIME_S);
    myFile.print("Locked position in degrees is: ");
    myFile.println(lockedPositionDeg);
    myFile.print("Unlocked position in degrees is: ");
    myFile.println(unlockedPositionDeg);
    myFile.println(", Status: READY");
  }

  if (!myFile) {
    printBoth("Couldn't write config data");
  } else {
    printBoth("Rocket primed. see you on the other side");
    state = preLaunch;
  }

  myFile.close();
  cardOpen = false;
}

void updateMatrices(float rawHeight, float timeFromLaunch) {

  float dt = max(timeFromLaunch - prevDPoint.time_s, .001f);

  float dt2 = dt * dt;
  float dt3 = dt2 * dt;
  float dt4 = dt3 * dt;
  Transition(0,1) = dt;
  ModelVarience = {
    dt4/4.0 * ACCELERATION_UNCERTAINTY * ACCELERATION_UNCERTAINTY,  dt3/2.0 * ACCELERATION_UNCERTAINTY * ACCELERATION_UNCERTAINTY,
    dt3/2.0 * ACCELERATION_UNCERTAINTY * ACCELERATION_UNCERTAINTY,  dt2     * ACCELERATION_UNCERTAINTY * ACCELERATION_UNCERTAINTY
  };
  Matrix <1,1> MeasurementPoint = rawHeight;

  State = Transition * State;
  //State(1,0) -= 9.81f * dt; //gravity duh
  sensorCovarience = Transition * sensorCovarience * ~Transition + ModelVarience;
  Matrix <1,1> Residual = MeasurementPoint - Measurement * State;
  Matrix <1,1> Innovation = Measurement * sensorCovarience * ~Measurement + R;
  KalmanGain = sensorCovarience * ~Measurement * Inverse(Innovation);

  State = State + KalmanGain * Residual;
  sensorCovarience = (I - KalmanGain * Measurement) * sensorCovarience;

  prevDPoint.time_s = timeFromLaunch;
  prevDPoint.data.rawHeight_m = rawHeight;
  prevDPoint.data.height_m = State(0,0);
  prevDPoint.data.velocity_m = State(1,0);

  if (prevDPoint.data.height_m > apogee) {
    apogee = prevDPoint.data.height_m;
  }

  if (recordingData) {
    if (dataPointIndex >= FLUSH_COUNT) {
      for (int i = 0; i<FLUSH_COUNT; i++) {
          dataPoints[i].printTo(myFile);
      }
      myFile.flush();
      dataPointIndex = 0;
    }

    dataPoints[dataPointIndex] = prevDPoint;
    dataPointIndex++;
  }

}

float deltaHToFullStop() {
  float velocity = prevDPoint.data.velocity_m;
  float dih = dhServoOpening(velocity);
  velocity += VELOCITY_GAIN_FROM_SPRINGS_M;
  dih += dhParachuteUnfurling(velocity);
  dih += dhParachuteOpened(velocity);
  return dih;
}

float dhServoOpening(float &velocity) {
  float t = SERVO_UNLOCK_DURATION_S;
  float dih = t * velocity + .5 * -9.81 * t * t;
  velocity += t * -9.81;
  return dih;
}

float dhParachuteUnfurling(float &velocity) {
  float t = PARACHUTE_FULL_UNFURL_DURATION_S;
  float dih = t * velocity + .5 * -9.81 * t * t;
  velocity += t * -9.81;
  return dih;
}

float dhParachuteOpened(float velocity) {
  float dih = 0;
  float dt = SIM_DT;
  while (velocity > 0) {
    dih += velocity * dt;
    float accel = -9.81 - (K_DURING_FULL_PARACHUTE / MASS_KG) * velocity * abs(velocity);
    velocity += accel * dt;
  }
  return dih;
}
```
