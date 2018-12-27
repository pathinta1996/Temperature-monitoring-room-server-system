# Temperature-monitoring-room-server-system
school project
#include <ESP8266WiFi.h>    //https://github.com/esp8266/Arduino
#include <DNSServer.h>
#include <ESP8266WebServer.h>
#include <WiFiManager.h>     //https://github.com/tzapu/WiFiManager
#define D0 16             // USER LED Wake
#define ledPin  D0        // ใส่ LED ติดที่ D0 
#define D1 5
#define ConfigWiFi_Pin D1 
#define ESP_AP_NAME  "ESP8266 Config AP"
#include "DHT.h"
#include <ESP8266WiFi.h>
#include "ThingSpeak.h"
#include <ESP8266HTTPClient.h> 
unsigned long myChannelNumber = 637029 ;  ///  channel id ได้มาจาก คลาวด์thingspeak
const char * myWriteAPIKey = "8T60KPRAT9GNGEPA";  // ได้มาจาก คลาวด์thingspeak
#define LINE_TOKEN "uPSu8RfJGaT9y7KTzbeFk5w4CuRj4Uv2FZIZakQeBFv" // รหัส Line Token
char tempF[6]; // บัฟเฟอร์สำหรับ temp //charใช้หน่วยความจำน้อย เก็บอักษร เลขจำนวนเต็ม
String message; // ตัวแปรแสดงข้อความ
String messagetemperature; //ตัวแปรเก็บค่าอุณหภูมิ
String messagehumidity; // ตัวแปรเก็บค่าความชื้นสัมพันธ์
int Buzzer = 2;
#define DHTPIN D2     // กำหนดให้ ขาเป็น D2 
#define DHTTYPE DHT22   // DHT 22 กำหนดรุ่นตัวเซ็นเซอร์วัดอุณหภูมิ
DHT dht(DHTPIN, DHTTYPE); // กำหนดให้ SENSOR อุณหภูมิความชื่น เข้าทาง D2

WiFiClient client;

void setup() 
  {
  Serial.begin(115200);
  digitalWrite(ledPin,LOW);//Turn on the LE
  WiFi.mode(WIFI_STA);

  WiFiManager wifiManager;
  if(digitalRead(ConfigWiFi_Pin) == LOW) // เมื่อกดปุ่มจะทำการรีเช็ตไวฟายที่เชื่อมไว้
  {
    wifiManager.resetSettings(); // เข้าไปที่ ip 192.168.4.1 เพื่อทำการ ใส่ชื่อและรหัสไวฟาย
  }
  //fetches ssid and password from EEPROM and tries to connect
  //if it does not connect, it starts an access point with the specified name
  //and goes into a blocking loop awaiting configuration
  wifiManager.autoConnect(ESP_AP_NAME); 
  while (WiFi.status() != WL_CONNECTED)  //รอจนกว่าเชื่อมต่อสำเร็จ
  {
     delay(500);
     Serial.print(".");
  }
  Serial.println("");
  Serial.println("WiFi connected");
  Serial.println("IP address: ");
  Serial.println(WiFi.localIP()); //แสดงค่า IP หลังเชื่อมต่อได้
  ThingSpeak.begin(client);
  dht.begin();
  pinMode(Buzzer,OUTPUT);
  pinMode(ledPin, OUTPUT);
  pinMode(ConfigWiFi_Pin,INPUT_PULLUP);

}

void Line_Notify(String message) {
 WiFiClientSecure client;

if (!client.connect("notify-api.line.me", 443)) {     //เชื่อมต่อกับ Line
 Serial.println("connection failed");                 //หากเชื่อมไม่ได้ ให้เริ่มรอบใหม่
 return;
 }

String req = "";
 req += "POST /api/notify HTTP/1.1\r\n";
 req += "Host: notify-api.line.me\r\n";
 req += "Authorization: Bearer " + String(LINE_TOKEN) + "\r\n";
 req += "Cache-Control: no-cache\r\n";
 req += "User-Agent: ESP8266\r\n";
 req += "Content-Type: application/x-www-form-urlencoded\r\n";
 req += "Content-Length: " + String(String("message=" + message).length()) + "\r\n";
 req += "\r\n";
 req += "message=" + message;
 Serial.println(req);
 client.print(req);
 delay(3000);

Serial.println("-------------");
 while (client.connected()) {
 String line = client.readStringUntil('\n');
 if (line == "\r") {
 break;
 }
 Serial.println(line);
 }
 Serial.println("-------------");
}

void loop(){
 {
 digitalWrite(D0, HIGH);  // turn off the LED  
 delay(2000);             // wait for two seconds
 digitalWrite(D0, LOW);   // turn on the LED
 delay(2000);             // wait for two seconds
{
 // กำหนดดีเลย์ก่อนทำการวัดค่า
  delay(30000); // 1 วินาที = 1000 มิลลิวินาที
  float h = dht.readHumidity(); //อ่านค่าความชื้นสัมพัทธ์
  float t = dht.readTemperature(); //อ่านค่าอุณหภูมิ
  float f = dht.readTemperature(true); // แสดงค่าอุณหภูมิเป็นองศาฟาเรนไฮต์ (isFahrenheit = true) ไว้ในตัวแปร f ชนิด float (ทศนิยม)
  float hif = dht.computeHeatIndex(f, h); //แสดงค่าดัชนีความร้อนิเป็นองศาฟาเรนไฮต์
  float hic = dht.computeHeatIndex(t, h, false); // แสดงค่าดัชนีความร้อนเป็นองศาเซลเซียส (isFahreheit = false) ไว้ในตัวแปร hic ชนิด float (ทศนิยม)
  Serial.print("Humidity: ");
  Serial.print(h);
  Serial.print(" %\t");
  Serial.print("Temperature: ");
  Serial.print(t);
  Serial.print(" *C ");
  Serial.println();

  ThingSpeak.writeField(myChannelNumber, 1,t, myWriteAPIKey); //แทนค่าอุณหภูมิไปที่ Field 1 
  ThingSpeak.writeField(myChannelNumber, 2,h, myWriteAPIKey); //แทนค่าความชื้นสัมพันธ์ไปที่ Field 2

 messagetemperature = dtostrf(t, 6, 2, tempF);
 messagehumidity = dtostrf(h, 6, 2, tempF);
if(t > 33){  // กำหนดค่าอุณหภูมิไว้ที่ 28 หากเกินจะทำการส่งข้อความแจ้งเตือนไปทาง Line
   message = "อุณหภูมิขณะนี้สูงกว่าที่กำหนด "+ messagetemperature + messagehumidity ;
    Line_Notify(message);
}else{
    Serial.print("Temperature: "); 
    Serial.print(t);
    Serial.print(" *C ");
    }
// กำหนดค่าอุณหภูมิเพื่อให้ Buzzer ทำงาน
if(t > 33){
  digitalWrite(Buzzer, HIGH);
  delay(3000); // 1 วินาที = 1000 มิลลิวินาที
}else{
  digitalWrite(Buzzer, LOW);
  delay(3000);
    }
  }
}
}
