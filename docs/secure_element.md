# Secure Element (U1, ST33K1M5) – Ledger Nano S+ / Flex / Stax

Status: **visual analysis only** (photos in `../pics/`). Nothing has been checked electrically yet.
Each statement below is tagged:

- **[seen]**: visible directly in a photo
- **[inferred]**: worked out from several photos together
- **[TBD]**: needs a multimeter or more imaging

## 1. Identification & package

| Property | Value | Source |
|---|---|---|
| Part | **STMicroelectronics ST33K1M5** (secure MCU, ST33K platform, B01). Probably the **`C`** variant: ST says "ST33K1M5C … mainly target[s] the Customer market, while the ST33K1M5T … the TPM market". Not confirmed; the master ID would tell (`C` = 0x0227, `T` = 0x0247) | user + platform ST §1.4 |
| IC maskset | **K460** (die shows `K460A`) | die photo + ANSSI-CC-2023-08 |
| Die marking | `K460A`, ST logo, `2020` (see §1.1) | [seen] `die_marking_K460A.png` |
| Package marking | Ledger custom laser marking (see below) | [seen] 165806, 171319 |
| Type | WLCSP / bumped die with RDL (bare silicon, polyimide + copper redistribution layer visible on bump side) | [seen] 171031, 171107 |
| Balls | 24, 4 × 6 full array | [seen] |
| Body | ≈ 2.60 × 1.80 mm | [seen] measured against 0.1 mm reticle in 171031 |
| Ball pitch | ≈ 0.40 mm both axes | [seen] |
| Refdes on PCB | U1 (Nano S+) | [seen] 171512 |

The same SE is used in the Nano S+, Flex and Stax.

### 1.0 Platform security target (primary hardware reference)

