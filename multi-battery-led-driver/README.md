# Multi-Battery White-LED Driver

A white-LED light that runs from whichever battery you have: AAA, AA, C/D,
coin (CR2032-type), or button (LR44/SR44-type) cells. A rotary switch picks
the battery bank and sets the LED current to suit that battery.

---

## 1. Design summary

| Item | Choice |
|---|---|
| Converter | **LT1932** constant-current boost LED driver (Linear Technology / Analog Devices) |
| Converter input range | 1 V to 10 V (datasheet) |
| LED load | One series string of **4 white LEDs** (about 11–14 V total) |
| LED current | Set by R<sub>SET</sub>, from 5 mA to 40 mA (datasheet). The rotary switch picks a different R<sub>SET</sub> for each battery type. |
| Selector | **2-pole, 6-position, break-before-make (non-shorting)** rotary switch |
| Positions | OFF · AAA · AA · C/D · Coin · Button |

### Why this architecture

The batteries cover a very wide voltage range:

| Bank | Cells | Fresh | End of life |
|---|---|---|---|
| AAA | 2–3 in series | 3.2–4.8 V | 2.0–3.0 V |
| AA | 1–4 in series | 1.6–6.4 V | 1.0–4.0 V |
| C / D | 1–2 in series | 1.6–3.2 V | 1.0–2.0 V |
| Coin (CR2032) | 1–3 in parallel | ~3.2 V | ~2.0 V |
| Button (LR44/SR44) | 1–3 in series | 1.55–4.7 V | 1.0–3.0 V |

So the input can be anywhere from about **1.0 V to 6.4 V**. A white LED needs
about 2.8–3.6 V, so some sources are below the LED voltage and some are above
it.

The design avoids that problem by putting the LEDs **in series**. Four white
LEDs need about 11–14 V, which is always higher than the highest input
(6.4 V). One plain **boost** converter can therefore handle every source. A
boost converter only works when the output voltage is above the input, so
**do not use fewer than 3 LEDs** (3 LEDs ≈ 9 V, still above 6.4 V).

A series string also gives every LED exactly the same current, so the LEDs
match in brightness and you don't need a ballast resistor for each LED.

---

## 2. Schematic

### 2a. Battery banks (all negatives go to common GND)

```
 AAA bank (3-cell series holder; with only 2 cells, put a dummy cell in slot 3)
   GND ──[AAA]──[AAA]──[AAA or dummy]──────────────────────► AAA+   (SW1A pos 2)

 AA bank (4-cell series holder; fill unused slots with AA dummy cells)
   GND ──[AA]──[AA or dummy]──[AA or dummy]──[AA or dummy]─► AA+    (SW1A pos 3)

 C/D bank (2-cell D series holder; C cells go in C-to-D adapter sleeves;
           a single cell uses one D dummy)
   GND ──[D/C]──[D/C or dummy]────────────────────────────► CD+    (SW1A pos 4)

 Coin bank (three single CR2032 holders in PARALLEL, each with its own diode)
   GND ──[CR2032 #1]──►|── D3 ─┐
   GND ──[CR2032 #2]──►|── D4 ─┼──────────────────────────► COIN+  (SW1A pos 5)
   GND ──[CR2032 #3]──►|── D5 ─┘
        (fill any 1, 2 or 3 holders)

 Button bank (3-cell series LR44 holder; unused slots get a conductive spacer)
   GND ──[LR44]──[LR44 or spacer]──[LR44 or spacer]────────► BTN+   (SW1A pos 6)
```

### 2b. Selector, protection and input filter

```
            SW1A (pole A — battery select)
 pos 1  (OFF, not connected)  o
 pos 2  AAA+  ────────────────o \
 pos 3  AA+   ────────────────o  \
 pos 4  CD+   ────────────────o   >── COM A ──[ F1 PTC ]──┬──────┬──────┬──────► VIN
 pos 5  COIN+ ────────────────o  /                        │      │      │
 pos 6  BTN+  ────────────────o /                         │     C1     C2
                                                    D1   ─┴─   47µF   4.7µF
                                                (crowbar) ▲    10V    10V
                                                          │      │      │
 GND ─────────────────────────────────────────────────────┴──────┴──────┴──────  GND
```

D1 goes from GND (anode) to VIN (cathode). It does nothing in normal use. If a
pack is put in backwards, D1 conducts, the PTC fuse heats up and trips, and the
LT1932 never sees a negative voltage.

### 2c. Boost LED driver

