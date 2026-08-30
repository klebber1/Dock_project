# Dock_project
# Truck Dock Status System — Arduino Uno

An Arduino Uno–based control system for a truck loading dock. Two status lights indicate whether the dock is occupied or clear, a servo motor represents the dock gate/door, and a warning horn (planned) alerts personnel before and during gate movement. The system is button-triggered and enforces safe transitions: the gate must be fully open or fully closed before a new command is accepted.

## Overview

- **Light 7 (green/clear)**: dock is free — starts ON by default.
- **Light 8 (red/occupied)**: dock is in use / truck docking.
- **Servo motor**: represents the dock gate. Moves to 0° when the dock is clear (Light 7 ON) and to 180° when the dock is occupied (Light 8 ON).
- **Button A**: general control — toggles the dock status back and forth (clear ↔ occupied).
- **Button B**: one-way trigger — signals "dock occupied" (turns on Light 8 / closes the gate to 180°). If the dock is already marked occupied, pressing it again has no effect; only Button A can bring it back to "clear."
- **Safety interlock**: while the gate (servo) is moving, both buttons are ignored. A new command is only accepted once the gate has fully reached one of its end positions (0° or 180°) — preventing conflicting commands mid-movement.

## Hardware

| Component        | Notes                                                |
|-------------------|-------------------------------------------------------|
| Arduino Uno        | —                                                     |
| 2x Status light (LED) | + ~220Ω resistors                                 |
| 2x Push button     | No external resistor needed (`INPUT_PULLUP`)         |
| 1x Servo motor      | Represents the gate; external 5V power recommended  |
| Warning horn/buzzer | Planned — not yet implemented in code (see Roadmap) |

## Wiring

| Component            | Arduino Pin | Notes                                       |
|------------------------|:-----------:|-----------------------------------------------|
| Light 7 (dock clear)   | 7           | Cathode → resistor → GND                     |
| Light 8 (dock occupied)| 8           | Cathode → resistor → GND                     |
| Button A (toggle)      | 4           | One terminal to pin, other to GND            |
| Button B (occupied only)| 12         | One terminal to pin, other to GND            |
| Servo (signal)         | 3           | Signal wire (usually orange/yellow)          |
| Servo (power)          | 5V / GND    | External 5V supply recommended for the servo |

> ⚠️ **Note:** servos can draw more current than the Arduino's 5V pin can safely supply, especially under load. If the servo stutters, stalls, or resets the board, power it from an external 5V source (with its GND tied to the Arduino's GND).

## Gate speed setting

The `tempo` variable controls how many milliseconds the servo waits between each degree of movement:

```cpp
int tempo = 15; // adjust as needed
```

- `tempo = 1` → fastest possible gate movement
- `tempo = 100` → slowest possible gate movement

Tune this to match the real gate mechanism's desired opening/closing speed.

## Code structure

- `ligarLed7()` / `ligarLed8()`: set the dock status lights and trigger the gate to move to the matching angle.
- `iniciarMovimentoServo()`: sets the target angle and raises the "moving" flag.
- `moverServoGradualmente()`: moves the servo one degree at a time, non-blocking, respecting the `tempo` interval.
- 50ms software debounce applied independently to each button.

## How to use

1. Wire the circuit as described above.
2. Open the `.ino` file in the Arduino IDE.
3. Select **Arduino Uno** as the board and the correct serial port.
4. Upload the code.
5. Use the buttons to change the dock status and watch the gate (servo) sync with it.

## Roadmap / Planned features

- **Warning horn**: should start beeping ~3 seconds before the gate begins moving, and continue beeping intermittently only while the gate is in motion. Beep interval to be configurable via a code variable. Suggested pin: digital pin 10.
- **Speed potentiometer**: replace the fixed `tempo` variable with a potentiometer (planned on analog pin A0, powered by the Arduino's own 5V rail — not the servo's external supply) to allow adjusting gate speed live, while the system is running.
- Not currently planned: additional lights/states beyond the two dock statuses, a cycle counter with display, or a fully automatic toggling mode.
