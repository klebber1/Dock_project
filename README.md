# Truck Dock Safety & Control System

This repository documents the development of a safety and signaling system for a truck loading dock: a status semaphore (red/green), an industrial gate, and a warning horn, controlled by an external trigger button. It contains both the small-scale Arduino prototype that validated the system's logic, and the technical basis for the real industrial installation that is now the project's goal.

## Project direction: from prototype to production

This project started as a small-scale Arduino Uno prototype — two LEDs standing in for the dock's status lights, a servo motor standing in for the gate, and a small speaker standing in for the warning horn. That scale-down was deliberate, not a shortcut: building and testing the full **operating logic** (the warning phase before movement, the non-blocking gradual movement, the interlock that blocks new commands until the gate is fully open or closed, the debounce handling) on cheap, disposable hardware let us validate the *behavior* of the system before committing to real actuators, real voltages, and a real gate.

Every core rule the production system needs was proven here first:
- A warning period must always precede movement, and the horn must keep sounding throughout the movement, not just at the start.
- The system must never accept a new command while the gate is mid-travel — only once it has fully reached one of its end positions.
- Movement speed must be adjustable without blocking the rest of the system (hence the potentiometer, read live, and the non-blocking step-by-step servo movement instead of `delay()`-based motion).

With that logic validated, the project now moves toward a **real industrial implementation**: an actual dock with a real LED semaphore, an industrial roll-up canvas gate, an emergency-type external button, and a siren — sized and wired for continuous industrial use rather than a desk prototype. The technical and safety justification for that installation, including the NR-12 (Brazilian workplace safety regulation) grounding and a full comparison between two implementation paths — a microcontroller-based system (the same ATmega328p logic proven here) and an industrial PLC — is documented separately in **`Proposta_Tecnica_Doca_NR12`** (PDF/DOCX). That document is the one to consult for the production-grade decision; this README documents the proof-of-concept that logic is based on.

## Proof-of-concept prototype (Arduino Uno)

The sections below describe the working small-scale prototype used to validate the system's logic.

### Overview

- **Light 7 (dock clear)**: starts ON by default.
- **Light 8 (dock occupied)**: turned on when a truck is docking.
- **Servo motor (gate)**: moves to 0° when the dock is clear (Light 7) and to 180° when the dock is occupied (Light 8).
- **Button A**: general control — toggles the dock status back and forth (clear ↔ occupied).
- **Button B**: one-way trigger — signals "dock occupied" only. If the dock is already occupied, pressing it again has no effect; only Button A can bring it back to "clear."
- **Potentiometer**: sets the gate's movement speed **live**, including while the gate is already moving.
- **Warning horn**: beeps intermittently for 3 seconds *before* the gate starts moving, and keeps beeping intermittently throughout the movement. It stops only once the gate fully reaches its destination.
- **Safety interlock**: while the system is in the warning phase or the gate is moving, both buttons are ignored. A new command is only accepted once everything is fully stopped.

In the real installation, Light 7 / Light 8 become the semaphore, the servo becomes the industrial gate motor, Button B becomes the external emergency-type button, and the horn becomes the siren — the mapping is direct, which is exactly why this prototype was useful.

### Operation phases

The system runs through three phases each time a valid button press is accepted:

| Phase      | What happens                                                        |
|------------|----------------------------------------------------------------------|
| `PARADO`   | Idle — buttons are active, horn is off, gate is at 0° or 180°        |
| `AVISANDO` | Warning phase — horn beeps intermittently for 3 seconds, gate hasn't moved yet |
| `MOVENDO`  | Gate is moving to its target angle, horn keeps beeping intermittently |

Once the gate reaches its target, the system returns to `PARADO` and the buttons are active again.

### Hardware

| Component            | Notes                                              |
|------------------------|------------------------------------------------------|
| Arduino Uno            | —                                                    |
| 2x Status light (LED)  | + ~220Ω resistors                                    |
| 2x Push button         | No external resistor needed (`INPUT_PULLUP`)         |
| 1x Servo motor         | Represents the gate; needs external 5V power         |
| 1x Potentiometer       | Sets gate speed live                                 |
| 1x Warning horn/buzzer | Active buzzer, driven directly with `digitalWrite`   |

