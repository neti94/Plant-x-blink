#include "ESP8266WiFi.h"
#include <ESP8266mDNS.h>
#include <WiFiClient.h>
#include <BlynkSimpleEsp8266.h>
#include <SimpleTimer.h>
#include <DHT.h>
#include <EEPROM.h>


//Temperature sensor
// Config DHT
#define DHTPIN D1
#define DHTTYPE DHT22
DHT dht(DHTPIN, DHTTYPE);


MDNSResponder mdns;
WiFiServer server(80);
SimpleTimer timer;

//WiFi and Blynk connection variables
char auth[] = "3bb5cd6abf8a419b809cc7f07e46a69b"; // Blynk token "YourAuthToken"
const char* APssid = "ESP8266"; // Name of access point

String st;
String rsid;
String rpass;
boolean newSSID = false;

//Thermostat variables
int TempDes = 70;
int PreviousTempDes = 70;
int TempAct = 70;
int TempCorrection = 0;
int UpdateFrequency = 5000; //Update frequency in milliseconds
float LastRead;

int RelayPin = D2; //Relay pin to turn on fan

int Hysteresis_W = D2; //Summer and Winter hysteresis levels
int Hysteresis_S = D2;

boolean Winter = true; 
boolean Home = true;
boolean FirstRead = true; //flag to cycle DHT until a good first read is made

int MenuItem = 0;

// Attach virtual serial terminal to Virtual Pin V1
WidgetTerminal terminal(V2);


void setup() {
  dht.begin(); //Start temperature sensor
  delay(1000);

  pinMode(RelayPin,OUTPUT);
  digitalWrite(RelayPin,HIGH);
 

  Serial.begin(115200);
  delay(10);
  Serial.println("Startup");
  Serial.println("");
  
  EEPROM.begin(20);  //Get saved temperature correction from EEPROM
  Serial.println("Loading presets from EEPROM");
  GetPresets();

  // if the stored SSID and password connected successfully, exit setup
  if ( testWifi()) {

          //Frequency of temperature reads and updates from Blynk
          timer.setInterval(UpdateFrequency, TempUpdate);
           
          Blynk.config(auth);
          while (Blynk.connect() == false) {
            // Wait until connected
          }
          terminal.println("PRESS SETTINGS BUTTON TO ACCESS MENU");
          terminal.println("");
          terminal.println("");
          terminal.flush();
          return;
      }
  // otherwise, set up an access point to input SSID and password     
  else
      Serial.println("");
      Serial.println("Connect timed out, opening AP"); 
      setupAP();
}

// WiFi connection ***********************************************************
//****************************************************************************

int testWifi(void) {
  int c = 0;
  Serial.println("Waiting for Wifi to connect");  
  while ( c < 20 ) {
    if (WiFi.status() == WL_CONNECTED) {
      Serial.println("WiFi connected.");
      return(1); 
      }      
    delay(500);
    Serial.print(WiFi.status());    
    c++;
  }
  return(0);
} 

void launchWeb(int webtype) {
    Serial.println("");
    Serial.println("WiFi connected");
    Serial.println(WiFi.softAPIP());
    
    // Start the server
    server.begin();
    Serial.println("Server started");   
    int b = 20;
    int c = 0;
    while(b == 20) { 
       b = mdns1(webtype);

       //If a new SSID and Password were sent, close the AP, and connect to local WIFI
       if (newSSID == true){
          newSSID = false;

          //convert SSID and Password sting to char
          char ssid[rsid.length()];
          rsid.toCharArray(ssid, rsid.length());         
          char pass[rpass.length()];
          rpass.toCharArray(pass, rpass.length());

          Serial.println("Connecting to local Wifi");
          delay(500);
    
          WiFi.begin(ssid,pass);
          delay(1000);
          if ( testWifi()) {
 //           Blynk.config(auth);
          ESP.restart();
            return;
          }

         else{
            Serial.println("");
            Serial.println("New SSID or Password failed. Reconnect to server, and try again.");
            setupAP();
            return;
         }
       }
     }
}


