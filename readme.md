# Smart Farm Lab: ESP32 → MQTT → Node-RED → InfluxDB → Dashboard

คู่มือฉบับนี้ใช้ประกอบการเรียนปฏิบัติการ Smart Farm โดยอาจารย์จะให้นักศึกษาประกอบระบบ ทดลองรับส่งข้อมูลผ่าน MQTT และสังเกตการทำงานตั้งแต่ ESP32 ไปจนถึง Dashboard

เมื่อทำตามขั้นตอนครบ นักศึกษาจะอ่านอุณหภูมิและความชื้นจาก DHT22 ส่งข้อมูลทุกประมาณ 5 วินาที แสดงผลบน Dashboard เก็บข้อมูลย้อนหลัง และส่งคำสั่งควบคุมรีเลย์กลับไปยัง ESP32 ได้

## ภาพรวมและลำดับการทำงานของระบบ

ก่อนลงมือ ให้นักศึกษาทำความเข้าใจเส้นทางข้อมูลหลักของระบบ ซึ่งแบ่งเป็นสองทิศทางดังนี้:

```text
ส่งข้อมูลและสถานะ:
ESP32 (DHT22 + GPIO) → MQTT telemetry/status → Node-RED Router
  ├─ telemetry → InfluxDB และ Gauge
  ├─ status → Device Status Text
  └─ telemetry/ACK → สถานะ Pump และ Valve Switch

ส่งคำสั่งควบคุม:
Dashboard Switch หรือ MQTT Explorer → MQTT cmd → ESP32
  → ตรวจสอบคำสั่ง → เขียน GPIO → MQTT ack → Node-RED Router → Dashboard Switch
```

### ลำดับการทำงานของโปรแกรม ESP32

ให้นักศึกษาอ่านลำดับการทำงานต่อไปนี้ควบคู่กับไฟล์ `src/main.cpp`:

1. **เริ่มระบบ:** โปรแกรมกำหนด GPIO 14 (Pump) และ GPIO 27 (Valve) เป็น OUTPUT และตั้งระดับเริ่มต้นเป็น LOW (`0`) จากนั้นเริ่ม LCD และ DHT22
2. **เชื่อมต่อ Wi-Fi:** WiFiManager จะเชื่อมต่อกับเครือข่ายที่บันทึกไว้ หากยังไม่เคยตั้งค่า โปรแกรมจะเปิด Access Point ชื่อ `parinya_AP` ให้นักศึกษาตั้งค่า Wi-Fi ตรงนี้นักศึกษาสามารถแก้ชื่อ AP เป็นของตัวเองได้ โดยให้กดปุ่ม CTRL ตามด้วยปุ่ม F บนแป้นพิมพ์ เพื่อค้นหาชื่อ AP `parinya_AP` หากเชื่อมต่อไม่สำเร็จ โปรแกรมจะรีสตาร์ต
3. **เตรียม MQTT:** โปรแกรมตั้งค่า Broker และ callback พร้อมลดเวลารอ TCP และ MQTT เป็นอย่างละ 1 วินาที ทุกส่วนทำงานตามลำดับใน `loop()` เดียว
4. **ตรวจปุ่มก่อน:** โปรแกรมตรวจปุ่ม GPIO 4 ซึ่งเป็น active-low และ debounce 50 ms เมื่อนักศึกษากดปุ่ม จะตัดการเชื่อมต่อ MQTT แล้วเปิดหน้า Wi-Fi setup ไม่เกิน 180 วินาที ระหว่างเปิดหน้าตั้งค่า โปรแกรมจะพักการอ่านเซนเซอร์และรับส่ง MQTT หากยังกดปุ่มค้างหลังตั้งค่าเสร็จ จะไม่เปิดหน้าตั้งค่าซ้ำจนกว่าจะปล่อยแล้วกดใหม่
5. **ดูแล MQTT:** หากยังไม่เชื่อมต่อ `reconnect()` จะลองเพียงครั้งเดียวแล้วกลับเข้า `loop()` ใช้ `millis()` นับเวลาให้ครบ 5 วินาทีหลังจบความพยายามก่อนลองใหม่ จึงไม่ต้องใช้ `while` วนรอหรือ `delay(5000)` เมื่อเชื่อมต่อสำเร็จ โปรแกรมจะ subscribe Topic `cmd` และประกาศสถานะ `online` ส่วน Last Will (LWT) จะประกาศ `offline` หากการเชื่อมต่อขาด
6. **รับข้อความ:** เมื่อเชื่อมต่ออยู่ โปรแกรมเรียก `client.loop()` ในแต่ละรอบเพื่อประมวลผลคำสั่งจาก MQTT
7. **อ่านและส่งเซนเซอร์:** ใช้ `millis()` ตรวจว่าผ่านไปประมาณ 5 วินาทีหรือยัง เมื่อครบเวลา จะอ่าน DHT22 อัปเดต LCD แล้วสร้าง JSON ที่มี `temperature`, `humidity`, `pump_state` และ `valve_state` ส่งไปยัง Topic `telemetry` หาก MQTT ยังไม่พร้อม จะข้ามการส่งรอบนั้น หากอ่าน DHT22 ไม่สำเร็จ จะข้ามทั้งการอัปเดตและส่งตัวอย่างนั้น
8. **รับคำสั่งรีเลย์:** callback จะรับเฉพาะข้อความจาก Topic `cmd` และตรวจสอบ JSON ก่อนเปลี่ยน GPIO คำสั่งต้องมี `cmd` เป็น `pump` หรือ `valve` และ `state` เป็นจำนวนเต็ม `0` หรือ `1` หากรูปแบบไม่ถูกต้อง โปรแกรมจะไม่เปลี่ยน GPIO และไม่ส่ง ACK สำเร็จ
9. **ตอบ ACK:** หลังเขียน GPIO แล้ว ESP32 จะอ่านระดับขานั้นกลับและส่ง `{"ack":"pump","state_set":1}` หรือรูปแบบเดียวกันสำหรับ Valve ไปยัง Topic `ack` ค่า `state_set` คือระดับ GPIO ที่อ่านได้ ไม่ใช่การยืนยันทางไฟฟ้าว่าปั๊มหรือวาล์วทำงานจริง

