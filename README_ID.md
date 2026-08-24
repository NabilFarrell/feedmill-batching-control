# Sistem Batching & Dosing Feedmill

Sistem batching feedmill simulasi dengan PLC state machine, komunikasi OPC UA, dan dashboard Node-RED.

> [English Version →](README.md)

## Ikhtisar

Sistem pemrosesan batch lengkap untuk produksi pakan ternak. Sistem mengontrol 4 bin bahan baku, hopper penimbang dengan load cell, mixer, dan gerbang discharge. Mendukung 4 resep yang sudah dikonfigurasi dengan penghitungan batch otomatis dan operasi berbasis target.

## Arsitektur

```
┌─────────────────────────────────────────────┐
│              CODESYS PLC                     │
│  ┌──────────────┐  ┌─────────────────────┐  │
│  │ Controller_ST │  │   Controller_LD     │  │
│  │ (State Mach.) │→ │ (Safety Interlocks) │  │
│  └──────────────┘  └─────────────────────┘  │
│         ↓                                    │
│  ┌──────────────┐                            │
│  │PlantModel_ST │                            │
│  │ (Physics)    │                            │
│  └──────────────┘                            │
└─────────────────┬───────────────────────────┘
                  │ OPC UA
┌─────────────────▼───────────────────────────┐
│          Node-RED Dashboard                  │
│  ┌────────┬──────────┬──────────┬─────────┐  │
│  │ Status │ Process  │ Controls │  Alarm  │  │
│  └────────┴──────────┴──────────┴─────────┘  │
└─────────────────────────────────────────────┘
```

## Fitur Utama

- **6 State Machine:** IDLE → DOSING → MIXING → DISCHARGING → COMPLETE → FAULT
- **4 Resep:** Poultry, Pig Growler, Pig Finisher, Custom
- **Safety Interlocks:** E-Stop latching, gate interlocks, mixer safety
- **Sistem Alarm:** Empty Bin, Underweight, Emergency Stop
- **Manual Discharge:** Override operator untuk maintenance
- **Auto-Restart:** Batching kontinu hingga target tercapai
- **Dashboard Real-Time:** Weight, level bin, status batch secara live

## Struktur Repository

```
├── README.md                    ← versi English
├── README_ID.md                 ← versi Indonesia (ini)
├── code/                        ← ekspor kode PLC (PDF)
│   ├── Controller-ST.pdf
│   ├── Controller-LD.pdf
│   ├── PlantModel-ST.pdf
│   └── GVL.pdf
├── screenshots/                 ← screenshot dashboard + CODESYS
├── video/                       ← rekaman demo
├── project-files/               ← flow JSON Node-RED
└── docs/                        ← dokumen spesifikasi
```

## Video Demo

<!-- Tambahkan embed video atau link setelah upload ke YouTube/Loom -->
[Tonton Video Demo →](video/)

## Screenshot

<!-- Tambahkan screenshot dashboard di sini -->
| IDLE | DOSING | COMPLETE | FAULT |
|------|--------|----------|-------|
| ![IDLE](screenshots/Dashboard%20Startup.png) | ![DOSING](screenshots/Dashboard%20Dosing.png) | ![COMPLETE](screenshots/Dashboard%20Complete.png) | ![FAULT](screenshots/Dashboard%20Emergency%20Stop.png) |

## Keputusan Desain Utama

1. **Pola Edge-to-Latch** — Input operator (Start, Stop, Alarm_Reset) menggunakan deteksi rising-edge dengan latching untuk mencegah pemicu berulang dari tombol yang ditahan
2. **E-Stop di LD** — Safety-critical latching diimplementasikan dalam ladder logic (bukan structured text) untuk perilaku scan-cycle yang deterministik
3. **Mixer Tetap Menyala Saat Discharge** — Mencegah loop self-defeating di mana Mixer_Done collapse saat Mixer_Activate turun pada transisi state
4. **Toggle Manual Discharge** — Toggle latch dengan kondisi guard (hanya bekerja di state IDLE atau FAULT)

## Yang Saya Pelajari

- Desain PLC state machine dengan transisi guard yang tepat
- Integrasi OPC UA antara CODESYS dan Node-RED
- Pola desain safety interlock (set/reset coils, gate mutual exclusion)
- Desain HMI industri dengan dark theme dan indikator status berwarna
- Debugging masalah simulasi (AccessViolation, stuck state, self-defeating loop)

## Teknologi

- **PLC:** CODESYS V3.5 SP22 Patch 3, CODESYS Control Win V3 x64
- **Komunikasi:** OPC UA (konfigurasi Symbol Set)
- **HMI:** Node-RED dengan node-red-dashboard, node-red-contrib-opcua
- **Bahasa Pemrograman:** IEC 61131-3 Structured Text (ST), Ladder Diagram (LD)

## Penulis

Muhammad Nabil Farrell — Insinyur Automasi Industri

- GitHub: [github.com/NabilFarrell](https://github.com/NabilFarrell)
- LinkedIn: [linkedin.com/in/nabil-farrell](https://www.linkedin.com/in/nabil-farrell/)
- Email: nabilfarrellid@gmail.com

## Lisensi

Projet ini untuk tujuan portofolio/demonstrasi.