void setupAP(void) {
  
  WiFi.mode(WIFI_STA);
  WiFi.disconnect();
  delay(100);
  int n = WiFi.scanNetworks();
  Serial.println("scan done");
  if (n == 0)
    Serial.println("no networks found");
  else
  {
    Serial.print(n);
    Serial.println(" networks found");
  }
  Serial.println(""); 
  st = "<ul>";
  for (int i = 0; i < n; ++i)
    {
      // Print SSID and RSSI for each network found
      st += "<li>";
      st += WiFi.SSID(i);
      st += " (";
      st += WiFi.RSSI(i);
      st += ")";
      st += (WiFi.encryptionType(i) == ENC_TYPE_NONE)?" ":"*";
      st += "</li>";
    }
  st += "</ul>";
  delay(100);
  WiFi.softAP(APssid);
  Serial.println("softAP");
  Serial.println("");
  launchWeb(1);
  WiFi.softAPdisconnect(true); // kill softAP after completing WiFi connection
}


String urldecode(const char *src){ //fix encoding
  String decoded = "";
    char a, b;
    
  while (*src) {     
    if ((*src == '%') && ((a = src[1]) && (b = src[2])) && (isxdigit(a) && isxdigit(b))) {      
      if (a >= 'a')
        a -= 'a'-'A';       
      if (a >= 'A')                
        a -= ('A' - 10);                   
      else               
        a -= '0';                  
      if (b >= 'a')                
        b -= 'a'-'A';           
      if (b >= 'A')                
        b -= ('A' - 10);            
      else                
        b -= '0';                        
      decoded += char(16*a+b);            
      src+=3;        
    } 
    else if (*src == '+') {
      decoded += ' ';           
      *src++;       
    }  
    else {
      decoded += *src;           
      *src++;        
    }    
  }
  decoded += '\0';        
  return decoded;
}


int mdns1(int webtype){
  
  // Check if a client has connected
  WiFiClient client = server.available();
  if (!client) {
    return(20);
  }
  Serial.println("");
  Serial.println("New client");

  // Wait for data from client to become available
  while(client.connected() && !client.available()){
    delay(1);
   }
  
  // Read the first line of HTTP request
  String req = client.readStringUntil('\r');
  
  // First line of HTTP request looks like "GET /path HTTP/1.1"
  // Retrieve the "/path" part by finding the spaces
  int addr_start = req.indexOf(' ');
  int addr_end = req.indexOf(' ', addr_start + 1);
  if (addr_start == -1 || addr_end == -1) {
    Serial.print("Invalid request: ");
    Serial.println(req);
    return(20);
   }
  req = req.substring(addr_start + 1, addr_end);
  Serial.print("Request: ");
  Serial.println(req);
  client.flush(); 
  String s;
  if ( webtype == 1 ) {
      if (req == "/")
      {
        IPAddress ip = WiFi.softAPIP();
        String ipStr = String(ip[0]) + '.' + String(ip[1]) + '.' + String(ip[2]) + '.' + String(ip[3]);
        s = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n<!DOCTYPE HTML>\r\n<html>";
        s += "<font face='arial,helvetica' size='7'>";
        s += "<b><label>Hello from ESP8266 at ";
        s += ipStr;
        s += "</label></b><p>";
        s += st;
        s += "<form method='get' action='a'><label>SSID: </label><input name='ssid' style='width:200px; height:60px; font-size:50px;'>   ";
        s += "<label>Password: </label><input name='pass' style='width:200px; height:60px; font-size:50px;'>";
  //      s += "<p><label>Blynk Token (optional): </label><input name='token' style='width:200px; height:60px; font-size:50px;'>";
        s += "<p><input type='submit' style='font-size:60px'></form>";
        s += "</html>\r\n\r\n";
        Serial.println("Sending 200");
      }
      else if ( req.startsWith("/a?ssid=") ) {

        newSSID = true;
        String qsid; //WiFi SSID 
        qsid = urldecode(req.substring(8,req.indexOf('&')).c_str()); //correct coding for spaces as "+"
        Serial.println(qsid);
        Serial.println("");
        rsid = qsid;
        
        String qpass; //Wifi Password
        qpass = urldecode(req.substring(req.lastIndexOf('=')+1).c_str());//correct for coding spaces as "+"
        Serial.println(qpass);
        Serial.println("");
        rpass = qpass;
 
        s = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n<!DOCTYPE HTML>\r\n<html>";
        s += "<font face='arial,helvetica' size='7'><b>Hello from ESP8266 </b>";
        s += "<p> New SSID and Password received</html>\r\n\r\n"; 
      }
      else
      {
        s = "HTTP/1.1 404 Not Found\r\n\r\n";
        Serial.println("Sending 404");
      }
  } 
  else
  {
      if (req == "/")
      {
        s = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n<!DOCTYPE HTML>\r\n<html>";
        s += "<font face='arial,helvetica' size='7'>Hello from ESP8266";
        s += "<p>";
        s += "</html>\r\n\r\n";
        Serial.println("Sending 200");
      }
      else
      {
        s = "HTTP/1.1 404 Not Found\r\n\r\n";
        Serial.println("Sending 404");
      }       
  }
  client.print(s);
  Serial.println("Done with client");
  return(20);
}

