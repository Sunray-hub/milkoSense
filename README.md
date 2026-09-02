# Milk Adulteration Detector

A portable, handheld device to detect adulterants (water, starch, detergent, urea) in milk using multi-sensor fusion and machine learning on RP2040.

## Problem

Milk adulteration is widespread in informal milk markets across India. Existing field tests rely on color-matching by eye, which is subjective and unreliable. No accessible digital detection tool exists for household/small-dairy use.

This device provides instant, quantitative readout of milk purity without lab infrastructure.

## How It Works

The device measures four physical properties of milk:

1. **Density (via float + Hall sensor)** — Water dilution directly lowers milk density; detects adulteration most directly
2. **Electrical conductivity (EC)** — Water and salts shift conductivity; temperature-compensated
3. **Turbidity (optical)** — Starch and detergent scatter IR light differently than milk fat
4. **Temperature** — Compensates EC readings across milk temps (room to warm-fresh milk)

A scoring algorithm fuses these four readings and displays a verdict: **Pure** / **Suspect** / **Adulterated**

## BOM

### EC Sensor
- 1µF capacitor (ceramic or film)
- 1N4148 diode
- 0.1µF capacitor (smoothing)
- 10kΩ resistor (sense resistor)
- 2x stainless steel or graphite rod electrodes
- Jumper wires

### Turbidity Sensor
- IR LED (850nm)
- 220Ω resistor
- Photodiode or phototransistor
- 10kΩ resistor (pull-down)
- Clear plastic/glass cuvette chamber
- Jumper wires

### Density Sensor (Float + Hall)
- Linear Hall effect sensor (SS49E or A3144)
- Small neodymium magnet (5-10mm disk)
- Sealed float rod (weighted plastic tube or hydrometer-style)
- Narrow clear tube (~10-12mm diameter)
- Jumper wires

### Temperature Sensor
- DS18B20 digital thermometer
- 4.7kΩ resistor (pull-up)
- Jumper wires

### MCU & Support
- RP2040 Pico board
- ADS1115 16-bit ADC module (I2C)
- SSD1306 OLED display (I2C, 128x64)
- 18650 Li-ion battery + TP4056 charger module
- Push button
- Power switch
- 3D-printed or ABS enclosure (IP65 recommended)

**Estimated cost:** ~₹1500-2000 for all components

## Circuit Diagram

### Pin Map (RP2040)
| Function | Pin |
|---|---|
| EC PWM out | GPIO15 |
| Turbidity ADC | GPIO26 (ADC0) |
| Density Hall ADC | GPIO27 (ADC1) |
| Temperature (1-Wire) | GPIO17 |
| I2C SDA (OLED + ADS1115) | GPIO4 |
| I2C SCL (OLED + ADS1115) | GPIO5 |
| Trigger button | GPIO16 |
| Power | VSYS (from TP4056) |

### EC Circuit
```
Pico GPIO15 (PWM 1-2kHz) 
    → 1µF coupling cap 
    → Electrode A
    
Electrode B 
    → 10kΩ sense resistor 
    → GND
    
Junction between electrode B and sense resistor
    → 1N4148 diode 
    → 0.1µF cap to GND
    
Capacitor top terminal → ADS1115 A0
```

### Turbidity Circuit
```
3.3V 
    → 220Ω resistor 
    → IR LED (850nm) 
    → GND

Photodiode (across sample gap, facing LED)
    → 10kΩ pull-down to GND
    
Junction → Pico GPIO26 (ADC0)
```

### Density Circuit
```
Hall sensor (SS49E)
    VDD → 3.3V
    GND → GND
    OUT → Pico GPIO27 (ADC1)

Physical: Float with magnet in vertical tube, 
Hall sensor mounted beside tube at float travel height
```

### Temperature Circuit
```
DS18B20
    VDD → 3.3V
    GND → GND
    DQ → Pico GPIO17 (with 4.7kΩ pull-up to 3.3V)
```

### Display & Power
```
OLED (SSD1306)
    VCC → 3.3V, GND → GND
    SDA → Pico GPIO4, SCL → Pico GPIO5

ADS1115
    VDD → 3.3V, GND → GND
    SDA → Pico GPIO4, SCL → Pico GPIO5 (shared I2C)
    A0 ← EC peak detector
    A1 ← pH op-amp (future: not yet in simple version)

Power
    18650 cell → TP4056 module
    TP4056 OUT+ → Pico VSYS
    TP4056 OUT- → GND
```

## Setup & Firmware

### Required Libraries (Arduino-Pico)
- `Adafruit_ADS1X15` (ADS1115)
- `Adafruit_SSD1306` (OLED)
- `OneWire` (DS18B20 temperature)

