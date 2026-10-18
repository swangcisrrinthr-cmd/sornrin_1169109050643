# คู่มือปฏิบัติการ Smart Farm Lab: Docker, ESP32, MQTT, InfluxDB และ Node-RED

> เอกสารนี้เป็นต้นฉบับบทปฏิบัติการ โค้ด `reconnect()` ด้านล่างยังเป็นตัวอย่างเดิมที่ใช้ `while` และ `delay(5000)` สำหรับการทดลองปัจจุบันให้ใช้ `src/main.cpp` และอ่านลำดับการทำงานใน `readme.md` ซึ่งปรับให้ใช้ `loop()` เดียวและ `millis()` เพื่อให้ตรวจปุ่มได้ระหว่างรอลองเชื่อมต่อ MQTT ใหม่

คู่มือนี้ออกแบบมาเพื่อให้นักศึกษาเข้าใจสถาปัตยกรรมระบบ Internet of Things (IoT) ตั้งแต่การสร้าง Server จำลองด้วย Docker, การเขียนโปรแกรมฝังตัวบน ESP32, ไปจนถึงการจัดการข้อมูลและสร้าง Dashboard ด้วย Node-RED

---

## ส่วนที่ 1: การเตรียม Server ด้วย Docker Compose

การจำลอง Server จะใช้โปรแกรม 4 ตัว ได้แก่ Mosquitto (MQTT Broker), InfluxDB (Database), Node-RED (Flow Editor), และ Grafana (Dashboard)

### ขั้นตอนที่ 1: เตรียมไฟล์ตั้งค่า MQTT (mosquitto.conf)

เนื่องจาก Mosquitto เวอร์ชัน 2.0 ขึ้นไป มีการบล็อกการเชื่อมต่อจากภายนอกเป็นค่าเริ่มต้น นักศึกษาจำเป็นต้องสร้างไฟล์ตั้งค่าก่อน โดยสร้างโครงสร้างโฟลเดอร์ดังนี้:

D:\\lab\_docker\\

├── docker-compose.yml

└── std\_lab\_data\\

    └── mosquitto\\

        └── config\\

            └── mosquitto.conf

เปิดไฟล์ `mosquitto.conf` ด้วย Notepad และพิมพ์คำสั่ง บรรทัดนี้ลงไป แล้วกด Save:

\# รับ MQTT ภายใน Container  
listener 1883 0.0.0.0  
protocol mqtt

\# อนุญาตเชื่อมต่อโดยไม่ใช้บัญชี สำหรับ Lab  
allow\_anonymous true

\# เก็บสถานะ MQTT เช่น Retained messages  
persistence true  
persistence\_location /mosquitto/data/

\# แสดง Log และบันทึกลงไฟล์  
log\_dest stdout  
log\_dest file /mosquitto/log/mosquitto.log

*(คำสั่งนี้เป็นการเปิดรับการเชื่อมต่อจากอุปกรณ์ ESP32 และอนุญาตให้เข้าใช้งานได้โดยไม่ต้องใส่รหัสผ่าน)*

### ขั้นตอนที่ 2: สร้างไฟล์ docker-compose.yml

นักศึกษาสามารถเลือกตั้งค่า Port ได้ 2 กรณีตามความเหมาะสมของเครื่องที่ใช้งาน

**กรณีที่ 1: ใช้ Port มาตรฐาน (Standard Ports)** ใช้ในกรณีที่ไม่มีโปรแกรมอื่นรันพอร์ตเหล่านี้อยู่

version: '3.8'

services:

  std-mosquitto:

    image: eclipse-mosquitto:latest

    container\_name: std-mosquitto

    restart: unless-stopped

    ports:

      \- "1883:1883"

    volumes:

      \- ./std\_lab\_data/mosquitto/config:/mosquitto/config

    networks:

      \- std-iot-net

  std-influxdb:

    image: influxdb:2.0

    container\_name: std-influxdb

    ports:

      \- "8086:8086"

    networks:

      \- std-iot-net

  std-nodered:

    image: nodered/node-red:latest

    container\_name: std-nodered

    ports:

      \- "1880:1880"

    networks:

      \- std-iot-net

    depends\_on:

      \- std-mosquitto

      \- std-influxdb

