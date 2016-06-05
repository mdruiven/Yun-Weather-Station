/* Mike Druiven, June 4, 2016
 * Source code taken from Internet of Things with the Arduino Yun by Marco Schwartz
 * and Save Arduino Yun data to a spreadsheet at https://temboo.com/arduino/yun/update-google-spreadsheet
 * Build a weather statio using a photoresistor for light level and a DHT22 for temperature and humidity.
 * Logs data from station to google spreadheet
 */


#include <Bridge.h> //interaction between processor and microcontroller on Yun
#include <Temboo.h>
#include <Process.h>
#include "TembooAccount.h" 
#include "DHT.h"

// Variables
int lightLevel;
float humidity;
float temperature;
unsigned long time;
Process date;  // Process to get the measurement time

// Your Google Docs data is appended to TembooAccount.h
// GOOGLE_CLIENT_ID = "######.apps.googleusercontent.com"
// GOOGLE_CLIENT_SECRET = "######"
// GOOGLE_REFRESH_TOKEN = "######"
// the title of the spreadsheet you want to send data to
// (Note that this must actually be the title of a Google spreadsheet
// that exists in your Google Drive/Docs account, and is configured
// as described above.)
const String SPREADSHEET_TITLE = "Yun";

// DHT22 sensor pins
#define DHTPIN 8 
#define DHTTYPE DHT22

// DHT instance
DHT dht(DHTPIN, DHTTYPE);

// Debug mode ?
boolean debug_mode = false;

void setup() {
  // Start Serial
  if (debug_mode == true){
    Serial.begin(115200);
    delay(4000);
    while(!Serial);
  }
  
  // Initialize DHT sensor
  dht.begin();
  
  // Start bridge
  Bridge.begin();
  
  // Start date process
  time = millis();
  if (!date.running())  {
    date.begin("date");
    date.addParameter("+%D-%T");
    date.run();
  }

  if (debug_mode == true){
    Serial.println("Setup complete. Waiting for sensor input...\n");
  }
} // end setup

void loop() {
  
  // Measure the humidity & temperature
  humidity = dht.readHumidity();
  temperature = dht.readTemperature();
    
  // Measure light level
  int lightLevel = analogRead(A0);

  if (debug_mode == true){
    Serial.println("\nCalling the /Library/Google/Spreadsheets/AppendRow Choreo...");
  }

  // Get Date data
  if(!date.running()){
    date.begin("date");
    date.addParameter("+%D-%T");
    date.run();
  String now = date.readString();
  
 // we need a Process object to send a Choreo request to Temboo
   TembooChoreo AppendRowChoreo;

    // invoke the Temboo client
    // NOTE that the client must be reinvoked and repopulated with
    // appropriate arguments each time its run() method is called.
    AppendRowChoreo.begin();
    
    // set Temboo account credentials from TembooAccount.h
    AppendRowChoreo.setAccountName(TEMBOO_ACCOUNT);
    AppendRowChoreo.setAppKeyName(TEMBOO_APP_KEY_NAME);
    AppendRowChoreo.setAppKey(TEMBOO_APP_KEY);
    
    // identify the Temboo Library choreo to run (Google > Spreadsheets > AppendRow)
    AppendRowChoreo.setChoreo("/Library/Google/Spreadsheets/AppendRow");
    
    // set the required Choreo inputs
    // see https://www.temboo.com/library/Library/Google/Spreadsheets/AppendRow/ 
    // for complete details about the inputs for this Choreo
    
    // your Google Client ID from TembooAccount.h
    AppendRowChoreo.addInput("ClientID", GOOGLE_CLIENT_ID);

    // your Google Client Secret
    AppendRowChoreo.addInput("ClientSecret", GOOGLE_CLIENT_SECRET);

    // your Google Refresh Token
    AppendRowChoreo.addInput("RefreshToken", GOOGLE_REFRESH_TOKEN);

    // the title of the spreadsheet you want to append to
    AppendRowChoreo.addInput("SpreadsheetTitle", SPREADSHEET_TITLE);

    // convert the time and sensor values to a comma separated string
    String rowData(now);
    rowData += ",";
    rowData += temperature;
    rowData += ",";
    rowData += humidity;
    rowData += ",";
    rowData += lightLevel;

    // add the RowData input item
    AppendRowChoreo.addInput("RowData", rowData);

    // run the Choreo and wait for the results
    // The return code (returnCode) will indicate success or failure 
    unsigned int returnCode = AppendRowChoreo.run();

    // return code of zero (0) means success
    if (debug_mode == true){
    if (returnCode == 0) {
      Serial.println("Success! Appended " + rowData);
      Serial.println("");
    } else {
      // return code of anything other than zero means failure  
      // read and display any error messages
      while (AppendRowChoreo.available()) {
        char c = AppendRowChoreo.read();
        Serial.print(c);
      }
    }
    }
    AppendRowChoreo.close();
  }       
  // Repeat every 10 minutes
  delay(600000);
} // end main loop
