# r01: servo test menu

```cpp
//r01: first servo test: a serial menu to run the servo for a few seconds or sweep through speeds. it doesn't compile yet (scanner isn't defined, the switch is typed wrong, a few typos) but its the essence of our code

#include <Servo.h>
servo myServo;
void setup() {
  myServo.attach(4);
  
}

void loop() {
  Serial.println("Welcome to glebs program!");
  Serial.println("Would you like too:\n1 - have the servo move in a direction for 3 seconds?\n2 - test all speeds of the servo?");

  int answer = scanner.toInt();
  case (answer) {
    case 1:
    while (answer != -1) {
      Serial.println("give command number, -1 to get out (nahh)");
      answer = scanner.toInt()
      Servo.write(answer);
    }
  case 2:
    Serial.println("commencing speed test. from 1 to 180 in intervalsof 10");
  for (int i = 0; i < 180; i += 10) {
    myServo.write(i);
    delay(500);
  }

      
  }
}
```
