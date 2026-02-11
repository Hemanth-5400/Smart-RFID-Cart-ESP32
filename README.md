# Smart-RFID-Cart-ESP32
#include <WiFi.h>
#include <WebSocketsServer.h>
#include <MFRC522.h>
#include <SPI.h>
#include <WebServer.h>
#include "SPIFFS.h"
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <map>

const char* ssid = "*******";
const char* password = "1234567890";

// Hardware pins
#define SS_PIN    5   // RFID SDA
#define RST_PIN   27  // RFID Reset
#define LED_PIN   2   // Built-in LED
#define SDA_PIN   21  // LCD I2C SDA
#define SCL_PIN   22  // LCD I2C SCL

#define DUPLICATE_SCAN_WINDOW 30000  
#define REMOVAL_SCAN_WINDOW 60000    
#define REMOVAL_SCAN_DELAY 10000   
LiquidCrystal_I2C lcd(0x27, 20, 4);
WebServer server(80);
WebSocketsServer webSocket = WebSocketsServer(81);
MFRC522 rfid(SS_PIN, RST_PIN);
bool isScanning = false;
float totalAmount = 0.0;
int itemCount = 0;

std::map<String, std::pair<String, float>> products = {
    {"946C3202", {"Organic Apples", 199.99f}},
    {"FD34973F", {"Whole Grain Bread", 49.99f}},
    {"835EACD9", {"Fresh Milk", 79.99f}}
};
std::map<String, unsigned long> lastScanTimes;

