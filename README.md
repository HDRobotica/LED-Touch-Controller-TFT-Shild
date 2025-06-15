#include <Adafruit_GFX.h>
#include <MCUFRIEND_kbv.h>
MCUFRIEND_kbv tft;
#include <TouchScreen.h>

// Farbdefinitionen
#define BLACK   0x0000
#define BLUE    0x001F
#define RED     0xF800
#define GREEN   0x07E0
#define CYAN    0x07FF
#define MAGENTA 0xF81F
#define YELLOW  0xFFE0
#define WHITE   0xFFFF

// Touchscreen Setup
#define MINPRESSURE 200
#define MAXPRESSURE 1000
const int XP = 8, XM = A2, YP = A3, YM = 9;
TouchScreen ts = TouchScreen(XP, YP, XM, YM, 300);
const int TS_LEFT = 248, TS_RT = 873, TS_TOP = 929, TS_BOT = 213;

// Pins
#define PIEZO 3
#define LED_R 4
#define LED_G 5
#define LED_B 6
#define LED_Y 7

// Zustände
bool ledState[4] = {false, false, false, false};
bool blinkMode = false;
bool touchHandled = false;
unsigned long lastBlink = 0;
bool blinkState = false;
int pixel_x = -1, pixel_y = -1;

Adafruit_GFX_Button btn_led[4];
Adafruit_GFX_Button btn_beep, btn_blink;

const char* btnLabels[4] = {"LED1", "LED2", "LED3", "LED4"};
uint16_t ledColors[4] = {GREEN, BLUE, YELLOW, RED};
int ledPins[4] = {LED_R, LED_G, LED_B, LED_Y};

bool Touch_getXY() {
  TSPoint p;
  do {
    p = ts.getPoint();
    pinMode(YP, OUTPUT);
    pinMode(XM, OUTPUT);
    digitalWrite(YP, HIGH);
    digitalWrite(XM, HIGH);
  } while ((p.z < MINPRESSURE || p.z > MAXPRESSURE));
  pixel_x = map(p.x, TS_LEFT, TS_RT, 0, tft.width());
  pixel_y = map(p.y, TS_TOP, TS_BOT, 0, tft.height());
  delay(300); // verhindert mehrfaches Auslösen
  return true;
}

void playBeep() {
  tone(PIEZO, 1000, 80);
}

void playMusicNote() {
  tone(PIEZO, 523, 200); delay(250);
  tone(PIEZO, 587, 200); delay(250);
  tone(PIEZO, 659, 200); delay(250);
  tone(PIEZO, 523, 200); delay(250);
  noTone(PIEZO);
}

void updateLEDs() {
  for (int i = 0; i < 4; i++) {
    digitalWrite(ledPins[i], ledState[i] ? HIGH : LOW);
  }
}

void drawStatusLamps() {
  int y = 280, radius = 12;
  int x_start = 40;
  for (int i = 0; i < 4; i++) {
    int x = x_start + i * 50;
    if (ledState[i]) {
      tft.fillCircle(x, y, radius, ledColors[i]);
    } else {
      tft.fillCircle(x, y, radius, BLACK);
      tft.drawCircle(x, y, radius, ledColors[i]);
    }
  }
}

void drawButtons() {
  for (int i = 0; i < 4; i++) {
    int x = (i % 2 == 0) ? 60 : 180;
    int y = (i < 2) ? 60 : 120;
    btn_led[i].initButton(&tft, x, y, 100, 40, BLACK, ledColors[i], BLACK, btnLabels[i], 2);
    btn_led[i].drawButton(false);
  }

  btn_beep.initButton(&tft, 60, 180, 100, 40, BLACK, MAGENTA, BLACK, "MUSIK", 2);
  btn_blink.initButton(&tft, 180, 180, 100, 40, BLACK, CYAN, BLACK, "All", 2);
  btn_beep.drawButton(false);
  btn_blink.drawButton(false);
}

void drawHeaderFooter() {
  tft.setTextColor(WHITE);
  tft.setTextSize(2);
  tft.setCursor(10, 10);
  tft.print("HD Robotics");
  tft.setTextSize(1);
  tft.setCursor(90, 310);
  tft.print("hdrobotics.de");
}

void setup() {
  Serial.begin(9600);
  for (int i = 0; i < 4; i++) pinMode(ledPins[i], OUTPUT);
  pinMode(PIEZO, OUTPUT);

  uint16_t ID = 0x9341;
  tft.begin(ID);
  tft.setRotation(0);
  tft.fillScreen(BLACK);

  drawHeaderFooter();
  drawButtons();
  drawStatusLamps();
  updateLEDs();
}

void loop() {
  if (Touch_getXY()) {
    playBeep();

    for (int i = 0; i < 4; i++) {
      if (btn_led[i].contains(pixel_x, pixel_y)) {
        ledState[i] = !ledState[i];
        updateLEDs();
        drawStatusLamps();
      }
    }

    if (btn_beep.contains(pixel_x, pixel_y)) playMusicNote();
    if (btn_blink.contains(pixel_x, pixel_y)) {
      blinkMode = !blinkMode;
      blinkState = false;
    }
  }

  if (blinkMode && millis() - lastBlink > 500) {
    blinkState = !blinkState;
    for (int i = 0; i < 4; i++) {
      digitalWrite(ledPins[i], blinkState ? HIGH : LOW);
    }
    lastBlink = millis();
  }

  if (!blinkMode) updateLEDs();
}