networks:

  std-iot-net:

    driver: bridge

**กรณีที่ 2: เปลี่ยนหมายเลข Port ฝั่ง Host (รันคู่ขนานกับโปรเจกต์เดิม)** ใช้ในกรณีที่ไม่ต้องการปิดคอนเทนเนอร์ของโปรเจกต์อื่นที่รันค้างอยู่

version: '3.8'

services:

  std-mosquitto:

    image: eclipse-mosquitto:latest

    container\_name: std-mosquitto

    ports:

      \- "1884:1883" \# \<--- เปลี่ยน Port ฝั่ง Host เป็น 1884

    volumes:

      \- ./std\_lab\_data/mosquitto/config:/mosquitto/config

    networks:

      \- std-iot-net

  \# ... (ส่วนอื่นตั้งค่าเหมือนกรณีที่ 1 แต่เปลี่ยนตัวเลขพอร์ตหน้าเครื่องหมาย colon)

*(เมื่อเตรียมไฟล์เสร็จแล้ว เปิด Terminal ในโฟลเดอร์นี้และสั่งรันด้วยคำสั่ง `docker-compose up -d`)*

---

## ส่วนที่ 2: โค้ด ESP32 และอัลกอริทึมการทำงาน

โค้ดนี้ใช้สำหรับบอร์ด ESP32 เพื่ออ่านค่า DHT22 (ขา 15), แสดงผลบน LCD, ควบคุม Relay 2 ช่อง (ขา 14, 27\) และมีปุ่มกด (ขา 4\) สำหรับรีเซ็ต WiFi มีการใช้ไลบรารี `ArduinoJson` ในการแพ็กข้อความขาส่งข้อมูล (Telemetry) และตอบกลับคำสั่ง (ACK)

### โค้ดฉบับสมบูรณ์ (C++)

\#include \<WiFi.h\>

\#include \<WiFiManager.h\>

\#include \<PubSubClient.h\>

\#include \<LiquidCrystal\_I2C.h\>

\#include \<DHT.h\>

\#include \<ArduinoJson.h\>

// \=========================================================

// 1\. การตั้งค่าฮาร์ดแวร์ (Hardware Configuration)

// \=========================================================

\#define DHTPIN 15          // ขาเชื่อมต่อเซนเซอร์ DHT22

\#define DHTTYPE DHT22      

\#define WIFI\_RESET\_BTN 4   // ขาสำหรับปุ่มกดเข้าโหมดตั้งค่า WiFi (Active LOW)

\#define RELAY\_PUMP 14      // ขาควบคุมปั๊มน้ำ

\#define RELAY\_VALVE 27     // ขาควบคุมวาล์วน้ำ

DHT dht(DHTPIN, DHTTYPE);

LiquidCrystal\_I2C lcd(0x27, 20, 4);

// \=========================================================

// 2\. การตั้งค่า MQTT และโครงสร้างหัวข้อ (Topic)

// \=========================================================

const char\* mqtt\_server \= "192.168.1.xxx"; // นักศึกษาต้องเปลี่ยนเป็น IP ของ Server 

const int mqtt\_port \= 1883;

WiFiClient espClient;

PubSubClient client(espClient);

String site\_id \= "site\_01";

String zone\_id \= "zone\_A";

String device\_id \= "node\_01"; // เปลี่ยนชื่อกลุ่มให้ไม่ซ้ำกันในห้องเรียน

String base\_topic \= "smart\_farm/" \+ site\_id \+ "/" \+ zone\_id \+ "/" \+ device\_id \+ "/";

unsigned long lastMsg \= 0; 

// \=========================================================

// 3\. ฟังก์ชัน Setup 

// \=========================================================