Source: **ST33K1M5C and ST33K1M5T B01 – Security Target for composition**, SMD_ST33K1M5_ST_21_001 Rev B01.1, July 2022
(STMicroelectronics). Public copy: <https://sec-certs.org/cc/9238e8b74c987b68/target.pdf>
(certificate record: <https://sec-certs.org/cc/9238e8b74c987b68/>). This is the chip's own CC document, evaluated under the
Dutch NSCIB scheme at **EAL6 augmented with ALC_FLR.1** and claiming conformance to BSI-CC-PP-0084-2014. Section numbers below refer to it.

**Identification** (§1.4, Table 1):

| IC maskset | Commercial product | Master ID | IC version | Firmware version |
|---|---|---|---|---|
| **K460** | ST33K1M5C / ST33K1M5T | 0x0227 / 0x0247 | B / C | 3.1.3 / 3.1.4 |

- "The IC maskset name is the product hardware identification." The IC version changes with any hardware change (some mask layers) or any change to the OST test software.
- The marking on the die, readable registers and specific instructions give the product information (§1.4 ¶18). The die marking `K460A` is part of that identification.
- All derivatives share the same hardware design and maskset. Each derivative has its own "Child product identification number" (listed in the datasheet and the FW user manual).
- Derivatives are configured in **Admin configuration**, by ST or by the customer (§1.5 Table 2):
  - NVM size: 512, 768, 1024, 1280 or 1534 KB
  - CPU and SPI frequency, temperature range: depend on the commercial product

**Hardware** (§1.5–1.6.1):

- CPU: **2 × Arm Cortex-M35P in lockstep**, 70 or 75 MHz, Thumb-2. Each has an Armv8-M MPU with up to 16 regions.
- **LPU** (Library Protection Unit): isolates protected code such as a library from the rest. It may be reserved by ST for its own libraries.
- Memory: **64 KB user RAM**, up to **1534 KB user Flash (NVM)**, ST ROM and System ROM, an OST ROM for test, and a Flash read cache.
- Crypto:
  - AES-128/192/256 accelerator (ECB, CBC)
  - 3-key Triple-DES accelerator "EDES+" (ECB, CBC)
  - **NESCRYPT LLP** public-key coprocessor with its own RAM
  - SM4 and MIFARE accelerators (out of evaluation scope)
  - 16/32-bit CRC block
- TRNG: AIS20/AIS31 PTG.2, accessed directly through registers.
- **Interfaces** (¶36):
  - **2 × ISO/IEC 7816-3 slave UARTs** (T=0, T=1)
  - **SWP** slave interface (single-wire protocol to an NFC router)
  - **SPI master/slave**
  - **I²C master/slave**
  - Every interface has its own RAM buffer.
- Timers: 3 × 16-bit general-purpose timers, a watchdog, and a permanent timer that keeps counting in low-power mode.
- Other blocks: reset manager, clock manager, "security administrator".
- **Pins shown in the block diagram (Figure 1): `VCC1`, `VCC2`, `GND1`, `GND2`, `10 GPIOs`**. See §3 for how this fits our ball count.

**Firmware and life-cycle states** (§1.6.2, §6.3):

- **OST ROM**: ST's test OS, unreachable after delivery.
- **System ROM + ST NVM firmware**, which includes:
  - The **Secure Flash Loader**: a fast, ciphered, integrity-checked download of the embedded software into NVM. It is available only in Admin configuration, and the customer can disable it irreversibly (§6.3 ¶334).
  - **Flash Drivers** that the embedded software uses to manage NVM (User configuration).
  - Protected test and profiling commands (not available in User configuration).
  - A small set of diagnostic commands reserved to ST.
  - **Secure Diagnostic**: the customer must enable it, and ST then runs it on its own audited sites. The customer can disable it permanently. Its status can be read: permanently disabled, permanently enabled, or still configurable (§6.5).
- Configurations: **Test** (ST only), **Admin** (loader and configuration), **User** (normal run), and **Diagnostic volatile** (ST only). Transitions are controlled by fuses, authentication and control registers (§6.3).
- In Admin configuration, the firmware can prove the chip is genuine to another device using a cryptographic mechanism (§6.17, FIA_API.1).
- Loader sessions are encrypted, integrity-checked and logged (§6.18–6.25).
- For Ledger, presumably: the OS is loaded through the Secure Flash Loader in Admin configuration, then the chip is switched to User configuration. **[TBD]** whether Ledger keeps the loader for OS updates or uses its own bootloader in User NVM.

**Security mechanisms** (§6):

- **Events that trigger a reset or interrupt** (§6.2): die-integrity violation (active shield), memory/register errors, glitches, high supply voltage, out-of-range temperature, CPU/MPU errors (including lockstep mismatch), faults on the crypto processors. The software can also trigger a reset itself.
- **Confidentiality**: memory scrambling and encryption, **bus encryption**, concealment of operation execution, MPU/LPU, active shields (§6.6, §6.10).
- **Integrity**: error-detecting codes (EDC) on the memories, plus MPU/LPU (§6.7).
- **Physical**: anti-probing countermeasures, and active shields that react automatically when die integrity is violated (§6.9). This matches the shield mesh visible in `die_marking_K460A.png`.

**Delivery** (§1.6.4, §1.7): wafers, micromodules or packages, with the firmware already on the chip. The evaluation covers phases 2–3 and optionally phase 4 (packaging).

**Sites** (Table 15, abridged):

- Wafer fab (FE): **ST Crolles**, which also makes masks
- Masks: also AMTC/Toppan Dresden, DNP (Japan), DPE (Agrate)
- Wafer test (EWS): ST Rousset, ST Toa Payoh, Ardentec (Taiwan)
- Packaging (BE): Amkor Taiwan (ATT1/3/6), Smartflex (Singapore), ST Bouskoura (Morocco), STS Shenzhen, UTAC Thailand, CMP Gardanne
- Design (DEV): Rousset, Crolles, Grenoble, Rennes, Sophia, Zaventem, Catania, Palermo, Ljubljana, Ang Mo Kio

**Public-key coprocessor scope**: the platform ST certifies only the AES and DES accelerators as crypto services. RSA and ECC run on the NESCRYPT LLP through ST's **NesLib** library, which is certified separately (NesLib 6.7.4: certificate CC-22-0322914/2, security target <https://www.commoncriteriaportal.org/files/epfiles/NSCIB-CC-0322914-STlite%20_v02_0.pdf>).

**Guidance documents** (Table 14). These are NDA-only and sent by ST as encrypted PDFs, but the names help when searching:

- `DS_ST33K1M5C` v4 / `DS_ST33K1M5T` v3: datasheets (these would contain the ballout)
- `UM_ST33K_FW` v6: platform firmware V3 user manual
- `AN_SECU_ST33K`: security guidance
- `UM_ST_TRNG14`, `AN_ST33K_TRNG`: TRNG documentation
- Arm Cortex-M35P TRM `100883_0101_00_en` (r1p1): public from Arm

### 1.0b What the ANSSI-CC-2023-08 (TPM) document adds

Source: **ANSSI-CC-2023-08, Security Target "ST33KTPM2XSPI & ST33KTPM2XI2C – TPM firmware 9.256"**
(STMicroelectronics, rev 01-02p, 8 Nov 2022), <https://messervices.cyber.gouv.fr/visas/ANSSI-CC-2023-08-cible.pdf>

> ⚠️ This is the security target for **ST's own TPM 2.0 product**, not for Ledger's firmware. It is useful
> because the TPM is a composite product built on the same hardware platform, so its platform section
> identifies our die. Everything about the TPM firmware, the SPI/I²C TPM interfaces and the EK keys applies
> to the TPM product, **not** to the Ledger SE.

**Hardware platform identification** (ST §2.1, quoted):

| Field | Value |
|---|---|
| Platform | ST33K1M5T B01 (configuration B01) |
| IC maskset name | **K460** ← matches the die marking `K460A` |
| Master identification number | 0x0247 |
| IC version | C |
| IC firmware version | 3.1.4 |
| Crypto library on top | NesLib 6.7.4 (ST's certified crypto library) |

"The chip package is not included in the TOE": the certification covers the die only. That fits Ledger's
custom-marked WLCSP.

**Platform certification** (ST §2.3.1 and Appendix A):

- Hardware certificate: **CC-22-0252712/2**, "ST33K1M5C and ST33K1M5T B01".
- Platform security target: *ST33K1M5C and ST33K1M5T B01, Security Target for composition*, Rev B01.1, July 2022. Now summarised in §1.0 above.
- Platform assurance level: **EAL6 augmented with ALC_FLR.1** (ST §8.2.4). The TPM composite is only EAL4+ (ALC_DVS.2, ALC_FLR.1, AVA_VAN.5).
- NesLib 6.7.4 on ST33K1M5C/T B01: certificate **CC-22-0322914/2**, EAL5+ (ALC_FLR.1, ALC_DVS.2, AVA_VAN.5).

**Manufacturing and development sites** (ST §2.4; listed for the TPM, but the hardware part presumably applies to the K460 die):

- HW development: ST Rousset (FR), ST Ang Mo Kio (SG)
- Embedded SW development (TPM): ST Rousset, ST Rennes (FR), ST Zaventem (BE)
- Die manufacturing (wafer fab): **ST Crolles (FR)**
- Test: ST Rousset (FR), ST Toa Payoh (SG)

**Platform security mechanisms the TPM composite relies on** (ST §8.2.3, Table 8, platform SFR names). These hint at what the hardware provides:

- FRU_FLT.2 / FPT_FLS.1: fault tolerance and secure state on failure (glitch/fault-injection detection)
- FPT_PHP.3: resistance to physical attack (consistent with the active shield seen on the die)
- FDP_ITT.1 / FPT_ITT.1 / FDP_IFC.1: protection of internal transfers and leakage control (side channels)
- FDP_SDC.1 / FDP_SDI.2: stored-data confidentiality and integrity (memory encryption/integrity)
- FCS_RNG.1 /PTG.2: physical TRNG (AIS-31 class PTG.2)
- FCS_COP.1: hardware crypto accelerators for AES, RSA, ECC, SHA-2, SHA-3 and DRBG, used through NesLib
- FMT_LIM.1/2 /Test, FAU_SAS.1: test features are locked after manufacturing, and an audit record is written during test
- FMT_LIM.1/2 /Loader, FIA_API.1, FTP_ITC.1 /Loader: the platform has a **Flash Loader** for secure download of user software. The TPM product deactivates it permanently. **[TBD]** how Ledger provisions its OS onto the SE (presumably through this loader, at Ledger's factory).
- FDP_ACC.2 / FDP_ACF.1 / FMT_MSA… /Memories: hardware memory access control (MPU/firewall)

### 1.1 Die marking (metal layer, `pics/die_marking_K460A.png`)

```
 K460A
 (C) STW  ■
          (C) ZMB  ■  ST3
          [ST logo]  (M) (C)
                      2020
```

- `K460A`: **confirmed**. The IC maskset name of the ST33K1M5C/T B01 platform is `K460`, according to ANSSI-CC-2023-08 §2.1. The trailing `A` is probably a mask revision or variant letter **[TBD]**.
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
- The horizontal RDL bus on the chip (B2–B6) lines up with the horizontal PCB trace bus in the same row.
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
   B      VCC    GND    GND    GND    GND    GND
   C      GND?   ?      ?      ?      ?      GND
   D      GND    ?      ?      ?      ?      GND
```

| Ball | Net | Confidence | Evidence |
|---|---|---|---|
| A1 | GND | [inferred] | RDL link to A2 (chip side) |
| A2 | GND | [inferred] | RDL link to B2 and A1; matching PCB link A2–B2 |
| A3 | signal → R1/R4 area | [inferred] | PCB trace leaves footprint downward toward R1/R4; own RDL feed from die edge pad |
| A4 | signal → R1/R4 area | [inferred] | same as A3 |
| A5 | GND | [inferred] | RDL link to B5 |
| A6 | GND | [inferred] | RDL link to B6 |
| B1 | **VCC** | **[measured]** (2026-11) | Confirmed by measurement. A closer look at 171355 agrees: B1 has its own short PCB trace to the via next to decoupling cap C2, with **no** copper link to B2, and no RDL link to B2 on the chip side. (The earlier reading that the PCB bus covered B1 was wrong.) |
| B2–B6 | GND | [seen]/[inferred] | Chip RDL bus joins B2…B6, and a matching PCB trace bus joins B2…B6 |
| C6 | GND | [inferred] | RDL links B6–C6–D6 |
| D6 | GND | [inferred] | RDL link to C6; PCB link C6–D6 |
| D1 | **GND** | **[measured]** (2026-11) | Confirmed by measurement. This also confirms that the wide RDL trace from the C1/D1 junction to the row-B GND bus (via the vertical trace between columns 2 and 3) is a real connection. |
| C1 | GND (likely) | [inferred] | Short RDL link to D1 on the die, so it is the same net as D1. **[TBD]** beep to confirm. |
| C2–C5, D2–D5 | signals / supply | [TBD] | Each has its own RDL trace to a die edge pad |

**Count:** 1 VCC (B1, measured) and 13 GND (D1 measured; A1, A2, A5, A6, B2–B6, C1, C6, D6 inferred). That leaves **10 balls** (A3, A4, C2–C5, D2–D5) to identify, and each has its own RDL trace to a die edge pad.

### 3.1 Pin budget vs. the platform block diagram

The block diagram in the platform ST (Figure 1) shows the chip's external pins as **VCC1, VCC2, GND1, GND2 and 10 GPIOs**.
The ISO7816 UARTs, SWP, SPI and I²C presumably come out on those GPIOs.

- **B1 = VCC** (measured), probably `VCC1`, the main supply, decoupled by C2 through the adjacent via.
- **D1 = GND** (measured), and C1 is RDL-tied to D1. So C1/D1 is **not** `VCC2`, which rules out my earlier hypothesis.
- That gives 24 balls = 13 GND + 1 VCC + **10 remaining balls (A3, A4, C2–C5, D2–D5)**, which matches the block diagram's **10 GPIOs** exactly.
- So `VCC2` does not seem to have its own ball in this package. Possible explanations **[TBD]**:
  - `VCC2` is tied to `VCC1` on the die or in the RDL (B1 serving both).
  - `VCC2` powers a domain that this package doesn't bring out, for example the SWP/contactless IO supply on parts with an SWP pin.
  - One of the 10 "GPIO" balls is actually `VCC2`. Checking each with a meter against B1, and under power, would settle it.
- `GND1`/`GND2` both land on the 13 GND balls.
- Reset and clock would then be GPIO functions or internal (the chip has its own clock and reset managers).

## 4. Other observations

- Bond pads (small copper dots) run around the whole die perimeter under the RDL. Several perimeter pads go to the GND bus, which is why it is so wide.
- Removal damage: **C2** pad on the chip is lifted/dark (the ball stayed on the PCB). **D2** and **D3** are partly de-wetted. **D1**'s PCB pad is bare (the ball came off with the chip).
- Passives next to U1 on the Nano S+: C1 (larger, likely bulk VCC decoupling), C2 (small, likely HF decoupling), and R7 on the col-1 side. R1/R4 are below the A-row signal balls. R1/R4 are probably series resistors or pull-ups on the A3/A4 lines **[TBD]**.

## 5. Next steps

1. Continuity-test every pad on the bare footprint against a known GND (USB shield / battery −) to confirm or correct the GND list. D1 is confirmed GND; beep C1 next (expected GND).
2. ~~Find VCC~~: B1 = VCC (measured). Next, check whether any of the 10 remaining balls is shorted to B1 or sits at a supply level under power (a possible `VCC2`).
3. Trace A3/A4 through R1/R4 to the MCU (STM32 on the S+). Likely candidates are the SE↔MCU link (ISO7816-style IO/CLK/RST or SPI) **[TBD]**.
4. Do the same on the Flex and Stax boards to confirm the footprint and pinout are identical.
