# Sistem Batching & Dosing Feedmill

Sistem kontrol CODESYS dua layer untuk jalur batching feedmill, dengan HMI Ignition Perspective di atas OPC UA.

> [English Version →](README.md)

## Ikhtisar

Stasiun batching feedmill simulasi: **4 bin bahan baku → weigh hopper (load cell) → mixer → gerbang discharge**. Sistem menjalankan siklus batch 6-state dengan resep, safety interlock, pengecekan toleransi, dan alarm latched, yang semuanya diekspos ke HMI melalui OPC UA.

**Pilihan desain utama** adalah pemisahan layer (layer separation). Controller ditulis persis seolah-olah berjalan pada perangkat keras asli — controller hanya membaca nilai sensor dan menulis variabel perintah, serta tidak mengetahui bagaimana fisik disimulasikan. Semua nilai sensor dan integrasi aliran berada di dalam plant model yang terpisah. Ini adalah pemisahan yang sama yang digunakan dalam factory acceptance testing (FAT), yang berarti logika kontrol dapat dipercayai tanpa memerlukan pengetahuan apa pun tentang simulasi.

## Arsitektur

```
┌──────────────────────────────────────────────────────┐
│              CODESYS Control Win V3 (soft PLC)        │
│  Task: MainTask — IEC task, 100 ms cycle              │
│                                                      │
│  PLC_PRG                                            │
│   ├── Controller_ST   ← REAL logic                    │
│   │                     state machine, recipes,      │
│   │                     tolerance, alarms             │
│   ├── Controller_LD   ← interlocks + E-Stop latch    │
│   └── PlantModel_ST   ← FAKE physics                  │
│                         flow integration, gate        │
│                         travel delay, clamps          │
└────────────────────┬─────────────────────────────────┘
                     │ OPC UA  (opc.tcp://localhost:4840)
┌────────────────────▼─────────────────────────────────┐
│        Ignition Perspective 8.3 — HMI                │
│  ┌────────────┬────────────┬────────────────────────┐  │
│  │  Overview  │  Controls  │  History               │  │
│  │  (process  │  (operator │  (trend + batch log +  │  │
│  │   mimic)   │   inputs)  │   alarm journal)       │  │
│  └────────────┴────────────┴────────────────────────┘  │
└──────────────────────────────────────────────────────┘
```

## Tiga Program

| Program | Language | Owns |
|---|---|---|
| `Controller_ST` | Structured Text | 6-state machine, pemilihan resep, logika target kumulatif, pengecekan toleransi ±2%, pengecekan bin kosong, alarm latching |
| `Controller_LD` | Ladder Diagram | Interlock fisik, set/reset latch E-Stop, eksklusi mutual gerbang, output aktuator |
| `PlantModel_ST` | Structured Text | Fisika saja — integrasi aliran per bin, tundaan perjalanan gerbang, laju discharge, clamping pada nol |

Layer-layer ini tidak pernah tercampur. `Controller_ST` tidak pernah menyentuh fisika; `PlantModel_ST` tidak pernah membuat keputusan kontrol.

## State Machine

| State | Name | Behaviour |
|---|---|---|
| 0 | IDLE | Semua gerbang tertutup, mixer mati. Menerima Start hanya ketika hopper < 5 kg dan tidak ada alarm aktif |
| 1 | DOSING | Bin 1→4 secara berurutan. Buka gerbang, integrasikan hingga target kumulatif, tutup, tunggu feedback tertutup |
| 2 | MIXING | Mixer berjalan selama MIX_TIME tetap |
| 3 | DISCHARGING | Mixer tetap menyala, gerbang discharge terbuka, hopper terkuras hingga < 5 kg |
| 4 | COMPLETE | `Batch_Counter += 1`, `Total_Weight += total resep`, tahan selama 2 detik, lalu auto-restart atau kembali ke IDLE |
| 5 | FAULT | Semua aktuator mati. Ditahan hingga alarm di-acknowledge dan di-reset |

## Resep

| Recipe | Bin 1 (Corn) | Bin 2 (Soybean Meal) | Bin 3 (Wheat Bran) | Bin 4 (Limestone) | Total |
|---|---|---|---|---|---|
| 1 — Poultry | 200 | 150 | 100 | 50 | **500 kg** |
| 2 — Cattle | 150 | 100 | 75 | 25 | **350 kg** |
| 3 — Supplement | 250 | 125 | 75 | 50 | **500 kg** |

`Recipe_Select` di-latch pada rising edge Start — mengubahnya di tengah batch hanya akan diterapkan pada batch berikutnya.