void setup() {

  Serial.begin(115200);

  

  pinMode(WIFI\_RESET\_BTN, INPUT\_PULLUP);

  

  pinMode(RELAY\_PUMP, OUTPUT);

  pinMode(RELAY\_VALVE, OUTPUT);

  digitalWrite(RELAY\_PUMP, 0); 

  digitalWrite(RELAY\_VALVE, 0);

  lcd.init();

  lcd.backlight();

  lcd.setCursor(0, 0);

  lcd.print("Smart Farm Lab");

  dht.begin();

  WiFiManager wm;

  lcd.setCursor(0, 1);

  lcd.print("Connecting WiFi...");

  

  if (\!wm.autoConnect("SmartFarm\_Setup")) {

    Serial.println("Failed to connect");

    delay(3000);

    ESP.restart(); 

  }

  

  lcd.setCursor(0, 1);

  lcd.print("WiFi Connected\!   ");

  lcd.setCursor(0, 2);

  lcd.print(WiFi.localIP().toString());

  delay(2000);

  lcd.clear();

  client.setServer(mqtt\_server, mqtt\_port);

  client.setCallback(mqttCallback);

}

// \=========================================================

// 4\. ฟังก์ชัน Loop 

// \=========================================================

void loop() {

  if (digitalRead(WIFI\_RESET\_BTN) \== LOW) {

    delay(50); 

    if (digitalRead(WIFI\_RESET\_BTN) \== LOW) {

      lcd.clear();

      lcd.setCursor(0, 0);

      lcd.print("WiFi Config Mode");

      

      WiFiManager wm;

      wm.setConfigPortalTimeout(180); 

      if (\!wm.startConfigPortal("SmartFarm\_Setup")) {

        delay(3000);

        ESP.restart();

      }

      lcd.clear();

    }

  }

  if (\!client.connected()) {

    reconnect();

  }

  client.loop(); 

  unsigned long now \= millis();

  if (now \- lastMsg \> 5000\) {

    lastMsg \= now;

    

    float h \= dht.readHumidity();

    float t \= dht.readTemperature();

    if (isnan(h) || isnan(t)) {

      Serial.println("Failed to read from DHT sensor\!");

      return;

    }

    lcd.setCursor(0, 0);

    lcd.print("T:" \+ String(t, 1\) \+ "C P:" \+ String(digitalRead(RELAY\_PUMP)));

    lcd.setCursor(0, 1);

    lcd.print("H:" \+ String(h, 1\) \+ "% V:" \+ String(digitalRead(RELAY\_VALVE)));

    // แพ็กข้อมูล Telemetry ด้วย ArduinoJson

    StaticJsonDocument\<256\> doc;

    doc\["temperature"\] \= t;

    doc\["humidity"\] \= h;

    doc\["pump\_state"\] \= digitalRead(RELAY\_PUMP);

    doc\["valve\_state"\] \= digitalRead(RELAY\_VALVE);

    

    char msgBuffer\[256\];

    serializeJson(doc, msgBuffer);

    String topic\_telemetry \= base\_topic \+ "telemetry";

    client.publish(topic\_telemetry.c\_str(), msgBuffer);

  }

}

// \=========================================================

// 5\. ฟังก์ชันเชื่อมต่อ MQTT และตั้งค่า LWT (พินัยกรรมสถานะ)

// \=========================================================

void reconnect() {

  while (\!client.connected()) {

    Serial.print("Attempting MQTT connection...");

    

    String topic\_status \= base\_topic \+ "status";

    String willMessage \= "{\\"status\\":\\"offline\\"}"; 

    

    if (client.connect(device\_id.c\_str(), NULL, NULL, topic\_status.c\_str(), 0, true, willMessage.c\_str())) {

      Serial.println("connected");

      

      String topic\_cmd \= base\_topic \+ "cmd";

      client.subscribe(topic\_cmd.c\_str());

      

      client.publish(topic\_status.c\_str(), "{\\"status\\":\\"online\\"}", true);

    } else {

      delay(5000);

    }

  }

}

// \=========================================================

// 6\. ฟังก์ชันรับคำสั่งและส่ง ACK ด้วย ArduinoJson

// \=========================================================

