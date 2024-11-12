#include <WiFi.h>
#include <DHT.h>
#include <ArduinoJson.h>
#include <PubSubClient.h>


// Defined IoT connection informations
#define MQTT_HOST "<Replace with the address of your MQTT broker>"
#define MQTT_PORT 1883
#define MQTT_DEVICEID "d:hwu.b31ot.h00455187"
#define MQTT_USER ""   // N/A
#define MQTT_TOKEN ""  // N/A
#define MQTT_TOPIC "d:hwu.b31ot.h00455187/evt/status/fmt/json"
#define MQTT_TOPIC_INTERVAL "d:hwu.b31ot.h00455187/cmd/report_interval/fmt/json"
#define MQTT_TOPIC_LED "d:hwu.b31ot.h00455187/cmd/led"

// GPIO pins used to connect devices
#define RED_PIN 33    // GPIO pin the red line of RGB LED is connected to
#define GREEN_PIN 26  // GPIO pin the green line of RGB LED is connected to
#define BLUE_PIN 25   // GPIO pin the blue line of RGB LED is connected to
#define DHT_PIN 4     // GPIO pin the data line of the DHT sensor is connected to

// DHT11 Sensor
#define DHTTYPE DHT11

// Define PWM properties for LED
const int freq = 12000;    // PWM frequency
const int resolution = 8;  // Use 8-bits resolution (0-255)

// Temperatures to set LED by (assume temp in C)
#define ALARM_COLD 0.0
#define ALARM_HOT 30.0
#define WARN_COLD 10.0
#define WARN_HOT 25.0

// Add WiFi connection information
char ssid[] = "XXXXX";     // Network SSID (name)
char pass[] = "XXXXX";  // Network password

// DHT11 Assignment
DHT dht(DHT_PIN, DHTTYPE);

// Reporting interval variable (default 10 seconds)
unsigned long reportingInterval = 10000;
unsigned long lastReportTime = 0;

// MQTT objects
void callback(char* topic, byte* payload, unsigned int length);
WiFiClient wifiClient;
PubSubClient mqtt(MQTT_HOST, MQTT_PORT, callback, wifiClient);

// variables to hold data
StaticJsonDocument<100> jsonDoc;
JsonObject payload = jsonDoc.to<JsonObject>();
JsonObject status = payload.createNestedObject("d");
static char msg[50];

//Temperature and Humidity Variables
float h = 0.0;
float t = 0.0;

void callback(char* topic, byte* payload, unsigned int length) {
  payload[length] = 0;
  StaticJsonDocument<100> doc;
  DeserializationError error = deserializeJson(doc, payload);
  Serial.print(topic);
  Serial.println((char*)payload);
  if (!error) {
    // Read brightness values for each LED color
    int redValue = doc["red"] | 0;
    int greenValue = doc["green"] | 0;
    int blueValue = doc["blue"] | 0;

    // Write colour brightness levels to each channel
    ledcWrite(RED_PIN, redValue);
    ledcWrite(GREEN_PIN, greenValue);
    ledcWrite(BLUE_PIN, blueValue);
    // Interval Update
    if (strcmp(topic, MQTT_TOPIC_INTERVAL) == 0) {
      int new_interval = doc["Interval"];
      reportingInterval = new_interval * 1000;
    }
  } else {
    Serial.println("Error parsing JSON");
  }
}

void setup() {
  // Initialize LEDC for RGB
  ledcAttach(RED_PIN, freq, resolution);  // 12 kHz PWM, 8-bit resolution
  ledcAttach(GREEN_PIN, freq, resolution);
  ledcAttach(BLUE_PIN, freq, resolution);

  // Start serial console
  Serial.begin(115200);
  Serial.setTimeout(2000);
  while (!Serial) {}
  Serial.println();
  Serial.println("ESP32 Sensor Application");

  // Start WiFi connection
  WiFi.begin(ssid, pass);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("");
  Serial.println("WiFi Connected");

  // Start connected devices
  dht.begin();

  // Connect to MQTT broker
  if (mqtt.connect(MQTT_DEVICEID, MQTT_USER, MQTT_TOKEN)) {
    Serial.println("MQTT Connected");
    mqtt.subscribe(MQTT_TOPIC_INTERVAL);
    mqtt.subscribe(MQTT_TOPIC_LED);
  }
}

void loop() {
  mqtt.loop();
  while (!mqtt.connected()) {
    Serial.print("Attempting MQTT connection...");
    // Attempt to connect
    if (mqtt.connect(MQTT_DEVICEID, MQTT_USER, MQTT_TOKEN)) {
      Serial.println("MQTT Connected");
      mqtt.subscribe(MQTT_TOPIC_INTERVAL);
      mqtt.subscribe(MQTT_TOPIC_LED);
      mqtt.loop();
    } else {
      Serial.println("MQTT Failed to connect!");
      delay(5000);
    }
  }

  unsigned long currentTime = millis();
  if (currentTime - lastReportTime >= reportingInterval) {
    h = dht.readHumidity();
    t = dht.readTemperature();  // Temperature in degree celcius

    // Check if any reads failed and exit early
    if (isnan(h) || isnan(t)) {
      Serial.println("Failed to read from DHT sensor!");
    } else {
      // Set RGB LED Colour based on temp
      if (t < ALARM_COLD) {
        // Blue for cold
        ledcWrite(RED_PIN, 0);     // Red off
        ledcWrite(GREEN_PIN, 0);   // Green off
        ledcWrite(BLUE_PIN, 255);  // Blue on
      } else if (t >= ALARM_HOT) {
        // Red for hot
        ledcWrite(RED_PIN, 255);  // Red on
        ledcWrite(GREEN_PIN, 0);  // Green off
        ledcWrite(BLUE_PIN, 0);   // Blue off
      } else {
        // Green for comfortable
        int greenBrightness = map(t, ALARM_COLD, WARN_HOT, 0, 255);
        int redBrightness = map(t, WARN_HOT, ALARM_HOT, 255, 0);
        ledcWrite(RED_PIN, redBrightness);
        ledcWrite(GREEN_PIN, greenBrightness);
        ledcWrite(BLUE_PIN, 0);  // Blue off
      }

      // Send data to MQTT
      status["temp"] = t;
      status["humidity"] = h;
      serializeJson(jsonDoc, msg, 50);
      Serial.println(msg);
      mqtt.publish(MQTT_TOPIC, msg);
      if (!mqtt.publish(MQTT_TOPIC, msg)) {
        Serial.println("MQTT Publish failed");
      }
    }
    lastReportTime = currentTime;
  }
}
