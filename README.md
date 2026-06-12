paste this code in arduino ino and upload it to esp c6 

```
#include <Arduino.h>
#include <BLEDevice.h>
#include <BLEUtils.h>
#include <BLEServer.h>
#include <ESP32Servo.h>
#include <BLE2902.h>

Servo pillGate;

// Certified Hardware Pin Allocations
const int irSensorPin = 7; 
const int servoPin = 6;    
const int buzzerPin = 4;   // OPTIMIZED: Moved to unrestricted GPIO 4 to prevent boot loops

// Unified System UUID Configuration (Perfect match for your Localhost Web App)
#define SERVICE_UUID        "4fafc201-1fb5-459e-8fcc-c5c9c331914b"
#define CHARACTERISTIC_UUID "beb5483e-36e1-4688-b7f5-ea07361b26a8"

// Global Clock & Tracking Variables
int currentHour = 0;
int currentMinute = 0;
unsigned long lastMillis = 0;
unsigned long lastBuzzerMillis = 0;

// Dynamic Matrix Storage Profiles (Up to 6 Doses Per Day)
int activeDosesCount = 2; 
int alarmHours[6]   = {9,  18,  0,  0,  0,  0}; 
int alarmMinutes[6] = {0,   1,  0,  0,  0,  0}; 
bool doseDispensed[6] = {false, false, false, false, false, false};

// Operational State Flags
bool alarmActive = false;
bool buzzerState = false;
int missedDoseCounter = 0;
int minutesElapsedSinceDrop = 0;
bool deviceConnected = false;

BLEServer* pServer = NULL;
BLECharacteristic *pCharacteristic = NULL;

// Forward Declarations of Telemetry Handshakes
void sendDataToApp(String eventStatus);
void executeDispense(int doseIndex);

// BLE Server Connection Event Management Lifecycle
class ServerCallbacks: public BLEServerCallbacks {
    void onConnect(BLEServer* pServer) { 
        deviceConnected = true;
        Serial.println("[BLE] Wireless UI Dashboard Link Active."); 
    }
    void onDisconnect(BLEServer* pServer) { 
        deviceConnected = false;
        Serial.println("[BLE] Connection dropped. Re-engaging advertising..."); 
        delay(500);
        pServer->startAdvertising(); 
    }
};

// Advanced Dynamic Char Parsing Callbacks
class DataCallbacks: public BLECharacteristicCallbacks {
    void onWrite(BLECharacteristic *pCharacteristic) {
        String value = pCharacteristic->getValue();
        if (value.length() == 0) return;

        Serial.print("[BLE Inbound]: ");
        Serial.println(value);

        // 1. Check for incoming Time Sync packet from Dashboard Connection Handshake
        if (value.startsWith("TIME:")) {
             int firstColon = value.indexOf(':');
             int secondColon = value.indexOf(':', firstColon + 1);
             currentHour = value.substring(firstColon + 1, secondColon).toInt();
             currentMinute = value.substring(secondColon + 1).toInt();
             Serial.printf("[RTC Sync] Internal Clock alignment: %02d:%02d\n", currentHour, currentMinute);
        }
        // 2. Loop-driven Token parsing for Dynamic Frequency configuration: "FREQ=4|T1=04:00..."
        else if (value.startsWith("FREQ=")) {
            char buf[128];
            value.toCharArray(buf, sizeof(buf));

            char* token = strtok(buf, "|");
            while (token != NULL) {
                String pair = String(token);
                int separatorIdx = pair.indexOf('=');
                
                if (separatorIdx != -1) {
                    String key = pair.substring(0, separatorIdx);
                    String valuePart = pair.substring(separatorIdx + 1);

                    if (key == "FREQ") {
                        activeDosesCount = valuePart.toInt();
                        if (activeDosesCount > 6) activeDosesCount = 6; // Upper limits security rail
                        Serial.printf("[MATRIX Sync] Dosage Profile: %d doses/day\n", activeDosesCount);
                    } 
                    else if (key.startsWith("T")) {
                        int slotNum = key.substring(1).toInt();
                        if (slotNum >= 1 && slotNum <= 6) {
                            int colonIdx = valuePart.indexOf(':');
                            if (colonIdx != -1) {
                                alarmHours[slotNum - 1] = valuePart.substring(0, colonIdx).toInt();
                                alarmMinutes[slotNum - 1] = valuePart.substring(colonIdx + 1).toInt();
                                doseDispensed[slotNum - 1] = false; // Reset the drop lock for the new time target
                                Serial.printf(" -> Slot #%d synchronized to execution window: %02d:%02d\n", 
                                              slotNum, alarmHours[slotNum - 1], alarmMinutes[slotNum - 1]);
                            }
                        }
                    }
                }
                token = strtok(NULL, "|");
            }
            Serial.println("[MATRIX Sync] Profile registers updated successfully.");
            sendDataToApp("CONFIG_SYNC");
        }
    }
};

void setup() {
  Serial.begin(115200);
  delay(1000);
  Serial.println("Booting Smart Pill Dispenser Core Substack...");
  
  // Initialize Physical Hardware Peripherals
  pinMode(irSensorPin, INPUT);
  pinMode(buzzerPin, OUTPUT);
  digitalWrite(buzzerPin, LOW); // Force absolute silence on startup

  // Allocate Timer Resources for Smooth Servo Trajectory Control
  ESP32PWM::allocateTimer(0);
  pillGate.setPeriodHertz(50); 
  pillGate.attach(servoPin, 500, 2400); 
  pillGate.write(0); // Lock structural trapdoor gate to initial rest point

  // Initialize Wireless BLE RF Architecture
  BLEDevice::init("Tejas_Pill_Dispenser");
  pServer = BLEDevice::createServer();
  pServer->setCallbacks(new ServerCallbacks());

  BLEService *pService = pServer->createService(SERVICE_UUID);
  pCharacteristic = pService->createCharacteristic(
                      CHARACTERISTIC_UUID,
                      BLECharacteristic::PROPERTY_READ   | 
                      BLECharacteristic::PROPERTY_WRITE  | 
                      BLECharacteristic::PROPERTY_NOTIFY |
                      BLECharacteristic::PROPERTY_INDICATE
                    );

  pCharacteristic->setCallbacks(new DataCallbacks());
  pCharacteristic->addDescriptor(new BLE2902()); // Crucial element to let Chrome capture notification packets

  pService->start();
  
  BLEAdvertising *pAdvertising = BLEDevice::getAdvertising();
  pAdvertising->addServiceUUID(SERVICE_UUID);
  pAdvertising->setScanResponse(true);
  
  BLEDevice::startAdvertising();
  lastMillis = millis();
  Serial.println("RF Layer initialized. Awaiting background dashboard connection...");
}

void loop() {
  // Strict 60-Second Software Real Time Clock Emulation
  if (millis() - lastMillis >= 60000) {
    currentMinute++;
    if (currentMinute >= 60) { currentMinute = 0; currentHour++; }
    if (currentHour >= 24) { 
      currentHour = 0; 
      for(int i = 0; i < 6; i++) doseDispensed[i] = false; // Clear full day matrix locks
    }
    lastMillis = millis();
    Serial.printf("[SYSTEM TICK] Real-Time Clock Matrix: %02d:%02d\n", currentHour, currentMinute);

    if (alarmActive) {
      minutesElapsedSinceDrop++;
    }
  }

  // Scanning Scheduled Timeline Array Registers
  for (int i = 0; i < activeDosesCount; i++) {
    if (currentHour == alarmHours[i] && currentMinute == alarmMinutes[i]) {
      if (!doseDispensed[i]) {
        executeDispense(i);
      }
    }
  }

  // Active Alarm Monitor and Telemetry Loop
  if (alarmActive) {
    // IR Sensor Check: Returns HIGH when target object is cleared out of the drop frame
    if (digitalRead(irSensorPin) == HIGH) { 
      alarmActive = false;
      digitalWrite(buzzerPin, LOW); 
      Serial.println("[SUCCESS] Patient extracted cartridge. Closing transaction.");
      sendDataToApp("TAKEN");
    } 
    // Timeout Safety Boundary (30-Minute Upper Fence)
    else if (minutesElapsedSinceDrop >= 30) {
      alarmActive = false;
      digitalWrite(buzzerPin, LOW); 
      missedDoseCounter++;          
      Serial.printf("[TIMEOUT] Cartridge neglected. Missed Count: %d\n", missedDoseCounter);
      sendDataToApp("MISSED");
    }
    // Safe Asynchronous Non-Blocking Buzzer Oscillation
    else {
      if (millis() - lastBuzzerMillis >= 500) {
        lastBuzzerMillis = millis();
        buzzerState = !buzzerState;
        digitalWrite(buzzerPin, buzzerState ? HIGH : LOW);
      }
    }
  }

  delay(20); // Standard execution padding to reduce ambient chip heat
}

void executeDispense(int doseIndex) {
  // Pre-Check: If the tray is already blocked before sweep, abort to prevent cartridge jamming
  if (digitalRead(irSensorPin) == LOW) {
    Serial.println("[CRITICAL ERROR] Mechanical slide blocked. Execution aborted.");
    doseDispensed[doseIndex] = true; 
    missedDoseCounter++; 
    sendDataToApp("BLOCKED");
    return;
  }

  // PRESENTATION TIP: If you want this code version to automatically trigger a demonstration 
  // sweep when clicking "Update Schedule" (instead of waiting for the clock), 
  // you can remove the next two lines from comment tags:
  // alarmActive = true; 
  // return;

  Serial.printf("[ACTUATING] Releasing Cartridge Profile for Slot #%d...\n", doseIndex + 1);
  
  // Sweep out gate to clear channel drop path
  pillGate.write(90); 
  delay(1500); 
  
  // Return actuator back to home anchor point
  pillGate.write(0);
  
  // POWER OVERHEAD DELAY: Let the servo finish movement and let the 
  // voltage rail bounce cleanly back up to 5.38V before powering the buzzer.
  delay(200); 
  
  doseDispensed[doseIndex] = true; 
  alarmActive = true; 
  minutesElapsedSinceDrop = 0; 
  lastBuzzerMillis = millis();
}

void sendDataToApp(String eventStatus) {
  if (!deviceConnected) return;

  // FIXED SCALAR OUTPUT: Added the raw status property string into your exact frame format
  String jsonPayload = "{\"missed\":" + String(missedDoseCounter) + 
                       ",\"bat\":100" + 
                       ",\"status\":\"" + eventStatus + "\"" +
                       ",\"slots\":[";
                         
  for (int i = 0; i < activeDosesCount; i++) {
    // Generate linear array string format matching the index.html parser loops
    jsonPayload += "\"" + String(alarmHours[i] < 10 ? "0" : "") + String(alarmHours[i]) + ":" + 
                          String(alarmMinutes[i] < 10 ? "0" : "") + String(alarmMinutes[i]) + "\"";
    if (i < activeDosesCount - 1) {
        jsonPayload += ",";
    }
  }
  jsonPayload += "]}";

  pCharacteristic->setValue(jsonPayload.c_str());
  pCharacteristic->notify(); 
  Serial.print("[BLE Transmit Telemetry Matrix]: ");
  Serial.println(jsonPayload);
}
```

later run index.html file to connect to app
