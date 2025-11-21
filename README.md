#include <Servo.h>
#include <TM1637Display.h>

Servo myservo; 

int buttonPin = 4;

const int CLK = 2;
const int DIO = 3;

TM1637Display display(CLK, DIO);

int servingDuration = 0; // duration serving seconds
bool servingInProgress = false; // flag indicate still serving or not
unsigned long servingEndTime = 0; // timestamp when end
bool hasMovedTo0Degrees = false; //zero degrees yet or not

const unsigned long portionDuration = 3;    
const unsigned long positionChangeDuration = 2000;

unsigned long buttonPressTime = 0;
int clickCount = 0;               
const unsigned long shortClickThreshold = 800;
const unsigned long longHoldThreshold = 3000; 

void setup() {
  myservo.attach(5);
  pinMode(buttonPin, INPUT_PULLUP);

  display.setBrightness(0x0f); 
  display.showNumberDec(0, false);

  myservo.write(0); 
}

void loop() {
  int buttonState = digitalRead(buttonPin);
  unsigned long currentTime = millis();

  if (buttonState == LOW) {
    if (buttonPressTime == 0) {
      buttonPressTime = currentTime;
    }

    unsigned long buttonPressDuration = currentTime - buttonPressTime;

    if (buttonPressDuration >= longHoldThreshold) {
      // long hold = release portion (3 seconds)
      servingInProgress = true;
      startServing(portionDuration);
      buttonPressTime = 0;
    }

  } else if (buttonPressTime != 0) {
    unsigned long buttonPressDuration = currentTime - buttonPressTime;

    if (buttonPressDuration >= shortClickThreshold) {
      // Short click detected
      clickCount++;

      if (clickCount == 1) {
        servingDuration = 10;
      } else if (clickCount == 2) {
        servingDuration = 15;
      } else if (clickCount == 3) {
        servingDuration = 20;
      } else {
        servingDuration = 0;
        clickCount = 0;
      }

      if (servingDuration > 0) {
        servingInProgress = true;
        startServing(servingDuration);
        hasMovedTo0Degrees = false;
      }
    }

    buttonPressTime = 0;
  }

  // timer countdown logic
  if (servingInProgress) {
    int remainingTime = (servingEndTime - currentTime) / 1000;
    display.showNumberDec(remainingTime, false);

    if (currentTime >= servingEndTime) {
      stopServing();
      delay(positionChangeDuration);
      myservo.write(0);
      hasMovedTo0Degrees = true;
    }

  } else if (!hasMovedTo0Degrees) {
    myservo.write(0);
  } else {
    myservo.write(90);
  }
}

void startServing(int duration) {
  myservo.write(90);
  display.showNumberDec(duration, false);
  servingEndTime = millis() + (duration * 1000);
}

void stopServing() {
  myservo.write(0); 
  display.showNumberDec(0, false);
  servingDuration = 0;
  servingInProgress = false;
}
