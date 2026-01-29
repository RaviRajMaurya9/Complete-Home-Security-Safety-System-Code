# Complete-Home-Security-Safety-System-Code
An intelligent home safety system that combines fire detection, gas leak monitoring, and automated fire suppression to protect your home from fire hazards and gas-related accidents.

#code


#include <Servo.h>
#include <SoftwareSerial.h>
#define MQ_SENSOR A0      // MQ-2/MQ-5 Gas Sensor (Analog)
#define FIRE_SENSOR 2     // IR Flame Sensor (Digital)
#define RELAY1 3          // Water Pump Relay
#define RELAY2 4          // Exhaust Fan/Additional Device Relay
#define BUZZER 5          // Buzzer for Audible Alert
#define SERVO_PIN 6       // Servo Motor Control
#define GSM_RX 7          // GSM Module RX Pin
#define GSM_TX 8          // GSM Module TX Pin

Servo waterServo;               
SoftwareSerial SIM900(GSM_RX, GSM_TX); 

// Threshold Configuration
int gasThreshold = 300;         
bool emergencyActive = false;    
bool callMade = false;       
unsigned long emergencyStartTime = 0;
const unsigned long EMERGENCY_TIMEOUT = 300000; // 5 minutes timeout

void setup() {
    // Initialize Serial Communication for Debugging
    Serial.begin(9600);
    Serial.println(F("======================================"));
    Serial.println(F("   SMART HOME SECURITY SYSTEM"));
    Serial.println(F("   Initializing..."));
    Serial.println(F("======================================"));
    
    // Initialize GSM Module
    SIM900.begin(9600);
    delay(2000); // Give GSM module time to initialize
    
    // Set Pin Modes
    pinMode(FIRE_SENSOR, INPUT);
    pinMode(RELAY1, OUTPUT);
    pinMode(RELAY2, OUTPUT);
    pinMode(BUZZER, OUTPUT);
    
    // Initialize Servo
    waterServo.attach(SERVO_PIN);
    waterServo.write(0); // Initial position (0 degrees)
    
    // Ensure all outputs are OFF initially
    digitalWrite(RELAY1, LOW);
    digitalWrite(RELAY2, LOW);
    digitalWrite(BUZZER, LOW);
    
    // Test sequence
    performStartupTest();
    
    Serial.println(F("======================================"));
    Serial.println(F("   SYSTEM READY"));
    Serial.println(F("   Monitoring for threats..."));
    Serial.println(F("======================================"));
    Serial.println();
}

// ============================
// MAIN LOOP
// ============================
void loop() {
    // Read Sensor Values
    int gasValue = analogRead(MQ_SENSOR);
    int fireValue = digitalRead(FIRE_SENSOR);
    
    // Display current readings on Serial Monitor
    displaySensorReadings(gasValue, fireValue);
    
    // Check for emergency conditions
    bool gasEmergency = (gasValue > gasThreshold);
    bool fireEmergency = (fireValue == LOW); // LOW = Fire detected
    
    // EMERGENCY HANDLING
    if (gasEmergency || fireEmergency) {
        handleEmergency(gasEmergency, fireEmergency, gasValue);
    } 
    // NORMAL OPERATION
    else {
        handleNormalOperation();
    }
    
    // Check for emergency timeout
    checkEmergencyTimeout();
    
    delay(500); // Main loop delay
}

// ============================
// EMERGENCY HANDLER FUNCTION
// ============================
void handleEmergency(bool gasDetected, bool fireDetected, int gasLevel) {
    if (!emergencyActive) {
        emergencyActive = true;
        emergencyStartTime = millis();
        callMade = false;
        
        Serial.println(F("======================================"));
        Serial.println(F("   EMERGENCY DETECTED!"));
        if (gasDetected) {
            Serial.print(F("   Gas Leak: "));
            Serial.print(gasLevel);
            Serial.println(F(" ppm"));
        }
        if (fireDetected) {
            Serial.println(F("   Fire Detected!"));
        }
        Serial.println(F("======================================"));
    }
    
    // Activate Emergency Systems
    activateEmergencySystems(gasDetected, fireDetected);
    
    // Make Emergency Call (only once per emergency)
    if (!callMade) {
        makeEmergencyCall(gasDetected, fireDetected, gasLevel);
        callMade = true;
    }
}

