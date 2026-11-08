# Secure Element (U1, ST33K1M5) – Ledger Nano S+ / Flex / Stax

Status: **visual analysis only** (photos in `../pics/`). Nothing has been checked electrically yet.
Each statement below is tagged:

- **[seen]**: visible directly in a photo
- **[inferred]**: worked out from several photos together
- **[TBD]**: needs a multimeter or more imaging

## 1. Identification & package

| Property | Value | Source |
|---|---|---|
| Part | **STMicroelectronics ST33K1M5** (secure MCU, ST33K family) | user, confirmed under microscope |
| Die marking | `K460A`, ST logo, `2020` (see §1.1) | [seen] `die_marking_K460A.png` |
| Package marking | Ledger custom laser marking (see below) | [seen] 165806, 171319 |
| Type | WLCSP / bumped die with RDL (bare silicon, polyimide + copper redistribution layer visible on bump side) | [seen] 171031, 171107 |
| Balls | 24, 4 × 6 full array | [seen] |
| Body | ≈ 2.60 × 1.80 mm | [seen] measured against 0.1 mm reticle in 171031 |
| Ball pitch | ≈ 0.40 mm both axes | [seen] |
| Refdes on PCB | U1 (Nano S+) | [seen] 171512 |

The same SE is used in the Nano S+, Flex and Stax.

### 1.1 Die marking (metal layer, `pics/die_marking_K460A.png`)

```
 K460A
 (C) STW  ■
          (C) ZMB  ■  ST3
          [ST logo]  (M) (C)
                      2020
```

- `K460A`: die/mask identifier, probably ST's internal die code for the ST33K1M5.
- `(C) STW` and `(C) ZMB`, each followed by a small square: copyright lines. These could be design-team, site or IP-block tags. **[TBD]** meaning.
- `ST3`: probably the product-family prefix (ST33…), cut off at the edge of the marking block.
- `(M)` and `(C)` in boxes: mask-work and copyright symbols.
- `2020`: design/mask year.
- On the right, a dense dot/via matrix next to the marking block, and fine metal lines with shielding mesh around it. These are consistent with ST33's active top-layer shield.

### Package top marking (laser, on die backside)

```
 !L!
 KC7A6
 (◎) 102
```

This is Ledger's own marking, not ST's standard top marking. `!L!` is probably Ledger's mark, and `KC7A6` / `102` are probably lot or date codes. The ST identity (`ST K460A`) comes from the die marking.

ST's public datasheets for the ST33K1M5 don't include the WLCSP-24 ballout (it is only available under NDA), so the ball map below is reverse-engineered.

## 2. Orientation

Two independent features mark the **same corner**:

1. **PCB silkscreen dot** next to the footprint corner (171512, 171355). **[seen]**
2. **Dark marker / blotch on the chip's bump side**, at the corner that sits over that dot once the chip is flipped (bottom-left in 171107). **[seen]**

The mapping between the chip's bottom side and the footprint was confirmed four ways **[inferred]**:
- The horizontal RDL bus on the chip lines up with the horizontal PCB trace bus in the same row.
- Every short vertical RDL link (A2–B2, C6–D6) lines up with a matching short PCB link.
- Where solder separated on removal, a ball missing on one side matches a ball still present on the other (D1, D2, C2).
- The two PCB traces going to R1/R4 come from balls that have their own die-pad RDL feed (A3, A4).

I use JEDEC-style naming with **A1 = the dot corner**:

```
TOP VIEW (looking down on the board / chip marking)
Marking reads top→bottom (rotated 90° CW) in this view.

            col:  1    2    3    4    5    6
               ┌──────────────────────────────┐
      row A    │ ●A1  ○    ○    ○    ○    ○   │   ← dot side (silk dot is
      row B    │ ○    ○    ○    ○    ○    ○   │     just outside A1)
      row C    │ ○    ○    ○    ○    ○    ○   │
      row D    │ ○    ○    ○    ○    ○    ○   │
               └──────────────────────────────┘
```

