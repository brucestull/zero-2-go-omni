# Zero2Go Omini Board Setup

[Zero2Go Omini Setup](https://chatgpt.com/share/67aa4a66-2e44-8002-8ddc-ad1dd7bf2434)

## The Hardware

- Raspberry Pi Zero 2 W
- [Zero2Go Omini – Multi-Channel Power Supply for Raspberry Pi](https://www.adafruit.com/product/4114)

Below is one common approach to getting your Raspberry Pi 4 to “wake up” when you press the Zero2Go Omini’s power‐button and then run a safe shutdown when the vehicle’s 12 V disappears (for example, when the engine is turned off).

> **Note:** The Zero2Go Omini is a hardware power‐management board. It is designed so that when you press its power button it “latches” the 5 V supply to the Pi. In your installation you wire one of its signal outputs (typically a “vehicle power detect” or “IGN” signal) to one of the Pi’s GPIO pins. Then, when the 12 V supply is removed (i.e. when the vehicle is turned off) the board can trigger a safe shutdown by way of your custom code before it finally cuts the power. (Please consult the Zero2Go Omini manual for the exact wiring details and signal polarity.) 

The basic idea is:

1. **Hardware:**  
   - **Power On:** The built-in button on the Zero2Go Omini turns on the 5 V supply to the Pi when pressed.  
   - **Shutdown Signal:** The board provides a signal (for example, a “vehicle power” signal) that indicates if the 12 V is present. (Typically, when the 12 V is present the GPIO reads HIGH; when it is absent it reads LOW—but verify this against your wiring.)  
   - Wire that signal to one of the Pi’s GPIO pins (for example, GPIO17).

2. **Software:**  
   Write a Python script that monitors the chosen GPIO pin. When the script sees that the signal has changed (i.e. 12 V is lost) and stays low (after a debounce period), it issues a shutdown command to the Pi.

3. **Packages:**  
   On Raspberry Pi OS, you can use either the low-level `RPi.GPIO` package or the higher-level `gpiozero` package. (Both are typically pre-installed on Raspberry Pi OS, but if needed you can install them via apt.) For example:  
   ```bash
   sudo apt-get update
   sudo apt-get install python3-gpiozero python3-rpi.gpio
   ```

Below are two example scripts.

---

### Example 1: Using RPi.GPIO

Create a file called `vehicle_shutdown.py` (for example in `/home/pi/`):

```python
#!/usr/bin/env python3
import RPi.GPIO as GPIO
import os
import time

# Use Broadcom pin numbering
GPIO.setmode(GPIO.BCM)

# Change this to match the GPIO pin you use for the 12V (vehicle power) signal
VEHICLE_POWER_PIN = 17

# Set up the pin as an input with an internal pull-up resistor.
GPIO.setup(VEHICLE_POWER_PIN, GPIO.IN, pull_up_down=GPIO.PUD_UP)

def shutdown_callback(channel):
    print("Vehicle power signal lost. Waiting for debounce...")
    time.sleep(2)  # Debounce delay; adjust as needed
    # Confirm that the signal is still low
    if GPIO.input(VEHICLE_POWER_PIN) == GPIO.LOW:
        print("Confirmed loss of 12V. Initiating shutdown.")
        os.system("sudo shutdown -h now")
    else:
        print("Vehicle power restored. Aborting shutdown.")

# Set up event detection on the falling edge (i.e. when 12V goes away)
GPIO.add_event_detect(VEHICLE_POWER_PIN, GPIO.FALLING,
                      callback=shutdown_callback, bouncetime=200)

print("Monitoring vehicle power on GPIO{}".format(VEHICLE_POWER_PIN))
try:
    while True:
        time.sleep(1)
except KeyboardInterrupt:
    print("Exiting and cleaning up.")
finally:
    GPIO.cleanup()
```

Make the script executable:

```bash
chmod +x /home/pi/vehicle_shutdown.py
```

---

### Example 2: Using gpiozero

Alternatively, using the higher-level gpiozero library, create a file (say `vehicle_shutdown_gpiozero.py`):

```python
#!/usr/bin/env python3
from gpiozero import Button
import os
import time

# Change this to the GPIO pin connected to the 12V detect signal.
VEHICLE_POWER_PIN = 17

# Instantiate a Button (which assumes a pull-up resistor by default).
vehicle_power = Button(VEHICLE_POWER_PIN, pull_up=True, bounce_time=0.1)

def shutdown_sequence():
    print("Vehicle power lost. Waiting 2 seconds for confirmation...")
    time.sleep(2)  # Delay to debounce/confirm
    if not vehicle_power.is_pressed:
        print("Confirmed 12V loss. Shutting down now!")
        os.system("sudo shutdown -h now")
    else:
        print("Vehicle power restored. Shutdown aborted.")

print("Monitoring vehicle power on GPIO{}".format(VEHICLE_POWER_PIN))
# This loop waits until the button (i.e. 12V signal) is released (goes low)
while True:
    vehicle_power.wait_for_release()
    shutdown_sequence()
```

Make this script executable as well:

```bash
chmod +x /home/pi/vehicle_shutdown_gpiozero.py
```

---

### Running on Boot

To have your shutdown-monitoring script run automatically on boot, you can set it up as a systemd service. For example, create a file called

`/etc/systemd/system/vehicle_shutdown.service`

with the following contents (adjust paths as needed):

```ini
[Unit]
Description=Vehicle Power Monitor Shutdown Service
After=network.target

[Service]
ExecStart=/usr/bin/python3 /home/pi/vehicle_shutdown.py
WorkingDirectory=/home/pi
StandardOutput=inherit
StandardError=inherit
Restart=always
User=pi

[Install]
WantedBy=multi-user.target
```

Then enable and start the service with:

```bash
sudo systemctl daemon-reload
sudo systemctl enable vehicle_shutdown.service
sudo systemctl start vehicle_shutdown.service
```

---

### Final Remarks

- **Wiring & Logic Levels:**  
  Make sure the wiring from the Zero2Go Omini to your Pi matches the assumptions in the code. For example, if your board pulls the GPIO low when 12 V is present (or vice versa), you may need to invert the logic in the script.

- **Power-on Button:**  
  The Zero2Go Omini’s built-in circuitry typically takes care of “latching” the 5 V power when you press its button. Your Pi will boot up automatically; your script then monitors the 12 V signal for a safe shutdown when the vehicle is turned off.

This approach (and similar variations) has been used successfully by many users. Adjust the code, debounce times, and GPIO pins to suit your particular wiring and requirements. If you have any questions or need further assistance, feel free to ask!
