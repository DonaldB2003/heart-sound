# About 
AI-based Heart Sound Analysis

# Resources

- [Sound Analysis Portal](https://github.com/silicon-vlsi/sound-analysis)

# Raspberrypi-4
-pin config
![image](https://github.com/user-attachments/assets/19fb46bd-9cc9-4fb3-ac5b-0b645cee1a35)

| Raspberry Pi 4 |INMP441 |
|--|---|
|3V3|VDD|
|GND|GND|
|GPIO 18 (PCM_CLK)|WS|
|GPIO 19(PCM_FS)|SCK|
|GPIO 20(PCM_DIN)|SD|
|GND|L/R|

## Code to generate a ws, sck ,sd in real time using raspberypi-4 and 12s mic(INMP441)
```PY
import numpy as np
import pyaudio
import pigpio
import time
import matplotlib.pyplot as plt
import matplotlib

# Enable GUI backend for Matplotlib
matplotlib.use('TkAgg')

# Constants
WS_FREQUENCY = 10000   # WS clock = 10 kHz
SCK_FREQUENCY = 320000  # SCK clock = 32 * WS = 320 kHz
GPIO_WS = 18           # WS pin (PWM-capable: 12, 13, 18, 19)(PWM)
GPIO_SCK = 19          # SCK pin (Another PWM-capable GPIO)(PWM)
GPIO_SD = 20           # (PCM pins)Serial Data (SD) input pin (Must be an input pin)
DUTY_CYCLE = 500000    # 50% duty cycle (Range: 0 - 1M)

# Initialize pigpio
pi = pigpio.pi()
if not pi.connected:
    exit("Cannot connect to pigpio daemon!")

# Configure SD and SCK pins
pi.set_mode(GPIO_SD, pigpio.INPUT)
pi.set_mode(GPIO_SCK, pigpio.INPUT)

# Start hardware PWM for WS (10 kHz) and SCK (320 kHz)
pi.hardware_PWM(GPIO_WS, WS_FREQUENCY, DUTY_CYCLE)   # 10 kHz WS signal
pi.hardware_PWM(GPIO_SCK, SCK_FREQUENCY, DUTY_CYCLE)  # 320 kHz SCK signal

# Function to read 24-bit live data from GPIO_SD on SCK clock edge without loop
def read_24bit_data(gpio, level, tick):
    static_data = getattr(read_24bit_data, "data", 0)
    static_data = ((static_data << 1) | pi.read(GPIO_SD)) & 0xFFFFFF
    read_24bit_data.data = static_data
    print(f"Live 24-bit Serial Data: {static_data:06X}")

# Attach callback to detect SCK clock edges and capture 24-bit data
pi.callback(GPIO_SCK, pigpio.RISING_EDGE, read_24bit_data)

# Generate sine wave using WS as sample rate
t = np.linspace(0, 1, WS_FREQUENCY, endpoint=False)  # WS defines sample rate
sine_wave = 0.5 * np.sin(2 * np.pi * WS_FREQUENCY * t)  # Normalized sine wave

# Convert to 24-bit PCM format
audio_data = (sine_wave * (2**23 - 1)).astype(np.int32)  # Scale to 24-bit range

# Initialize PyAudio
p = pyaudio.PyAudio()
stream = p.open(format=pyaudio.paInt32,
                channels=1,
                rate=WS_FREQUENCY,  # Set WS as the sample rate (10 kHz)
                output=True)

# Play the sine wave using PyAudio
print(f"Playing sine wave with WS={WS_FREQUENCY} Hz and generating SCK={SCK_FREQUENCY} Hz on GPIO {GPIO_WS} & {GPIO_SCK}")
stream.write(audio_data.tobytes())

# Perform FFT
fft_result = np.fft.fft(audio_data)
frequencies = np.fft.fftfreq(len(audio_data), 1 / WS_FREQUENCY)

# Plot FFT and display
plt.figure()
plt.plot(frequencies[:len(frequencies)//2], np.abs(fft_result)[:len(frequencies)//2])
plt.title("FFT of Generated Sine Wave")
plt.xlabel("Frequency (Hz)")
plt.ylabel("Magnitude")
plt.grid()
plt.show()  # Display plot instead of saving

# Cleanup
stream.stop_stream()
stream.close()
p.terminate()

# Keep PWM running and reading until user stops it
try:
    input("Press Enter to stop WS, SCK, and SD signal monitoring...")
finally:
    pi.hardware_PWM(GPIO_WS, 0, 0)   # Stop WS
    pi.hardware_PWM(GPIO_SCK, 0, 0)  # Stop SCK
    pi.stop()
```

