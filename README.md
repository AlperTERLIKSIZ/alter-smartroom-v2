#include <IRremote.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int IR_PIN = 3;
const int CEILING_PIN = 4;
const int BOOKSHELF_PIN = 5;
const int RGB_PIN = 6;
const int FAN_PIN = 7;
const int BUZZER_PIN = 8;

const int BUTTON_1 = 0x45;
const int BUTTON_2 = 0x46;
const int BUTTON_3 = 0x47;
const int BUTTON_4 = 0x44;
const int BUTTON_5 = 0x40;
const int BUTTON_6 = 0x43;
const int BUTTON_STAR = 0x16;

int activeMode = 0;
bool fanState = false;
bool bookshelfState = false;
bool ceilingState = false;
bool rgbState = false;

String receivedData = "";
int cpuTemp = 0;

unsigned long messageTime = 0;
bool messageActive = false;

LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  Serial.begin(9600);

  pinMode(FAN_PIN, OUTPUT);
  pinMode(BOOKSHELF_PIN, OUTPUT);
  pinMode(CEILING_PIN, OUTPUT);
  pinMode(RGB_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);

  turnAllOff();

  IrReceiver.begin(IR_PIN, ENABLE_LED_FEEDBACK);

  lcd.init();
  lcd.backlight();
  lcd.print("SYSTEM READY");

  buzzerSignal(2);
  updateStatusLine();
}

void loop() {
  unsigned long currentTime = millis();

  if (messageActive && currentTime - messageTime >= 2000) {
    messageActive = false;
    updateStatusLine();
  }

  while (Serial.available() > 0) {
    char character = Serial.read();

    if (character == '\n') {
      cpuTemp = receivedData.toInt();
      receivedData = "";

      if (activeMode == 2 && !messageActive) {
        lcd.setCursor(0, 1);
        lcd.print("CPU: ");
        lcd.print(cpuTemp);
        lcd.print((char)223);
        lcd.print("C    ");
      }
    } else {
      receivedData += character;
    }
  }

  if (IrReceiver.decode()) {
    uint16_t command = IrReceiver.decodedIRData.command;

    if (command == BUTTON_1) {
      activeMode = 1;
      setupStandardMode();
    }
    else if (command == BUTTON_2) {
      activeMode = 2;
      setupGameMode();
    }
    else if (command == BUTTON_STAR) {
      sleepMode();
    }
    else if (command == BUTTON_3) {
      bookshelfState = !bookshelfState;
      digitalWrite(BOOKSHELF_PIN, bookshelfState ? LOW : HIGH);
      showMessage("Bookshelf", bookshelfState);
    }
    else if (command == BUTTON_4) {
      ceilingState = !ceilingState;
      digitalWrite(CEILING_PIN, ceilingState ? LOW : HIGH);
      showMessage("Ceiling", ceilingState);
    }
    else if (command == BUTTON_5) {
      fanState = !fanState;
      digitalWrite(FAN_PIN, fanState ? LOW : HIGH);
      showMessage("Fan", fanState);
    }
    else if (command == BUTTON_6) {
      rgbState = !rgbState;
      digitalWrite(RGB_PIN, rgbState ? LOW : HIGH);
      showMessage("RGB Light", rgbState);
    }

    IrReceiver.resume();
  }
}

void updateStatusLine() {
  if (activeMode == 2) return;

  int lightsOn = 0;

  if (bookshelfState) lightsOn++;
  if (ceilingState) lightsOn++;

  lcd.setCursor(0, 1);
  lcd.print("                ");
  lcd.setCursor(0, 1);
  lcd.print("Lights 2/");
  lcd.print(lightsOn);
  lcd.print("  ");
}

void showMessage(String device, bool state) {
  buzzerSignal(1);

  lcd.setCursor(0, 1);
  lcd.print("                ");
  lcd.setCursor(0, 1);
  lcd.print(device);
  lcd.print(state ? ": ON" : ": OFF");

  messageTime = millis();
  messageActive = true;
}

void setupStandardMode() {
  lcd.backlight();
  buzzerSignal(1);

  bookshelfState = true;
  ceilingState = true;
  rgbState = false;
  fanState = false;

  digitalWrite(BOOKSHELF_PIN, LOW);
  digitalWrite(CEILING_PIN, LOW);
  digitalWrite(RGB_PIN, HIGH);
  digitalWrite(FAN_PIN, HIGH);

  lcd.clear();
  lcd.print("[STANDARD MODE]");
  updateStatusLine();
}

void setupGameMode() {
  lcd.backlight();
  buzzerSignal(1);

  bookshelfState = false;
  ceilingState = false;
  rgbState = true;
  fanState = true;

  digitalWrite(BOOKSHELF_PIN, HIGH);
  digitalWrite(CEILING_PIN, HIGH);
  digitalWrite(RGB_PIN, LOW);
  digitalWrite(FAN_PIN, LOW);

  lcd.clear();
  lcd.print("[GAME MODE]");
}

void sleepMode() {
  buzzerSignal(2);
  activeMode = 0;
  turnAllOff();

  lcd.clear();
  lcd.print("SLEEP MODE...");
  delay(1500);
  lcd.noBacklight();
}

void turnAllOff() {
  digitalWrite(FAN_PIN, HIGH);
  digitalWrite(BOOKSHELF_PIN, HIGH);
  digitalWrite(CEILING_PIN, HIGH);
  digitalWrite(RGB_PIN, HIGH);

  bookshelfState = false;
  ceilingState = false;
  rgbState = false;
  fanState = false;
}

void buzzerSignal(int count) {
  for (int i = 0; i < count; i++) {
    digitalWrite(BUZZER_PIN, HIGH);
    delay(80);
    digitalWrite(BUZZER_PIN, LOW);

    if (count > 1) delay(80);
  }
}
