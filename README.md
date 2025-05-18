# PYNQ-Z2 Realtime Synthesizer

Dit project is een real-time softwarematige synthesizer die draait op de **PYNQ-Z2** (Xilinx Zynq-7000 ARM Cortex-A9). De synthesizer genereert audio via software, met behulp van een timer-interruptmechanisme en directe communicatie met een audiomodule via GPIO.

## Functionaliteiten

- Real-time audioverwerking via interrupts
- Ondersteuning voor meerdere golfvormen:
  - Sinus
  - Driehoek
  - Zaagtand
  - Rechte lijn
- Modulatie van frequentie en volume
- Directe sample-output naar audiopoort via GPIO
- Instelbare sample rate en buﬀergrootte
- Timer-gebaseerde interruptverwerking via SCU-timer

---

## Architectuur

### 1. **Main Loop**

- Initialisatie van hardwarecomponenten:
  - `XGpio`
  - `XScuTimer`
- Initialisatie van synthesizerparameters
- Timer start → interrupt activeert periodiek `timer_interrupt_handler`
- Set GPIO pins as inputs

---

### 2. **Timer Interrupt Setup**  

Deze functie `Timer_Intr_Setup` configureert en activeert een hardware timer interrupt op een Xilinx Zynq-systeem.

#### Functie: `Timer_Intr_Setup`

```c
static int Timer_Intr_Setup(XScuGic * IntcInstancePtr, XScuTimer *TimerInstancePtr, u16 TimerIntrId)
```

#### Wat doet de functie?
Deze functie zorgt ervoor dat de timer hardware een interrupt kan triggeren die door de CPU wordt herkend, en dat bij deze interrupt de Timer_ISR wordt aangeroepen. Dit is cruciaal om periodieke taken uit te voeren, zoals het genereren van audio samples of het bijhouden van tijd.

### 3. **Timer Interrupt Service Routine (ISR)**

Deze functie `Timer_ISR` wordt aangeroepen telkens wanneer de timer interrupt plaatsvindt. Het is verantwoordelijk voor het lezen van audio-input, toepassen van effecten en het schrijven van audio-output.

#### Functie: `Timer_ISR`

```c
static void Timer_ISR(void *CallBackRef)
```

#### Wat doet de functie?
1. Wis de interrupt status van de timer
XScuTimer_ClearInterruptStatus zorgt dat de interrupt wordt gereset zodat deze opnieuw kan triggeren.

2. Lees de status van de GPIO knoppen
XGpio_DiscreteRead leest welke knop(en) ingedrukt zijn via gpio_0.

3. Lees audio data van de linker- en rechterkanaal
Via registers I2S_DATA_RX_L_REG en I2S_DATA_RX_R_REG wordt de binnenkomende audio gelezen.

4. Pas audio-effect toe afhankelijk van welke knop ingedrukt is

- 1 → SpacePhaserEffect: creëert een phaser-effect op het geluid.

- 2 → GeneratePianoTone: genereert een pianoton.

- 4 → PacManSoundEffect: past het Pac-Man geluidseffect toe.

- 8 → TremoloEffect: creëert een tremolo-effect op het geluid.
Bij geen knop ingedrukt wordt het signaal ongefilterd doorgezet.

5. Schrijf de verwerkte audio data terug naar de output registers
Via I2S_DATA_TX_L_REG en I2S_DATA_TX_R_REG wordt de output gestuurd naar de audio hardware.

**Console output**
Bij elke effectkeuze wordt een bericht uitgeprint via xil_printf ter indicatie welk effect actief is.

#### Samenvatting
*Timer_ISR* handelt de realtime audio verwerking af tijdens elke timer interrupt, leest input, voert effecten uit op het audio signaal afhankelijk van de knopstatus, en schrijft de output terug. Hierdoor ontstaat een interactieve synthesizer met verschillende geluidseffecten.