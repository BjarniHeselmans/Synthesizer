# PYNQ-Z2 Real-time Synthesizer

Dit project is een real-time softwarematige synthesizer die draait op de PYNQ-Z2 (Xilinx Zynq-7000 ARM Cortex-A9). De synthesizer genereert en verwerkt audio in realtime via een timer-interrupt en communiceert direct met een audiomodule via GPIO-registraties.

## Functionaliteiten

- Real-time audioverwerking via SCU-timer interrupts
- Ondersteuning voor meerdere golfvormen en geluidseffecten:
  - Sinusgolf
  - Driehoeksgolf (pulsating)
  - Zaagtandgolf
  - Rechte lijn
- Specifieke geluidseffecten: Pac-Man, Phaser, Tremolo, Piano, Chorus, Rising Laser
- Modulatie van frequentie en volume
- Directe sample-output naar audiopoort via I2S GPIO-interfaces
- Instelbare sample rate en buffer grootte
- Console-uitvoer ter debugging en effectstatus via `xil_printf`

## Architectuur

### 1. Main Loop

- Initialisatie van hardwarecomponenten:
  - `XGpio` (voor input buttons/switches)
  - `XScuTimer` (hardware timer voor interrupt)
- Initialisatie van synthesizerparameters en buffers
- Timer start → periodieke interrupts triggeren `Timer_ISR`
- GPIO pins ingesteld als input voor knopdetectie

### 2. Timer Interrupt Setup

**Functie:** `Timer_Intr_Setup`

```c
static int Timer_Intr_Setup(XScuGic * IntcInstancePtr, XScuTimer *TimerInstancePtr, u16 TimerIntrId)
```
Deze functie zorgt ervoor dat de timer interrupt wordt ingeschakeld in de interrupt controller en dat de Timer_ISR wordt aangeroepen wanneer de interrupt optreedt. Dit is essentieel voor de realtime verwerking van audio samples.

3. Timer Interrupt Service Routine (ISR)


Functie: Timer_ISR

```c
static void Timer_ISR(void *CallBackRef)
```
Workflow van Timer_ISR:

1. Clear interrupt status: reset de interrupt flag met XScuTimer_ClearInterruptStatus.

2. Lees GPIO input: lees de status van de knoppen via XGpio_DiscreteRead.

3. Lees binnenkomende audio samples: via registers I2S_DATA_RX_L_REG en I2S_DATA_RX_R_REG.

4. Selecteer audio-effect op basis van knoppen:

- Knop 1 → SpacePhaserEffect

- Knop 2 → GeneratePianoTone

- Knop 4 → PacManSoundEffect

- Knop 8 → TremoloEffect


Als geen knop ingedrukt is, worden de switches gecontroleerd:

- Switch 1 → PulsatingTriangleWave

- Switch 2 → RisingLaserSound


Bij geen input wordt output op nul gezet.

5. Schrijf verwerkte audio naar output registers: via I2S_DATA_TX_L_REG en I2S_DATA_TX_R_REG.

6. Console-uitvoer: xil_printf toont welk effect actief is voor debugging.

### 3. Uitleg van Audio-Effect Functies
1. PacManSoundEffect
```c
void PacManSoundEffect(u32* inputBufferL, u32* inputBufferR, u32* outputBufferL, u32* outputBufferR, int bufferSize)
```
- Combineert twee sinussen (600 Hz en 900 Hz) voor een karakteristiek Pac-Man geluid.

- Per sample wordt het signaal opgebouwd en in outputbuffers geschreven.

- Inputbuffers worden genegeerd.

2. ChorusEffect
```c
void ChorusEffect(u32* inputBufferL, u32* inputBufferR, u32* outputBufferL, u32* outputBufferR, int bufferSize, float rate, float depth, float delayTime)
```
- Voegt een chorus-effect toe door gebruik te maken van een delaybuffer met sinusvormige modulatie (LFO).

- Delay en amplitude variëren, wat resulteert in een voller, rijker geluid.

3. SpacePhaserEffect
```c
void SpacePhaserEffect(u32* inputBufferL, u32* inputBufferR, u32* outputBufferL, u32* outputBufferR, int bufferSize, float rate, float depth)
```
- Phaser-effect waarbij amplitude gemoduleerd wordt met een sinus-LFO.

- Linker- en rechterkanaal staan 180 graden uit fase voor stereo-effect.

4. GeneratePianoTone
```c
void GeneratePianoTone(u32* outputBufferL, u32* outputBufferR, int bufferSize)
```
- Simpele sinusgolf voor pianoton.

- Per sample wordt een sinuswaarde berekend en omgezet naar 32-bit PCM.

5. TremoloEffect
```c
void TremoloEffect(u32* inputBufferL, u32* inputBufferR, u32* outputBufferL, u32* outputBufferR, int bufferSize, float rate, float depth)
```
- Amplitudemodulatie met sinus-LFO om trillend, golvend effect te creëren.

- Modulatie tussen 1 - depth en 1 + depth.

6. PulsatingTriangleWave
```c
void PulsatingTriangleWave(u32* outputBufferL, u32* outputBufferR, int bufferSize)
```
- Driehoeksgolf met constante 440 Hz, waarbij amplitude pulseert (bijv. 2 Hz).

- Creëert een ademend effect.

7. RisingLaserSound
```c
void RisingLaserSound(u32* outputBufferL, u32* outputBufferR, int bufferSize)
```
- Laserachtig geluid met stijgende frequentie van 300 Hz naar 2000 Hz, daarna reset.

- Herhalend elektronisch sweep-effect.