// ============================
// ACTIVATE EMERGENCY SYSTEMS
// ============================
void activateEmergencySystems(bool gasDetected, bool fireDetected) {
    // Always activate buzzer for audible alert
    tone(BUZZER, 1000, 500);
    delay(100);
    tone(BUZZER, 1500, 500);
    
    if (fireDetected) {
        // Fire-specific response
        digitalWrite(RELAY1, HIGH);    // Activate water pump
        digitalWrite(RELAY2, HIGH);    // Activate additional safety measures
        
        // Servo sweeping for fire fighting
        for (int pos = 0; pos <= 180; pos += 30) {
            waterServo.write(pos);
            delay(200);
            // Pause at critical angles for better coverage
            if (pos == 60 || pos == 120) delay(500);
        }
        for (int pos = 180; pos >= 0; pos -= 30) {
            waterServo.write(pos);
            delay(200);
            if (pos == 120 || pos == 60) delay(500);
        }
    } 
    else if (gasDetected) {
        // Gas leak response - activate ventilation
        digitalWrite(RELAY1, LOW);     // Keep water pump off
        digitalWrite(RELAY2, HIGH);    // Activate exhaust fan
        
        // Servo stays in neutral position
        waterServo.write(90);
        
        // Pulsing buzzer for gas alert
        tone(BUZZER, 800, 300);
        delay(1000);
    }
}

// ============================
// MAKE EMERGENCY CALL
// ============================
void makeEmergencyCall(bool gasDetected, bool fireDetected, int gasLevel) {
    Serial.println(F("   Making Emergency Call..."));
    
    // Prepare call with AT commands
    SIM900.println("AT");
    delay(1000);
    
    // Dial emergency number - REPLACE WITH YOUR NUMBER
    // Format: ATD+[CountryCode][PhoneNumber];
    SIM900.println("ATD+91xxxxxxxx;");
    
    Serial.println(F("   Calling emergency contact..."));
    
    // Let the call ring for 20 seconds
    for (int i = 0; i < 20; i++) {
        Serial.print(".");
        delay(1000);
        
        // Check if emergency is over
        if (!emergencyActive) {
            SIM900.println("ATH"); // Hang up immediately
            Serial.println(F("\n   Emergency resolved, call cancelled."));
            return;
        }
    }
    
    // Hang up after 20 seconds
    SIM900.println("ATH");
    Serial.println(F("\n   Call ended."));
    
    // Log the emergency
    Serial.println(F("   Emergency logged:"));
    if (gasDetected) Serial.println(F("     - Gas Leak Detected"));
    if (fireDetected) Serial.println(F("     - Fire Detected"));
}

// ============================
// NORMAL OPERATION HANDLER
// ============================
void handleNormalOperation() {
    if (emergencyActive) {
        // Emergency just ended
        Serial.println(F("======================================"));
        Serial.println(F("   EMERGENCY RESOLVED"));
        Serial.println(F("   Returning to normal operation..."));
        Serial.println(F("======================================"));
        
        // Deactivate all emergency systems
        digitalWrite(RELAY1, LOW);
        digitalWrite(RELAY2, LOW);
        noTone(BUZZER);
        
        // Return servo to home position
        waterServo.write(0);
        
        // Reset emergency flags
        emergencyActive = false;
        callMade = false;
    }
}

// ============================
// EMERGENCY TIMEOUT CHECK
// ============================
void checkEmergencyTimeout() {
    if (emergencyActive) {
        unsigned long currentTime = millis();
        if (currentTime - emergencyStartTime > EMERGENCY_TIMEOUT) {
            Serial.println(F("   Emergency timeout reached."));
            Serial.println(F("   System resetting..."));
            
            // Force system reset
            digitalWrite(RELAY1, LOW);
            digitalWrite(RELAY2, LOW);
            noTone(BUZZER);
            waterServo.write(0);
            
            emergencyActive = false;
            callMade = false;
        }
    }
}

