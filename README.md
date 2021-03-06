In this repository I will post my insights during experimenting with the Raspberry Pi Zero W.

## Contents

1. Preparing the Raspberry Pi
    * Access the Pi on Windows
    * Set a fixed IPv6 address
    * Accessing the internet via the OTG cable
2. Using a PiTFT display
3. Using sensors and other inputs
    * Research about API integration
    * Research about the temperature sensor
    * Research about the LCD display
4. The weather script

## Preparing the Raspberry Pi

This section discusses installing the Pi and configure its networks so I can access the Pi from my host computer and vice versa, as well as connecting to the internet on the Pi.

### Access the Pi on Windows

First install the latest Raspbian Stretch from the Raspberry Pi website and update the packages (using `sudo apt update` and `sudo apt upgrade`). Now to access the Pi on a Windows machine, take the following steps:

1. Boot into the Pi using a display and keyboard.
2. Make sure an SSH server is running (use `raspi-config` to enable).
3. Add the following line to the file `/boot/config.txt`. This will load the correct USB driver in the kernel to use the Pi as an On-The-Go device:

```
dtoverlay=dwc2
```

4. Add the following lines to the file `/etc/modules`. This enables the modules to use the Pi as a virtual network device:

```
dwc2
g_ether
```

5. Reboot the Pi and connect an USB cable from the pc to the USB port of the Pi.

The first time I did this the Pi wasn't discovered as an *USB Ethernet/RNDIS Gadget*, so I followed the steps on [this French website](http://domotique.caron.ws/cartes-microcontroleurs/raspberrypi/pi-zero-otg-ethernet/) to install the correct driver:

6. Download the RNDIS driver provided by the website and extract it.
7. In the Windows Device Manager, locate the Pi, right click on it and select *Properties*.
8. Install the driver by locating it with the *Install driver...*  option.

Now you can SSH into the Pi using its IPv6 address on the `usb0` interface, e.g.:

```shell
ssh pi@fe80::5563:28e9:b94e:8c3c%eth3
```

Note: make sure that your Pi is stored safe, so that the SD card cannot bend or break and you don't have to buy a new SD card and re-install all the software ;).

### Set a fixed IPv6 address

Every time the Pi boots the IPv6 address changes, due to the virtual network that is created. There are some tutorial to set up an automatic DNS service, so that the hostname `raspberrypi.local` is automatically linked to the correct IP address. This is done using a *Zeroconf* or *Bomnjour* service for Windows. However none of the tutorials I found worked, so I decided to give the Pi a fixed IP address, e.g. `169.254.64.64` and `fe80::40:40`.

To set a fixed IPv6 address, add the following lines to the `/etc/dhcpcd.conf` file:

```
interface usb0
static ip_address=169.254.64.64
static ip6_address=fe80::40:40
```

Reboot the Pi. You can now SSH into the Pi using the IPv6 address:

```shell
ssh pi@fe80::40:40
```

### Accessing the internet via the OTG cable