void mqttCallback(char\* topic, byte\* payload, unsigned int length) {

  String messageTemp;

  for (int i \= 0; i \< length; i++) {

    messageTemp \+= (char)payload\[i\];

  }

  

  // แกะข้อมูล JSON ที่รับเข้ามา

  StaticJsonDocument\<256\> doc;

  DeserializationError error \= deserializeJson(doc, messageTemp);

  if (error) return;

  if (doc.containsKey("cmd") && doc.containsKey("state")) {

    String command \= doc\["cmd"\].as\<String\>();

    int state \= doc\["state"\].as\<int\>();

    // สั่งการฮาร์ดแวร์

    if (command \== "pump") {

      digitalWrite(RELAY\_PUMP, state);

    } 

    else if (command \== "valve") {

      digitalWrite(RELAY\_VALVE, state);

    }

    // แพ็กข้อมูลตอบกลับ (ACK) ด้วย ArduinoJson

    StaticJsonDocument\<256\> ackDoc;

    ackDoc\["ack"\] \= command;          

    ackDoc\["state\_set"\] \= state;      

    

    char ackBuffer\[256\];

    serializeJson(ackDoc, ackBuffer); 

    

    String topic\_ack \= base\_topic \+ "ack";

    client.publish(topic\_ack.c\_str(), ackBuffer);

  }

}

### คำอธิบาย Algorithm ของแต่ละ Function ในโปรแกรม

1. **`setup()`:** ทำงานครั้งแรกเมื่อจ่ายไฟ เริ่มต้นจากการกำหนดโหมดอินพุต/เอาต์พุตของขาสัญญาณ เปิดการทำงานจอ LCD และเซนเซอร์ จากนั้นเรียกใช้ฟังก์ชันจากไลบรารี `WiFiManager` เพื่อเชื่อมต่ออินเทอร์เน็ต หากเชื่อมต่อไม่ได้บอร์ดจะเปิดโหมด AP ปล่อย Hotspot เพื่อให้ผู้ใช้ตั้งค่า WiFi ใหม่  
2. **`loop()`:** เป็นฟังก์ชันหลักที่ทำงานวนลูปไม่รู้จบ มี 3 หน้าที่หลัก:  
   * ดักจับการกดปุ่ม (WIFI\_RESET\_BTN) เพื่อเรียก WiFiManager กลับมาตั้งค่าใหม่  
   * ตรวจสอบสถานะ MQTT ตลอดเวลา หากหลุดจะเรียก `reconnect()` และเรียก `client.loop()` เพื่อประมวลผลข้อความเบื้องหลัง  
   * จับเวลาด้วย `millis()` เมื่อครบ 5 วินาที จะอ่านค่าเซนเซอร์ แพ็กข้อมูลด้วย `ArduinoJson` และ Publish ไปที่หัวข้อ `.../telemetry`  
3. **`reconnect()`:** ควบคุมการเชื่อมต่อกับเซิร์ฟเวอร์ MQTT อัลกอริทึมเด่นคือ **LWT (Last Will and Testament)** โดยการสั่งผ่านฟังก์ชันเชื่อมต่อว่า *ถ้าบอร์ดขาดการเชื่อมต่อไปกะทันหัน ให้ Broker ประกาศข้อความพินัยกรรม (offline) แทนบอร์ด* แต่เมื่อเชื่อมต่อสำเร็จ บอร์ดจะทับพินัยกรรมด้วยสถานะ online และ Subscribe รอรับคำสั่งควบคุม  
4. **`mqttCallback()`:** ทำงานเมื่อมีข้อความเข้ามาในหัวข้อ `.../cmd` อัลกอริทึมจะแกะ JSON (Deserialize) เพื่ออ่านค่าสั่งการเปิด/ปิด Relay จากนั้นจะใช้ `ArduinoJson` ในการสร้างข้อความตอบกลับใหม่ (Serialize) และ Publish ส่งไปที่หัวข้อ `.../ack` เพื่อยืนยันว่าบอร์ดทำงานตามคำสั่งแล้ว

---

## ส่วนที่ 3: การตั้งค่า Node-RED และ Dashboard

ก่อนเริ่ม ให้นักศึกษาติดตั้งปลั๊กอิน Dashboard เวอร์ชันคลาสสิก โดยไปที่เมนู **Manage palette** และค้นหาเพื่อติดตั้งแพ็กเกจ `node-red-dashboard`

