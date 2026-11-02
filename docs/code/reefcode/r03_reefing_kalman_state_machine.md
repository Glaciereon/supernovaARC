# r03: reefing kalman state machine

```cpp
//r03: swapped the timer for the barometer and a kalman filter, with a state machine that waits for the descent to start and then winches the line in or out to land at 37.5 s, utilizing code from supernova. also logs velocity and closing time to eeprom and has a ground menu to tune the servo and read the flight back

//plan:
/*
kalman filter will continue to be height and velocity
state machine cause it's convenient, having a state for reading the eeprom correctly. also program a wireless and wired version
if it's convenient won't be afraid to make a compression algorhythm
https://chatgpt.com/c/6a7d3553-1110-83ea-9b4f-e1a62be49cf9 -- if you want to use flash (it's hard)
otherwise choose wisely how to you want to read and write to the eeprom
*/

// What this program does:
/*
boot up
if height > 10 meters gets ready to listen to velocity readings
once velocity is stable and negative,  then start the reefing algorithm
when you hit the ground blink success!!
*/

// operation instructions
/*
it doesn't know the position of the servo. you have to tune it manually before and after.
if untuned it might tighten the string to point of breaking
*/

//servo rotating is also blocking. there is a way to make it not but  . . . yah

#include <Wire.h>
#include <SPI.h>
#include <Adafruit_Sensor.h>
#include "Adafruit_BMP3XX.h"
#include <Servo.h>
#include <BasicLinearAlgebra.h>
#include <EEPROM.h>

using namespace BLA;

Adafruit_BMP3XX bmp;
Servo myServo;

struct DataPacket {
  float height_m = 0;
  float velocity_m = 0;
  float rawHeight_m = 0;
};

struct DataPoint {
  float time_s;
  DataPacket data;
};

#define SERVO_PIN (4)
#define ONBOARD_LED (13)
#define SEALEVELPRESSURE_HPA (1013.25)
#define BASE_VARIANCE (.06) //variance in the height data
#define ACCELERATION_UNCERTAINTY (10) //idk really 

#define WINDOW_LENGTH (10)
#define USING_CABLE_BOOL (1)
#define LAUNCH_HEIGHT_M (5)
#define APOGEE_CUTOFF_M (10)
#define VELOCITY_CUTOFF (.5)
#define STD_CUTOFF (.8)
#define TARGET_TIME (37.5)
#define TIGHTEN_SERVO_COMMAND (180)
#define LOOSEN_SERVO_COMMAND (0)
#define SERVO_DELAY_MS (100)
#define CRASH_VELOCITY (15)
#define CLOSING_BOUND_S (10)
#define SLOWING_WINDOW_S (4)

byte state = 1;
float dataRecordingTimeInit_s = .0067;
float height0M = 0;
float apogee = 0;
DataPoint prevDPoint;
float velocities[WINDOW_LENGTH];
byte velocityIndex = 0;
int EEPROM_Index = 0;
float closingTime_s = 0;

Matrix<2,1> State;
Matrix<2,2> sensorCovarience;
Matrix<2,2> Transition;
Matrix<2,2> ModelVarience;
Matrix <1,2> Measurement;
Matrix <1,1> R;
Matrix <2,1> KalmanGain;
Matrix <2,2> I;

enum States {onGround = 1, preLaunch = 2, preReef = 3, reefing = 4, crashing = 5}; //on ground means the led is blkinking

void setup() {
  EEPROM_Index += sizeof(apogee) + sizeof(float);
  
  prevDPoint.time_s = 0;
  State = {0,0};
  sensorCovarience = {1, 0, 0, 1};
  Transition = {1, .02, 0, 1};
  Measurement = {1, 0};
  ModelVarience = {1, 0, 0, 1};
  R = {BASE_VARIANCE};
  I = {1, 0, 0, 1};
  
  Serial.begin(115200);

  Serial.println("Program start");
  Serial.print(" start and stop running the reefing algorithm at ");
  Serial.print(LAUNCH_HEIGHT_M);
  Serial.println(" meters\n");

  pinMode(ONBOARD_LED, OUTPUT);
  setupServo();
  setupSensor();

  if (!USING_CABLE_BOOL) {
    for (int i = 0; i < 5; i++) {
      blink_LED();
    }

    state = preLaunch;
  }

  Serial.println("\nSetup complete, gl\n\n");
}

void loop() {
 static unsigned long nextSample_us = micros();
  if (micros() >= nextSample_us) {
    nextSample_us += 30000; //30 ms
    switch (state) {
      case onGround:
        onGroundCycle();
        break;
      case preLaunch:
        preLaunchCycle();
        break;
      case preReef:
        preReefCycle();
        break;
      case reefing:
        reefingCycle();
        break;
      case crashing:
        crashingCycle();
        break;
    }
  }
}

void onGroundCycle() {
  Serial.println("1 - adjust servo pos");
  Serial.println("2 - reinitialize height_0");
  Serial.println("3 - Read last flight's EEPROM data");
  Serial.println("4 - primeRocket");
  Serial.println("5 - goon of course\n");

  int answer = scanner().toInt();
  switch (answer) {
    case 1:
      adjustServoPos();
      break;
    case 2:
      initializeHeight0M();
      break;
    case 3:
      readFlightDetails();
      break;
    case 4: //this means that all previous flights are scrapped.
      state = preLaunch;
      break;
    case 5:
      Serial.println("boi");
      break;
    case 99998888:
      for (int i = 0; i < EEPROM.length(); i++) {
        EEPROM.update(i,0);
      }
      Serial.println("deletion successfull");
      break;
  }

  blink_LED();
}
/*
float apogee, float flight duration, alternating pairs of byte velocity and byte net time the servo was closing 
*/
void readFlightDetails() { //chat cooked on this one
  int adressTemp = 0;
  float apogeeTemp;
  EEPROM.get(adressTemp, apogeeTemp);
  adressTemp += sizeof(float);
  float flightDuration;
  EEPROM.get(adressTemp, flightDuration);
  adressTemp += sizeof(float);

  Serial.println("===== FLIGHT DATA =====");
  Serial.print("Apogee: ");
  Serial.println(apogeeTemp);
  Serial.print("Flight duration: ");
  Serial.print(flightDuration);
  Serial.println(" s");
  Serial.println();
  Serial.println("Velocity / Servo Closing Time:");
  Serial.println("Velocity\tNet Time");

  while (adressTemp + 1 < EEPROM.length()) {

    int8_t  velocityTemp;
    int8_t  netTime;

    EEPROM.get(adressTemp, velocityTemp);
    adressTemp += sizeof(int8_t );

    EEPROM.get(adressTemp, netTime);
    adressTemp += sizeof(int8_t );

    Serial.print(velocityTemp);
    Serial.print("\t\t");
    Serial.println(netTime);
  }

  Serial.println();
  Serial.println("===== END OF EEPROM =====");
}

void adjustServoPos() {
  Serial.println("1 - Tighten for .25s");
  Serial.println("2 - Loosen for .25s");
  Serial.println("3 - done");

  int answer = 0;

  while (answer != 3) {
    answer = scanner().toInt();
    if (answer == 1){
      myServo.write(TIGHTEN_SERVO_COMMAND);
      delay(1000);
      myServo.write(90);
    } else if (answer == 2) {
      myServo.write(LOOSEN_SERVO_COMMAND);
      delay(1000);
      myServo.write(90);
    }
  }

}

void blink_LED() {
  digitalWrite(ONBOARD_LED, HIGH);
  delay(500);
  digitalWrite(ONBOARD_LED, LOW);
  delay(500);
}

void preLaunchCycle() {
  updateMeasurements();

  if (prevDPoint.data.height_m > LAUNCH_HEIGHT_M) {
    state = preReef;
    dataRecordingTimeInit_s = prevDPoint.time_s;
    return;
  }

}

void preReefCycle() {
  updateMeasurements();
  
  if (prevDPoint.data.height_m > apogee) {
    apogee = prevDPoint.data.height_m;
  }

  bool heightCutoff = prevDPoint.data.height_m + APOGEE_CUTOFF_M < apogee;
  bool velocityCutoff = (getAverageVelocity() < (-1 * VELOCITY_CUTOFF)) && (getVelocity_STD() < STD_CUTOFF);
  if (heightCutoff && velocityCutoff) {
    state = reefing;
  }
}

float getAverageVelocity() {
  float sum = 0;
  for (int i = 0; i < WINDOW_LENGTH; i++) {
    sum += velocities[i];
  }
  return sum/WINDOW_LENGTH;
}

float getVelocity_STD() {
  float avg = getAverageVelocity();
  float diffSum = 0;
  for (int i = 0; i < WINDOW_LENGTH; i++) {
    diffSum += (avg - velocities[i]) * (avg - velocities[i]);
  }
  return sqrt(diffSum/WINDOW_LENGTH);
}

void reefingCycle() {
  updateMeasurements();
  
  if (close() && (closingTime_s < CLOSING_BOUND_S)) {
    myServo.write(TIGHTEN_SERVO_COMMAND);
    closingTime_s += float(SERVO_DELAY_MS)/1000.0f;
  } else if (closingTime_s > 0) {
    myServo.write(LOOSEN_SERVO_COMMAND);
    closingTime_s -= float(SERVO_DELAY_MS)/1000.0f;
  }
  unsigned long time_temp = millis(); //eeprom writing is slow
  if (EEPROM_Index < EEPROM.length()-2*sizeof(byte)) {
    EEPROM.put(EEPROM_Index, int8_t ((prevDPoint.data.velocity_m)));
    EEPROM_Index += sizeof(int8_t );
    EEPROM.put(EEPROM_Index, int8_t(closingTime_s));
    EEPROM_Index += sizeof(int8_t );
  }
  
  while (time_temp + SERVO_DELAY_MS > millis()) {
    delay(5);
  }
  
  if (abs(prevDPoint.data.velocity_m) > abs(CRASH_VELOCITY) && prevDPoint.data.height_m < SLOWING_WINDOW_S * abs(CRASH_VELOCITY)) { //how many seconds are we giving it to slow down
    state = crashing;
  }

  if (prevDPoint.data.height_m < LAUNCH_HEIGHT_M) {
    state = crashing;
  }
}

void crashingCycle() {
  updateMeasurements();
  if (closingTime_s > 0) {
      myServo.write(LOOSEN_SERVO_COMMAND);
      closingTime_s -= (float)SERVO_DELAY_MS/1000.0f;
      delay(SERVO_DELAY_MS);
      myServo.write(90);
  }

  if (prevDPoint.data.height_m < LAUNCH_HEIGHT_M) {
    EEPROM.put(0, apogee);
    EEPROM.put(0 + sizeof(apogee),float(float(millis())/1000.0f - dataRecordingTimeInit_s));
    while (closingTime_s > 0) {
      myServo.write(TIGHTEN_SERVO_COMMAND);
      delay(SERVO_DELAY_MS);
      closingTime_s -= float(SERVO_DELAY_MS)/1000.0f;
    }
    state = onGround;
  }
}

String scanner() {
  while (Serial.available() == 0) {
    delay(5);
  }

  String response = Serial.readStringUntil('\n');
  response.trim();
  return response;
}

float getHeight() {
  if (!bmp.performReading()) {
    return prevDPoint.data.rawHeight_m;
  }
  return bmp.readAltitude(SEALEVELPRESSURE_HPA) - height0M;
}

void setupServo() {
  myServo.attach(SERVO_PIN);
  myServo.write(90);
  Serial.println("Servo attached. Speed: ");
  Serial.println(myServo.read());
}

void setupSensor() {
  Serial.println("Setting up sensor: ");
  Serial.println("Adafruit BMP390 test");

  bool bmp_I2C_init = bmp.begin_I2C();
  Serial.println("BMP390 I2C initialization: ");
  Serial.println(bmp_I2C_init ? "Doing the most" : "FAIL: Could not find a valid BMP3 sensor, check wiring!");

  bmp.setTemperatureOversampling(BMP3_OVERSAMPLING_4X);
  bmp.setPressureOversampling(BMP3_OVERSAMPLING_4X);
  bmp.setIIRFilterCoeff(BMP3_IIR_FILTER_COEFF_3);
  bmp.setOutputDataRate(BMP3_ODR_50_HZ);

  bool bmp_read = bmp.performReading();
  Serial.println("Sensor read: ");
  Serial.println(bmp_read ? "SUCCESS" : "FAIL: BMP390 not producing data");

  int avgSampleCnt = 20;
  Serial.println("Warming and clearing sensor for ");
  Serial.println(2*avgSampleCnt*30);
  Serial.println(" ms");
  
  for (int i = 0; i<avgSampleCnt; i++) {
    bmp.performReading();
    height0M = bmp.readAltitude(SEALEVELPRESSURE_HPA);
    delay(30);
  } 
  height0M = 0.0f; //lmao 😒
    
  for (int i = 0; i<avgSampleCnt; i++) {
    bmp.performReading();
    height0M += (1.0f/avgSampleCnt) * bmp.readAltitude(SEALEVELPRESSURE_HPA);
    delay(30);
  }
  Serial.println(height0M);
  
  Serial.println("Sample reading:");  delay(5);
  Serial.println("-------------------------- Initial Sample");delay(5);
  Serial.println("Temperature: ");delay(5);
  Serial.println((int)bmp.temperature);delay(5);
  Serial.println(" C");delay(5);
  Serial.println("Pressure: ");delay(5);
  Serial.println((int)(bmp.pressure / 100.0));delay(5);
  Serial.println(" hPa");delay(5);
  Serial.println("Altitude (std atm): ");delay(5);
  Serial.println((int)bmp.readAltitude(SEALEVELPRESSURE_HPA));delay(5);
  Serial.println(" m");delay(5);
  Serial.println("-------------------------------");delay(5);
}

void initializeHeight0M() {
  Serial.println("Old height_0: ");
  Serial.println(height0M);
  height0M = 0;

  int avgSampleCnt = 20;
  for (int i = 0; i<avgSampleCnt; i++) { // gets average of og height
    bmp.performReading();
    height0M += (1.0f/avgSampleCnt) * bmp.readAltitude(SEALEVELPRESSURE_HPA);
    delay(30);
  }

  Serial.println("New height_0: ");
  Serial.println(height0M);
}

//you fill up the window from the top down
void updateMeasurements() {
  float h = getHeight();
  updateMatrices(h, (float)millis()/1000.0f - dataRecordingTimeInit_s);
  if (velocityIndex == WINDOW_LENGTH) {
    velocityIndex = 0;
  }
  velocities[velocityIndex] = prevDPoint.data.velocity_m;
  velocityIndex++;
}

void updateMatrices(float rawHeight, float timeFromLaunch) {
  float dt = max(timeFromLaunch - prevDPoint.time_s, .01f);
  float dt2 = dt * dt;
  float dt3 = dt2 * dt;
  float dt4 = dt3 * dt;
  Transition(0,1) = dt;
  ModelVarience = {
    dt4/4.0 * ACCELERATION_UNCERTAINTY * ACCELERATION_UNCERTAINTY,  dt3/2.0 * ACCELERATION_UNCERTAINTY * ACCELERATION_UNCERTAINTY,
    dt3/2.0 * ACCELERATION_UNCERTAINTY * ACCELERATION_UNCERTAINTY,  dt2     * ACCELERATION_UNCERTAINTY * ACCELERATION_UNCERTAINTY};
  Matrix <1,1> MeasurementPoint = rawHeight;
  State = Transition * State;
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

}

bool close() {
  float timeToTouchDown = prevDPoint.data.height_m/abs(prevDPoint.data.velocity_m);
  float idealTimeToTouchDown = TARGET_TIME - ((float)millis()/1000.0f - dataRecordingTimeInit_s);
  if (timeToTouchDown > idealTimeToTouchDown) {
    return true;
  }
  return false;
}
```