### Wiring

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

### Gate speed (potentiometer)

The gate speed is read continuously from the potentiometer on pin `A0` and mapped to the servo's step delay:

```cpp
tempo = map(leituraPote, 0, 1023, tempoMinimo, tempoMaximo);
```

- Potentiometer turned fully one way → `tempo = 1` → fastest possible gate movement
- Potentiometer turned fully the other way → `tempo = 100` → slowest possible gate movement

Because this is read every loop cycle, turning the knob **while the gate is already moving** changes its speed immediately.

### Warning horn behavior

- `intervaloBeep` (default 300ms) controls how fast the horn beeps on/off — lower is faster, higher is more spaced out.
- `tempoAviso` (default 3000ms) controls how long the warning phase lasts before the gate starts moving.
- The horn beeps during both the warning phase and the movement phase, and turns off completely once the gate is idle.

### Code structure

- `iniciarTroca()`: starts a status change — enters the `AVISANDO` phase and starts the horn, without touching the LEDs or the gate yet.
- `atualizarFaseOperacao()`: advances the phase machine (`AVISANDO` → `MOVENDO` → `PARADO`).
- `ligarLed7()` / `ligarLed8()`: set the dock status lights and trigger the gate to move to the matching angle.
- `moverServoGradualmente()`: moves the servo one degree at a time, non-blocking, respecting the current `tempo` value.
- `atualizarVelocidadePeloPotenciometro()`: reads the potentiometer every loop and updates `tempo`.
- `atualizarBuzina()`: handles the intermittent beep during the warning and movement phases.
- 50ms software debounce applied independently to each button.

### How to use

1. Wire the circuit as described above.
2. Open the `.ino` file in the Arduino IDE.
3. Select **Arduino Uno** as the board and the correct serial port.
4. Upload the code.
5. Press a button — the horn will beep for 3 seconds, then the light and gate will switch, with the horn continuing to beep until the gate finishes moving.
6. Turn the potentiometer to adjust gate speed, even mid-movement.

### Notes

- `ligarLed7()` and `ligarLed8()` each contain two `delay(300)` calls, used intentionally to work around a button debounce issue. These briefly pause the whole program on every transition (600ms total), including the horn and gate updates during that window — this is expected behavior, not a bug.
- Not currently planned at the prototype scale: additional lights/states beyond the two dock statuses, a cycle counter with display, or a fully automatic toggling mode. These are separate from the production system's requirements, which are defined in the technical document referenced above.

## Next steps

- [ ] Review `Proposta_Tecnica_Doca_NR12` and confirm the implementation path (microcontroller vs. PLC) for the production system.
- [ ] Source the real components: LED semaphore, industrial roll-up canvas gate + motor, emergency-type external button, siren, electrical panel.
- [ ] Re-map the validated prototype logic (phase machine, interlock, warning timing) onto the chosen production controller.
- [ ] Have the final installation reviewed by a qualified occupational safety professional for formal NR-12 compliance.


# Sistema de Segurança e Controle de Doca de Caminhões

Este repositório documenta o desenvolvimento de um sistema de sinalização e segurança para uma doca de carga de caminhões: um semáforo de status (vermelho/verde), um portão industrial e uma sirene de aviso, acionados por um botão externo. Ele contém tanto o protótipo em pequena escala feito com Arduino, que validou a lógica do sistema, quanto a base técnica para a instalação industrial real que agora é o objetivo do projeto.

## Direção do projeto: do protótipo à produção

Este projeto começou como um protótipo em pequena escala com Arduino Uno — dois LEDs representando as luzes de status da doca, um servo motor representando o portão, e um pequeno alto-falante representando a sirene de aviso. Essa redução de escala foi proposital, não um atalho: construir e testar toda a **lógica de operação** (a fase de aviso antes do movimento, o movimento gradual e não-bloqueante, o interbloqueio que impede novos comandos até o portão estar completamente aberto ou fechado, o tratamento de debounce) em um hardware barato e descartável permitiu validar o *comportamento* do sistema antes de investir em atuadores reais, tensões reais e um portão de verdade.

