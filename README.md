# Portfolio-Brandon-Le
Current project: Embedded DC Motor Control & Monitoring System — In Development

Arduino-based embedded control system for a brushed DC motor, integrating PWM speed control, an OLED user interface, transistor-based motor switching, and motor protection circuitry. The system is being expanded with real-time RPM and temperature monitoring and fault detection.

<img width="1145" height="692" alt="image" src="https://github.com/user-attachments/assets/40ad2dad-25cd-46e8-a117-5e157632004c" />

CODE: 

#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

#define MOTOR_PIN 3
#define POT_PIN A0
#define LED_PIN 9
#define BUTTON_PIN 6

#define DISPLAY_UPDATE_INTERVAL 150  // ms, throttles redraw while motor is ON

bool motorState = false;
int lastDrawnPercent = -1;           // forces first draw
unsigned long lastDisplayUpdate = 0;

void drawEngineOn(int percent) {
  display.clearDisplay();

  display.setTextSize(2);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(10, 10);
  display.println("ENGINE ON");

  display.setTextSize(1);
  display.setCursor(30, 40);
  display.print("POWER: ");
  display.print(percent);
  display.println("%");

  display.display();
}

void drawEngineOff() {
  display.clearDisplay();

  display.setTextSize(2);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 25);
  display.println("ENGINE OFF");
  display.display();
}

void setup() {
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  pinMode(LED_PIN, OUTPUT);
  pinMode(MOTOR_PIN, OUTPUT);

  digitalWrite(MOTOR_PIN, LOW);
  digitalWrite(LED_PIN, LOW);

  display.begin(SSD1306_SWITCHCAPVCC, 0x3C);
  display.clearDisplay();
  display.display();
  display.ssd1306_command(SSD1306_DISPLAYOFF);
}

void loop() {
  int potValue = analogRead(POT_PIN);
  int motorSpeed = map(potValue, 0, 1023, 0, 255);
  int powerPercent = map(potValue, 0, 1023, 0, 100);

  // --- Button handling ---
  if (digitalRead(BUTTON_PIN) == LOW) {
    delay(20); // debounce

    if (digitalRead(BUTTON_PIN) == LOW) {
      motorState = !motorState;
      digitalWrite(LED_PIN, motorState);

      // Screen was off (or mid-cycle) — always make sure it's on before drawing
      display.ssd1306_command(SSD1306_DISPLAYON);

      if (motorState) {
        analogWrite(MOTOR_PIN, motorSpeed);
        drawEngineOn(powerPercent);
        lastDrawnPercent = powerPercent;
        lastDisplayUpdate = millis();
      } else {
        analogWrite(MOTOR_PIN, 0);

        drawEngineOff();
        delay(200); // show "ENGINE OFF" briefly

        display.ssd1306_command(SSD1306_DISPLAYOFF);
      }

      // Wait for release so one press = one toggle
      while (digitalRead(BUTTON_PIN) == LOW) {
        delay(5);
      }
      delay(50);
    }
  }

  // --- Continuous update while motor is running ---
  if (motorState) {
    analogWrite(MOTOR_PIN, motorSpeed);

    unsigned long now = millis();
    bool percentChanged = (powerPercent != lastDrawnPercent);
    bool intervalElapsed = (now - lastDisplayUpdate >= DISPLAY_UPDATE_INTERVAL);

    if (percentChanged && intervalElapsed) {
      drawEngineOn(powerPercent);
      lastDrawnPercent = powerPercent;
      lastDisplayUpdate = now;
    }
  }
}


