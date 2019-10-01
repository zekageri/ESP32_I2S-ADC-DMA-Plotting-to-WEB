#include <Arduino.h>
#include <WebSocketsServer.h>
#include <WiFi.h>
#include <WebAuthentication.h>
#include <AsyncTCP.h>
#include <ESPAsyncWebServer.h>
#define FS_NO_GLOBALS
#include <FS.h>
#include "SPIFFS.h"
#include <driver/i2s.h>
#include <driver/adc.h>
extern "C" {
    #include "soc/syscon_reg.h"
    #include "soc/syscon_struct.h"
}

AsyncWebServer    server(80);
WebSocketsServer  webSocket = WebSocketsServer(81);
AsyncEventSource  events("/events");
AsyncWebServerRequest *request;
AsyncWebSocket    ws("/test");

#define HTTP_USER     "admin"
#define HTTP_PASS     "1234"
#define HTTP_REALM    "realm"

/** ANALOG DIGITAL CONVERTER SETTINGS DMA **/
#define ADC_CHANNEL   ADC1_CHANNEL_0 //GPIO36
int samples = 10500;
int NUM_SAMPLES = samples;
int samplingFrequency = 44000;
uint32_t collected_samples = 0;
uint32_t last_sample_count = 0;
uint16_t adc_reading;
TaskHandle_t TaskHandle_2;
#define DUMP_INTERVAL 1000           // dump samples at this interval
int32_t  last_millis = DUMP_INTERVAL;
size_t bytes_read;
long Start_Sending_Millis = 0;
boolean olvasunk_e       = true;   // ADC enabler from web


AsyncWebSocketClient * globalClient = NULL;
void onWsEvent(AsyncWebSocket * server, AsyncWebSocketClient * client, AwsEventType type, void * arg, uint8_t *data, size_t len){
  if(type == WS_EVT_CONNECT){
    Serial.println("Async_WebSocket_Connected");
    globalClient = client;
  } else if(type == WS_EVT_DISCONNECT){
    globalClient = NULL;
    Serial.println("Async_WebSocket_Disconnected");
  }
}

void serverons(){

server.on("/", HTTP_GET, [](AsyncWebServerRequest *request) {
    if(!request->authenticate(generateDigestHash(HTTP_USER, HTTP_PASS, HTTP_REALM).c_str())) {
      return request->requestAuthentication(HTTP_REALM, true);
    }
    AsyncWebServerResponse* response = request->beginResponse(SPIFFS, "/Canvas_Test.html.gz", "text/html");
    response->addHeader("Content-Encoding", "gzip");
    request->send(response);
  });

server.on("/favicon.ico", HTTP_GET, [](AsyncWebServerRequest *request){
  request->send(SPIFFS, "/favicon.ico","text/css");
});

server.on("/jquery.js", HTTP_GET, [](AsyncWebServerRequest* request) {
        AsyncWebServerResponse* response = request->beginResponse(SPIFFS, "/jquery.js.gz", "text/javascript");
        response->addHeader("Content-Encoding", "gzip");
        request->send(response);
    });
server.on("/CanvasJs", HTTP_GET, [](AsyncWebServerRequest* request) {
        AsyncWebServerResponse* response = request->beginResponse(SPIFFS, "/CanvasJs.js.gz", "text/javascript");
        response->addHeader("Content-Encoding", "gzip");
        request->send(response);
    });
server.on("/MeresStop", HTTP_GET, [](AsyncWebServerRequest * request) {
    request->send(204);
    olvasunk_e = false;
  });

server.on("/MeresOk", HTTP_GET, [](AsyncWebServerRequest * request) {
    request->send(204);
    olvasunk_e = true;
  });
}

#define I2S_NUM         (0)

void configure_i2s(){
  i2s_config_t i2s_config = 
    {
    .mode = (i2s_mode_t)(I2S_MODE_MASTER | I2S_MODE_RX | I2S_MODE_ADC_BUILT_IN),  // I2S receive mode with ADC
    .sample_rate = samplingFrequency,                                             // sample rate
    .bits_per_sample = I2S_BITS_PER_SAMPLE_16BIT,                                 // 16 bit I2S
    .channel_format = I2S_CHANNEL_FMT_ALL_LEFT,                                   // only the left channel
    .communication_format = (i2s_comm_format_t)(I2S_COMM_FORMAT_I2S | I2S_COMM_FORMAT_I2S_MSB),   // I2S format
    .intr_alloc_flags = 1,                                                        // none
    .dma_buf_count = 8,                                                           // number of DMA buffers
    .dma_buf_len = 1000,                                                   // number of samples
    .use_apll = 0,                                                                // no Audio PLL
  };
  adc1_config_channel_atten(ADC_CHANNEL, ADC_ATTEN_11db);
  adc1_config_width(ADC_WIDTH_12Bit);
  i2s_driver_install(I2S_NUM_0, &i2s_config, 0, NULL);
 
  i2s_set_adc_mode(ADC_UNIT_1, ADC_CHANNEL);
  SET_PERI_REG_MASK(SYSCON_SARADC_CTRL2_REG, SYSCON_SARADC_SAR1_INV);
  i2s_adc_enable(I2S_NUM_0);
}

static const inline void Plotting(uint16_t* Buffer){
  String data;
  long Millis_Now = millis();
  if ((Millis_Now-Start_Sending_Millis) >= DUMP_INTERVAL)
  {
    Start_Sending_Millis = Millis_Now;
    for(int i = 0;i<NUM_SAMPLES;i++)
    {
      data = String((Buffer[i])/(float)40.95);
      if(olvasunk_e){
      data += "}";
      webSocket.broadcastTXT(data.c_str(), data.length());
      }
    }
  }
}

static const inline void Sampling(){
  uint16_t i2s_read_buff[NUM_SAMPLES];
  i2s_read(I2S_NUM_0, (char*)i2s_read_buff,NUM_SAMPLES * sizeof(uint16_t), &bytes_read, portMAX_DELAY);
   if(I2S_EVENT_RX_DONE){
     Plotting(i2s_read_buff);
   }
}

void WIFI_Setup(){
  WiFi.disconnect();
  WiFi.mode(WIFI_AP);
  WiFi.softAP("ADC","12345678");
  Serial.print("WiFi IP: ");
  Serial.println(WiFi.softAPIP());
}

void setup() {
  configure_i2s();
  Serial.begin(115200);
  SPIFFS.begin() ? Serial.println("SPIFFS.OK") : Serial.println("SPIFFS.FAIL");
  WIFI_Setup();
  server.begin();
  serverons();
  ws.onEvent(onWsEvent);
  server.addHandler(&ws);
  webSocket.begin();
  xTaskCreatePinnedToCore ( loop0, "v_getIMU0", 50000, NULL, 0, &TaskHandle_2, 0 );
  Start_Sending_Millis  = millis();
}

static void loop0(void * pvParameters){
  for( ;; ){
    vTaskDelay(1);          // REQUIRED TO RESET THE WATCH DOG TIMER IF WORKFLOW DOES NOT CONTAIN ANY OTHER DELAY
    Sampling();
  }
}

void loop() {
  webSocket.loop();       // WEBSOCKET PACKET LOOP
}