// ============================
// STARTUP TEST FUNCTION
// ============================
void performStartupTest() {
    Serial.println(F("   Performing System Test..."));
    
    // Test Buzzer
    Serial.println(F("   Testing Buzzer..."));
    tone(BUZZER, 1000, 200);
    delay(300);
    noTone(BUZZER);
    
    // Test Servo
    Serial.println(F("   Testing Servo Motor..."));
    for (int pos = 0; pos <= 180; pos += 45) {
        waterServo.write(pos);
        delay(200);
    }
    waterServo.write(0);
    
    // Test Relays (brief activation)
    Serial.println(F("   Testing Relays..."));
    digitalWrite(RELAY1, HIGH);
    delay(100);
    digitalWrite(RELAY1, LOW);
    digitalWrite(RELAY2, HIGH);
    delay(100);
    digitalWrite(RELAY2, LOW);
    
    // Test GSM Module
    Serial.println(F("   Testing GSM Module..."));
    SIM900.println("AT");
    delay(1000);
    
    Serial.println(F("   System Test Complete!"));
    delay(1000);
}

// ============================
// DISPLAY SENSOR READINGS
// ============================
void displaySensorReadings(int gasValue, int fireValue) {
    static unsigned long lastDisplayTime = 0;
    unsigned long currentTime = millis();
    
    // Update display every 2 seconds to avoid flooding serial
    if (currentTime - lastDisplayTime > 2000) {
        Serial.println(F("--------------------------------------"));
        Serial.print(F("   Gas Level: "));
        Serial.print(gasValue);
        Serial.print(F(" ppm"));
        
        if (gasValue > gasThreshold) {
            Serial.print(F(" [DANGER]"));
        } else if (gasValue > gasThreshold * 0.7) {
            Serial.print(F(" [WARNING]"));
        } else {
            Serial.print(F(" [SAFE]"));
        }
        
        Serial.print(F("   |   Fire Status: "));
        if (fireValue == LOW) {
            Serial.println(F("DETECTED [DANGER]"));
        } else {
            Serial.println(F("CLEAR [SAFE]"));
        }
        
        Serial.print(F("   System Status: "));
        if (emergencyActive) {
            Serial.println(F("EMERGENCY ACTIVE"));
        } else {
            Serial.println(F("NORMAL OPERATION"));
        }
        
        lastDisplayTime = currentTime;
    }
}

// ============================
// ADDITIONAL SAFETY FEATURES
// ============================

/**
 * Manual Override Function
 * Can be called to manually activate/deactivate systems
 */
void manualOverride(bool activate) {
    if (activate) {
        digitalWrite(RELAY1, HIGH);
        digitalWrite(RELAY2, HIGH);
        tone(BUZZER, 1000, 1000);
        waterServo.write(90);
        Serial.println(F("   Manual Override: Systems ACTIVATED"));
    } else {
        digitalWrite(RELAY1, LOW);
        digitalWrite(RELAY2, LOW);
        noTone(BUZZER);
        waterServo.write(0);
        Serial.println(F("   Manual Override: Systems DEACTIVATED"));
    }
}

/**
 * Calibration Function
 * Call this function to calibrate gas sensor
 */
void calibrateGasSensor() {
    Serial.println(F("======================================"));
    Serial.println(F("   GAS SENSOR CALIBRATION"));
    Serial.println(F("   Ensure clean air environment"));
    Serial.println(F("   Starting calibration in 5 seconds..."));
    Serial.println(F("======================================"));
    
    delay(5000);
    
    long total = 0;
    int readings = 100;
    
    Serial.println(F("   Calibrating... Please wait."));
    
    for (int i = 0; i < readings; i++) {
        total += analogRead(MQ_SENSOR);
        delay(10);
        if (i % 10 == 0) Serial.print(".");
    }
    
    int baseline = total / readings;
    gasThreshold = baseline + 100; // Set threshold 100 above baseline
    
    Serial.println();
    Serial.print(F("   Calibration Complete!"));
    Serial.print(F("   Baseline: "));
    Serial.print(baseline);
    Serial.print(F("   Threshold: "));
    Serial.println(gasThreshold);
    Serial.println(F("======================================"));
}
