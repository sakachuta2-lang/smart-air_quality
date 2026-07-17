# smart-air_quality
#include <WiFi.h>
#include <PubSubClient.h>

// ======================
// KONFIGURASI WIFI
// ======================
const char* ssid = "Caa";
const char* password = "wiuwiuwiu";

// ======================
// THINGSBOARD
// ======================
const char* mqtt_server = "thingsboard.cloud";
const char* token = "8BTTUD4Oh12jpZ9uyAKc";

// ======================
// PIN
// ======================
const int mq135Pin = 34;
const int ledPin = 2;

// ======================
// MQTT CLIENT
// ======================
WiFiClient espClient;
PubSubClient client(espClient);

// ======================
// WIFI CONNECT
// ======================
void setup_wifi() {

  Serial.println();
  Serial.print("Menghubungkan ke WiFi : ");
  Serial.println(ssid);

  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println("");
  Serial.println("WiFi Terhubung");
  Serial.print("IP Address : ");
  Serial.println(WiFi.localIP());
}

// ======================
// MQTT RECONNECT
// ======================
void reconnect() {

  while (!client.connected()) {

    Serial.print("Menghubungkan ke ThingsBoard...");

    if (client.connect("ESP32", token, NULL)) {
      Serial.println("Berhasil");
    } else {
      Serial.print("Gagal, rc=");
      Serial.print(client.state());
      Serial.println(" coba lagi 5 detik");
      delay(5000);
    }
  }
}

// ======================
// SETUP
// ======================
void setup() {

  Serial.begin(115200);

  pinMode(ledPin, OUTPUT);

  setup_wifi();

  client.setServer(mqtt_server, 1883);
}

// ======================
// LOOP
// ======================
void loop() {

  if (!client.connected()) {
    reconnect();
  }

  client.loop();

  // ======================
  // BACA MQ135
  // ======================
  int airQuality = analogRead(mq135Pin);

  // ======================
  // ESTIMASI GAS
  // ======================
  float co2 = map(airQuality, 0, 4095, 400, 5000);
  float co  = map(airQuality, 0, 4095, 0, 100);
  float nh3 = map(airQuality, 0, 4095, 0, 50);
  float nox = map(airQuality, 0, 4095, 0, 20);

  // ======================
  // KUALITAS UDARA
  // ======================
  String statusUdara;
  int ledStatus;

  if (airQuality <= 1000) {

    statusUdara = "Baik";
    ledStatus = 0;
    digitalWrite(ledPin, LOW);

  }

  else if (airQuality <= 2000) {

    statusUdara = "Sedang";
    ledStatus = 0;
    digitalWrite(ledPin, LOW);

  }

  else {

    statusUdara = "Buruk";
    ledStatus = 1;
    digitalWrite(ledPin, HIGH);

  }

  // ======================
  // SERIAL MONITOR
  // ======================

  Serial.println("====================================");
  Serial.print("Nilai MQ135 : ");
  Serial.println(airQuality);

  Serial.print("Status Udara : ");
  Serial.println(statusUdara);

  Serial.print("CO2 : ");
  Serial.print(co2);
  Serial.println(" ppm");

  Serial.print("CO : ");
  Serial.print(co);
  Serial.println(" ppm");

  Serial.print("NH3 : ");
  Serial.print(nh3);
  Serial.println(" ppm");

  Serial.print("NOx : ");
  Serial.print(nox);
  Serial.println(" ppm");

  Serial.print("LED : ");
  Serial.println(ledStatus == 1 ? "ON" : "OFF");

  // ======================
  // JSON THINGSBOARD
  // ======================

  String payload = "{";

  payload += "\"air_quality\":";
  payload += airQuality;
  payload += ",";

  payload += "\"status\":\"";
  payload += statusUdara;
  payload += "\",";

  payload += "\"co2\":";
  payload += String(co2);
  payload += ",";

  payload += "\"co\":";
  payload += String(co);
  payload += ",";

  payload += "\"nh3\":";
  payload += String(nh3);
  payload += ",";

  payload += "\"nox\":";
  payload += String(nox);
  payload += ",";

  payload += "\"led\":";
  payload += String(ledStatus);

  payload += "}";

  // ======================
  // KIRIM KE THINGSBOARD
  // ======================

  client.publish("v1/devices/me/telemetry", payload.c_str());

  Serial.println("Data terkirim:");
  Serial.println(payload);

  delay(5000);
}