**เหตุผลที่ปุ่มยังใช้งานได้เมื่อ Broker ล่ม:** หลังลองเชื่อมต่อไม่สำเร็จ โปรแกรมกลับมาตรวจปุ่มและอ่านเซนเซอร์ต่อ ระหว่างพัก 5 วินาทีก็ยังวน `loop()` ตามปกติ อย่างไรก็ตาม `client.connect()` ยังเป็นการรอแบบ synchronous ปุ่มจึงอาจตอบสนองช้าระหว่างรอ TCP/MQTT และการค้นชื่อ DNS อาจรอนานกว่าค่า timeout นี้ หากต้องการตัดเวลารอ DNS ในห้อง Lab ให้ใช้ IP ของ Broker เป็น `mqtt_server` นักศึกษาควรกดปุ่มค้างจนเห็น `WiFi Config Mode` เพราะการกดสั้นระหว่างรอเครือข่ายอาจถูกพลาดได้

> `0` หมายถึง GPIO LOW และ `1` หมายถึง GPIO HIGH รีเลย์บางรุ่นเป็น Active LOW จึงอาจทำงานกลับความหมายได้ ให้ทดสอบวงจรและโหลดจริงอย่างปลอดภัยก่อนใช้งาน

### ลำดับการทำงานของ Node-RED Router

ให้นักศึกษาตรวจดูสายจาก MQTT In ใน Node-RED: โหนดนี้รับทุก Topic ใต้ `smart_farm/site_01/zone_A/node_01/` แล้วส่งข้อความไปยัง Debug และ Function `Router & Influx Format` ตามลำดับ Function มี 5 Outputs และเลือกเส้นทางจากส่วนท้ายของ Topic กับชนิดของ `msg.payload` ดังนี้:

1. **ตรวจ payload:** หากไม่มี `msg.payload` หรือ payload ไม่ใช่ object Function จะคืน `null` ทั้ง 5 outputs และไม่ส่งข้อความต่อ
2. **แยก Topic:** Function แบ่ง Topic ด้วย `/` เพื่ออ่าน `site`, `zone`, `device` และชนิดข้อความ (`telemetry`, `status`, `ack`)
3. **เมื่อเป็น `telemetry`:**
   - Output 1 สร้างข้อมูล InfluxDB: `msg.measurement = "sensor_data"`, fields ได้แก่ `temperature`, `humidity`, `pump`, `valve` และ tags ได้แก่ `site`, `zone`, `device`
   - Output 2 ส่ง object telemetry เดิมไปอัปเดต Temperature และ Humidity Gauge
   - Output 3 เว้นว่าง
   - Output 4 และ 5 แปลง `pump_state` / `valve_state` เป็น `{ "cmd": "...", "state": 0|1 }` แล้วส่งกลับเข้า Switch ที่ตรงกัน เพื่อซิงก์หน้าจอกับสถานะล่าสุดจากบอร์ด
4. **เมื่อเป็น `status`:** Output 3 ส่งเฉพาะ `msg.payload.status` ไปแสดงใน Device Status Text; outputs อื่นเว้นว่าง
5. **เมื่อเป็น `ack`:** ถ้า `msg.payload.ack` เป็น `pump` จะส่ง `state_set` ไป Output 4 เท่านั้น; ถ้าเป็น `valve` จะส่งไป Output 5 เท่านั้น ทำให้ UI แสดงผลตามค่าที่บอร์ดตอบกลับจริง
6. **ข้อความชนิดอื่นหรือ ACK ที่ไม่มี state ถูกต้อง:** ไม่ส่งไปยัง outputs ใด เพราะ `switchFeedback()` ยอมรับเฉพาะเลข `0` หรือ `1`

สังเกตการตั้งค่าของ UI Switch ด้วย: `passthru: false` และ `decouple: true` ทำให้ข้อความสถานะจาก Router ปรับหน้าจอได้โดยไม่ถูกส่งออกจาก Switch ไปยัง MQTT Out ซ้ำ จึงไม่เกิดการวนส่งคำสั่ง ส่วนเมื่อผู้ใช้กด Switch ข้อความที่ออกจาก Switch ยังคงเป็น JSON command และส่งไปยัง MQTT Out ตามปกติ

## 1. เตรียมอุปกรณ์และโปรแกรม

ก่อนเริ่ม ให้นักศึกษาจัดเตรียมอุปกรณ์ดังต่อไปนี้:

- ESP32 DOIT DevKit V1 และสาย USB ที่รับส่งข้อมูลได้
- DHT22, จอ LCD I2C 20×4 ที่อยู่ `0x27`, รีเลย์ 2 ช่อง และปุ่มกด
- Breadboard และสายเชื่อมต่อ
- คอมพิวเตอร์ Windows และ Wi-Fi 2.4 GHz ที่ ESP32 เข้าถึงคอมพิวเตอร์ได้

ติดตั้งโปรแกรมต่อไปนี้ให้พร้อมก่อนทำ Lab:

1. Docker Desktop พร้อม WSL2 แล้วเปิด Docker Desktop ให้ Engine ทำงาน
2. Visual Studio Code และส่วนขยาย **PlatformIO IDE**
3. MQTT Explorer สำหรับตรวจข้อความและทดลองส่งคำสั่ง
4. เว็บเบราว์เซอร์
5. สมัครบัญชี GitHub ที่ [github.com](https://github.com/) หากยังไม่มีบัญชี
![alt text](image.png)

ให้นักศึกษาเชื่อมคอมพิวเตอร์และ ESP32 เข้ากับเครือข่ายเดียวกัน และหลีกเลี่ยง Guest Wi-Fi ที่แยกอุปกรณ์ออกจากกัน เครือข่ายอินเทอร์เน็ตของมหาวิทยาลัยอาจมี Firewall ปิดกั้นพอร์ต หากเชื่อมต่อไม่ได้ ให้ตรวจสอบข้อจำกัดของเครือข่ายหรือสอบถามผู้ดูแลก่อน

## 2. เปิดโปรเจกต์และทำความเข้าใจพอร์ต

ให้นักศึกษาเปิดโปรเจกต์และตรวจสอบตำแหน่งทำงานตามขั้นตอนนี้:

1. เปิด VS Code เลือก **File → Open Folder** แล้วเลือกโฟลเดอร์โปรเจกต์
2. เปิด **Terminal → New Terminal** และเลือกใช้ PowerShell
3. ตรวจสอบว่า Terminal อยู่ในโฟลเดอร์เดียวกับ `docker-compose.yml` และ `platformio.ini`

ตัวอย่างต่อไปนี้ใช้ตำแหน่งโปรเจกต์ของเครื่องในห้องปฏิบัติการ หากนักศึกษาเก็บโปรเจกต์ไว้ที่อื่น ให้เปลี่ยน path ให้ตรงกับเครื่องของตน:

```powershell
cd D:\code_iot\ESP32_data_vistual
docker version
```

**ตรวจสอบผล:** คำสั่ง `docker version` ควรแสดงทั้ง Client และ Server หากไม่พบข้อมูล Server ให้เปิด Docker Desktop แล้วรอให้ Engine พร้อม

ระหว่างทำ Lab นักศึกษาจะใช้ไฟล์หลักต่อไปนี้:

| ไฟล์ | หน้าที่ |
|---|---|
| `docker-compose.yml` | เปิด Mosquitto, InfluxDB, Node-RED และ Grafana |
| `src/main.cpp` | โปรแกรม ESP32 และค่าการเชื่อมต่อ MQTT |
| `platformio.ini` | รุ่นบอร์ด ไลบรารี และความเร็ว Serial Monitor |
| `std_lab_data/mosquitto/config/mosquitto.conf` | ตั้งค่า MQTT Broker |
| `docs/node-red-flow.json` | Flow ตัวอย่างสำหรับนักศึกษานำเข้า Node-RED |
| `std_lab_data/nodered/flows.json` | Flow ที่ Node-RED บันทึกขณะใช้งาน |
| `std_lab_data/` | ข้อมูลถาวรของบริการ ห้ามลบระหว่างทำ Lab |
| `.gitignore` | กำหนดไฟล์และข้อมูลที่ Git ไม่ต้องติดตาม |

### ทำความเข้าใจไฟล์ `.gitignore`

ก่อนส่งงาน ให้นักศึกษาทำความเข้าใจว่า `.gitignore` ระบุไฟล์ที่ Git ควรละเว้น เพื่อป้องกันไม่ให้ไฟล์ build ไฟล์เฉพาะเครื่อง และข้อมูล runtime ถูกเพิ่มเข้า repository โดยไม่ตั้งใจ โปรเจกต์นี้ตั้งค่าให้ละเว้นรายการต่อไปนี้:

| รูปแบบที่ละเว้น | คำอธิบาย |
|---|---|
| `.pio` | ไฟล์ build และผลลัพธ์จาก PlatformIO ซึ่งสร้างใหม่ได้ |
| `.vscode/.browse.c_cpp.db*` | ฐานข้อมูลสำหรับค้นหาและเรียกดูโค้ด C/C++ ที่ VS Code สร้างขึ้น |
| `.vscode/c_cpp_properties.json` | การตั้งค่า IntelliSense C/C++ ที่อาจแตกต่างกันในแต่ละเครื่อง |
| `.vscode/launch.json` | การตั้งค่าเปิดโปรแกรม Debug ซึ่งอาจเฉพาะกับเครื่องหรือวิธี Debug ของผู้ใช้ |
| `.vscode/ipch` | ไฟล์แคชของ IntelliSense ที่สร้างใหม่ได้ |
| `/std_lab_data/mosquitto/data/`, `/std_lab_data/mosquitto/log/` | ข้อมูลและ log ที่ Mosquitto สร้างระหว่างทำงาน |
| `/std_lab_data/influxdb/` | ฐานข้อมูล InfluxDB ที่เก็บข้อมูล runtime |
| `/std_lab_data/grafana/` | ข้อมูลและการตั้งค่าที่ Grafana สร้างหรือเปลี่ยนระหว่างใช้งาน |
| `/std_lab_data/nodered/.config.*` | ไฟล์ตั้งค่าภายในของ Node-RED |
| `/std_lab_data/nodered/*.backup` | ไฟล์สำรองที่ Node-RED สร้างขึ้น |
| `/std_lab_data/nodered/flows_cred.json` | ข้อมูล credentials ของ Node-RED ซึ่งอาจมีข้อมูลลับ |
| `/std_lab_data/nodered/node_modules/` | แพ็กเกจ Node.js ที่ติดตั้งสำหรับ Node-RED และติดตั้งใหม่ได้ |

เครื่องหมาย `/` หน้ารูปแบบใน `std_lab_data` หมายถึงให้จับคู่จากโฟลเดอร์หลักของ repository ส่วน `.vscode/...` และ `.pio` ใช้กับรายการที่ระบุในโปรเจกต์นี้เท่านั้น โปรดจำไว้ว่า `.gitignore` มีผลกับไฟล์ที่ Git ยังไม่ได้ติดตาม หากเพิ่มไฟล์เข้า Git ไปแล้ว การเพิ่มชื่อไฟล์ใน `.gitignore` อย่างเดียวจะไม่หยุดติดตามไฟล์นั้น

พอร์ตที่ใช้ในโปรเจกต์ปัจจุบัน:

| บริการ | จากคอมพิวเตอร์/ESP32 | ระหว่าง container |
|---|---|---|
| MQTT | `<SERVER_IP>:11883` | `std-mosquitto:1883` |
| Node-RED Editor | `http://localhost:11880` | `std-nodered:1880` |
| Node-RED Dashboard | `http://localhost:11880/ui` | — |
| InfluxDB | `http://localhost:8086` | `http://std-influxdb:8086` |
| Grafana | `http://localhost:3000` | `std-grafana:3000` |

ในตาราง `<SERVER_IP>` หมายถึง IPv4 ของคอมพิวเตอร์ที่เปิด Docker ไม่ใช่ IP ของ ESP32 ส่วน `localhost` ใช้เมื่อเปิดเว็บจากคอมพิวเตอร์เครื่องนั้น หากนักศึกษาเปิดเว็บจากเครื่องอื่น ให้แทน `localhost` ด้วย `<SERVER_IP>`

พอร์ตใน Compose เขียนในรูปแบบ `HOST:CONTAINER` เช่น `11883:1883` หมายความว่า ESP32 เชื่อมต่อผ่านพอร์ต `11883` ส่วนบริการภายใน Docker ใช้พอร์ต `1883` นักศึกษาไม่ควรเปลี่ยนพอร์ตทั้งสองฝั่งให้เป็นค่าเดียวกันโดยไม่ตรวจการตั้งค่าที่เกี่ยวข้อง

## 3. ตรวจ MQTT Broker และเปิด Server

ให้นักศึกษาตรวจและเปิด MQTT Broker ตามขั้นตอนต่อไปนี้:

1. เปิด `std_lab_data/mosquitto/config/mosquitto.conf` และตรวจว่ามีค่าต่อไปนี้ ชื่อไฟล์ต้องไม่ลงท้ายด้วย `.txt`:

```conf
listener 1883 0.0.0.0
protocol mqtt
allow_anonymous true
persistence true
persistence_location /mosquitto/data/
log_dest stdout
log_dest file /mosquitto/log/mosquitto.log
```

การตั้งค่าแบบไม่ใช้บัญชีเหมาะสำหรับเครือข่าย Lab ที่ควบคุมได้เท่านั้น ไม่ควรนำไปใช้กับ Broker ที่เปิดสู่เครือข่ายสาธารณะ

2. เปิด `docker-compose.yml` และตรวจว่า Mosquitto ใช้ `11883:1883` ส่วน Node-RED ใช้ `11880:1880`
3. จากโฟลเดอร์โปรเจกต์ รันคำสั่งต่อไปนี้:

```powershell
docker compose config --quiet
docker compose up -d
docker compose ps -a
```

ในการเปิดใช้งานครั้งแรก Docker ต้องดาวน์โหลด image จากอินเทอร์เน็ต ขั้นตอนนี้อาจใช้เวลาหลายนาที

**ตรวจสอบผล:** `std-mosquitto`, `std-influxdb`, `std-nodered` และ `std-grafana` ควรแสดงสถานะ `Up` หรือ `running` หากพบ `Created`, `Exited` หรือ `Restarting` ให้นักศึกษาตรวจสอบหัวข้อแก้ปัญหาท้ายคู่มือ

4. ทดลองเปิด Node-RED ที่ `http://localhost:11880` และ InfluxDB ที่ `http://localhost:8086`

## 4. ตั้งค่า InfluxDB และสร้าง Token

ให้นักศึกษาตั้งค่า InfluxDB และสร้าง Token ก่อนตั้งค่าโหนดบันทึกข้อมูลใน Node-RED:

1. เปิด `http://localhost:8086`
2. หากเป็นการตั้งค่าครั้งแรก ให้เลือก **Get Started** แล้วกำหนดค่าตามตาราง:

| รายการ | ค่าที่ใช้ |
|---|---|
| Username | ตั้งชื่อบัญชีของนักศึกษาเอง |
| Password | ตั้งรหัสผ่านและเก็บไว้ |
| Organization | `lab` |
| Bucket | `smartfarm` |

3. หากพบหน้า Login แสดงว่าฐานข้อมูลตั้งค่าไว้แล้ว ให้นักศึกษาใช้บัญชีเดิมและตรวจ Organization กับ Bucket ที่มีอยู่ อย่าลบโฟลเดอร์ฐานข้อมูลเพื่อเริ่มใหม่
4. ไปที่ **Load Data → API Tokens** แล้วสร้าง **Custom API Token** ที่มีสิทธิ์อ่านและเขียน Bucket `smartfarm` สำหรับ Node-RED
5. เก็บ Token ไว้ในที่ปลอดภัยและนำไปตั้งค่าใน Node-RED ห้ามใส่ Token ในรายงานหรือ commit ลง Git
6. สร้าง Token อีกตัวที่อ่าน Bucket `smartfarm` ได้ เพื่อใช้กับ Grafana

หากนักศึกษาใช้ Organization หรือ Bucket ชื่ออื่น ให้เปลี่ยนค่าใน Node-RED, Grafana และตัวอย่าง query ในคู่มือนี้ให้ตรงกัน

**ตรวจสอบผล:** ควรมี Bucket `smartfarm` และ Token สำหรับเขียนข้อมูล ข้อมูลจะเริ่มปรากฏหลังจากเปิด ESP32 และ Deploy Flow แล้ว

## 5. ต่อวงจร ESP32

ก่อนต่อวงจร ให้นักศึกษาถอดสาย USB ออกจากบอร์ด และต่อ GND ร่วมกันสำหรับวงจรสัญญาณ จากนั้นต่ออุปกรณ์ตามตาราง:

| อุปกรณ์/สัญญาณ | ขา ESP32 | หมายเหตุ |
|---|---|---|
| DHT22 DATA | GPIO 15 | เซนเซอร์เปล่าใช้ตัวต้านทาน pull-up DATA ไป 3.3 V ประมาณ 4.7–10 kΩ; โมดูลอาจมีอยู่แล้ว |
| DHT22 VCC / GND | 3.3 V / GND | ตรวจลำดับขาจากอุปกรณ์จริง |
| รีเลย์ช่องปั๊ม IN | GPIO 14 | ตรวจว่าโมดูลรับสัญญาณ 3.3 V ได้ |
| รีเลย์ช่องวาล์ว IN | GPIO 27 | จ่ายไฟโมดูลตามพิกัด ไม่จ่ายไฟโหลดจาก GPIO |
| ปุ่มตั้งค่า Wi-Fi | GPIO 4 กับ GND | โปรแกรมใช้ `INPUT_PULLUP`; กดแล้วเป็น LOW |
| LCD SDA / SCL | GPIO 21 / GPIO 22 | ขา I2C เริ่มต้นของบอร์ดที่ใช้ |
| LCD VCC / GND | ตามพิกัดโมดูล / GND | หากวงจร I2C ดึงสัญญาณขึ้น 5 V ให้ใช้ตัวแปลงระดับสัญญาณก่อนเข้า ESP32 |

ก่อนต่อปั๊มหรือวาล์วจริง ให้นักศึกษาทดสอบรีเลย์ด้วย LED หรือโหลดแรงดันต่ำก่อน โปรแกรมเขียนค่า `state` ลง GPIO โดยตรง (`1` คือ HIGH และ `0` คือ LOW) รีเลย์แบบ Active LOW อาจเปิดเมื่อส่ง `0` จึงต้องตรวจสอบรุ่นของรีเลย์ให้แน่ใจก่อนใช้งาน

## 6. ตั้งค่าโปรแกรมและอัปโหลดลง ESP32

ให้นักศึกษาตั้งค่าโปรแกรมและอัปโหลดลง ESP32 ตามขั้นตอนนี้:

1. เปิด PowerShell แล้วรัน:

```powershell
ipconfig
```

2. จด **IPv4 Address ของ Wi-Fi หรือ Ethernet ที่ใช้งานจริง** อย่าเลือก IP ของอะแดปเตอร์ `vEthernet`, Docker หรือ WSL
3. เปิด `src/main.cpp` แล้วแก้ค่าการเชื่อมต่อให้ตรงกับเครื่องของนักศึกษา ตัวอย่างเช่น:

```cpp
const char* mqtt_server = "192.168.1.72"; // เปลี่ยนเป็น IPv4 ของเครื่องที่เปิด Docker
const int mqtt_port = 11883;

String site_id = "site_01";
String zone_id = "zone_A";
String device_id = "node_01";
```

หากมีหลายบอร์ดใช้ Broker เดียวกัน นักศึกษาต้องกำหนด `device_id` ให้แต่ละบอร์ดไม่ซ้ำกัน เช่น `node_02` และแก้ Topic ใน MQTT Explorer กับ Node-RED ให้ตรงกับค่าที่กำหนด

4. บันทึกไฟล์ ต่อ ESP32 ผ่าน USB แล้วรอให้ PlatformIO ติดตั้งแพ็กเกจตาม `platformio.ini`
5. ใน VS Code เปิด **PlatformIO → Project Tasks → esp32doit-devkit-v1 → General → Build**
6. เมื่อ Build แสดง `SUCCESS` ให้เลือก **Upload**
7. เปิด **Monitor** ที่ความเร็ว **115200 baud** หาก Upload ไม่พบพอร์ต ให้ตรวจสาย USB, Device Manager และไดรเวอร์ของบอร์ด

หากนักศึกษาต้องการใช้ PlatformIO Core CLI ให้เปิด PlatformIO Terminal แล้วรัน:

```powershell
pio run
pio run --target upload
pio device monitor --baud 115200
```

หากระบบไม่พบคำสั่ง `pio` ให้ใช้ Project Tasks ใน VS Code แทนโดยไม่ต้องแก้ PATH และอย่าลืมปิด Serial Monitor ก่อนอัปโหลดครั้งถัดไป หากพอร์ตยังถูกใช้งานอยู่

**ตรวจสอบผล:** Build และ Upload ควรสำเร็จ และ Serial Monitor ควรแสดง `Smart Farm starting...`

## 7. เชื่อมต่อ Wi-Fi และตรวจ MQTT

ให้นักศึกษาตั้งค่า Wi-Fi และตรวจการเชื่อมต่อ MQTT ตามขั้นตอนนี้:

1. หาก ESP32 ยังไม่มีข้อมูล Wi-Fi ให้ใช้มือถือหรือคอมพิวเตอร์เชื่อมต่อ Access Point ชื่อ **parinya_AP**
2. เปิดหน้าตั้งค่า หากหน้าเว็บไม่เปิดขึ้นเอง ให้เข้า `http://192.168.4.1`
3. เลือก Wi-Fi 2.4 GHz ที่เข้าถึง Server ได้ กรอกรหัสผ่านแล้วบันทึก
4. เมื่อตั้งค่าเสร็จ ให้มือถือหรือคอมพิวเตอร์กลับไปเชื่อมต่อเครือข่ายปกติ
5. เปิด Serial Monitor แล้วตรวจข้อความ:

```text
[WiFi] Connected. IP=...
[MQTT] Broker=192.168.1.72:11883, client=node_01
[MQTT] Subscribe OK: smart_farm/site_01/zone_A/node_01/cmd
[MQTT] Telemetry publish OK: ...
```

IP ใน log ต้องตรงกับ IP ของเครื่อง Server หากนักศึกษาต้องการเปลี่ยน Wi-Fi ภายหลัง ให้กดปุ่ม GPIO 4 เพื่อเปิด Access Point `parinya_AP` อีกครั้ง โดย portal จะเปิดได้นานสูงสุด 180 วินาที

6. เปิด MQTT Explorer แล้วสร้าง Connection โดยใช้ค่าต่อไปนี้:

| รายการ | ค่า |
|---|---|
| Protocol | MQTT แบบไม่ใช้ TLS |
| Host | IPv4 ของ Server |
| Port | `11883` |
| Username / Password | เว้นว่าง |

7. กด **Connect** แล้วตรวจ Topic ของ `node_01` ตามตาราง:

| Topic | การใช้งาน |
|---|---|
| `smart_farm/site_01/zone_A/node_01/telemetry` | ค่าเซนเซอร์ทุกประมาณ 5 วินาที |
| `smart_farm/site_01/zone_A/node_01/status` | สถานะ `online` / `offline` แบบ retained |
| `smart_farm/site_01/zone_A/node_01/cmd` | ส่งคำสั่งจากผู้ใช้ไปยัง ESP32 |
| `smart_farm/site_01/zone_A/node_01/ack` | ESP32 ตอบรับคำสั่ง |

ตัวอย่าง Telemetry:

```json
{"temperature":28.5,"humidity":65.2,"pump_state":0,"valve_state":0}
```

8. ให้นักศึกษาทดลองควบคุมปั๊มด้วยช่อง Publish ของ MQTT Explorer ส่ง JSON ต่อไปนี้ไปยัง Topic `smart_farm/site_01/zone_A/node_01/cmd` และ **ปิด Retain**:

```json
{"cmd":"pump","state":1}
```

**ตรวจสอบผล:** GPIO 14 ควรเปลี่ยนเป็น HIGH และ Topic `ack` ควรได้รับ `{"ack":"pump","state_set":1}` จากนั้นให้นักศึกษาทดลองส่ง `state:0` เพื่อตั้ง GPIO เป็น LOW และเปลี่ยน `cmd` เป็น `valve` เพื่อทดสอบ GPIO 27

## 8. เตรียม Flow ใน Node-RED

ให้นักศึกษาตรวจสอบและเตรียม Flow ใน Node-RED ดังนี้:

1. เปิด `http://localhost:11880`
2. ไปที่ **☰ → Manage palette → Install** แล้วตรวจว่ามีแพ็กเกจต่อไปนี้ หากยังไม่มี ให้ติดตั้ง:
   - `node-red-dashboard` สำหรับ Dashboard แบบคลาสสิกที่ Flow นี้ใช้
   - `node-red-contrib-influxdb` สำหรับบันทึกข้อมูล
3. หากมี Flow และโหนด `Router & Influx Format` อยู่แล้ว ให้นักศึกษาตรวจจำนวน outputs และสายตามขั้นตอนที่ 9 หากเป็น Flow รุ่นเก่า ให้นำเข้า Flow รุ่นล่าสุดแทน
4. หากยังไม่มี Flow ให้เลือก **☰ → Import → select a file to import** เลือก [docs/node-red-flow.json](docs/node-red-flow.json) แล้ว Import
5. หากมีตัวเลือก ให้นำเข้าเป็น Flow ใหม่ จากนั้นตรวจว่ามี MQTT In, Function, InfluxDB Out, Gauge, Text, Switch และ Debug ครบ และไม่มีโหนด unknown

ให้นักศึกษาใช้ไฟล์ `docs/node-red-flow.json` สำหรับนำเข้า อย่าคัดลอก JSON จาก `instruction.md` เพราะตัวอย่างในเอกสารมีอักขระ escape จากการจัดรูปแบบ ไฟล์ Flow ไม่มี Token จึงต้องตั้ง Token ของตนเองในขั้นตอนถัดไป

ใน Flow มี Debug อยู่สองจุด: **debug 1** ใช้ตรวจ Topic และ payload ที่รับจาก MQTT In ส่วน **debug 2** ใช้ตรวจ JSON คำสั่งที่ออกจาก Switch ให้นักศึกษาเปิด Debug sidebar เพื่อดูข้อความ และปิด Debug nodes ได้เมื่อทดสอบเสร็จ โปรดทราบว่า Debug ทั้งสองจุดไม่ได้ตรวจผลการเขียน InfluxDB

## 9. ตั้งค่าการเชื่อมต่อใน Node-RED และ Deploy

ให้นักศึกษาตั้งค่า Broker, InfluxDB และตรวจสายของ Flow ก่อน Deploy ตามขั้นตอนต่อไปนี้:

1. ดับเบิลคลิกโหนด **MQTT In** แล้วกดปุ่มดินสอข้าง Server จากนั้นกำหนด Broker:

| รายการ | ค่า |
|---|---|
| Server | `std-mosquitto` |
| Port | `1883` |
| TLS | ไม่เปิดสำหรับ Lab นี้ |
| Username / Password | เว้นว่าง |
| Client ID | เว้นว่างให้สร้างอัตโนมัติ หรือกำหนดชื่อที่ไม่ซ้ำกับ ESP32 |

2. ตั้ง Topic เป็น `smart_farm/site_01/zone_A/node_01/#` และ Output เป็น **a parsed JSON object** จากนั้นตรวจว่า MQTT Out ใช้ Broker เดียวกัน
3. ดับเบิลคลิก **Save to InfluxDB** แล้วกดปุ่มดินสอเพื่อแก้ Server configuration:

| รายการ | ค่า |
|---|---|
| Version | `2.0` |
| URL | `http://std-influxdb:8086` |
| Token | Token ที่เขียน Bucket `smartfarm` ได้จากขั้นตอนที่ 4 |

4. ในโหนด **Save to InfluxDB** ตั้ง **Organization = `lab`**, **Bucket = `smartfarm`** และ Precision เป็น `ms` โดย Flow จะกำหนด measurement เป็น `sensor_data` ผ่าน `msg.measurement`
5. เปิด Function **Router & Influx Format** แล้วตรวจว่ามี Outputs จำนวน **5** และต่อสายตามตาราง:

| Output | ต่อไปที่ | ข้อมูล |
|---|---|---|
| 1 | Save to InfluxDB | `msg.payload = [fields, tags]` |
| 2 | Temperature และ Humidity Gauge | JSON ค่าเซนเซอร์ |
| 3 | Device Status Text | ข้อความ online/offline |
| 4 | Pump Control Switch | สถานะ pump จาก telemetry หรือ ACK |
| 5 | Valve Control Switch | สถานะ valve จาก telemetry หรือ ACK |

ข้อมูลที่ส่งไปบันทึกประกอบด้วย Fields `temperature`, `humidity`, `pump`, `valve` และ Tags `site`, `zone`, `device` ส่วน Output 4–5 ส่ง payload รูปแบบเดียวกับค่า On/Off ของ UI Switch เช่น `{"cmd":"pump","state":1}` เพื่อให้ Switch เปรียบเทียบและอัปเดตสถานะได้

6. ดับเบิลคลิก Switch **Pump Control** และ **Valve Control** แล้วตรวจว่าแต่ละตัวส่งไปยัง Topic `smart_farm/site_01/zone_A/node_01/cmd` และกำหนดค่า On/Off เป็นชนิด **JSON**:

| Switch | On | Off |
|---|---|---|
| Pump | `{"cmd":"pump","state":1}` | `{"cmd":"pump","state":0}` |
| Valve | `{"cmd":"valve","state":1}` | `{"cmd":"valve","state":0}` |

ตรวจด้วยว่า MQTT Out ไม่ได้เปิด Retain สำหรับคำสั่งควบคุม

ให้นักศึกษาตรวจการตั้งค่าของ UI Switch ทั้งสองเพิ่มเติม: `passthru` ต้องเป็น **false** และ `decouple` ต้องเป็น **true** เพื่อให้ feedback จาก Outputs 4–5 อัปเดต Dashboard โดยไม่ถูกส่งกลับเป็นคำสั่ง MQTT

7. กด **Done** แล้วกด **Deploy** เปิด Debug sidebar และตรวจว่าไม่มี error หากต้องการตรวจข้อมูลก่อนบันทึก ให้นักศึกษาเพิ่ม Debug node ต่อจาก Output 1 ชั่วคราว

**ตรวจสอบผล:** MQTT nodes ควรแสดงสถานะ connected และไม่ควรพบ `401 unauthorized` จาก InfluxDB

## 10. ทดสอบ Dashboard และข้อมูลย้อนหลัง

ให้นักศึกษาทดสอบการแสดงผลและการบันทึกข้อมูลตามขั้นตอนนี้:

1. เปิด Dashboard ที่ `http://localhost:11880/ui`
2. ตรวจว่าเกจ Temperature และ Humidity เปลี่ยนตามค่าจาก DHT22 และ Device Status แสดง `online`
3. ส่งคำสั่ง pump หรือ valve จาก Dashboard หรือ MQTT Explorer แล้วตรวจ ACK ใน MQTT Explorer จากนั้นสังเกตว่า UI Switch เปลี่ยนตามค่า `state_set` ที่บอร์ดตอบกลับ สถานะจาก telemetry จะซิงก์ UI อีกครั้งทุกครั้งที่บอร์ดส่งข้อมูล หากส่งคำสั่งจาก MQTT Explorer แล้วไม่ได้ ACK ให้ตรวจ Topic `cmd`, รูปแบบ JSON, การเชื่อมต่อ MQTT ของ ESP32 และ Topic ที่ MQTT Explorer ใช้รับ ACK
4. เปิด InfluxDB → **Data Explorer** เลือก Bucket `smartfarm`, measurement `sensor_data` และ field `temperature` หรือ `humidity` ตั้งช่วงเวลาเป็น 15 นาทีล่าสุด แล้วกด **Submit**

หากนักศึกษาใช้ Script Editor ให้ทดลอง query Flux ต่อไปนี้:

```flux
from(bucket: "smartfarm")
  |> range(start: -15m)
  |> filter(fn: (r) => r._measurement == "sensor_data")
  |> filter(fn: (r) => r._field == "temperature" or r._field == "humidity")
  |> filter(fn: (r) => r.device == "node_01")
```

5. ถอดไฟ ESP32 แล้วรอให้ Broker ตรวจพบว่าการเชื่อมต่อขาด สถานะควรเปลี่ยนเป็น `offline` ผ่าน LWT ซึ่งอาจใช้เวลาสักครู่ จากนั้นเสียบไฟกลับและตรวจว่าเปลี่ยนเป็น `online`

**ตรวจสอบผล:** ควรพบข้อมูลใหม่ในฐานข้อมูล Dashboard แสดงค่า และสั่ง GPIO ได้ หากยังไม่พบข้อมูล ให้นักศึกษาแก้ปัญหาก่อนเริ่มตั้งค่า Grafana

## 11. สร้างกราฟใน Grafana

เมื่อตรวจข้อมูลใน InfluxDB ได้แล้ว ให้นักศึกษาสร้างกราฟใน Grafana:

1. เปิด `http://localhost:3000` หากเป็นระบบใหม่ที่ยังไม่ได้เปลี่ยนรหัสผ่าน ให้เข้าสู่ระบบด้วย `admin` / `admin` แล้วตั้งรหัสผ่านใหม่ หากมีข้อมูลเดิม ให้ใช้บัญชีเดิม
2. ไปที่ **Connections → Data sources → Add data source** แล้วเลือก **InfluxDB** ชื่อเมนูอาจแตกต่างเล็กน้อยตามเวอร์ชัน
3. กำหนดค่าตามตาราง:

| รายการ | ค่า |
|---|---|
| Query language | Flux |
| URL | `http://std-influxdb:8086` |
| Organization | `lab` |
| Token | Token ที่อ่าน Bucket `smartfarm` ได้ |
| Default Bucket | `smartfarm` |

4. กด **Save & test** และตรวจว่าการเชื่อมต่อสำเร็จ
5. สร้าง Dashboard ใหม่ เพิ่ม Visualization เลือก data source ที่เพิ่งสร้าง แล้ววาง query:

```flux
from(bucket: "smartfarm")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "sensor_data")
  |> filter(fn: (r) => r._field == "temperature")
  |> filter(fn: (r) => r.device == "node_01")
  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)
```

6. ตั้งชื่อกราฟเป็น Temperature เลือกหน่วย Celsius แล้วบันทึก Dashboard จากนั้นเพิ่มกราฟ Humidity โดยเปลี่ยน field เป็น `humidity` และเลือกหน่วย Percent (0–100)
7. ตั้งช่วงเวลาเป็น 15 นาทีล่าสุด และกำหนด Refresh ประมาณ 5–10 วินาที

**ตรวจสอบผล:** Grafana ควรแสดงกราฟจากชุดข้อมูลเดียวกับที่นักศึกษาตรวจพบใน InfluxDB

## 12. วิธีเปิดใช้งานครั้งถัดไปและปิดเมื่อเลิกเรียน

เมื่อนักศึกษากลับมาใช้โปรเจกต์ในครั้งถัดไป ให้ทำตามขั้นตอนนี้:

1. เปิด Docker Desktop รอให้ Engine พร้อม
2. เปิด Terminal ในโฟลเดอร์โปรเจกต์แล้วรัน:

```powershell
docker compose up -d
docker compose ps -a
```

3. ตรวจ IPv4 ของ Server หากเปลี่ยน ให้แก้ค่า `mqtt_server` แล้ว Upload โปรแกรมลง ESP32 ใหม่
4. จ่ายไฟให้ ESP32 และเปิด Dashboard นักศึกษาไม่จำเป็นต้องสร้างบัญชี InfluxDB หรือ Import Flow ซ้ำ

เมื่อทำ Lab เสร็จ ให้นักศึกษาปิดโหลดทดลองก่อน แล้วหยุดบริการด้วยคำสั่ง:

```powershell
docker compose stop
```

คำสั่ง `docker compose stop` จะหยุดบริการแต่เก็บ container และข้อมูลไว้ ส่วน `docker compose down` จะลบ container และ network ของโปรเจกต์ แต่ข้อมูลที่ bind mount ใน `std_lab_data/` ยังอยู่ อย่าลบโฟลเดอร์นี้หากต้องการเก็บฐานข้อมูล Flow และ Dashboard

## 13. แก้ปัญหาที่พบบ่อย

### Docker เชื่อมต่อ Engine ไม่ได้

หากคำสั่ง Docker เชื่อมต่อ Engine ไม่ได้ ให้นักศึกษาเปิด Docker Desktop และรอจน Engine พร้อม จากนั้นตรวจด้วย `docker version` หากพบ `permission denied` ให้ตรวจสิทธิ์ของ Terminal ที่ใช้งานก่อน ข้อความ error จากเครื่องมือที่มีข้อจำกัดด้านสิทธิ์ไม่ได้หมายความว่า Engine หยุดทำงานเสมอไป

### เปิด container ไม่ได้: ports are not available / forbidden by its access permissions

หากเปิดบริการไม่ได้เพราะพอร์ตถูกใช้งานหรือถูกสงวนไว้ ให้นักศึกษาตรวจสถานะบริการและช่วงพอร์ตของ Windows ด้วยคำสั่งต่อไปนี้:

```powershell
docker compose ps -a
netsh interface ipv4 show excludedportrange protocol=tcp
Get-NetTCPConnection -LocalPort 11883 -ErrorAction SilentlyContinue
```

แม้ไม่มีโปรแกรมกำลังฟังพอร์ต Windows ก็อาจสงวนพอร์ตนั้นไว้ได้ เครื่องที่ใช้จัดทำคู่มือนี้เคยสงวนช่วง `1823–1922` ทำให้ใช้ `1883`, `1884` และ `1880` ไม่ได้ จึงเปลี่ยนมาใช้ `11883` และ `11880` ช่วงพอร์ตที่สงวนอาจเปลี่ยนหลังรีสตาร์ต ให้นักศึกษาตรวจสอบเครื่องที่ใช้จริงทุกครั้ง

หากจำเป็นต้องเปลี่ยนพอร์ตอีก ให้นักศึกษาเลือกพอร์ต Host ที่ไม่ถูกใช้งานหรือสงวนไว้ แล้วแก้ค่าใน Compose และ `mqtt_port` ใน ESP32 พร้อม Upload โปรแกรมใหม่ ส่วนพอร์ตภายใน Mosquitto และ Broker configuration ของ Node-RED ยังคงเป็น `1883`

การหยุด WinNAT ชั่วคราวเป็นทางเลือกเฉพาะเครื่อง Lab ที่ผู้ดูแลอนุญาตเท่านั้น หากได้รับอนุญาต ให้นักศึกษาเปิด PowerShell แบบ Administrator แล้วรันทีละบรรทัด:

```powershell
net stop winnat
docker compose up -d
net start winnat
```

วิธีนี้อาจปลดการสงวนพอร์ตที่เกี่ยวกับ WinNAT แต่ไม่รับประกันผล และอาจทำให้เครือข่ายเสมือนสะดุด นักศึกษาต้องเปิด WinNAT กลับ แม้คำสั่ง Compose จะล้มเหลว วิธีที่แนะนำคือเลือกใช้พอร์ตที่ตรวจแล้วว่าว่าง

### ESP32 เชื่อมต่อ MQTT ไม่ได้

ให้นักศึกษาตรวจทีละข้อเรียงตามลำดับ: ESP32 เชื่อมต่อ Wi-Fi แล้วหรือไม่ → Server IP ถูกต้องหรือไม่ → Mosquitto container ทำงานหรือไม่ → ESP32 ใช้พอร์ต `11883` หรือไม่ → Windows Firewall อนุญาต TCP พอร์ตนี้บนเครือข่าย Lab หรือไม่ → เครือข่ายแยกอุปกรณ์ออกจากกันหรือไม่

นักศึกษาสามารถทดสอบพอร์ตจากคอมพิวเตอร์อีกเครื่องในเครือข่ายเดียวกัน โดยเปลี่ยน IP ตัวอย่างให้เป็น IP ของ Server:

```powershell
Test-NetConnection 192.168.1.72 -Port 11883
```

ผลที่ต้องการคือ `TcpTestSucceeded : True` และอย่าใช้ `localhost` เป็น Broker address ใน ESP32

### Node-RED บันทึก InfluxDB ไม่ได้: 401 unauthorized

หากพบ `401 unauthorized` ให้นักศึกษาตรวจว่า Token ถูกต้อง ยังไม่ถูกยกเลิก และมีสิทธิ์เขียน Bucket ที่เลือก ตรวจ Organization และ Bucket ให้ตรงกับ InfluxDB แล้ว Deploy อีกครั้ง โปรดจำไว้ว่ารหัสผ่านหน้า Login ไม่ใช่ API Token และการ Restart container ไม่ช่วยแก้ Token ที่ไม่ถูกต้อง

### Dashboard ไม่มีค่า / Node-RED มีโหนด unknown

หาก Dashboard ยังไม่มีข้อมูลหรือพบโหนด unknown ให้นักศึกษาตรวจว่าติดตั้ง `node-red-dashboard` และ `node-red-contrib-influxdb` แล้ว ตั้ง MQTT In ให้ Output เป็น JSON object ตรวจ Topic ให้ตรงกับ `device_id` และตรวจสายทั้ง 5 Outputs ของ Function ตามตารางในหัวข้อ 9 จากนั้นจึง Deploy ใหม่

### DHT อ่านค่าไม่ได้ / LCD ไม่แสดงผล / รีเลย์ทำงานกลับด้าน

- หาก Serial Monitor แสดง `[DHT] Read failed` ให้นักศึกษาตรวจ GPIO 15, ไฟเลี้ยง, GND และตัวต้านทาน pull-up
- หาก LCD ไม่แสดงผล ให้ตรวจสาย SDA/SCL และที่อยู่ I2C โดยโปรแกรมกำหนด `0x27` และจอขนาด 20×4
- หากรีเลย์ทำงานกลับด้าน ให้ตรวจว่าเป็น Active HIGH หรือ Active LOW ค่าใน Telemetry แสดงระดับ GPIO ไม่ได้ยืนยันว่าโหลดทำงานจริง

หากยังหาสาเหตุไม่พบ ให้นักศึกษาตรวจ log ของแต่ละบริการเพิ่มเติม:

```powershell
docker compose logs --tail 50 std-mosquitto
docker compose logs --tail 50 std-nodered
docker compose logs --tail 50 std-influxdb
```

คำเตือน `version is obsolete` จาก Compose ไม่ใช่สาเหตุที่ทำให้ container เริ่มทำงานไม่ได้

## 14. ตรวจสอบและส่งงาน

ก่อนส่งงาน ให้นักศึกษาตรวจรายการต่อไปนี้ให้ครบ:

- [ ] Container ทั้ง 4 ทำงาน และเปิดหน้าเว็บได้ตามพอร์ตในตาราง
- [ ] ESP32 เชื่อมต่อ Wi-Fi และ MQTT ได้ โดย `device_id` ไม่ซ้ำกับอุปกรณ์อื่น
- [ ] MQTT Explorer รับ Telemetry ทุกประมาณ 5 วินาที
- [ ] เมื่อส่งคำสั่ง pump/valve พบ ACK และ GPIO เปลี่ยนตามคำสั่ง
- [ ] Node-RED Dashboard แสดงค่าเซนเซอร์และสถานะ
- [ ] InfluxDB มีข้อมูล measurement `sensor_data` พร้อม tags ของบอร์ด
- [ ] Grafana แสดงกราฟ Temperature และ Humidity
- [ ] ถอดและจ่ายไฟ ESP32 ใหม่ แล้วตรวจสถานะ `offline`/`online` ได้

ให้นักศึกษาส่งหลักฐานเป็นภาพสถานะ container, Serial Monitor, MQTT Telemetry/ACK, Node-RED Flow และ Dashboard, ข้อมูลใน InfluxDB และกราฟ Grafana พร้อมอธิบายเส้นทางข้อมูลและความแตกต่างระหว่างพอร์ต Host กับ Container ปิดบังรหัสผ่านและ Token ในภาพทุกภาพ

## เอกสารประกอบ

- [ต้นฉบับบทปฏิบัติการ](instruction.md)
- [การตั้งค่า InfluxDB 2.x](https://docs.influxdata.com/influxdb/v2/get-started/setup/)
- [โหนด Node-RED สำหรับ InfluxDB](https://flows.nodered.org/node/node-red-contrib-influxdb)
- [การตั้งค่าการเชื่อมต่อ Grafana กับ InfluxDB](https://grafana.com/docs/learning-paths/influxdb-data-source/configure-connection/)