## HMI — Ignition Perspective

Tiga view, dengan docked header bersama yang menyediakan navigasi dan status sistem.

**Overview** — process mimic dengan level bin live, berat per bin, pewarnaan status valve, pengisian mixer, berat hopper vs target, dan produksi kumulatif. Kotak status persisten melaporkan `OK` / `EMERGENCY STOP` / `FAULT`.

**Controls** — pemilihan resep, target jumlah batch, auto-restart, start/stop, E-Stop, safety release, reset alarm, dan manual discharge.

**History** — tren load cell, tabel riwayat batch, dan alarm journal.

Konvensi HMI yang diterapkan:

- **Disiplin warna ISA-101** — abu-abu untuk peralatan idle, warna dicadangkan untuk state, merah hanya untuk alarm
- **Manajemen alarm native** — alarm adalah event alarm Ignition nyata dengan prioritas (High/Medium), siklus acknowledgement, dan jurnal persisten. `Alarm_EmptyBin` dan `Alarm_Underweight` di-latch di PLC dan di-acknowledge serta dibersihkan dari HMI
- **Identifikasi fault per-bin** — PLC mengekspos satu flag global `Alarm_EmptyBin`, sehingga HMI menurunkan indikasi per-bin dari tag `BinLevel_n` individu terhadap threshold kosong. Ini memungkinkan operator melihat bin *mana* yang habis, yang tidak dapat disampaikan oleh flag PLC saja
- **Navigasi berbasis halaman** dengan region docked header, dan tombol navigasi yang menyoroti view aktif

## Migrasi HMI: Node-RED → Ignition Perspective

Versi pertama menggunakan dashboard Node-RED sebagai layer prototyping cepat. Dashboard tersebut digantikan dengan Ignition Perspective karena satu alasan substansial:

**Node-RED tidak dapat melakukan manajemen alarm.** Dashboard asli merender alarm sebagai teks berwarna dan tabel yang dibuat secara manual. Ignition menyediakan sistem alarm nyata — prioritas, siklus acknowledgement, status shelve/cleared, dan journal — yang merupakan hal yang benar-benar diharapkan oleh operator industri dan standar manajemen alarm (ISA-18.2). Segala hal lain tentang HMI juga berpindah: pewarnaan state ISA-101, identifikasi fault per-bin, navigasi berbasis halaman, dan banner alarm persisten.

Versi Node-RED disimpan di bawah `v1-node-red/` dan `screenshots/v1-node-red/` sebagai layer prototyping v1.

## Fitur

- **Siklus batch 6-state** dengan guard transisi dan penanganan safe-state
- **3 resep** dengan logika target kumulatif di 4 bin
- **Safety interlock** — latching E-Stop, eksklusi mutual gerbang, interlock mixer-gerbang
- **Sistem alarm latched** — bin kosong, underweight (toleransi ±2%), dengan acknowledgement dan reset
- **Software latch E-Stop** yang mensimulasikan relay safety perangkat keras, dengan aksi release dan reset terpisah
- **Manual discharge override** untuk maintenance, diguard ke state IDLE dan FAULT saja
- **Auto-restart** untuk batching kontinu hingga target hitungan tercapai
- **Batch history dan alarm journal** dengan waktu, prioritas, dan state

## Struktur Repository

```
├── README.md                    ← this file
├── README_ID.md                 ← Bahasa Indonesia
├── .gitignore                   ← artefak build CODESYS + setting mesin tidak ikut di-commit
├── code/                        ← export kode PLC (PDF) + source project CODESYS
│   ├── Controller-ST.pdf
│   ├── Controller-LD.pdf
│   ├── PlantModel-ST.pdf
│   ├── GVL.pdf
│   └── Feedmill Control.project ← project CODESYS (import untuk menjalankan PLC)
├── docs/
│   ├── mock_project_1_feedmill_batching_spec.md   ← dokumen spesifikasi lengkap
│   └── hardware-reference.md     ← pemetaan variabel → hardware nyata
├── hmi/                         ← archive project Ignition Perspective
│   └── designer project export.zip
├── screenshots/
│   ├── hmi/                     ← capture client + designer HMI saat ini
│   │   ├── client-*.jpg
│   │   └── designer-*.jpg
│   ├── codesys/                 ← project tree, symbol set OPC UA
│   └── v1-node-red/             ← dashboard Node-RED (arsip)
├── video/
│   ├── feedmill-batching-demo.mp4   ← demo HMI lengkap
│   └── v1-node-red/             ← demo v1 (arsip)
└── v1-node-red/                 ← flow JSON Node-RED (layer HMI v1)
    └── flows_codesys_opcua.json
```

