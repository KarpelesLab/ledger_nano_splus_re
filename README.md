# Ledger Nano S+ reverse engineering

Hardware reverse-engineering notes for the Ledger Nano S+. The current focus is the
secure element, which the Nano S+, Flex and Stax all share.

## Contents

| Path | What |
|---|---|
| [`docs/secure_element.md`](docs/secure_element.md) | U1 secure element (ST33K1M5, die `K460A`): package, markings, orientation, ball map |
| `pics/` | Board, footprint, chip and die micrographs |

## Status

- **Secure element:** identified as STMicroelectronics ST33K1M5. It's a 24-ball 4×6 WLCSP
  (≈2.6 × 1.8 mm, 0.4 mm pitch) with Ledger's own laser marking (`!L! KC7A6`).
  Orientation is established (A1 = silkscreen-dot corner). The ball map comes from
  photos only: 12 balls are likely GND, and the rest still need continuity checks.

## Conventions

- Ball names are JEDEC-style, viewed from the top: rows A–D, columns 1–6, with A1 at the
  PCB silkscreen-dot corner. See `docs/secure_element.md` §2 for how this maps onto each photo.
- Each claim in the docs is tagged **[seen]** (visible in a photo), **[inferred]**
  (worked out from several photos) or **[TBD]** (needs confirming).