### Install Arduino-Pico
1. Arduino IDE → Preferences → Additional Boards URLs
2. Add: `https://github.com/earlephilhower/arduino-pico/releases/download/global/package_rp2040_index.json`
3. Tools → Board Manager → Search "pico" → Install "Raspberry Pi Pico/RP2040"

### Basic Firmware Structure
```cpp
#include <Wire.h>
#include <Adafruit_ADS1X15.h>
#include <Adafruit_SSD1306.h>
#include <OneWire.h>

Adafruit_ADS1115 ads;
Adafruit_SSD1306 display(128, 64, &Wire, -1);
OneWire ds(17); // DS18B20 on GPIO17

void setup() {
  ads.begin();
  display.begin(SSD1306_SWITCHCAPVCC, 0x3C);
  pinMode(16, INPUT_PULLUP); // button
  pinMode(15, OUTPUT); // EC PWM
}

void loop() {
  if (digitalRead(16) == LOW) { // button pressed
    readAllSensors();
    computeVerdict();
    displayResult();
  }
}
```

### Sensor Reading Functions
- `readEC()` — Read ADS1115 channel A0, apply temp compensation
- `readTurbidity()` — Read GPIO26 ADC0
- `readDensity()` — Read GPIO27 ADC1
- `readTemp()` — Query DS18B20 via 1-Wire

## Calibration Procedure

### EC Calibration
1. Prepare 3-4 known solutions: pure milk, milk + 5% water, milk + 10% water, milk + 20% water
2. For each, measure EC reading at a reference temp (e.g., 25°C)
3. Record ADC value and corresponding dilution %
4. Fit a linear or polynomial curve: `dilution% = f(ADC_value)`
5. Store coefficients in firmware

### Density Calibration
1. Same water-dilution samples as EC
2. Record Hall sensor ADC reading for each
3. Fit curve: `dilution% = f(Hall_ADC)`

### Turbidity Calibration
1. Prepare pure milk sample
2. Prepare adulterant solutions (starch paste, detergent solution)
3. Read turbidity ADC for each concentration level
4. Build lookup table or curve

### Temperature Compensation (EC)
- Standard formula: `EC_actual = EC_measured / (1 + 0.02 * (T - 25))`
- Read temp at start of measurement, apply before scoring

## Usage

1. Pour ~10mL of milk sample into the cuvette chamber
2. Ensure all three probe tips (EC electrodes, turbidity sensor, Hall float) are submerged at consistent depth
3. Press the trigger button
4. Device displays: **PURE** / **SUSPECT** / **ADULTERATED**
5. Optional: log result with timestamp on the OLED or via serial (future: Pico W + WiFi)

## Verdict Algorithm

Simple weighted scoring (can be tuned):
```
score = 0.4 * density_score + 0.3 * ec_score + 0.2 * turbidity_score + 0.1 * temp_consistency

if score > 0.9: PURE
elif score > 0.6: SUSPECT
else: ADULTERATED
```

Each sensor's score ranges 0-1, where 1 = "strongly indicates pure milk."

## PCB Design (Future)

Compact two-layer board planned for final build:
- EC circuit (diode + cap + sense resistor)
- Hall sensor breakout
- Pico module footprint
- ADS1115 footprint
- OLED header
- Battery + TP4056 connector
- Push button

EasyEDA project: [link coming]

## Known Limitations

- **Accuracy:** screening-grade only; does not match lab-grade AAS analysis. Meant to flag suspect samples for confirmation at a certified lab.
- **Reagents:** no chemical reagents needed, unlike Gutzeit-based arsenic/fluoride tests.
- **Temperature drift:** EC compensation assumes linear temp coefficient; actual milk may vary slightly.
- **Sample prep:** all three probes must sit at consistent depth; probe holder design is critical.

## Future Improvements

- pH probe + amplifier circuit (detects detergent/starch better)
- LoRa module for remote alerting to dairy owner/inspector
- SD card logging of time-stamped results
- ML model (trained on real adulterated samples) replacing simple weighted scoring
- Pico W WiFi for mobile app companion
- Rechargeable solar + Li-ion dock

## Competition Notes

**Problem statement:** Milk adulteration in informal markets; field color-match tests are unreliable.

**Novel gap:** No maker/DIY digital detection tool; existing kits are expensive lab equipment or disposable test strips.

**Technical story:** Four-sensor fusion with temperature compensation; requires calibration curve fitting and understanding of milk physical properties (density, conductivity, turbidity).

**Demo:** Dip probe in pure sample → shows "PURE"; dip in water-diluted sample → shows "ADULTERATED." Live, instant, quantitative.

## License

MIT

## Author

Silicon Acid (Sun)

---

**Questions / Issues:** Open a GitHub issue or contact [@sunray-hub]