Toda regra essencial que o sistema de produção vai precisar já foi comprovada aqui primeiro:
- Um período de aviso sempre precisa preceder o movimento, e a sirene deve continuar tocando durante todo o movimento, não só no início.
- O sistema nunca deve aceitar um novo comando enquanto o portão está em trânsito — só quando ele tiver chegado completamente a uma das posições extremas.
- A velocidade do movimento precisa ser ajustável sem travar o resto do sistema (por isso o potenciômetro, lido em tempo real, e o movimento do servo grau a grau e não-bloqueante, em vez de movimento baseado em `delay()`).

Com essa lógica validada, o projeto agora avança para uma **implementação industrial real**: uma doca de verdade, com um semáforo de LED real, um portão de lona industrial enrolável, um botão de acionamento externo do tipo emergência, e uma sirene — dimensionados e cabeados para uso industrial contínuo, e não para um protótipo de mesa. A justificativa técnica e de segurança para essa instalação, incluindo a fundamentação na NR-12 e uma comparação completa entre duas vias de implementação — um sistema baseado em microcontrolador (a mesma lógica do ATmega328p já comprovada aqui) e um CLP industrial — está documentada separadamente em **`Proposta_Tecnica_Doca_NR12`** (PDF/DOCX). É esse documento que deve ser consultado para a decisão de nível de produção; este README documenta a prova de conceito na qual essa lógica se baseia.

## Protótipo de prova de conceito (Arduino Uno)

As seções abaixo descrevem o protótipo funcional em pequena escala usado para validar a lógica do sistema.

### Visão geral

- **Luz 7 (doca livre)**: inicia ligada por padrão.
- **Luz 8 (doca ocupada)**: acende quando um caminhão está atracando.
- **Servo motor (portão)**: vai para 0° quando a doca está livre (Luz 7) e para 180° quando está ocupada (Luz 8).
- **Botão A**: controle geral — alterna o status da doca nos dois sentidos (livre ↔ ocupada).
- **Botão B**: acionamento de mão única — sinaliza apenas "doca ocupada". Se a doca já estiver ocupada, apertar de novo não faz nada; só o Botão A consegue voltar para "livre".
- **Potenciômetro**: define a velocidade de movimento do portão **em tempo real**, inclusive enquanto o portão já está se movendo.
- **Buzina de aviso**: apita de forma intermitente por 3 segundos *antes* do portão começar a se mover, e continua apitando intermitentemente durante todo o movimento. Só para completamente quando o portão chega ao destino.
- **Trava de segurança**: enquanto o sistema estiver na fase de aviso ou o portão estiver se movendo, os dois botões são ignorados. Um novo comando só é aceito quando tudo estiver completamente parado.

Na instalação real, a Luz 7 / Luz 8 se tornam o semáforo, o servo se torna o motor do portão industrial, o Botão B se torna o botão externo de emergência, e a buzina se torna a sirene — o mapeamento é direto, e é exatamente por isso que este protótipo foi útil.

### Fases de operação

O sistema passa por três fases sempre que um botão válido é apertado:

| Fase       | O que acontece                                                         |
|------------|---------------------------------------------------------------------------|
| `PARADO`   | Ocioso — botões ativos, buzina desligada, portão em 0° ou 180°            |
| `AVISANDO` | Fase de aviso — buzina apita intermitentemente por 3 segundos, portão ainda parado |
| `MOVENDO`  | Portão se movendo até o ângulo alvo, buzina continua apitando intermitentemente |

Assim que o portão chega ao destino, o sistema volta para `PARADO` e os botões voltam a funcionar.

### Hardware

