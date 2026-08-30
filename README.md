# Truck Dock Status System — Arduino Uno

An Arduino Uno–based control system for a truck loading dock. Two status lights indicate whether the dock is occupied or clear, a servo motor represents the dock gate, a potentiometer sets the gate speed live, and a warning horn alerts personnel before and during gate movement.

## Overview

- **Light 7 (dock clear)**: starts ON by default.
- **Light 8 (dock occupied)**: turned on when a truck is docking.
- **Servo motor (gate)**: moves to 0° when the dock is clear (Light 7) and to 180° when the dock is occupied (Light 8).
- **Button A**: general control — toggles the dock status back and forth (clear ↔ occupied).
- **Button B**: one-way trigger — signals "dock occupied" only. If the dock is already occupied, pressing it again has no effect; only Button A can bring it back to "clear."
- **Potentiometer**: sets the gate's movement speed **live**, including while the gate is already moving.
- **Warning horn**: beeps intermittently for 3 seconds *before* the gate starts moving, and keeps beeping intermittently throughout the movement. It stops only once the gate fully reaches its destination.
- **Safety interlock**: while the system is in the warning phase or the gate is moving, both buttons are ignored. A new command is only accepted once everything is fully stopped.

## Operation phases

The system runs through three phases each time a valid button press is accepted:

| Phase      | What happens                                                        |
|------------|----------------------------------------------------------------------|
| `PARADO`   | Idle — buttons are active, horn is off, gate is at 0° or 180°        |
| `AVISANDO` | Warning phase — horn beeps intermittently for 3 seconds, gate hasn't moved yet |
| `MOVENDO`  | Gate is moving to its target angle, horn keeps beeping intermittently |

Once the gate reaches its target, the system returns to `PARADO` and the buttons are active again.

## Hardware

| Component            | Notes                                              |
|------------------------|------------------------------------------------------|
| Arduino Uno            | —                                                    |
| 2x Status light (LED)  | + ~220Ω resistors                                    |
| 2x Push button         | No external resistor needed (`INPUT_PULLUP`)         |
| 1x Servo motor         | Represents the gate; needs external 5V power         |
| 1x Potentiometer       | Sets gate speed live                                 |
| 1x Warning horn/buzzer | Active buzzer, driven directly with `digitalWrite`   |

## Wiring

| Component              | Arduino Pin | Notes                                                        |
|--------------------------|:-----------:|-----------------------------------------------------------------|
| Light 7 (dock clear)     | 7           | Cathode → resistor → GND                                        |
| Light 8 (dock occupied)  | 8           | Cathode → resistor → GND                                        |
| Button A (toggle)        | 4           | One terminal to pin, other to GND                                |
| Button B (occupied only) | 12          | One terminal to pin, other to GND                                |
| Servo (signal)           | 3           | Signal wire (usually orange/yellow)                              |
| Servo (power)            | External 5V | **External** 5V supply — GND tied to Arduino GND                 |
| Potentiometer            | A0          | Outer terminals → Arduino's own 5V and GND; wiper → A0           |
| Warning horn             | 10          | Positive terminal → pin 10, negative → GND                       |

> ⚠️ **Servo power:** servos can draw more current than the Arduino's 5V pin can safely supply. Power the servo from an **external** 5V source, with that source's GND tied to the Arduino's GND.
>
> ⚠️ **Potentiometer power:** wire the potentiometer to the Arduino's **own** 5V rail, not the servo's external supply — this keeps current spikes from the servo out of the analog reading.

## Gate speed (potentiometer)

The gate speed is read continuously from the potentiometer on pin `A0` and mapped to the servo's step delay:

```cpp
tempo = map(leituraPote, 0, 1023, tempoMinimo, tempoMaximo);
```

- Potentiometer turned fully one way → `tempo = 1` → fastest possible gate movement
- Potentiometer turned fully the other way → `tempo = 100` → slowest possible gate movement

Because this is read every loop cycle, turning the knob **while the gate is already moving** changes its speed immediately.

## Warning horn behavior

- `intervaloBeep` (default 300ms) controls how fast the horn beeps on/off — lower is faster, higher is more spaced out.
- `tempoAviso` (default 3000ms) controls how long the warning phase lasts before the gate starts moving.
- The horn beeps during both the warning phase and the movement phase, and turns off completely once the gate is idle.

## Code structure

- `iniciarTroca()`: starts a status change — enters the `AVISANDO` phase and starts the horn, without touching the LEDs or the gate yet.
- `atualizarFaseOperacao()`: advances the phase machine (`AVISANDO` → `MOVENDO` → `PARADO`).
- `ligarLed7()` / `ligarLed8()`: set the dock status lights and trigger the gate to move to the matching angle.
- `moverServoGradualmente()`: moves the servo one degree at a time, non-blocking, respecting the current `tempo` value.
- `atualizarVelocidadePeloPotenciometro()`: reads the potentiometer every loop and updates `tempo`.
- `atualizarBuzina()`: handles the intermittent beep during the warning and movement phases.
- 50ms software debounce applied independently to each button.

## How to use

1. Wire the circuit as described above.
2. Open the `.ino` file in the Arduino IDE.
3. Select **Arduino Uno** as the board and the correct serial port.
4. Upload the code.
5. Press a button — the horn will beep for 3 seconds, then the light and gate will switch, with the horn continuing to beep until the gate finishes moving.
6. Turn the potentiometer to adjust gate speed, even mid-movement.

## Notes

- `ligarLed7()` and `ligarLed8()` each contain two `delay(300)` calls. These briefly pause the whole program on every transition (600ms total), which pauses the horn and gate updates during that window — worth keeping in mind if very precise timing between the light and the horn/gate matters.
- Not currently planned: additional lights/states beyond the two dock statuses, a cycle counter with display, or a fully automatic toggling mode.