### อัลกอริทึมของ Node-RED Flow

1. **การรับข้อมูล (MQTT In):** Subscribe หัวข้อ `smart_farm/site_01/zone_A/node_01/#` (รับทุก Topic ย่อยของ node\_01)  
2. **การคัดแยก (Function Node แบบ 3 ขา):**  
   * **ขาที่ 1 (`telemetry`):** นำค่าที่ได้มาจัดฟอร์แมตเป็น Array 2 ก้อน `[fields, tags]` เพื่อส่งไปบันทึกลง InfluxDB  
   * **ขาที่ 2 (`telemetry`):** ส่ง JSON ก้อนเดิมไปแสดงผลบนเกจวัดอุณหภูมิและความชื้น (Dashboard)  
   * **ขาที่ 3 (`status`):** ดึงเฉพาะค่า status (online/offline) เพื่อแสดงสถานะการเชื่อมต่อบนหน้าจอ (Dashboard)  
3. **การสั่งงาน (Switch Node บน Dashboard):** เมื่อผู้ใช้สับสวิตช์ปิด/เปิด Relay สวิตช์จะพ่น JSON payload (เช่น `{"cmd":"pump","state":1}`) เข้าสู่โหนด MQTT Out เพื่อยิงคำสั่งไปที่หัวข้อ `.../cmd` ของบอร์ด ESP32 ทันที

### ชุดโค้ด JSON สำหรับ Import Flow บน Node-RED สมบูรณ์แบบ

นักศึกษาสามารถคัดลอกโค้ดด้านล่างนี้ ไปที่เมนู **Import** ใน Node-RED เพื่อสร้าง Flow แยก 3 ขา พร้อมหน้า Dashboard ให้อัตโนมัติ:

