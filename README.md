#include <Arduino.h>
#include <esp_task_wdt.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_adc_cal.h"
#include <SPI.h>
#include "pin_config.h"
#include <WiFi.h>
#include <HTTPClient.h>
#include <ArduinoJson.h>
#include <TFT_eSPI.h>
#include <time.h>

String ssid = "Ludwigs iPhone (2)";
String password = "123456789";

TFT_eSPI tft = TFT_eSPI();
#define DISPLAY_WIDTH 320
#define DISPLAY_HEIGHT 170

WiFiClient wifi_client;

int screen_state = 0; // 0 = boot, 1 = forecast, 2 = menu
unsigned long boot_start = 0;

void drawBootScreen() {
  tft.fillScreen(TFT_BLACK);
  tft.setTextColor(TFT_CYAN, TFT_BLACK);
  tft.setTextSize(2);
  tft.drawString("WeatherApp v1.0", 40, 50);
  tft.drawString("Team 22", 110, 80);
}

void drawForecast(String temp) {
  tft.fillScreen(TFT_BLACK);
  tft.setTextColor(TFT_WHITE, TFT_BLACK);
  tft.setTextSize(2);
  tft.drawString("Forecast for:", 10, 10);
  tft.drawString("Karlskrona", 10, 35);
  tft.drawString("Temp: " + temp + " C", 10, 70);
}

void drawMenu() {
  tft.fillScreen(TFT_BLACK);
  tft.setTextColor(TFT_YELLOW, TFT_BLACK);
  tft.setTextSize(2);
  tft.drawString("1. Forecast", 10, 30);
  tft.drawString("2. Settings", 10, 60);
}

String fetchTemperature() {
  HTTPClient http;
  http.begin(wifi_client, "https://opendata-download-metfcst.smhi.se/api/category/pmp3g/version/2/geotype/city/name/Karlskrona/forecast.json");
  int httpCode = http.GET();
  if (httpCode > 0) {
    String payload = http.getString();
    DynamicJsonDocument doc(20000);
    deserializeJson(doc, payload);
    for (JsonObject timeSeries : doc["timeSeries"].as<JsonArray>()) {
      for (JsonObject param : timeSeries["parameters"].as<JsonArray>()) {
        if (param["name"] == "t") {
          float temp = param["values"][0];
          return String(temp);
        }
      }
    }
  }
  return "--";
}

void setup() {
  Serial.begin(115200);
  while (!Serial);
  Serial.println("Starting ESP32 program...");
  tft.init();
  tft.setRotation(1);
  pinMode(PIN_BUTTON_1, INPUT_PULLUP);
  pinMode(PIN_BUTTON_2, INPUT_PULLUP);

  tft.fillScreen(TFT_BLACK);
  tft.setTextColor(TFT_WHITE, TFT_BLACK);
  tft.drawString("Connecting to WiFi...", 10, 10);

  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.println("Attempting to connect to WiFi...");
  }

  tft.fillScreen(TFT_BLACK);
  tft.setTextColor(TFT_GREEN, TFT_BLACK);
  tft.drawString("Connected to WiFi", 10, 10);
  Serial.println("Connected to WiFi");
  delay(1000);

  drawBootScreen();
  boot_start = millis();
}

void loop() {
  static bool button1_pressed = false;
  static bool button2_pressed = false;

  if (millis() - boot_start < 3000 && screen_state == 0) {
    return; // Show boot screen for 3 sec
  }

  if (screen_state == 0) {
    screen_state = 1;
    drawForecast(fetchTemperature());
  }

  if (!digitalRead(PIN_BUTTON_1)) {
    if (!button1_pressed) {
      screen_state++;
      if (screen_state > 2) screen_state = 1;
      if (screen_state == 2) drawMenu();
      else if (screen_state == 1) drawForecast(fetchTemperature());
      button1_pressed = true;
    }
  } else button1_pressed = false;

  if (!digitalRead(PIN_BUTTON_2)) {
    static unsigned long press_start = 0;
    if (press_start == 0) press_start = millis();
    else if (millis() - press_start > 2000) {
      drawMenu();
      screen_state = 2;
    }
    button2_pressed = true;
  } else {
    button2_pressed = false;
  }
}


// TFT Pin check
//////////////////
// DO NOT TOUCH //
//////////////////
#if PIN_LCD_WR  != TFT_WR || \
    PIN_LCD_RD  != TFT_RD || \
    PIN_LCD_CS    != TFT_CS   || \
    PIN_LCD_DC    != TFT_DC   || \
    PIN_LCD_RES   != TFT_RST  || \
    PIN_LCD_D0   != TFT_D0  || \
    PIN_LCD_D1   != TFT_D1  || \
    PIN_LCD_D2   != TFT_D2  || \
    PIN_LCD_D3   != TFT_D3  || \
    PIN_LCD_D4   != TFT_D4  || \
    PIN_LCD_D5   != TFT_D5  || \
    PIN_LCD_D6   != TFT_D6  || \
    PIN_LCD_D7   != TFT_D7  || \
    PIN_LCD_BL   != TFT_BL  || \
    TFT_BACKLIGHT_ON   != HIGH  || \
    170   != TFT_WIDTH  || \
    320   != TFT_HEIGHT
#error  "Error! Please make sure <User_Setups/Setup206_LilyGo_T_Display_S3.h> is selected in <TFT_eSPI/User_Setup_Select.h>"
#endif

#if ESP_IDF_VERSION >= ESP_IDF_VERSION_VAL(5,0,0)
#error  "The current version is not supported for the time being, please use a version below Arduino ESP32 3.0"
#endif

