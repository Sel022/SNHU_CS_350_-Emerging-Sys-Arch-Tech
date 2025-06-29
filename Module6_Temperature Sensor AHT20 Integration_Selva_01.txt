import board
import busio
import digitalio
import adafruit_character_lcd.character_lcd as character_lcd
import adafruit_ahtx0
import time
from datetime import datetime

# --- LCD Setup ---
lcd_columns = 16
lcd_rows = 2

lcd_rs = digitalio.DigitalInOut(board.D26)
lcd_en = digitalio.DigitalInOut(board.D19)
lcd_d4 = digitalio.DigitalInOut(board.D13)
lcd_d5 = digitalio.DigitalInOut(board.D6)
lcd_d6 = digitalio.DigitalInOut(board.D5)
lcd_d7 = digitalio.DigitalInOut(board.D11)

lcd = character_lcd.Character_LCD_Mono(
    lcd_rs, lcd_en,
    lcd_d4, lcd_d5, lcd_d6, lcd_d7,
    lcd_columns, lcd_rows
)

# --- Button Setup (GPIO 21) ---
button = digitalio.DigitalInOut(board.D21)
button.direction = digitalio.Direction.INPUT
button.pull = digitalio.Pull.UP  # Assumes active-low button wiring

# --- I2C Sensor Setup ---
i2c = busio.I2C(board.SCL, board.SDA)
sensor = adafruit_ahtx0.AHTx0(i2c)

lcd.clear()

# --- Temp Display Mode ---
show_fahrenheit = True
prev_button_state = True  # Start as unpressed (because pull-up is True)

try:
    while True:
        # --- Time ---
        now = datetime.now()
        line1 = now.strftime("%b %d %H:%M:%S")[:16]

        # --- Sensor Data (AHT20) ---
        temp_c = round(sensor.temperature, 1)
        temp_f = round((temp_c * 9 / 5) + 32, 1)
        hum = round(sensor.relative_humidity, 1)

        # --- Button Edge Detection ---
        current_state = button.value
        if prev_button_state and not current_state:  # Detect falling edge (press)
            show_fahrenheit = not show_fahrenheit
        prev_button_state = current_state

        # --- Format Line 2 ---
        if show_fahrenheit:
            temp_display = f"T:{temp_f:.1f}F"
        else:
            temp_display = f"T:{temp_c:.1f}C"

        line2 = f"{temp_display} H:{hum:.1f}%"
        line2 = line2[:16].ljust(16)

        # --- Update LCD ---
        lcd.cursor_position(0, 0)
        lcd.message = line1
        lcd.cursor_position(0, 1)
        lcd.message = line2

        time.sleep(1)

except KeyboardInterrupt:
    lcd.clear()
    lcd.message = "Goodbye!"
    time.sleep(1)
    lcd.clear()
