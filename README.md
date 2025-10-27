# AGRI-ASSIT 
#include <WiFi.h>
#include <Firebase_ESP_Client.h>
#include <HardwareSerial.h>
#include <ModbusMaster.h>

// ---------------- WiFi ----------------
#define WIFI_SSID     "Hotspot name"
#define WIFI_PASSWORD "00000000"

// ---------------- Firebase ----------------
#define API_KEY "YOUR API KEY"
#define DATABASE_URL "YOUR DATA BASE URL"

FirebaseData fbdo;
FirebaseAuth auth;
FirebaseConfig config;

// ------------------- NPK Sensor (UART2 + RS485) -------------------
HardwareSerial rs485Serial(2);
#define NPK_RXD 16
#define NPK_TXD 17
#define MAX485_DE_RE 4

ModbusMaster node;

// ------------------- pH Sensor (UART1) -------------------
HardwareSerial phSerial(1);
#define PH_RXD 21
#define PH_TXD 22

float pH_offset = 0.0;   // Adjust after calibration
float pHValue = 0.0;

// ------------------- Soil Moisture Sensor -------------------
#define SOIL_PIN 34

int soilMoistureRaw = 0;
int soilMoisturePercent = 0;

// Calibration values for soil sensor
#define SOIL_DRY  3000
#define SOIL_WET  1200

// ------------------- RS485 Transmission -------------------
void preTransmission() {
  digitalWrite(MAX485_DE_RE, 1);
}
void postTransmission() {
  digitalWrite(MAX485_DE_RE, 0);
}

void setup() {
  Serial.begin(115200);

  // --- WiFi ---
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  Serial.print("Connecting to Wi-Fi");
  while (WiFi.status() != WL_CONNECTED) {
    Serial.print(".");
    delay(500);
  }
  Serial.println("\n✅ Connected to Wi-Fi");

  // --- Firebase ---
  config.api_key = API_KEY;
  config.database_url = DATABASE_URL;

  if (Firebase.signUp(&config, &auth, "", "")) {
    Serial.println("✅ Firebase SignUp OK");
  } else {
    Serial.printf("❌ Firebase SignUp Failed: %s\n", config.signer.signupError.message.c_str());
  }

  Firebase.begin(&config, &auth);
  Firebase.reconnectWiFi(true);

  // --- NPK Sensor ---
  pinMode(MAX485_DE_RE, OUTPUT);
  digitalWrite(MAX485_DE_RE, 0);
  rs485Serial.begin(9600, SERIAL_8N1, NPK_RXD, NPK_TXD);
  node.begin(1, rs485Serial);
  node.preTransmission(preTransmission);
  node.postTransmission(postTransmission);

  // --- pH Sensor ---
  phSerial.begin(9600, SERIAL_8N1, PH_RXD, PH_TXD);

  // --- Soil Moisture Sensor ---
  pinMode(SOIL_PIN, INPUT);

  Serial.println("System Ready: Reading pH + NPK + Soil Moisture + Firebase Upload");
}

void loop() {
  float calibrated_pH = 0;
  int nitrogen = 0, phosphorus = 0, potassium = 0;

  // --------- pH Sensor Reading ---------
  if (phSerial.available()) {
    String data = phSerial.readStringUntil('\n');
    int phIndex = data.indexOf("PH:");
    if (phIndex != -1) {
      int commaIndex = data.indexOf(',', phIndex);
      if (commaIndex != -1) {
        String phStr = data.substring(phIndex + 3, commaIndex);
        pHValue = phStr.toFloat();

        calibrated_pH = pHValue + pH_offset;
        if (calibrated_pH < 0) calibrated_pH = 0;
        if (calibrated_pH > 14) calibrated_pH = 14;
      }
    }
  }

  // --------- Soil Moisture Sensor Reading ---------
  soilMoistureRaw = analogRead(SOIL_PIN);
  soilMoisturePercent = map(soilMoistureRaw, SOIL_DRY, SOIL_WET, 0, 100);
  if (soilMoisturePercent < 0) soilMoisturePercent = 0;
  if (soilMoisturePercent > 100) soilMoisturePercent = 100;

  // --------- NPK Sensor Reading ---------
  uint8_t result = node.readHoldingRegisters(0x1E, 3);
  if (result == node.ku8MBSuccess) {
    nitrogen    = node.getResponseBuffer(0);
    phosphorus  = node.getResponseBuffer(1);
    potassium   = node.getResponseBuffer(2);
  } else {
    Serial.print("❌ NPK Read Error: ");
    Serial.println(result);
  }

  // --------- Print to Serial ---------
  Serial.println("-------------------------");
  Serial.print("pH Value: "); Serial.println(calibrated_pH, 2);
  Serial.print("Soil Moisture: "); Serial.print(soilMoisturePercent); Serial.println(" %");
  Serial.print("Nitrogen (N): "); Serial.print(nitrogen); Serial.println(" mg/kg");
  Serial.print("Phosphorus (P): "); Serial.print(phosphorus); Serial.println(" mg/kg");
  Serial.print("Potassium (K): "); Serial.print(potassium); Serial.println(" mg/kg");

  // --------- Push Data to Firebase ---------
  if (Firebase.ready()) {
    Firebase.RTDB.setFloat(&fbdo, "/SensorData/pH", calibrated_pH);
    Firebase.RTDB.setInt(&fbdo, "/SensorData/SoilMoisture", soilMoisturePercent);
    Firebase.RTDB.setInt(&fbdo, "/SensorData/Nitrogen", nitrogen);
    Firebase.RTDB.setInt(&fbdo, "/SensorData/Phosphorus", phosphorus);
    Firebase.RTDB.setInt(&fbdo, "/SensorData/Potassium", potassium);
    Serial.println("✅ Data uploaded to Firebase");
  }

  delay(3000);  // Update interval
}