```
 VIN ──┬──────────────┬────────── L1 6.8µH ──────────┬──── D2 ──►|──┬────────────┬──── VOUT
       │              │                              │   (Schottky) │            │
       │      ┌───────┴───────────────┐              │              │           ─┴─
       │      │ VIN                SW ├──────────────┘             C3        ZD1 ▲ 18V 1W
       └──────┤ SHDN                  │                          1µF 35V       │ (open-LED
              │        LT1932         │                             │          │  clamp)
              │                   LED ├───────────────┐            GND         │
              │ GND              RSET ├──┐            │                        │
              └───┬───────────────────┘  │            │   VOUT ──►|──►|──►|──►|─┤  LED1..LED4
                 GND                     │            └───────────────────────┴─ (string cathode)
                                         │
                       SW1B (pole B — LED current select, ganged with SW1A)
                                         │
                                       COM B
            pos 1 OFF   ─ not connected ─o
            pos 2 AAA   ─[ R1 1.50k ]────o──┐
            pos 3 AA    ─[ R2 1.13k ]────o──┤
            pos 4 C/D   ─[ R3 1.13k ]────o──┼── GND
            pos 5 Coin  ─[ R4 4.53k ]────o──┤
            pos 6 Button─[ R5 4.53k ]────o──┘
```

How it connects:
- **LED string:** the anode of LED1 goes to VOUT, the four LEDs are in series,
  and the cathode of LED4 goes to the LT1932 **LED** pin (an internal current
  sink).
- **ZD1** goes from VOUT (cathode) to the LED pin (anode), in parallel with the
  LED string. If an LED fails open or the string comes unplugged, the regulated
  current flows through the Zener instead. That holds VOUT near 18 V, well
  below the 36 V SW/LED pin rating, so the converter can't push its own output
  up until something breaks.
- **SHDN** is tied to VIN. It needs more than 0.85 V to turn on, so the chip
  runs whenever the switch selects a battery. The OFF position cuts VIN
  completely.
- **Pole B** connects the RSET pin to GND through a different 1 % resistor at
  each position. In the OFF position, and for an instant between positions,
  RSET is open and the LED current drops to zero. That is a safe state.

> The schematic labels the LT1932 by pin **name**, not pin number. Wire it
> from the pinout drawing in the datasheet for the package you buy.

---

## 3. LED current per position

From the LT1932 datasheet: the RSET pin is regulated to 100 mV, and the
LED current is 225 × (100 mV / R<sub>SET</sub>).

| Pos | Source | R<sub>SET</sub> | LED current | Reason |
|---|---|---|---|---|
| 1 | OFF | open | 0 | — |
| 2 | AAA | 1.50 kΩ | 15 mA | Small cells; trades a little brightness for runtime |
| 3 | AA | 1.13 kΩ | 20 mA | Typical rated current for 5 mm / PLCC white LEDs |
| 4 | C/D | 1.13 kΩ | 20 mA | Lots of capacity. You can use 750 Ω (30 mA) if your LEDs are rated for it |
| 5 | Coin | 4.53 kΩ | 5 mA | Coin cells have high internal resistance; 5 mA is the LT1932's minimum setting |
| 6 | Button | 4.53 kΩ | 5 mA | Small capacity and limited current |

To change brightness for a position, swap its resistor for another value from
the datasheet table: 562 Ω = 40 mA, 750 Ω = 30 mA, 1.13 k = 20 mA,
1.50 k = 15 mA, 2.26 k = 10 mA, 4.53 k = 5 mA.

**Low-voltage behavior:** the LT1932 switch current limit is 400 mA minimum
(550 mA typical). From a single nearly-dead 1.0–1.2 V cell, the converter may
hit that limit before reaching the set LED current. The LEDs then dim gradually
instead of shutting off. This is expected, and it acts as a low-battery
warning.

---

## 4. Bill of materials

| Ref | Part | Notes |
|---|---|---|
| U1 | LT1932 (ThinSOT-6) | Constant-current boost LED driver, V<sub>IN</sub> 1–10 V |
| L1 | 6.8 µH inductor | I<sub>sat</sub> ≥ 0.8 A (above the 780 mA max switch limit), DCR ≤ 0.2 Ω, shielded |
| D2 | Schottky, ≥ 30 V, ≥ 0.5 A (e.g. 1N5819 or an SOD-123 equivalent) | Boost rectifier |
| C1 | 47 µF, ≥ 10 V, low-ESR (ceramic or polymer) | Input reservoir; especially helps coin and button cells |
| C2 | 4.7 µF X5R/X7R, ≥ 10 V | Input decoupling, placed right at the U1 VIN pin |
| C3 | 1 µF X7R, ≥ 35 V | Output capacitor (use a 35–50 V rating; ceramics lose capacitance under DC bias) |
| ZD1 | 18 V, 1 W Zener (e.g. 1N4746A) | Open-LED clamp. 20 mA × 18 V = 0.36 W during a fault |
| D1 | Schottky, ≥ 1 A (e.g. 1N5817) | Reverse-polarity crowbar |
| F1 | Resettable PTC fuse, I<sub>hold</sub> ≈ 1 A, R ≤ 0.15 Ω | Keep resistance low; it is in series with a 1 V source |
| D3–D5 | Low-V<sub>F</sub> Schottky (e.g. 1N5817 / SS14) | Coin-cell isolation. **Required**: lithium coin cells must never be charged by each other |
| R1 | 1.50 kΩ 1 % | AAA |
| R2, R3 | 1.13 kΩ 1 % | AA, C/D |
| R4, R5 | 4.53 kΩ 1 % | Coin, Button |
| LED1–LED4 | White LEDs, I<sub>F</sub> rating ≥ 20 mA (30 mA if you use the 750 Ω option) | Match bin and part number |
| SW1 | 2-pole 6-position rotary, **non-shorting (break-before-make)** | Shorting types would briefly connect two banks together |
| BT holders | 3×AAA series; 4×AA series; 2×D series; 3× single CR2032; 3×LR44 series | Plus AAA/AA/D dummy cells, C-to-D sleeves, LR44 spacers |