With this settings the Pi is accessible from the host computer, but it does not have access to the internet yet. This is because the RNDIS driver creates an own virtual network on the host computer which is not visible to the main network and vice versa. I tried to overcome this by setting a fixed MAC address (thanks to [this tutorial](https://www.raspberrypi.org/forums/viewtopic.php?f=28&t=199588&p=1245602#p1245457) by adding the following line to the file `/etc/modprobe.d/g_ether.conf`. Replace xx with any hexadecimal numbers that don't match other MAC addresses on your network:

```
options g_ether dev_addr=xx:xx:xx:xx:xx:xx
```

Unfortunately that didn't work. I also tried to create a network bridge from the virtual network to the ethernet network on my pc, but without any luck. So for now I use the `wlan0` WiFi interface to connect to a hotspot created on my pc.


## Using a PiTFT display

I use a [PiTFT Plus 2.8" capacitive touch display](https://www.adafruit.com/product/2298) to easily access the terminal on the Pi Zero. It connects directly to the GPIO pins, so it doesn't need an extra HDMI or USB port.

Unfortunately the [original AdaFruit tutorial](https://learn.adafruit.com/adafruit-pitft-28-inch-resistive-touchscreen-display-raspberry-pi/easy-install) didn't work due to certificate problems. I used [this script](https://forums.adafruit.com/viewtopic.php?f=24&t=54246&sid=19ba7b71b9dcca00a538d2da6d6121e3&start=15#p630193) proviced by AdaFruit support to install the PiTFT.

One caveat is that I cannot use any SPI sensors while using the PiTFT, because the driver claims the SPI drives so that is not accessible to other programs. In the end product I thus did not use the PiTFT display in favor of ohter sensors. Moreover the PiTFT seems to be only useable with a GPIO bridge, because the redirected header pins are hidden between the display and the Pi.


## Using sensors and other inputs

A very handy website is [pinout.xyz](https://pinout.xyz/), which displays a RPi pinout with all harware and GPIO pin numbers and other useful information.

### Research about API integration

For the outside temperature I use the [current weather API of openweathermap.org](http://openweathermap.org/current). I created a Python script to get the data, using the `requests` package:

```python
import requests

url = 'https://api.openweathermap.org/data/2.5/find?q={}'.format(query)
response = requests.get(url, params = {'appid': '6b1a98fea6d95bbb8239e5ab471d5dd7', 'units': 'metric'})
responseJson = response.json()

temperature = responseJson['list'][0]['main']
print(temperature)
```

This script displays the current temperature in degrees Celsius, measured by a weather sation.

### Research about the temperature sensor

The temperature sensor is an analog sensor and the Pi Zero only has digital inputs, thus the circuit needs an analog to digital convertor. Domotix.com has a [nice tutorial](http://domoticx.com/raspberry-pi-temperatuur-sensor-tmp36-gpiomcp3008/) on this topic. It uses the **MCP3008** IC for the analog to digital conversion and the **TMP36** temperature sensor.

Needed components:
* **TMP36** temperature sensor
* **MCP3008** integrated circuit
* 0.1 μF condensator

Wiring (original source: imformit.com):

![TMP36 + MCP3008 wiring diagram](http://ptgmedia.pearsoncmg.com/images/art_blum1_rasppi_analog/elementLinks/blum1_fig02_alt.jpg)

* MCP3008 pin **9** (DGND) to Pi GND
* MCP3008 pin **10** (CS) to Pi SPI0 CE0 (hardware pin 24) or Pi SPI1 CE0 (hardware pin 12)
* MCP3008 pin **11** (DIN) to Pi SPI0 MOSI (hardware pin 19) or Pi SPI1 MOSI (hardware pin 38)
* MCP3008 pin **12** (DOUT) to Pi SPI0 MISO (hardware pin 21) or Pi SPI1 MISO (hardware pin 35)
* MCP3008 pin **13** (CLK) to Pi SPI0 SCLK (hardware pin 23) or Pi SPI1 SCLK (hardware pin 40)
* MCP3008 pin **14** (AGND) to Pi GND
* MCP3008 pin **15** (VREF) and **16** (VDD) to Pi 5V
* One of the TMP36 outer pins to MCP3008 pin **16** (VDD)
* The other of the TMP36 outer pins to MCP3008 pin **14** (AGND)
* The TMP36 middle pin to MCP3008 pin **1** (CH0)
* The 0.1 μF condensator between the TMP36 middle pin and the TMP36 GND pin

For the reading of the MCP3008 I use the [AdaFruit MCP3008 library](https://learn.adafruit.com/raspberry-pi-analog-to-digital-converters/mcp3008).

Example code to test the temperature sensor (`tempSensorTest.py` in the repository):
```python
import time
import Adafruit_GPIO.SPI as SPI
from Adafruit_MCP3008 import MCP3008

# Constants for the SPI connection
spi_port = 0
spi_device = 0
mcp = MCP3008(spi=SPI.SpiDev(spi_port,spi_device))

while True:
  channeldata = mcp.read_adc(0)
  volts = channeldata * (5.0 / 1024.0)
  temperature = (volts - 0.5) * 100.0
  print("Data = {}, Voltage = {:.3f} V, Temperature = {:.1f} °C".format(channeldata,volts,temperature))
  time.sleep(1)
```

In the `tempSensorTest` folder in the repository is a Arduino script that tests the MCP3008 on the Arduino. I used this script to test if the IC was working, because I got strange results when connecting it to the Pi. Eventually I found out that I mixed the Python code of two tutorials, causing the script to read the voltage in millivolts where the calculation was expecting a value in volts. I easily corrected this and the MCP3008 worked as espected.

### Research about the LCD display

I already have an  LCD display laying around for about 3 years so I thought it would be nice to incorporate that. AdaFruit has a [tutorial](https://learn.adafruit.com/character-lcd-with-raspberry-pi-or-beaglebone-black/wiring) on how to connect an LCD display directly on the GPIO pins. One could also connect it through a I2C bridge, but since I don't need many pins I decided to use this manner. Below is the wiring scheme for the LCD:

Needed components:
* LCD display 16x2 characters
* **3362P** 10 kΩ variable potentiometer (or similar)

Wiring (original source: domoticx.com):
* LCD pin **1** (VSS) to Pi GND
* LCD pin **2** (VDD) to Pi 5V
* LCD pin **3** (V0) to the pontentiometer middle pin
* LCD pin **4** (RS) to Pi GPIO 27 (hardware pin 13)
* LCD pin **5** (RW) to Pi GND
* LCD pin **6** (E) to Pi GPIO 22 (hardware pin 15)
* LCD pin **11** (D4) to Pi GPIO 25 (hardware pin 22)
* LCD pin **12** (D5) to Pi GPIO 24 (hardware pin 18)
* LCD pin **13** (D6) to Pi GPIO 23 (hardware pin 16)
* LCD pin **14** (D7) to Pi GPIO 18 (hardware pin 12)
* LCD pin **15** (A) to Pi 5V
* LCD pin **16** (K) to Pi GND
* The potentiometer outer pin to Pi 5V
* The potentiometer inner pin to Pi GND

At first the LCD display didn't work correctly; it didn't receive any data. Re-soldering the header om the Pi resolved this issue, so probably some pins weren't connected properly.

For the interaction with the LCD display I use the [AdaFruit CHarLCD library](https://github.com/adafruit/Adafruit_Python_CharLCD).

Example code to test the display (`lcdTest.py` in the repository):
```python
import Adafruit_CharLCD as LCD

# Raspberry Pi pin configuration
LCD_RS = 27
LCD_EN = 22
LCD_D4 = 25
LCD_D5 = 24
LCD_D6 = 23
LCD_D7 = 18
LCD_BACKLIGHT = 4

# Define LCD column and row size for 16x2 LCD
COLUMNS = 16
ROWS = 2

# Initialize the LCD using the pins above
lcd = LCD.Adafruit_CharLCD(LCD_RS,LCD_EN,LCD_D4,LCD_D5,LCD_D6,LCD_D7,COLUMNS,ROWS,LCD_BACKLIGHT)

# Print a two line message
lcd.message('Hello!\nraspberry')
```

## The weather script

I wanted to make a little weather device that can interact in the following ways:
* It can read the temperature of its environment by using a temperature sensor.
* It can poll the outside temperature by using open weather data.
* Depending on those two temperature variables (and possibly other weather variables) the device outputs algorithmic music.
* The device has a LCD display to indicate its state in a visual way.

I unfortunately didn't have time to work on the sound part, but the temperature sensor, the weather API and the display are working. I use the following script to display date, time and both temperature values (`weather.py` in the repository):

```python
import requests
import time
from Adafruit_GPIO.SPI import SpiDev as SPI
from Adafruit_MCP3008 import MCP3008
from Adafruit_CharLCD import Adafruit_CharLCD as LCD

# Weather device class
class WeatherDevice:
  # Constants for the Pi pins
  lcd_rs = 27
  lcd_en = 22
  lcd_d4 = 25
  lcd_d5 = 24
  lcd_d6 = 23
  lcd_d7 = 18

  # Constants for the SPI connection
  spi_port = 0
  spi_device = 0

  # Other constants
  lcd_columns = 16
  lcd_rows = 2

  # Constructor
  def __init__(self):
    # Create the MCP input
    self.mcp = MCP3008(spi=SPI(self.spi_port,self.spi_device))

    # Create the LCD output
    self.lcd = LCD(self.lcd_rs,self.lcd_en,self.lcd_d4,self.lcd_d5,self.lcd_d6,self.lcd_d7,self.lcd_columns,self.lcd_rows)

  # Poll the current internal temperature
  def poll_internal_temperature(self):
    channeldata = self.mcp.read_adc(0)
    volts = channeldata * (5.0 / 1024.0)
    return (volts - 0.5) * 100.0

  # Poll the current external temperature
  def poll_external_temperature(self, query = "Utrecht, NL"):
    url = "https://api.openweathermap.org/data/2.5/find?q={}".format(query)
    response = requests.get(url, params = {'appid': '6b1a98fea6d95bbb8239e5ab471d5dd7', 'units': 'metric'})
    responseJson = response.json()
    return responseJson['list'][0]['main']['temp']


# Main function
def main():
  # Create the weather device
  device = WeatherDevice()
  device.lcd.create_char(0,[12,18,18,12,0,0,0,0])

  # Create a timer and a state for the LCD
  timer = 0
  state = -1

  # Variables for data storage
  location = "Utrecht, NL"
  int_temp = 0
  ext_temp = 0

  # Main loop
  while True:
    # Poll API data every 30 seconds
    if timer % 30 == 0:
      ext_temp = device.poll_external_temperature(location)

    # Change state every 5 seconds
    if timer % 5 == 0:
      state = (state + 1) % 2

    # Pol lthe internal temperature
    int_temp = device.poll_internal_temperature()

    # Clear the display
    #device.lcd.clear()
    device.lcd.home()

    # Print according to the state
    if state == 0:
      # Print date and time
      device.lcd.message("{0:^16}\n{1:^16}".format(time.strftime("%H:%M:%S"),time.strftime("%d-%m-%Y")))
    elif state == 1:
      # Print temperatures
      device.lcd.message("Ext:  {0:>7.1f} {2}C\nInt:  {1:>7.1f} {2}C".format(ext_temp,int_temp,chr(0)))

    # Increase the timer
    timer += 1

    # Sleep for some time
    time.sleep(1.0)


# Execute the main function
if __name__ == '__main__':
  main()
```
