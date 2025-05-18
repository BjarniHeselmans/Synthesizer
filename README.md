# PYNQ-Z2 Realtime Synthesizer

Dit project is een real-time softwarematige synthesizer die draait op de **PYNQ-Z2** (Xilinx Zynq-7000 ARM Cortex-A9). De synthesizer genereert audio via software, met behulp van een timer-interruptmechanisme en directe communicatie met een audiomodule via GPIO.

## ⚙️ Functionaliteiten

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
