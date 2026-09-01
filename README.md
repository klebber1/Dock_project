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

- `ligarLed7()` and `ligarLed8()` each contain two `delay(300)` calls, used intentionally to work around a button debounce issue. These briefly pause the whole program on every transition (600ms total), including the horn and gate updates during that window — this is expected behavior, not a bug.
- Not currently planned: additional lights/states beyond the two dock statuses, a cycle counter with display, or a fully automatic toggling mode.

# Sistema de Status de Doca de Caminhões — Arduino Uno

Sistema de controle baseado em Arduino Uno para uma doca de caminhões. Duas luzes de status indicam se a doca está ocupada ou livre, um servo motor representa o portão da doca, um potenciômetro ajusta a velocidade do portão em tempo real, e uma buzina de aviso alerta a equipe antes e durante o movimento do portão.

## Visão geral

- **Luz 7 (doca livre)**: inicia ligada por padrão.
- **Luz 8 (doca ocupada)**: acende quando um caminhão está atracando.
- **Servo motor (portão)**: vai para 0° quando a doca está livre (Luz 7) e para 180° quando está ocupada (Luz 8).
- **Botão A**: controle geral — alterna o status da doca nos dois sentidos (livre ↔ ocupada).
- **Botão B**: acionamento de mão única — sinaliza apenas "doca ocupada". Se a doca já estiver ocupada, apertar de novo não faz nada; só o Botão A consegue voltar para "livre".
- **Potenciômetro**: define a velocidade de movimento do portão **em tempo real**, inclusive enquanto o portão já está se movendo.
- **Buzina de aviso**: apita de forma intermitente por 3 segundos *antes* do portão começar a se mover, e continua apitando intermitentemente durante todo o movimento. Só para completamente quando o portão chega ao destino.
- **Trava de segurança**: enquanto o sistema estiver na fase de aviso ou o portão estiver se movendo, os dois botões são ignorados. Um novo comando só é aceito quando tudo estiver completamente parado.

## Fases de operação

O sistema passa por três fases sempre que um botão válido é apertado:

| Fase       | O que acontece                                                         |
|------------|---------------------------------------------------------------------------|
| `PARADO`   | Ocioso — botões ativos, buzina desligada, portão em 0° ou 180°            |
| `AVISANDO` | Fase de aviso — buzina apita intermitentemente por 3 segundos, portão ainda parado |
| `MOVENDO`  | Portão se movendo até o ângulo alvo, buzina continua apitando intermitentemente |

Assim que o portão chega ao destino, o sistema volta para `PARADO` e os botões voltam a funcionar.

## Hardware

| Componente             | Observação                                            |
|--------------------------|----------------------------------------------------------|
| Arduino Uno              | —                                                          |
| 2x Luz de status (LED)   | + resistores de ~220Ω                                      |
| 2x Botão (push button)   | Sem necessidade de resistor externo (`INPUT_PULLUP`)       |
| 1x Servo motor           | Representa o portão; precisa de alimentação externa de 5V  |
| 1x Potenciômetro         | Define a velocidade do portão em tempo real                |
| 1x Buzina de aviso       | Buzina ativa, acionada diretamente com `digitalWrite`       |

## Ligações

| Componente               | Pino Arduino | Observação                                                     |
|----------------------------|:------------:|-------------------------------------------------------------------|
| Luz 7 (doca livre)         | 7            | Catodo → resistor → GND                                          |
| Luz 8 (doca ocupada)       | 8            | Catodo → resistor → GND                                          |
| Botão A (alterna)          | 4            | Um terminal no pino, outro no GND                                 |
| Botão B (só ocupa)         | 12           | Um terminal no pino, outro no GND                                 |
| Servo (sinal)              | 3            | Fio de sinal (geralmente laranja/amarelo)                          |
| Servo (alimentação)        | Fonte externa 5V | Fonte **externa** de 5V — GND ligado ao GND do Arduino         |
| Potenciômetro              | A0           | Terminais externos → 5V e GND do próprio Arduino; cursor → A0     |
| Buzina de aviso            | 10           | Terminal positivo → pino 10, negativo → GND                       |

> ⚠️ **Alimentação do servo:** servos podem consumir mais corrente do que o pino 5V do Arduino fornece com segurança. Alimente o servo com uma fonte **externa** de 5V, com o GND dessa fonte ligado ao GND do Arduino.
>
> ⚠️ **Alimentação do potenciômetro:** ligue o potenciômetro ao 5V do **próprio Arduino**, não à fonte externa do servo — isso evita que oscilações de corrente do servo interfiram na leitura analógica.

## Velocidade do portão (potenciômetro)

A velocidade do portão é lida continuamente do potenciômetro no pino `A0` e convertida no intervalo entre cada grau de movimento do servo:

```cpp
tempo = map(leituraPote, 0, 1023, tempoMinimo, tempoMaximo);
```

- Potenciômetro todo para um lado → `tempo = 1` → movimento o mais rápido possível
- Potenciômetro todo para o outro lado → `tempo = 100` → movimento o mais lento possível

Como essa leitura acontece a cada volta do loop, girar o potenciômetro **durante o movimento do portão** já muda a velocidade imediatamente.

## Comportamento da buzina

- `intervaloBeep` (padrão 300ms) controla a velocidade do apito (liga/desliga) — menor é mais rápido, maior é mais espaçado.
- `tempoAviso` (padrão 3000ms) controla quanto tempo dura a fase de aviso antes do portão começar a se mover.
- A buzina apita tanto na fase de aviso quanto na fase de movimento, e desliga completamente assim que o portão fica parado.

## Estrutura do código

- `iniciarTroca()`: inicia uma troca de status — entra na fase `AVISANDO` e liga a buzina, sem mexer ainda nas luzes ou no portão.
- `atualizarFaseOperacao()`: avança a máquina de fases (`AVISANDO` → `MOVENDO` → `PARADO`).
- `ligarLed7()` / `ligarLed8()`: ajustam as luzes de status da doca e disparam o movimento do portão para o ângulo correspondente.
- `moverServoGradualmente()`: move o servo um grau por vez, de forma não-bloqueante, respeitando o valor atual de `tempo`.
- `atualizarVelocidadePeloPotenciometro()`: lê o potenciômetro a cada loop e atualiza `tempo`.
- `atualizarBuzina()`: controla o apito intermitente durante as fases de aviso e movimento.
- Debounce de 50ms aplicado individualmente a cada botão.

## Como usar

1. Monte o circuito conforme a tabela de ligações.
2. Abra o arquivo `.ino` na IDE do Arduino.
3. Selecione a placa **Arduino Uno** e a porta serial correta.
4. Faça o upload do código.
5. Aperte um botão — a buzina apitará por 3 segundos, depois a luz e o portão trocam, com a buzina continuando a apitar até o portão terminar de se mover.
6. Gire o potenciômetro para ajustar a velocidade do portão, mesmo durante o movimento.

## Observações

- As funções `ligarLed7()` e `ligarLed8()` contêm dois `delay(300)` cada, usados propositalmente para contornar um problema de debounce nos botões. Isso pausa brevemente todo o programa a cada troca (600ms no total), incluindo a buzina e a atualização do portão nesse intervalo — é um comportamento esperado, não um bug.
- Não está planejado por enquanto: mais de duas luzes/estados, contador de ciclos com display, ou um modo totalmente automático de alternância.