---

## 5. Handling "one or more" cells per bank

- **Series banks (AAA, AA, C/D, button):** each holder is wired for the maximum
  number of cells. To run on fewer cells, put a **dummy cell** (a
  battery-shaped metal conductor, sold as "battery spacers") in the empty
  slots. For LR44 cells, use a conductive slug about 11.6 mm in diameter and
  5.4 mm tall, such as turned brass or stacked steel washers. More cells mean a
  higher voltage, lower input current and longer runtime. The converter
  accepts any count.
- **C or D:** use one D-size holder. C cells go in C-to-D adapter sleeves.
- **Coin cells:** wired in **parallel through diodes**. You can fit 1, 2 or 3
  cells in any slots. The roughly 0.2–0.3 V diode drop doesn't matter because
  even a tired CR2032 (2.0 V) is well above the 1 V minimum.
- **Why the series banks don't use parallel diodes too:** a single 1.0 V
  end-of-life alkaline cell minus a 0.3 V diode drop would fall below the
  LT1932's 1 V minimum. That would throw away much of the battery.

**Alternative (no dummy cells):** use a second switch deck, or a 2-deck
12-position wafer switch, to tap each series bank after cell 1, cell 2, and
so on. This takes more switch positions and wiring but needs no spacers.

---

## 6. Rough runtime estimates

These numbers are approximate. They assume 4 LEDs at about 3.1 V each, about
75 % converter efficiency, and typical alkaline/lithium energy at moderate
drain. Real capacity falls a lot at high current, especially for coin and
button cells.

| Position | LED current | Battery draw (power) | Approx. runtime |
|---|---|---|---|
| AAA ×3 | 15 mA | ~0.25 W | ~12–15 h |
| AA ×2 | 20 mA | ~0.33 W | ~12–15 h (scales with cell count) |
| D ×2 | 20 mA | ~0.33 W | ~80–90 h |
| C ×2 | 20 mA | ~0.33 W | ~35–40 h |
| Coin ×3 | 5 mA | ~0.08 W | a few hours (CR2032s lose much of their capacity at ~10 mA per cell) |
| Button ×3 (LR44) | 5 mA | ~0.08 W | ~3–5 h |

---

## 7. Build and layout notes

1. Keep the switching loop (VIN cap C2 → L1 → SW pin → D2 → C3 → GND) short
   and tight. Put C2 and C3 right next to U1.
2. The RSET node is a sensitive, low-current analog node. Route the wire from
   U1's RSET pin to SW1B's common short, or put R1–R5 on the PCB with the
   switch close by. Keep it away from the SW node.
3. Use a **non-shorting** rotary switch. A shorting (make-before-break) switch
   would briefly connect two battery banks in parallel, for example 6.4 V AA
   straight into 3 V coin cells. D3–D5 protect the coin cells, but the other
   banks have no such protection.
4. Never mix chemistries or old and new cells within one series bank.
   Rechargeable NiMH cells (1.2 V) work fine in the AAA/AA/C/D banks.
5. Do **not** put the large C1 capacitor on the battery side of the switch.
   Its leakage would slowly drain coin cells while the light is off.

## 8. Bring-up / test checklist

1. Before fitting the LEDs, power VIN from a current-limited bench supply at
   3 V / 100 mA limit. VOUT should rise to about 18 V (the ZD1 clamp). This
   confirms the open-LED protection works.
2. Fit the LEDs. In each switch position, measure the voltage across the RSET
   resistor (should be about 100 mV). Measure the LED current with a meter in
   series with the string; it should be within about 10 % of the table in §3.
3. Sweep the bench supply from 6.5 V down to 1.0 V in the AA position. The
   current should stay regulated down to about 1.5–2 V, then droop gradually.
4. Reverse the supply polarity with the 1 A limit set. D1 should conduct and
   F1 should warm and trip. U1 should survive.
5. Test each battery bank with real cells. Then rotate the switch back and
   forth through all positions while it is running to confirm no glitches.

---

### Datasheet references

LT1932 figures used in this design (1–10 V input, 5–40 mA LED current, R<sub>SET</sub>
table and 225× current ratio, 36 V SW/LED rating, 400/550/780 mA switch limit,
0.85 V SHDN turn-on threshold, 1.2 MHz switching, 6.8 µH typical inductor) come
from the LT1932 datasheet:
https://www.analog.com/media/en/technical-documentation/data-sheets/1932f.pdf
— check them against the current revision before ordering parts.
