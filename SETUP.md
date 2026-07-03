# SETUP.md — Canonical Device Inventory

This file is the canonical hardware inventory for the operator's devices. When
asked to "list my devices" / "devices with specs" / anything about hardware,
read this file — it wins over any cached/chat memory on conflict.

_Last updated: 2026-07-03_

## Devices

| # | Device | Motherboard | CPU | RAM | GPU | Storage (key) |
|---|--------|-------------|-----|-----|-----|---------------|
| 1 | **PC1 / SHOP-AI** | Gigabyte Z890 Aorus Elite WiFi7 | Ultra 9 285K | 128GB DDR5-4800 | RTX 5090 32GB | C: Kioxia Exceria 2TB NVMe · D: Crucial P310 4TB · F: Samsung 860 EVO 4TB SATA (POS clone target) |
| 2 | **PC2 / HOME-DESK** | Gigabyte B860M Eagle WiFi6 V2 | Ultra 7 265 | 64GB DDR5-6400 | RTX 5080 16GB | Crucial P310 4TB Gen4 · Crucial P510 2TB Gen5 (SMART pending) |
| 3 | **PC3 / SHOP-DESK** | Gigabyte Z370 HD3 | i7-8700 | 64GB DDR4-3200 | GTX 1060 6GB | C: BX500 2TB · D: IronWolf 12TB ⚠️ replace · E: IronWolf Pro 16TB |
| 4 | **LENOVO** X1 Nano Gen 1 | OEM (20UQS1XT01) | i5-1140G7 | 16GB | iGPU | 238GB SSD (1TB upgrade planned) |
| 5 | **HP** Dragonfly G4 | OEM | i7-1365U | 32GB | Iris Xe | 1TB SK Hynix BC901 Gen4 |
| 6 | **HOME-TRUENAS** | CWWK QS-Q670-PLUS | i5-14400 | 64GB DDR5 | — | 26TB Exos mirror · 4TB IronWolf Pro mirror (ex-WD Reds) · APPS: 2× SanDisk SSD · boot: Kingston NVMe |
| 7 | **SHOP-TRUENAS** | CWWK QS-Q670-PLUS | i5-14400 | 64GB DDR5 | — | 28TB Exos mirror · 4TB IronWolf Pro mirror (ex-3TB pool) · APPS: 2× T500 NVMe · boot: SP A55 · TX401 10GbE |

## Notes / open items

- NAS motherboards recorded as **CWWK QS-Q670-PLUS** (both boxes, same spec).
  To verify the exact baseboard string, run on each TrueNAS shell:
  `dmidecode -t baseboard`
- PC3 **D: IronWolf 12TB** flagged for replacement (SMART caution).
- Stale NAS figures from older chats (3TB/28TB, 4TB/26TB RAID-1) are
  **superseded** by the post-rebuild layout above.
