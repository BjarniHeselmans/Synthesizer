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
Indien er geen knop is ingedrukt controleren we de switches:

- 1 → PulsatingTriangleWave: creëert een pulserende driehoekgolf op het geluid.

- 2 → RisingLaserSound: creëert elektronisch "whoop" of "sweep" geluid, de frequentie stijgt snel, dit herhaalt zich.
Indien er niks wordt ingegeven, wordt het signaal stopgezet.

5. Schrijf de verwerkte audio data terug naar de output registers
Via I2S_DATA_TX_L_REG en I2S_DATA_TX_R_REG wordt de output gestuurd naar de audio hardware.

**Console output**
Bij elke effectkeuze wordt een bericht uitgeprint via xil_printf ter indicatie welk effect actief is.

#### Samenvatting
*Timer_ISR* handelt de realtime audio verwerking af tijdens elke timer interrupt, leest input, voert effecten uit op het audio signaal afhankelijk van de knopstatus, en schrijft de output terug. Hierdoor ontstaat een interactieve synthesizer met verschillende geluidseffecten.

# Uitleg van Audio-Effect Functies

---

## 1. PacManSoundEffect

```c
void PacManSoundEffect(u32* inputBufferL, u32* inputBufferR, u32* outputBufferL, u32* outputBufferR, int bufferSize)
```
### Wat doet deze functie?
- Genereert een uniek "Pac-Man" geluidssignaal.
- Combineert twee sinusgolven met verschillende frequenties (600 Hz en 900 Hz).
- Voor elk sample wordt de som van de sinussen berekend, geschaald, en naar linker- en rechter outputbuffer geschreven.
- De fases van de sinussen worden per sample bijgewerkt voor een continu geluid.

### Parameters
- inputBufferL, inputBufferR: worden niet gebruikt in deze functie.
- outputBufferL, outputBufferR: buffers waarin het geluid wordt geplaatst.
- bufferSize: aantal samples.

## 2. ChorusEffect
```c
void ChorusEffect(u32* inputBufferL, u32* inputBufferR, u32* outputBufferL, u32* outputBufferR, int bufferSize, float rate, float depth, float delayTime)
```
### Wat doet deze functie?
- Voegt een chorus-effect toe dat het geluid voller maakt.
- Maakt gebruik van een delay-buffer met variërende vertragingstijd gestuurd door een LFO (sinusfunctie).
- Vertraagde samples worden gemixt met de originele input voor een breed, rijk geluid.

### Parameters
- inputBufferL, inputBufferR: input samples.
- outputBufferL, outputBufferR: output samples met chorus-effect.
- bufferSize: aantal samples.
- rate: frequentie van de LFO.
- depth: amplitude van de LFO (hoeveel de delay varieert).
- delayTime: basisvertraging (in deze code niet actief gebruikt).

## 3. SpacePhaserEffect
```c
void SpacePhaserEffect(u32* inputBufferL, u32* inputBufferR, u32* outputBufferL, u32* outputBufferR, int bufferSize, float rate, float depth)
```
### Wat doet deze functie?
- Voegt een phaser-effect toe, waarbij de amplitude van het signaal wordt gemoduleerd.
- Het linker- en rechterkanaal hebben fases die 180 graden uit fase zijn, wat een stereo-effect creëert.
- De amplitude wordt gemoduleerd door een sinus-LFO, waardoor het geluid “beweegt”.

### Parameters
- inputBufferL, inputBufferR: input audio samples.
- outputBufferL, outputBufferR: output samples met phaser-effect.
- bufferSize: aantal samples.
- rate: snelheid van de modulatie.
- depth: sterkte van het effect.

## 4. GeneratePianoTone
```c
void GeneratePianoTone(u32* outputBufferL, u32* outputBufferR, int bufferSize)
```
### Wat doet deze functie?
- Genereert een simpele pianoton als een sinusgolf.
- Per sample wordt een sinuswaarde berekend en omgezet naar 32-bit unsigned PCM-waarde.
- Plaatst het signaal in beide (links en rechts) outputbuffers.

### Parameters
- outputBufferL, outputBufferR: buffers voor het pianogeluid.
- bufferSize: aantal samples.

## 5. TremoloEffect
```c
void TremoloEffect(u32* inputBufferL, u32* inputBufferR, u32* outputBufferL, u32* outputBufferR, int bufferSize, float rate, float depth)
```
### Wat doet deze functie?
- Voegt een tremolo-effect toe door de amplitude snel te laten fluctueren.
- Een sinus-LFO moduleert de amplitude tussen 1 - depth en 1 + depth.
- Hierdoor krijgt het geluid een trillend, golvend effect.

### Parameters
- inputBufferL, inputBufferR: originele audio samples.
- outputBufferL, outputBufferR: audio samples na het tremolo-effect.
- bufferSize: aantal samples.
- rate: frequentie van de amplitude-modulatie.
- depth: sterkte van de modulatie.

## 6. PulsatingTriangleWave
```c
void PulsatingTriangleWave(u32* outputBufferL, u32* outputBufferR, int bufferSize)
```
### Wat doet deze functie?
- Genereert een driehoeksgolf met een constante toonhoogte van 440 Hz.
- De amplitude van de golf pulseert met een lage frequentie (bijv. 2 Hz), wat zorgt voor een ritmisch op en neer gaan van het volume.
- Het resultaat is een zuivere toon die "ademt" of langzaam in- en uitfadeert.

### Parameters
- outputBufferL, outputBufferR: buffers waarin de gegenereerde audio wordt opgeslagen (links en rechts).
- bufferSize: aantal audio-samples dat gegenereerd moet worden.

## 7. RisingLaserSound
```c
void RisingLaserSound(u32* outputBufferL, u32* outputBufferR, int bufferSize)
```
### Wat doet deze functie?
- Genereert een elektronisch geluid met een stijgende toonhoogte.
- De frequentie begint bij 300 Hz en stijgt snel naar 2000 Hz, waarna deze reset.
- Het resultaat is een herhalend, opwaarts glijdend geluid zoals een laser- of sci-fi sweep.

### Parameters
- outputBufferL, outputBufferR: buffers voor de gegenereerde audio in het linker- en rechterkanaal.
- bufferSize: aantal samples dat moet worden gegenereerd.