\[{"id":"mqtt\_in","type":"mqtt in","z":"lab\_flow","name":"","topic":"smart\_farm/site\_01/zone\_A/node\_01/\#","qos":"0","datatype":"json","broker":"local\_mqtt","inputs":0,"x":150,"y":160,"wires":\[\["router\_func"\]\]},{"id":"router\_func","type":"function","z":"lab\_flow","name":"Router & Influx Format","func":"if (\!msg.payload || typeof msg.payload \!== 'object') return \[null, null, null\]; \\n\\nlet parts \= msg.topic.split(\\"/\\");\\nlet site \= parts\[1\];  \\nlet zone \= parts\[2\];  \\nlet device \= parts\[3\];\\nlet msg\_type \= parts\[4\]; \\n\\nif (msg\_type \=== \\"telemetry\\") {\\n    let msgInflux \= {\\n        measurement: \\"sensor\_data\\",\\n        payload: \[\\n            {\\n                temperature: parseFloat(msg.payload.temperature),\\n                humidity: parseFloat(msg.payload.humidity),\\n                pump: parseInt(msg.payload.pump\_state || 0),\\n                valve: parseInt(msg.payload.valve\_state || 0)\\n            },\\n            { site: site, zone: zone, device: device }\\n        \]\\n    };\\n    let msgDashboard \= { payload: msg.payload };\\n    return \[msgInflux, msgDashboard, null\];\\n} \\nelse if (msg\_type \=== \\"status\\") {\\n    return \[null, null, { payload: msg.payload.status }\];\\n}\\nreturn \[null, null, null\];","outputs":3,"timeout":"","noerr":0,"initialize":"","finalize":"","libs":\[\],"x":420,"y":160,"wires":\[\["influx\_out"\],\["gauge\_temp","gauge\_hum"\],\["text\_status"\]\]},{"id":"influx\_out","type":"influxdb out","z":"lab\_flow","influxdb":"local\_influx","name":"Save to InfluxDB","measurement":"","precision":"","retentionPolicy":"","database":"database","precisionV18FluxV20":"ms","retentionPolicyV18Flux":"","org":"lab","bucket":"smartfarm","x":710,"y":100,"wires":\[\]},{"id":"gauge\_temp","type":"ui\_gauge","z":"lab\_flow","name":"","group":"dashboard\_group","order":1,"width":0,"height":0,"gtype":"gage","title":"Temperature","label":"°C","format":"{{msg.payload.temperature}}","min":0,"max":"50","colors":\["\#00b500","\#e6e600","\#ca3838"\],"seg1":"","seg2":"","className":"","x":710,"y":160,"wires":\[\]},{"id":"gauge\_hum","type":"ui\_gauge","z":"lab\_flow","name":"","group":"dashboard\_group","order":2,"width":0,"height":0,"gtype":"gage","title":"Humidity","label":"%","format":"{{msg.payload.humidity}}","min":0,"max":"100","colors":\["\#00b500","\#e6e600","\#ca3838"\],"seg1":"","seg2":"","className":"","x":710,"y":200,"wires":\[\]},{"id":"text\_status","type":"ui\_text","z":"lab\_flow","group":"dashboard\_group","order":3,"width":0,"height":0,"name":"","label":"Device Status","format":"{{msg.payload}}","layout":"row-spread","className":"","x":720,"y":260,"wires":\[\]},{"id":"switch\_pump","type":"ui\_switch","z":"lab\_flow","name":"","label":"Pump Control","tooltip":"","group":"dashboard\_group","order":4,"width":0,"height":0,"passthru":true,"decouple":"false","topic":"smart\_farm/site\_01/zone\_A/node\_01/cmd","style":"","onvalue":"{\\"cmd\\":\\"pump\\",\\"state\\":1}","onvalueType":"json","onicon":"","oncolor":"","offvalue":"{\\"cmd\\":\\"pump\\",\\"state\\":0}","offvalueType":"json","officon":"","offcolor":"","animate":false,"className":"","x":160,"y":340,"wires":\[\["mqtt\_out\_cmd"\]\]},{"id":"switch\_valve","type":"ui\_switch","z":"lab\_flow","name":"","label":"Valve Control","tooltip":"","group":"dashboard\_group","order":5,"width":0,"height":0,"passthru":true,"decouple":"false","topic":"smart\_farm/site\_01/zone\_A/node\_01/cmd","style":"","onvalue":"{\\"cmd\\":\\"valve\\",\\"state\\":1}","onvalueType":"json","onicon":"","oncolor":"","offvalue":"{\\"cmd\\":\\"valve\\",\\"state\\":0}","offvalueType":"json","officon":"","offcolor":"","animate":false,"className":"","x":160,"y":400,"wires":\[\["mqtt\_out\_cmd"\]\]},{"id":"mqtt\_out\_cmd","type":"mqtt out","z":"lab\_flow","name":"Send Command","topic":"","qos":"","retain":"","respTopic":"","contentType":"","userProps":"","correl":"","expiry":"","broker":"local\_mqtt","x":440,"y":360,"wires":\[\]},{"id":"local\_mqtt","type":"mqtt-broker","name":"Mosquitto Broker","broker":"std-mosquitto","port":"1883","clientid":"","autoConnect":true,"usetls":false,"protocolVersion":"4","keepalive":"60","cleansession":true,"autoUnsubscribe":true,"birthTopic":"","birthQos":"0","birthRetain":"false","birthPayload":"","birthMsg":{},"closeTopic":"","closeQos":"0","closeRetain":"false","closePayload":"","closeMsg":{},"willTopic":"","willQos":"0","willRetain":"false","willPayload":"","willMsg":{},"userProps":"","sessionExpiry":""},{"id":"local\_influx","type":"influxdb","hostname":"127.0.0.1","port":"8086","protocol":"http","database":"database","name":"InfluxDB","usetls":false,"tls":"","influxdbVersion":"2.0","url":"http\://std-influxdb:8086","rejectUnauthorized":true},{"id":"dashboard\_group","type":"ui\_group","name":"Smart Farm Node 01","tab":"dashboard\_tab","order":1,"disp":true,"width":"6","collapse":false,"className":""},{"id":"dashboard\_tab","type":"ui\_tab","name":"IoT Lab Dashboard","icon":"dashboard","disabled":false,"hidden":false}\]