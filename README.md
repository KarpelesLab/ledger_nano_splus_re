# Ledger Nano S+ reverse engineering

Hardware reverse-engineering notes for the Ledger Nano S+. The current focus is the
secure element, which the Nano S+, Flex and Stax all share.

## Contents

| Path | What |
|---|---|
| [`docs/secure_element.md`](docs/secure_element.md) | U1 secure element (ST33K1M5, maskset K460): package, markings, orientation, ball map |
| `pics/` | Board, footprint, chip and die micrographs |

## Status

- **Secure element:** identified as STMicroelectronics ST33K1M5 (B01 platform), maskset K460.
  This is confirmed by the die marking and by ST's TPM security target
  [ANSSI-CC-2023-08](https://messervices.cyber.gouv.fr/visas/ANSSI-CC-2023-08-cible.pdf),
  which names K460 as the maskset of its ST33K1M5C/T B01 hardware platform. It's a 24-ball 4×6 WLCSP
  (≈2.6 × 1.8 mm, 0.4 mm pitch) with Ledger's own laser marking (`!L! KC7A6`).
  Orientation is established (A1 = silkscreen-dot corner). The ball map comes from
  photos, plus one measurement: **B1 = VCC** (measured). 11 balls are likely GND, and the rest
  still need continuity checks. The current hypothesis is C1/D1 = VCC2 and the 10 remaining balls = the 10 GPIOs.

## External references

- [ST33K1M5C and ST33K1M5T B01 – Security Target for composition](https://sec-certs.org/cc/9238e8b74c987b68/target.pdf)
  (SMD_ST33K1M5_ST_21_001 Rev B01.1, [certificate record](https://sec-certs.org/cc/9238e8b74c987b68/)).
  This is the chip's own CC document (EAL6+): CPU, memories, interfaces, pins, life cycle and security mechanisms.
- [ANSSI-CC-2023-08 security target](https://messervices.cyber.gouv.fr/visas/ANSSI-CC-2023-08-cible.pdf):
  ST33KTPM2X TPM, built on the same ST33K1M5 B01 / K460 platform. It is the source of the maskset, platform
  certificate and fab-site details summarised in `docs/secure_element.md` §1.0.

## Conventions

- Ball names are JEDEC-style, viewed from the top: rows A–D, columns 1–6, with A1 at the
  PCB silkscreen-dot corner. See `docs/secure_element.md` §2 for how this maps onto each photo.
- Each claim in the docs is tagged **[seen]** (visible in a photo), **[inferred]**
  (worked out from several photos) or **[TBD]** (needs confirming).
