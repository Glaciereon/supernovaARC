# r02: timed servo sequence

```cpp
//r02: actually dropped menu and made fully automatic: the led blinks faster as a timer counts down, then the servo turns for 3 seconds and back, and the rotation time gets saved to eeprom. no sensor yet, it's all just timing - this was all for the mini thing made to test servo rotation that didn't even end up working lol

//EEPROM will hold how many seconds the servo rotated

// delay a certain amount, and then tell when to drop with LED turning off. It will start turning when time to reach terminal velocity has passed, and stop turning when the time to ground from that point has passed

#include <Servo.h>
#include <EEPROM.h>

// ------------------------------------------------------------------
// USER-CONFIGURED TIMES -- edit these two values
// ------------------------------------------------------------------
const unsigned long DELAY_BEFORE_ROTATION_S = 30;  // delay before rotation starts
const unsigned long ROTATION_DURATION_S     = 3;   // assumed time of descent (rotate duration)
const float DELAY_BEFORE_TERMINAL_S = 3;

const byte SERVO_OPEN = 180;
const byte SERVO_CLOSE = 0;

const uint8_t SERVO_PIN = 4;
const uint8_t LED_PIN   = LED_BUILTIN;


// How often (ms) to write elapsed rotation time to EEPROM during rotation.
// EEPROM.put() only rewrites bytes that actually changed, and this interval
// keeps total write count well within the EEPROM's rated endurance.
const unsigned long EEPROM_WRITE_INTERVAL_MS = 250;

Servo myServo;

void setup() {
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);

  myServo.attach(SERVO_PIN);
  myServo.write(90);  // make sure it's stopped

  // ---- Phase 1: pre-rotation delay, LED blinks progressively faster ----
  unsigned long phaseStart = millis();
  unsigned long elapsed = 0;

  while (elapsed < DELAY_BEFORE_ROTATION_S*1000) {
    elapsed = millis() - phaseStart;

    float progress = (float)elapsed / (float)(DELAY_BEFORE_ROTATION_S*1000);
    if (progress > 1.0) progress = 1.0;

    // Blink period shrinks from 500ms (slow) down to 60ms (fast) as we approach rotation.
    unsigned long blinkPeriod = 500UL - (unsigned long)(440.0 * progress);

    digitalWrite(LED_PIN, HIGH);
    delay(blinkPeriod / 2);
    digitalWrite(LED_PIN, LOW);
    delay(blinkPeriod / 2);
  }

  // ---- Phase 2: solid LED for 3 seconds before rotation starts ----
  digitalWrite(LED_PIN, HIGH);
  delay(3000);
  digitalWrite(LED_PIN, LOW);
  delay(DELAY_BEFORE_TERMINAL_S*1000);

  // ---- Phase 3: rotate, logging elapsed time to EEPROM ----
  unsigned long rotationStart = millis();
  unsigned long lastEepromWrite = 0;

  myServo.write(SERVO_OPEN);

  while (true) {
    unsigned long rotElapsed = millis() - rotationStart;
    if (rotElapsed >= ROTATION_DURATION_S*1000) break;

    if (millis() - lastEepromWrite >= EEPROM_WRITE_INTERVAL_MS) {
      EEPROM.put(0, rotElapsed);
      lastEepromWrite = millis();
    }
  }

  // Final write: record the full, exact rotation duration completed.
  EEPROM.put(0, ROTATION_DURATION_S);

  // ---- Phase 4: stop ----
  myServo.write(SERVO_CLOSE);
  delay(ROTATION_DURATION_S*1000);
  digitalWrite(LED_PIN, LOW);
  myServo.write(90);
}

void loop() {
  // One-shot sequence; nothing to repeat.
}
```