How this maps onto the photos:

| Photo | View | Where A1 is | Rows/cols direction |
|---|---|---|---|
| 165806 (chip on board, marking upright) | top | **bottom-left** corner of chip, next to white dot | rows A→D go left→right, cols 1→6 go bottom→top |
| 171512 / 171355 (footprint, "U1" label on left) | top (PCB) | **bottom-right** pad, next to white dot | rows A→D go bottom→top, cols 1→6 go right→left |
| 171031 / 171107 (chip bump side) | bottom | **bottom-left** ball, next to dark marker | rows A→D go bottom→top, cols 1→6 go left→right |

Note that the chip's own marking does **not** put A1 at the top-left. With the marking upright, A1 is at the bottom-left.

## 3. Ball map

```
TOP VIEW, A1 top-left
            1      2      3      4      5      6
   A      GND    GND    sig    sig    GND    GND
   B      GND    GND    GND    GND    GND    GND
   C      ?      ?      ?      ?      ?      GND
   D      ?      ?      ?      ?      ?      GND
```

| Ball | Net | Confidence | Evidence |
|---|---|---|---|
| A1 | GND | [inferred] | RDL link to A2 (chip side) |
| A2 | GND | [inferred] | RDL link to B2 and A1; matching PCB link A2–B2 |
| A3 | signal → R1/R4 area | [inferred] | PCB trace leaves footprint downward toward R1/R4; own RDL feed from die edge pad |
| A4 | signal → R1/R4 area | [inferred] | same as A3 |
| A5 | GND | [inferred] | RDL link to B5 |
| A6 | GND | [inferred] | RDL link to B6 |
| B1 | GND | [inferred] | PCB row-B trace bus (no RDL link to B2 on chip side, so it is tied on the PCB) |
| B2–B6 | GND | [seen]/[inferred] | Chip RDL bus joins B2…B6; PCB bus joins B1…B6 and goes to a via next to C2 |
| C6 | GND | [inferred] | RDL links B6–C6–D6 |
| D6 | GND | [inferred] | RDL link to C6; PCB link C6–D6 |
| C1, D1 | ? (possibly GND or supply) | [TBD] | Short RDL link C1–D1 on chip; a wide RDL trace from there may join the GND bus (hard to resolve optically). Closest balls to decoupling caps C1/C2 |
| C2–C5, D2–D5 | signals / supply | [TBD] | Each has its own RDL trace to a die edge pad |

**Count:** 12 balls very likely GND (A1, A2, A5, A6, B1–B6, C6, D6). That leaves **12 functional balls** at most (A3, A4, C1–C5, D1–D5), fewer if C1/D1 turn out to be GND.

## 4. Other observations

- Bond pads (small copper dots) run around the whole die perimeter under the RDL. Several perimeter pads go to the GND bus, which is why it is so wide.
- Removal damage: **C2** pad on the chip is lifted/dark (the ball stayed on the PCB). **D2** and **D3** are partly de-wetted. **D1**'s PCB pad is bare (the ball came off with the chip).
- Passives next to U1 on the Nano S+: C1 (larger, likely bulk VCC decoupling), C2 (small, likely HF decoupling), and R7 on the col-1 side. R1/R4 are below the A-row signal balls. R1/R4 are probably series resistors or pull-ups on the A3/A4 lines **[TBD]**.

## 5. Next steps

1. Continuity-test every pad on the bare footprint against a known GND (USB shield / battery −) to confirm or correct the GND list. Check C1/D1 first.
2. Trace C1/C2 cap pads to balls to find VCC (and maybe a separate core/IO supply).
3. Trace A3/A4 through R1/R4 to the MCU (STM32 on the S+). Likely candidates are the SE↔MCU link (ISO7816-style IO/CLK/RST or SPI) **[TBD]**.
4. Do the same on the Flex and Stax boards to confirm the footprint and pinout are identical.