const char index_html[] PROGMEM = R"rawliteral(
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Smart Cart</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
            font-family: 'Poppins', sans-serif;
        }

        body {
            background: linear-gradient(-45deg, #ee7752, #e73c7e, #23a6d5, #23d5ab);
            background-size: 400% 400%;
            animation: gradientBG 15s ease infinite;
            min-height: 100vh;
            display: flex;
            justify-content: center;
            align-items: center;
            padding: 20px;
        }

        @keyframes gradientBG {
            0% { background-position: 0% 50%; }
            50% { background-position: 100% 50%; }
            100% { background-position: 0% 50%; }
        }

        .container {
            max-width: 1000px;
            width: 95%;
            background: rgba(255, 255, 255, 0.95);
            border-radius: 20px;
            padding: 40px;
            box-shadow: 0 10px 30px rgba(0, 0, 0, 0.1);
            backdrop-filter: blur(10px);
        }

        .screen {
            display: none;
            animation: fadeIn 0.5s ease-out;
        }

        .screen.active {
            display: block;
        }

        @keyframes fadeIn {
            from { opacity: 0; transform: translateY(20px); }
            to { opacity: 1; transform: translateY(0); }
        }

        h1, h2 {
            color: #333;
            text-align: center;
            margin-bottom: 30px;
            font-size: 2.5em;
            text-transform: uppercase;
            letter-spacing: 2px;
            animation: glow 2s ease-in-out infinite alternate;
        }

        @keyframes glow {
            from { text-shadow: 0 0 5px #fff, 0 0 10px #fff, 0 0 15px #e73c7e; }
            to { text-shadow: 0 0 10px #fff, 0 0 20px #fff, 0 0 30px #23a6d5; }
        }

        .welcome-text {
            text-align: center;
            color: #666;
            margin-bottom: 40px;
            font-size: 1.2em;
            line-height: 1.6;
        }

        .cart-items {
            margin-bottom: 30px;
            max-height: 400px;
            overflow-y: auto;
            padding-right: 10px;
        }

        .cart-items::-webkit-scrollbar {
            width: 8px;
        }

        .cart-items::-webkit-scrollbar-track {
            background: #f1f1f1;
            border-radius: 4px;
        }

        .cart-items::-webkit-scrollbar-thumb {
            background: #888;
            border-radius: 4px;
        }

        .item {
            display: flex;
            justify-content: space-between;
            align-items: center;
            padding: 15px;
            margin-bottom: 10px;
            background: white;
            border-radius: 10px;
            box-shadow: 0 2px 10px rgba(0, 0, 0, 0.05);
            transition: all 0.3s ease;
            position: relative;
            overflow: hidden;
        }

        .item::before {
            content: '';
            position: absolute;
            left: 0;
            top: 0;
            height: 100%;
            width: 5px;
            background: linear-gradient(45deg, #e73c7e, #23a6d5);
            border-radius: 5px 0 0 5px;
        }

        .item:hover {
            transform: translateX(10px);
            box-shadow: 0 5px 15px rgba(0, 0, 0, 0.1);
        }

        .item.removing {
            animation: removeItem 0.5s ease forwards;
        }

        @keyframes removeItem {
            to {
                transform: translateX(-100%);
                opacity: 0;
            }
        }

        .item-name {
            font-weight: 500;
            color: #333;
            flex-grow: 1;
        }

        .item-price {
            color: #e73c7e;
            font-weight: 600;
            margin-left: 20px;
        }

        .total {
            text-align: right;
            font-size: 1.4em;
            font-weight: 600;
            margin: 20px 0;
            color: #333;
            padding: 20px;
            background: rgba(255, 255, 255, 0.9);
            border-radius: 10px;
            box-shadow: 0 2px 10px rgba(0, 0, 0, 0.05);
        }

        .btn {
            display: block;
            width: fit-content;
            margin: 30px auto;
            padding: 15px 40px;
            background: linear-gradient(45deg, #e73c7e, #23a6d5);
            color: white;
            border: none;
            border-radius: 30px;
            font-size: 1.1em;
            cursor: pointer;
            transition: all 0.3s ease;
            text-transform: uppercase;
            letter-spacing: 1px;
        }

        .btn:hover {
            transform: translateY(-3px);
            box-shadow: 0 5px 15px rgba(0, 0, 0, 0.2);
            background: linear-gradient(45deg, #23a6d5, #e73c7e);
        }

        .qr-container {
            width: 300px;
            height: 300px;
            margin: 30px auto;
            padding: 20px;
            background: white;
            border-radius: 15px;
            box-shadow: 0 5px 15px rgba(0, 0, 0, 0.1);
            transition: transform 0.3s ease;
        }

        .qr-container:hover {
            transform: scale(1.05);
        }

        #qrCode {
            width: 100%;
            height: 100%;
        }

        .connection-status {
            position: fixed;
            top: 20px;
            right: 20px;
            padding: 10px 20px;
            border-radius: 20px;
            font-size: 0.9em;
            color: white;
            background: #333;
            transition: all 0.3s ease;
            z-index: 1000;
        }

        .connection-status.connected {
            background: #4CAF50;
            animation: pulse 2s infinite;
        }

        .connection-status.disconnected {
            background: #f44336;
        }

        @keyframes pulse {
            0% { transform: scale(1); }
            50% { transform: scale(1.05); }
            100% { transform: scale(1); }
        }

        @media (max-width: 768px) {
            .container {
                padding: 20px;
            }

            h1 {
                font-size: 2em;
            }

            .qr-container {
                width: 250px;
                height: 250px;
            }
        }
    </style>
</head>
<body>
    <div class="connection-status" id="connectionStatus">Connecting...</div>
    <div class="container">
        <div id="welcomeScreen" class="screen active">
            <h1>Manav Rachna Cart</h1>
            <p class="welcome-text">Welcome to our smart shopping experience. Scan your items and pay seamlessly.</p>
            <button class="btn" onclick="startShopping()">Start Shopping</button>
        </div>

        <div id="shoppingScreen" class="screen">
            <h2>Your Cart</h2>
            <div class="cart-items" id="cartItems"></div>
            <div class="total">Total: ₹<span id="totalAmount">0.00</span></div>
            <button class="btn" onclick="completeShopping()">Proceed to Pay</button>
        </div>

        <div id="paymentScreen" class="screen">
            <h2>Payment</h2>
            <div class="qr-container">
                <canvas id="qrCode"></canvas>
            </div>
            <div class="total">Amount: ₹<span id="paymentAmount">0.00</span></div>
            <button class="btn" onclick="confirmPayment()">Payment Done</button>
        </div>

        <div id="thankYouScreen" class="screen">
            <h2>Thank You!</h2>
            <p class="welcome-text">Your payment was successful. Visit us again!</p>
            <button class="btn" onclick="startShopping()">New Shopping</button>
        </div>
    </div>

    <script>
        const espIp = 'IP_PLACEHOLDER';
        let totalAmount = 0;
        let ws;
        let cartItems = new Map();
        let products = {};

        // Initialize WebSocket
        function connectWebSocket() {
            ws = new WebSocket('ws://' + espIp + ':81');
            
            ws.onopen = () => {
                document.getElementById('connectionStatus').textContent = 'Connected';
                document.getElementById('connectionStatus').classList.add('connected');
                ws.send('GET_PRODUCTS');
            };

            ws.onerror = (error) => {
                console.error("WebSocket Error:", error);
                document.getElementById('connectionStatus').textContent = 'Error';
                document.getElementById('connectionStatus').classList.remove('connected');
            };

            ws.onclose = () => {
                document.getElementById('connectionStatus').textContent = 'Disconnected';
                document.getElementById('connectionStatus').classList.remove('connected');
                setTimeout(connectWebSocket, 2000);
            };

            ws.onmessage = (event) => {
                try {
                    const data = JSON.parse(event.data);
                    
                    if (data.type === 'products') {
                        // Update product list
                        products = data.data;
                        console.log("Updated product list:", products);
                    } 
                    else if (data.type === 'item') {
                        // Handle scanned item based on action
                        if (data.action === 'add') {
                            addItemToCart(data.id + '-' + data.name, data.name, data.price);
                        } else if (data.action === 'remove') {
                            removeItemFromCart(data.id + '-' + data.name, data.price);
                        }
                    }
                } catch (e) {
                    console.log("Raw message:", event.data);
                }
            };
        }

        // Cart Management
        function addItemToCart(key, name, price) {
            cartItems.set(key, { name: name, price: price });
            totalAmount += price;
            updateCartUI(key, name, price, false);
        }

        function removeItemFromCart(key, price) {
            const item = document.getElementById(key);
            if (item) {
                item.classList.add('removing');
                setTimeout(() => {
                    item.remove();
                    cartItems.delete(key);
                    totalAmount -= price;
                    updateTotal();
                }, 500);
            }
        }

        // UI Updates
        function updateCartUI(key, name, price, isRemoving) {
            if (!isRemoving) {
                const cartItemsDiv = document.getElementById('cartItems');
                const item = document.createElement('div');
                item.className = 'item';
                item.id = key;
                item.innerHTML = `
                    <span class="item-name">${name}</span>
                    <span class="item-price">₹${price.toFixed(2)}</span>
                `;
                cartItemsDiv.appendChild(item);
            }
            updateTotal();
        }

        function updateTotal() {
            document.getElementById('totalAmount').textContent = totalAmount.toFixed(2);
            document.getElementById('paymentAmount').textContent = totalAmount.toFixed(2);
        }

        // Screen Navigation
        function showScreen(screenId) {
            document.querySelectorAll('.screen').forEach(screen => {
                screen.classList.remove('active');
            });
            document.getElementById(screenId).classList.add('active');
        }

        // Event Handlers
        function startShopping() {
            showScreen('shoppingScreen');
            ws.send('START_SCAN');
            // Clear cart when starting new session
            cartItems.clear();
            totalAmount = 0;
            document.getElementById('cartItems').innerHTML = '';
            updateTotal();
        }

        function completeShopping() {
            showScreen('paymentScreen');
            ws.send('STOP_SCAN');
            generateQRCode();
        }

        function generateQRCode() {
            const txnId = 'TXN' + Date.now();
            const upiUrl = 'upi://pay?pa=i&pn=MR_Cart&am=' + totalAmount.toFixed(2) + '&tn=' + txnId;
            
            // Clear previous QR code if any
            const qrCanvas = document.getElementById('qrCode');
            qrCanvas.getContext('2d').clearRect(0, 0, qrCanvas.width, qrCanvas.height);
            
            // Generate new QR code
            new QRious({
                element: qrCanvas,
                value: upiUrl,
                size: 200,
                background: 'white',
                foreground: 'black',
                level: 'H'
            });
        }

        function confirmPayment() {
            showScreen('thankYouScreen');
            ws.send('PAYMENT_COMPLETE');
        }

        // Initialize
        window.addEventListener('load', () => {
            connectWebSocket();
            // Include QRious library dynamically
            const script = document.createElement('script');
            script.src = 'https://cdn.jsdelivr.net/npm/qrious@4.0.2/dist/qrious.min.js';
            document.head.appendChild(script);
        });
    </script>
</body>
</html>
)rawliteral";

void updateLCDWelcome() {
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Welcome to MR Cart");
    lcd.setCursor(0, 1);
    lcd.print("Scan items to begin");
    updateLCDLine(2, "Items: 0");
    updateLCDLine(3, "Total: Rs.0.00");
}

void updateLCDLine(uint8_t line, const String &text) {
    lcd.setCursor(0, line);
    lcd.print(text);
    // Clear the rest of the line
    for (uint8_t i = text.length(); i < 20; i++) {
        lcd.print(' ');
    }
}

void handleRoot() {
    String html = FPSTR(index_html);
    html.replace("IP_PLACEHOLDER", WiFi.localIP().toString());
    server.send(200, "text/html", html);
}

void webSocketEvent(uint8_t num, WStype_t type, uint8_t *payload, size_t length) {
    switch (type) {
        case WStype_DISCONNECTED:
            Serial.printf("[%u] Disconnected!\n", num);
            break;
            
        case WStype_CONNECTED: {
            IPAddress ip = webSocket.remoteIP(num);
            Serial.printf("[%u] Connected from %d.%d.%d.%d\n", num, ip[0], ip[1], ip[2], ip[3]);
            
            // Send product list to newly connected client
            String json = "{\"type\":\"products\",\"data\":{";
            for (auto& p : products) {
                json += "\"" + p.first + "\":{\"name\":\"" + p.second.first + 
                        "\",\"price\":" + String(p.second.second) + "},";
            }
            if (products.size() > 0) {
                json.remove(json.length()-1); // Remove trailing comma
            }
            json += "}}";
            webSocket.sendTXT(num, json);
            break;
        }
            
        case WStype_TEXT: {
            String message = (char*)payload;
            message.trim();
            
            if (message == "START_SCAN") {
                isScanning = true;
                itemCount = 0;
                totalAmount = 0;
                lastScanTimes.clear(); // Reset scan times
                updateLCDWelcome();
                Serial.println("Scanning mode activated");
                digitalWrite(LED_PIN, HIGH);
                delay(100);
                digitalWrite(LED_PIN, LOW);
            } 
            else if (message == "STOP_SCAN") {
                isScanning = false;
                updateLCDLine(0, "Scan QR to Pay");
                updateLCDLine(3, "Total: Rs." + String(totalAmount, 2));
                Serial.println("Scanning mode deactivated");
            }
            else if (message == "PAYMENT_COMPLETE") {
                isScanning = false;
                totalAmount = 0;
                itemCount = 0;
                lastScanTimes.clear(); // Reset scan times
                updateLCDWelcome();
                Serial.println("Payment completed, cart reset");
            }
            break;
        }
            
        default:
            break;
    }
}

void handleRFIDScan(String tagID) {
    if (!products.count(tagID)) {
        Serial.println("Unknown product!");
        // Blink LED for unknown product
        for (int i = 0; i < 3; i++) {
            digitalWrite(LED_PIN, HIGH);
            delay(100);
            digitalWrite(LED_PIN, LOW);
            delay(100);
        }
        return;
    }

    auto product = products[tagID];
    unsigned long currentTime = millis();
    String action = "add"; // Default action is to add item
    
    // Check if this item was scanned before
    if (lastScanTimes.count(tagID)) {
        unsigned long lastScanTime = lastScanTimes[tagID];
        unsigned long timeSinceLastScan = currentTime - lastScanTime;
        
        if (timeSinceLastScan > REMOVAL_SCAN_WINDOW) {
            // If scanned after 1 minute, check if scanned again within 10 seconds
            if (timeSinceLastScan < (REMOVAL_SCAN_WINDOW + REMOVAL_SCAN_DELAY)) {
                action = "remove"; // Remove item
            }
            // Else treat as new scan (add item)
        } else if (timeSinceLastScan < DUPLICATE_SCAN_WINDOW) {
            // If scanned within 30 seconds, add another item
            action = "add";
        } else {
            // If scanned between 30 seconds and 1 minute, ignore
            Serial.println("Duplicate scan ignored");
            return;
        }
    }
    
    // Update last scan time
    lastScanTimes[tagID] = currentTime;
    
    // Prepare item data
    String itemJson = "{\"type\":\"item\",\"id\":\"" + tagID + 
                     "\",\"name\":\"" + product.first + 
                     "\",\"price\":" + String(product.second) + 
                     ",\"action\":\"" + action + "\"}";
    
    // Send to all clients
    webSocket.broadcastTXT(itemJson);
    
    // Update LCD and totals
    if (action == "add") {
        itemCount++;
        totalAmount += product.second;
        Serial.print("Added: ");
    } else {
        itemCount--;
        totalAmount -= product.second;
        Serial.print("Removed: ");
    }
    
    Serial.print(product.first);
    Serial.print(" - Rs.");
    Serial.println(product.second);
    
    updateLCDLine(2, "Items: " + String(itemCount));
    updateLCDLine(3, "Total: Rs." + String(totalAmount, 2));
    
    // Visual feedback
    digitalWrite(LED_PIN, HIGH);
    delay(100);
    digitalWrite(LED_PIN, LOW);
}

void setup() {
    Serial.begin(115200);
    while (!Serial); // Wait for serial port to connect
    
    pinMode(LED_PIN, OUTPUT);
    digitalWrite(LED_PIN, LOW);
    
    // Initialize LCD
    Wire.begin(SDA_PIN, SCL_PIN);
    lcd.init();
    lcd.backlight();
    lcd.begin(20, 4);
    updateLCDWelcome();
    
    // Initialize RFID
    SPI.begin();
    rfid.PCD_Init();
    delay(4); // Optional delay for RC522
    Serial.println("RFID Reader Initialized");
    rfid.PCD_DumpVersionToSerial();
    
    // Connect to WiFi
    WiFi.begin(ssid, password);
    Serial.print("Connecting to WiFi");
    int wifiTimeout = 20; // ~10 seconds timeout
    while (WiFi.status() != WL_CONNECTED && wifiTimeout-- > 0) {
        delay(500);
        Serial.print(".");
    }
    
    if (WiFi.status() != WL_CONNECTED) {
        Serial.println("\nWiFi connection failed!");
        updateLCDLine(0, "WiFi Connect Fail");
        updateLCDLine(1, "Check Credentials");
        while (1); // Halt if WiFi fails
    }
    
    Serial.println();
    Serial.print("Connected! IP address: ");
    Serial.println(WiFi.localIP());
    updateLCDLine(1, "IP: " + WiFi.localIP().toString());
    
    // Start servers
    server.on("/", handleRoot);
    server.begin();
    webSocket.begin();
    webSocket.onEvent(webSocketEvent);
    
    Serial.println("System ready");
}

void loop() {
    server.handleClient();
    webSocket.loop();
    
    // Handle RFID scanning
    if (isScanning) {
        // Reset the loop if no new card present
        if (!rfid.PICC_IsNewCardPresent()) {
            return;
        }
        
        // Verify if the NUID has been read
        if (!rfid.PICC_ReadCardSerial()) {
            return;
        }
        
        String tagID = "";
        for (byte i = 0; i < 4; i++) {  // Read all 4 bytes
            tagID += (rfid.uid.uidByte[i] < 0x10 ? "0" : "");
            tagID += String(rfid.uid.uidByte[i], HEX);
        }
        tagID.toUpperCase();
        
        Serial.print("Scanned RFID: ");
        Serial.println(tagID);
        
        handleRFIDScan(tagID);
        
        // Halt PICC
        rfid.PICC_HaltA();
        // Stop encryption on PCD
        rfid.PCD_StopCrypto1();
        delay(300);
    }
}