## Demo

**Saat ini — HMI Ignition Perspective**

| Skenario | Video |
|---|---|
| Run lengkap: idle → dosing → mixing → discharging → complete, plus penanganan E-Stop dan alarm | [feedmill-batching-demo.mp4](video/feedmill-batching-demo.mp4) |

**v1 — Layer HMI Node-RED** (diarsipkan)

| Skenario | Video |
|---|---|
| Alarm bin kosong, status error, release | [Idle → Control → Start → Empty Bin → Release Error Status](video/v1-node-red/Idle-Control-Start-Empty%20Bin-Release%20Error%20Status.mp4) |
| Aktivasi E-Stop dan pelepasan safety | [Idle → Control → Start → Emergency → Release Emergency](video/v1-node-red/Idle-Control-Start-Emergency-Release%20Emergency.mp4) |

## Keputusan Desain Utama

1. **Arsitektur dua layer** — Logika controller ditulis seolah-olah pada hardware asli, plant model memiliki seluruh fisika. Pemisahan yang sama seperti pengujian FAT; controller tidak pernah tahu bahwa ia sedang disimulasikan
2. **Pola edge-to-latch** — Input operator (Start, Stop, Alarm_Reset) menggunakan deteksi rising-edge dengan latching untuk mencegah pemicu berulang dari tombol yang ditahan
3. **E-Stop di ladder, bukan ST** — Safety latching diimplementasikan dalam LD untuk perilaku scan-cycle yang deterministik, dengan aksi release dan reset terpisah sehingga restart selalu menjadi tindakan multi-langkah yang disengaja
4. **Mixer tetap menyala selama discharge** — Mencegah loop self-defeating di mana `Mixer_Done` collapse saat `Mixer_Activate` turun pada transisi state
5. **Manual discharge sebagai toggle dengan guard** — Hanya diizinkan di state IDLE atau FAULT, dan berhenti sendiri saat hopper kosong
6. **HMI menurunkan indikasi per-bin, PLC memiliki alarm** — HMI tidak membuat keputusan kontrol, hanya menampilkan `BinLevel_n` terhadap threshold yang sama yang digunakan PLC
7. **Node-RED → Ignition untuk manajemen alarm** — Lihat bagian migrasi di atas

## Yang Saya Pelajari

- Desain PLC state machine dengan guard transisi yang tepat dan penanganan safe-state
- Mengapa pemisahan layer itu penting: membuat logika kontrol dapat diverifikasi secara independen
- Integrasi OPC UA antara CODESYS dan Ignition — konfigurasi symbol set, pembuatan sertifikat, langganan tag
- Tag Groups (scan classes) dan mengapa rate default 1000 ms adalah jebakan latensi untuk tampilan mimic live
- Pola safety interlock — koil set/reset, eksklusi mutual gerbang, pengurutan reset yang disengaja
- Manajemen alarm Ignition — prioritas, siklus acknowledgement, dan mengapa flag global PLC saja tidak cukup bagi operator
- Disiplin warna HMI ISA-101 dan cara mengenkode state tanpa warna dekoratif
- Arsitektur halaman Perspective — Pages dengan mounted views vs view swapping, dan mengapa region docked diukur dalam `Size` sementara flex children menggunakan `position.basis`
- Debugging masalah simulasi — `AccessViolation`, stuck state, dan self-defeating loop

## Teknologi

- **PLC:** CODESYS V3.5 SP22 Patch 3, CODESYS Control Win V3 x64
- **Bahasa:** IEC 61131-3 Structured Text (ST) dan Ladder Diagram (LD)
- **Komunikasi:** OPC UA server, konfigurasi IEC Symbol Set
- **HMI:** Ignition 8.3 Perspective (Maker Edition), OPC UA driver
- **Standar yang direferensikan:** ISA-101 (desain HMI), ISA-18.2 (manajemen alarm), IEC 61131-3
- **Prototyping v1:** Node-RED, node-red-dashboard, node-red-contrib-opcua

## Penulis

Muhammad Nabil Farrell — Lulusan Baru Teknik Elektro, calon Automation Engineer

- GitHub: [github.com/NabilFarrell](https://github.com/NabilFarrell)
- LinkedIn: [linkedin.com/in/nabil-farrell](https://www.linkedin.com/in/nabil-farrell/)
- Email: nabilfarrellid@gmail.com

## Lisensi

Proyek ini untuk tujuan portofolio dan demonstrasi.