| Componente             | Observação                                            |
|--------------------------|----------------------------------------------------------|
| Arduino Uno              | —                                                          |
| 2x Luz de status (LED)   | + resistores de ~220Ω                                      |
| 2x Botão (push button)   | Sem necessidade de resistor externo (`INPUT_PULLUP`)       |
| 1x Servo motor           | Representa o portão; precisa de alimentação externa de 5V  |
| 1x Potenciômetro         | Define a velocidade do portão em tempo real                |
| 1x Buzina de aviso       | Buzina ativa, acionada diretamente com `digitalWrite`       |

### Ligações

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

### Velocidade do portão (potenciômetro)

A velocidade do portão é lida continuamente do potenciômetro no pino `A0` e convertida no intervalo entre cada grau de movimento do servo:

```cpp
tempo = map(leituraPote, 0, 1023, tempoMinimo, tempoMaximo);
```

- Potenciômetro todo para um lado → `tempo = 1` → movimento o mais rápido possível
- Potenciômetro todo para o outro lado → `tempo = 100` → movimento o mais lento possível

Como essa leitura acontece a cada volta do loop, girar o potenciômetro **durante o movimento do portão** já muda a velocidade imediatamente.

### Comportamento da buzina

- `intervaloBeep` (padrão 300ms) controla a velocidade do apito (liga/desliga) — menor é mais rápido, maior é mais espaçado.
- `tempoAviso` (padrão 3000ms) controla quanto tempo dura a fase de aviso antes do portão começar a se mover.
- A buzina apita tanto na fase de aviso quanto na fase de movimento, e desliga completamente assim que o portão fica parado.

### Estrutura do código

- `iniciarTroca()`: inicia uma troca de status — entra na fase `AVISANDO` e liga a buzina, sem mexer ainda nas luzes ou no portão.
- `atualizarFaseOperacao()`: avança a máquina de fases (`AVISANDO` → `MOVENDO` → `PARADO`).
- `ligarLed7()` / `ligarLed8()`: ajustam as luzes de status da doca e disparam o movimento do portão para o ângulo correspondente.
- `moverServoGradualmente()`: move o servo um grau por vez, de forma não-bloqueante, respeitando o valor atual de `tempo`.
- `atualizarVelocidadePeloPotenciometro()`: lê o potenciômetro a cada loop e atualiza `tempo`.
- `atualizarBuzina()`: controla o apito intermitente durante as fases de aviso e movimento.
- Debounce de 50ms aplicado individualmente a cada botão.

### Como usar

1. Monte o circuito conforme a tabela de ligações.
2. Abra o arquivo `.ino` na IDE do Arduino.
3. Selecione a placa **Arduino Uno** e a porta serial correta.
4. Faça o upload do código.
5. Aperte um botão — a buzina apitará por 3 segundos, depois a luz e o portão trocam, com a buzina continuando a apitar até o portão terminar de se mover.
6. Gire o potenciômetro para ajustar a velocidade do portão, mesmo durante o movimento.

### Observações

- As funções `ligarLed7()` e `ligarLed8()` contêm dois `delay(300)` cada, usados propositalmente para contornar um problema de debounce nos botões. Isso pausa brevemente todo o programa a cada troca (600ms no total), incluindo a buzina e a atualização do portão nesse intervalo — é um comportamento esperado, não um bug.
- Não está planejado, na escala do protótipo: mais de duas luzes/estados, contador de ciclos com display, ou um modo totalmente automático de alternância. Esses pontos são independentes dos requisitos do sistema de produção, que estão definidos no documento técnico referenciado acima.

## Próximos passos

- [ ] Revisar o `Proposta_Tecnica_Doca_NR12` e confirmar a via de implementação (microcontrolador ou CLP) para o sistema de produção.
- [ ] Providenciar os componentes reais: semáforo de LED, portão de lona industrial enrolável + motor, botão externo tipo emergência, sirene, painel elétrico.
- [ ] Remapear a lógica já validada no protótipo (máquina de fases, interbloqueio, tempo de aviso) para o controlador de produção escolhido.
- [ ] Ter a instalação final revisada por um profissional habilitado em segurança do trabalho, para conformidade formal com a NR-12.

