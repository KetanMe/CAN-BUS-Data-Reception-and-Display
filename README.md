# CAN-BUS-Data-Reception-and-Display

## Table of Contents
- [CAN-BUS-Data-Reception-and-Display](#can-bus-data-reception-and-display)
  - [Introduction](#introduction)
  - [Oled Display](#oled-display)
  - [Pin Connections](#pin-connections)
  - [Code](#code)
  - [Explaination of code](#explaination-of-code)
  - [Output](#output)


## Introduction
The data transmitted via the BUS in [UNIT 4](https://github.com/KetanMe/RPM-speed-and-tempreature-sending-using-CAN/tree/main?tab=readme-ov-file#can-bus-data-transmission-rpm-speed-temperature) is received at the receiving end. Subsequently, this data is printed on the serial monitor and simultaneously displayed on the OLED display.

## Oled Display

![image](https://github.com/KetanMe/CAN-BUS-Data-Reception-and-Display/assets/121623546/a58e30f2-1336-4b26-b586-f6ffae853c40)

This 1.3″ I2C OLED Display is an OLED monochrome 128×64 dot matrix display module with I2C Interface. It is perfect when you need an ultra-small display. Comparing to LCD, OLED screens are way more competitive, which has a number of advantages such as high brightness, self-emission, high contrast ratio, slim outline, wide viewing angle, wide temperature range, and low power consumption. It is compatible with any 3.3V-5V microcontroller, such as Arduino.

**I2C** [Read here](https://www.circuitbasics.com/basics-of-the-i2c-communication-protocol/)

## Pin Connections

| OLED Display Pin | ESP8266 Pin | Function      |
|------------------|-------------|---------------|
| VCC              | 3.3V        | Power (3.3V)  |
| GND              | GND         | Ground        |
| SDA              | D1 (GPIO5)  | I2C Data      |
| SCL              | D2 (GPIO4)  | I2C Clock     |


## Code 
```cpp
#include <Wire.h>
#include <mcp2515.h>
#include <U8g2lib.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define MAX_TEMP_VALUES 10

const int spiCS = D8;

MCP2515 mcp2515Receiver(spiCS);

struct can_frame canMsg;

U8G2_SSD1306_128X64_NONAME_F_HW_I2C display(U8G2_R0, /* reset=*/ U8X8_PIN_NONE);

float lastReceivedRPM = 0.0;
float tempValues[MAX_TEMP_VALUES];
int tempCount = 0;

void drawTemps() {
  display.setFont(u8g2_font_profont12_mf);
  int xPosition = 2;
  display.setCursor(xPosition, 60);
  for (int i = 0; i < tempCount; ++i) {
    display.print(tempValues[i], 2);
    if (i < tempCount - 1) {
      display.print("  ");
    }
  }
}

void draw() {
  display.setFont(u8g2_font_profont29_mf);
  char rpmCharArray[10];
  dtostrf(lastReceivedRPM, 1, 0, rpmCharArray);
  int rpmTextWidth = display.getStrWidth(rpmCharArray);
  int xPosition = (SCREEN_WIDTH - rpmTextWidth) / 2;
  int yPosition = 30;
  display.setCursor(xPosition, yPosition);
  display.print(rpmCharArray);
}

void setup() {
  Serial.begin(9600);
  display.begin();
  mcp2515Receiver.reset();
  if (mcp2515Receiver.setBitrate(CAN_500KBPS, MCP_8MHZ) != MCP2515::ERROR_OK) {
    Serial.println("Error setting bitrate!");
    while (1);
  }
  mcp2515Receiver.setNormalMode();
}

void loop() {
  if (mcp2515Receiver.readMessage(&canMsg) == MCP2515::ERROR_OK) {
    switch (canMsg.can_dlc) {
      case 8:
        float receivedRPM, receivedSpeed;
        memcpy(&receivedRPM, canMsg.data, sizeof(receivedRPM));
        memcpy(&receivedSpeed, canMsg.data + sizeof(receivedRPM), sizeof(receivedSpeed));
        lastReceivedRPM = receivedRPM;
        display.clearBuffer();
        draw();
        drawTemps();
        display.sendBuffer();
        delay(1000);
        break;

      case 4:
        float receivedTemp;
        memcpy(&receivedTemp, canMsg.data, sizeof(receivedTemp));
        if (tempCount < MAX_TEMP_VALUES) {
          tempValues[tempCount] = receivedTemp;
          tempCount++;
        } else {
          for (int i = 0; i < MAX_TEMP_VALUES - 1; ++i) {
            tempValues[i] = tempValues[i + 1];
          }
          tempValues[MAX_TEMP_VALUES - 1] = receivedTemp;
        }
        display.clearBuffer();
        draw();
        drawTemps();
        display.sendBuffer();
        delay(1000);
        break;

      default:
        break;
    }
  }
}
```
##  Explaination of code

Sure, let's break down the code part by part and explain each section:

### Libraries and Constants
```cpp
#include <Wire.h>
#include <mcp2515.h>
#include <U8g2lib.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define MAX_TEMP_VALUES 10
```
- This section includes necessary libraries: `Wire.h` for I2C communication, `mcp2515.h` for CAN communication, and `U8g2lib.h` for interfacing with the OLED display.
- Constants are defined for the screen width, height, and maximum temperature values to store.

### Global Variables
```cpp
const int spiCS = D8;

MCP2515 mcp2515Receiver(spiCS);

struct can_frame canMsg;

U8G2_SSD1306_128X64_NONAME_F_HW_I2C display(U8G2_R0, /* reset=*/ U8X8_PIN_NONE);

float lastReceivedRPM = 0.0;
float tempValues[MAX_TEMP_VALUES];
int tempCount = 0;
```
- `spiCS` holds the chip select pin for the MCP2515 CAN controller.
- An instance of `MCP2515` class named `mcp2515Receiver` is created with the chip select pin.
- `can_frame` struct `canMsg` is declared to hold CAN messages.
- An OLED display object named `display` is created for a 128x64 resolution display.
- Variables are declared to store the last received RPM, an array for temperature values, and a counter for the number of stored temperature values.

### Function Definitions
#### `drawTemps()`
```cpp
void drawTemps() {
  display.setFont(u8g2_font_profont12_mf);
  int xPosition = 2;
  display.setCursor(xPosition, 60);
  for (int i = 0; i < tempCount; ++i) {
    display.print(tempValues[i], 2);
    if (i < tempCount - 1) {
      display.print("  ");
    }
  }
}
```
- This function sets the font size to a fixed-width font (`u8g2_font_profont12_mf`) suitable for displaying temperatures.
- It positions the cursor at (2, 60) on the display.
- It loops through the stored temperature values and prints each value with two decimal places.
- Between each temperature value, it prints two spaces, except for the last value.

#### `draw()`
```cpp
void draw() {
  display.setFont(u8g2_font_profont29_mf);
  char rpmCharArray[10];
  dtostrf(lastReceivedRPM, 1, 0, rpmCharArray);
  int rpmTextWidth = display.getStrWidth(rpmCharArray);
  int xPosition = (SCREEN_WIDTH - rpmTextWidth) / 2;
  int yPosition = 30;
  display.setCursor(xPosition, yPosition);
  display.print(rpmCharArray);
}
```
- This function sets the font size to a larger fixed-width font (`u8g2_font_profont29_mf`) suitable for displaying RPM values.
- It converts the last received RPM value to a character array with one decimal place using `dtostrf()` function.
- It calculates the width of the RPM text to center it horizontally on the display.
- It positions the cursor at the calculated position and prints the RPM value.

### Setup Function
```cpp
void setup() {
  Serial.begin(9600);
  display.begin();
  mcp2515Receiver.reset();
  if (mcp2515Receiver.setBitrate(CAN_500KBPS, MCP_8MHZ) != MCP2515::ERROR_OK) {
    Serial.println("Error setting bitrate!");
    while (1);
  }
  mcp2515Receiver.setNormalMode();
}
```
- Serial communication is initialized at a baud rate of 9600.
- The OLED display is initialized.
- The MCP2515 CAN controller is reset, and the bitrate is set to 500 kbps with an 8MHz clock.
- If there's an error setting the bitrate, an error message is printed, and the program enters an infinite loop.
- Finally, the MCP2515 controller is set to normal mode.

### Loop Function
```cpp
void loop() {
  if (mcp2515Receiver.readMessage(&canMsg) == MCP2515::ERROR_OK) {
    switch (canMsg.can_dlc) {
      case 8:
        // Handling RPM and Speed data
        // ...
        break;

      case 4:
        // Handling Temperature data
        // ...
        break;

      default:
        break;
    }
  }
}
```
- This function continuously checks for incoming CAN messages.
- If a message is successfully received, it checks the DLC (Data Length Code) of the message.
- Depending on the DLC, it handles RPM and Speed data or Temperature data.
- Inside each case, the appropriate data is extracted from the CAN message, and the display is updated with the new values.

## Output


https://github.com/KetanMe/CAN-BUS-Data-Reception-and-Display/assets/121623546/8575dc22-a704-4db5-ba40-d36013858855