//HVAC Control*********************************************************************
//*********************************************************************************

//Match temp gauge to slider in Blynk app 
BLYNK_WRITE(3){
  TempDes = param.asInt();
  Blynk.virtualWrite(1,TempDes);
}


// Turn the radiator fan on or off 
void TempUpdate (){
  float ReadF = dht.readTemperature(true);
    
  if (isnan(ReadF)) {
    Serial.println("Failed to read from DHT sensor!");
    return;
  }
  
  if (FirstRead == true){
    TempAct = (int)(ReadF + TempCorrection);
    FirstRead = false;
    Serial.print("First temperature reading (corrected): ");
    Serial.println(TempAct);
    LastRead = ReadF;
    return;   
  }
    
  else   { //Read gets averaged with previous read and limited to 1 degree at a time change 
    int TempAvg = (int)((ReadF + LastRead + (2 * TempCorrection))/2);
    if (TempAvg >= TempAct + 1){
      TempAct = TempAct + 1;
    }
    if (TempAvg <= TempAct - 1){
      TempAct = TempAct -1;
    }

    LastRead = ReadF;
  }
  Blynk.virtualWrite(0,TempAct); //Report actual temperature in app
  Serial.print("Actual temperature (corrected): ");
  Serial.println(TempAct);

  if (Winter){
    if (TempAct < TempDes){
      digitalWrite(RelayPin,LOW);
    }
    else if (TempAct >= (TempDes + Hysteresis_W)) {
    digitalWrite(RelayPin,HIGH);
    }
  }
  else if (!Winter){
    if (TempAct > TempDes){
    digitalWrite(RelayPin,LOW);
    }
    else if (TempAct <= (TempDes - Hysteresis_S)){
      digitalWrite(RelayPin,HIGH);
    }
  else{
    digitalWrite(RelayPin,HIGH);
  }
 }

 if (TempDes != PreviousTempDes){ //update the EEPROM if desired temperature had changed.
  EEPROM.write(3,TempDes);
  EEPROM.commit();
  Serial.print("New desired temperature saved to EEPROM: ");
  Serial.println(TempDes);
  PreviousTempDes = TempDes;  
 }
}


// Menu button. Selects settings menu item. 
BLYNK_WRITE(V4) {
  if (param.asInt()){
    MenuItem += 1;
    if (MenuItem > 7){
      MenuItem = 1;
    }
    switch(MenuItem){
      case 1:
        if (Winter){
          terminal.println("Mode: Winter / heating. CHANGE?");
        }
        else terminal.println("Mode: Summer / cooling. CHANGE?");
        break;

      case 2:
        if (Winter){
          terminal.print("Winter hysteresis: ");
          terminal.print(Hysteresis_W);
          terminal.println(" degrees. CHANGE?");   
        }
        else{
          terminal.print("Summer hysteresis: ");
          terminal.print(Hysteresis_S);
          terminal.println(" degrees. CHANGE?");    
        }
        break;

      case 3:
        terminal.print("Sensor correction: ");
        terminal.print(TempCorrection);
        terminal.println("degree(s). CHANGE?");
        break;

      case 4:
        if (Home){
          terminal.println("Location: home. CHANGE?");
        }
        else terminal.println("Location: away. CHANGE?");
        break;

      case 5:
        terminal.println("CLEAR WiFi SETTINGS?");
        break;

      case 6:
         terminal.println("RESET THERMOSTAT DEFAULTS?");
         break;
        
      case 7:
        terminal.println("EXIT SETTINGS?");
    }
  }
  // Move to top of terminal window, and ensure everything is sent
  terminal.println("");
  terminal.println("");
  terminal.flush();
}

