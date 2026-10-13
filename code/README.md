#define BLYNK_TEMPLATE_ID   "TMPL3JwGK35rl"
#define BLYNK_TEMPLATE_NAME "smart industries system"
#define BLYNK_AUTH_TOKEN    "*****"

#include <WiFi.h>
#include <BlynkSimpleEsp32.h>
#include <Wire.h>
#include <Adafruit_INA219.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <OneWire.h>
#include <DallasTemperature.h>

// Wi-Fi credentials
char ssid[] = "Mokesh";
char pass[] = "12345678";

// ESP32 pins
#define TEMP_PIN       4
#define VIBRATION_PIN  27
#define BUZZER_PIN     25

// OLED configuration
#define SCREEN_WIDTH   128
#define SCREEN_HEIGHT  64
#define OLED_ADDRESS   0x3C

Adafruit_INA219 ina219;

Adafruit_SSD1306 display(
  SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1
);

OneWire oneWire(TEMP_PIN);
DallasTemperature tempSensor(&oneWire);

// Temperature alert limit
const float HIGH_TEMP_LIMIT = 40.0;

// Timing
const unsigned long VIBRATION_HOLD_MS = 2000;
const unsigned long UPDATE_INTERVAL = 1000;

unsigned long lastVibrationTime = 0;
unsigned long lastUpdate = 0;

// Sensor readings
float temperatureC = 0;
float currentA = 0;
float voltageV = 0;
float powerW = 0;

// System states
bool tempValid = false;
bool vibrationDetected = false;
bool buzzerOn = false;
bool highTemperature = false;

String machineStatus = "DETECTING";

void setup() {
  Serial.begin(115200);

  pinMode(VIBRATION_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);

  digitalWrite(BUZZER_PIN, LOW);

  // Initialize I2C for OLED and INA219
  Wire.begin(21, 22);

  // Initialize INA219
  if (!ina219.begin()) {
    Serial.println("ERROR: INA219 not detected!");

    while (true) {
      digitalWrite(BUZZER_PIN, LOW);
      delay(100);
    }
  }

  // Initialize DS18B20 temperature sensor
  tempSensor.begin();
  tempSensor.setResolution(10);

  // Initialize OLED
  if (!display.begin(
        SSD1306_SWITCHCAPVCC, OLED_ADDRESS)) {
    Serial.println("ERROR: OLED not detected!");

    while (true) {
      digitalWrite(BUZZER_PIN, LOW);
      delay(100);
    }
  }

  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.println("PUMP MONITOR");
  display.println("Connecting WiFi...");
  display.display();

  Serial.println("Connecting to Blynk...");

  Blynk.begin(BLYNK_AUTH_TOKEN, ssid, pass);

  Serial.println("System started.");
}

void loop() {
  Blynk.run();

  // Detect vibration.
  // This assumes the sensor outputs HIGH on vibration.
  if (digitalRead(VIBRATION_PIN) == HIGH) {
    lastVibrationTime = millis();
  }

  vibrationDetected =
      (lastVibrationTime != 0) &&
      (millis() - lastVibrationTime < VIBRATION_HOLD_MS);

  // Update sensor readings every second
  if (millis() - lastUpdate >= UPDATE_INTERVAL) {
    lastUpdate = millis();

    // Read temperature
    tempSensor.requestTemperatures();
    temperatureC = tempSensor.getTempCByIndex(0);

    tempValid =
        temperatureC != DEVICE_DISCONNECTED_C &&
        temperatureC >= -55.0 &&
        temperatureC <= 125.0;

    // Read INA219
    currentA = ina219.getCurrent_mA() / 1000.0;
    voltageV = ina219.getBusVoltage_V();
    powerW = ina219.getPower_mW() / 1000.0;

    // Check temperature limit
    highTemperature =
        tempValid &&
        temperatureC >= HIGH_TEMP_LIMIT;

    // Buzzer: vibration OR high temperature
    buzzerOn = vibrationDetected || highTemperature;

    digitalWrite(
      BUZZER_PIN,
      buzzerOn ? HIGH : LOW
    );

    // UPDATED MACHINE STATUS
    if (!tempValid) {
      machineStatus = "DETECTING";
    }
    else if (vibrationDetected || highTemperature) {
      machineStatus = "ABNORMAL";
    }
    else {
      machineStatus = "NORMAL";
    }

    // SERIAL MONITOR
    Serial.println("\n--- PUMP MONITOR ---");

    if (tempValid) {
      Serial.printf(
        "Temperature: %.1f C\n", temperatureC
      );
    } else {
      Serial.println("Temperature: DETECTING");
    }

    Serial.printf("Current: %.3f A\n", currentA);
    Serial.printf("Voltage: %.2f V\n", voltageV);
    Serial.printf("Power: %.2f W\n", powerW);

    Serial.printf(
      "Vibration: %s\n",
      vibrationDetected ? "DETECTED" : "NORMAL"
    );

    Serial.print("Machine Status: ");
    Serial.println(machineStatus);

    Serial.printf(
      "Buzzer: %s\n",
      buzzerOn ? "ON" : "OFF"
    );

    // OLED DISPLAY
    display.clearDisplay();
    display.setCursor(0, 0);

    display.println("PUMP MONITOR");

    if (tempValid) {
      display.printf(
        "Temp: %.1f C\n", temperatureC
      );
    } else {
      display.println("Temp: DETECTING");
    }

    display.printf("Current: %.2f A\n", currentA);
    display.printf("Voltage: %.1f V\n", voltageV);
    display.printf("Power: %.1f W\n", powerW);

    display.printf(
      "Vibration: %s\n",
      vibrationDetected ? "YES" : "NO"
    );

    display.print("Status: ");
    display.println(machineStatus);

    display.printf(
      "Buzzer: %s\n",
      buzzerOn ? "ON" : "OFF"
    );

    display.display();

    // BLYNK DASHBOARD
    if (Blynk.connected()) {

      // V0: Temperature
      if (tempValid) {
        Blynk.virtualWrite(V0, temperatureC);
      }

      // V1: Vibration status (0 or 1)
      Blynk.virtualWrite(
        V1, vibrationDetected ? 1 : 0
      );

      // V2: Machine status (text)
      Blynk.virtualWrite(V2, machineStatus);

      // V3: Vibration status (0 or 1)
      Blynk.virtualWrite(
        V3, vibrationDetected ? 1 : 0
      );

      // V4: Current
      Blynk.virtualWrite(V4, currentA);

      // V5: Voltage
      Blynk.virtualWrite(V5, voltageV);

      // V6: Power
      Blynk.virtualWrite(V6, powerW);

      // V7: Buzzer status (0 or 1)
      Blynk.virtualWrite(
        V7, buzzerOn ? 1 : 0
      );
    }
  }
}
