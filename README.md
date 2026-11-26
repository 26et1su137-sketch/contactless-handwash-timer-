
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define IR_SENSOR 27
#define BUZZER 25
#define RELAY_PIN 23

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET -1

Adafruit_SSD1306 display(
  SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET
);

const unsigned long SOAP_TIME = 3000;
const unsigned long WASH_TIME = 20000;

bool timerRunning = false;
bool finished = false;

unsigned long startTime = 0;
unsigned long pumpStart = 0;
unsigned long lastDisplayUpdate = 0;

// Your tested relay logic
void relayOn() {
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW);
}

void relayOff() {
  pinMode(RELAY_PIN, INPUT);
}

void showReady() {
  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(2);
  display.setCursor(10, 18);
  display.println("SYSTEM");
  display.println("READY");
  display.display();
}

void beepComplete() {
  for (int i = 0; i < 3; i++) {
    digitalWrite(BUZZER, LOW);
    delay(300);
    digitalWrite(BUZZER, HIGH);
    delay(300);
  }
}

void setup() {
  Serial.begin(115200);

  pinMode(IR_SENSOR, INPUT);
  pinMode(BUZZER, OUTPUT);

  digitalWrite(BUZZER, HIGH);
  relayOff();

  Wire.begin(21, 22);

  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println("OLED not found!");
    while (true) {
      relayOff();
      delay(100);
    }
  }

  showReady();
  Serial.println("Handwash System Ready");
}

void loop() {
  int sensorValue = digitalRead(IR_SENSOR);

  // Start cycle when hand is detected
  if (!timerRunning && !finished && sensorValue == LOW) {

    timerRunning = true;

    startTime = millis();
    pumpStart = millis();

    relayOn();

    Serial.println("Hand Detected");
    Serial.println("Soap Dispensing");
  }

  // Stop pump after 3 second
  if (timerRunning &&
      millis() - pumpStart >= SOAP_TIME) {

    relayOff();
    Serial.println("Pump OFF");
  }
  // Run independent 20-second timer
  if (timerRunning) {

    unsigned long elapsed = millis() - startTime;

    if (elapsed >= WASH_TIME) {

      timerRunning = false;
      finished = true;

      relayOff();

      display.clearDisplay();
      display.setTextSize(2);
      display.setCursor(15, 20);
      display.println("TIME UP!");
      display.display();

      Serial.println("Handwash Complete");

      beepComplete();
    }
    else if (millis() - lastDisplayUpdate >= 200) {

      lastDisplayUpdate = millis();

      int remaining = (WASH_TIME - elapsed + 999) / 1000;

      display.clearDisplay();

      display.setTextSize(1);
      display.setCursor(20, 5);
      display.println("HANDWASH TIMER");

      display.setTextSize(3);
      display.setCursor(40, 27);
      display.print(remaining);
      display.println("s");

      display.display();
    }
  }

  // Reset only after hand is removed
  if (finished && sensorValue == HIGH) {

    finished = false;

    showReady();

    Serial.println("Ready for Next Cycle");
  }

  delay(20);
}