//Select button. Executes change of selected menu item 
BLYNK_WRITE(V5){
  if ((MenuItem > 0) && (param.asInt())){
    switch(MenuItem){
      //Change season
      case 1:
        if (Winter){
          terminal.println("Mode: Summer / cooling. CHANGE?");
          Winter = false;
          EEPROM.write(4,0);
          EEPROM.commit();
        }
        else {
          terminal.println("Mode: Winter / heating. CHANGE?");
          Winter = true;
          EEPROM.write(4,1);
          EEPROM.commit();
        } 
        break;
        
      //Change hysteresis level of currently selected season
      case 2:
        if (Winter){
          Hysteresis_W += 1;
          if (Hysteresis_W > 6){
            Hysteresis_W = 1;
          }
          EEPROM.write(1,(Hysteresis_W));
          EEPROM.commit();
          terminal.print("Winter hysteresis: ");
          terminal.print(Hysteresis_W);
          terminal.println(" degrees. CHANGE?");
        }
        else{
          Hysteresis_S += 1;
          if (Hysteresis_S > 6){
            Hysteresis_S = 1;
          }
          EEPROM.write(1,(Hysteresis_S));
          EEPROM.commit();
          terminal.print("Summer hysteresis: ");
          terminal.print(Hysteresis_S);
          terminal.println(" degrees. CHANGE?");
          }
        break;

      case 3:
        TempCorrection +=1;
        if (TempCorrection > 5){
          TempCorrection = -5;
        }
        EEPROM.write(0,(TempCorrection + 5));
        EEPROM.commit();
        terminal.print("Temp Sensor correction: ");
        terminal.print(TempCorrection);
        terminal.println(". CHANGE?");
        break;

      //Change location manually
      case 4:
        if (Home){
          Home = false;
          terminal.println("Location: away. CHANGE");
        }
        else {
          Home = true;
          terminal.println("Location: home. CHANGE?");
        }
        break;

      //Clear stored SSID and password
      case 5:
        terminal.println("Erasing SSID and restarting unit.");
        terminal.flush();
        WiFi.begin("FakeSSID","FakePW"); //replace current WiFi credentials with fake ones
        delay(1000);
        ESP.restart();
        break;

      //Clear current temperature settings
      case 6:
        terminal.println("All settings reset to default.");
        Winter = true;
        Hysteresis_W = 2;
        Hysteresis_S = 2;
        break;

      //Exit Settings menu
      case 7:
        terminal.println("PRESS SETTINGS BUTTON TO ACCESS MENU");
        break;
        
    }
  }
  terminal.println("");
  terminal.println("");
  terminal.flush();
}

// Load available presets from EEPROM 
void GetPresets(){
  TempCorrection = EEPROM.read(0);
  if ((TempCorrection < 0) || (TempCorrection > 10)){
    TempCorrection = 0;
    Serial.println("No saved temperature correction in EEPROM.");
  }
  else{
    TempCorrection -= 5; // 5 was added at EEPROM save to account for negative values
    Serial.print("Temperature correction from EEPROM: ");
    Serial.println(TempCorrection);   
  }

  Hysteresis_W = EEPROM.read(1);
  if ((Hysteresis_W < 2) || (Hysteresis_W > 6)){
    Hysteresis_W = 2;
    Serial.println("No saved Winter hysteresis in EEPROM.");
  }
  else{
    Serial.print("Winter hysteresis from EEPROM: ");
    Serial.println(Hysteresis_W);   
  }

  Hysteresis_S = EEPROM.read(2);
  if ((Hysteresis_S < 2) || (Hysteresis_S > 6)){
    Hysteresis_S = 2;
    Serial.println("No saved Summer hysteresis in EEPROM.");
  }
  else{
    Serial.print("Winter hysteresis from EEPROM: ");
    Serial.println(Hysteresis_S);   
  }
  TempDes = EEPROM.read(3);
  if ((TempDes < 50) || (TempDes > 80)){
    TempDes = 70;
    Serial.println("No deisred temperature in EEPROM. Default temp setting: 70");
  }
  else {
    Serial.print("Desired temperature from EEPROM: ");
    Serial.println(TempDes);
  }
  PreviousTempDes = TempDes;
  Winter = EEPROM.read(4);
  if (Winter){
    Serial.println("Season setting from EEPROM: Winter / heating");
  }
  else Serial.println("Season setting from EEPROM: Summer / cooling");
}


void loop() {

  Blynk.run();
  timer.run();
  yield();
}
