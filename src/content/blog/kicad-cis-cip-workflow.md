---
title: 'Bringing Orcad CIS / CIP to KiCad — Database-Driven Component Management'
description: "How to leverage KiCad 9.0's SQLite database library feature to create an Orcad-style component information system with automated Digikey integration."
pubDate: 'Apr 28 2026'
heroImage: '../../assets/blog-placeholder-5.jpg'
---

If you've ever designed a PCB with a team using Orcad, you've probably used **CIS** (Component Information System) and **CIP** (Component Information Portal). These tools let you manage a centralized component database — storing manufacturer part numbers, distributor SKUs, parametric data, datasheet URLs, and links to KiCad symbols and footprints — all in one place.

KiCad has had a database library feature since version 7, but it really came into its own in **KiCad 9.0** with improved SQLite support. The problem? There's no built-in GUI tool for populating and managing that database.

Until now.

## The Tool: `kicad-lib-gen`

I built a command-line tool called [`kicad-lib-gen`](https://github.com/your-org/kicad_lib_gen) that brings the CIS/CIP workflow to KiCad. It connects to the **Digikey API**, fetches real-time product data, and stores it directly into a SQLite database that KiCad's symbol and footprint choosers can read natively.

Here's what it does:

- **Search Digikey** by manufacturer part number and pull down descriptions, parameters, pricing, datasheet URLs, and availability
- **Store everything** into a SQLite database with the exact schema KiCad expects
- **Link components** to KiCad symbols and footprints (with tab-completion for existing entries)
- **Batch import** hundreds of parts from a CSV file for team-scale setup
- **Auto-refresh** Digikey OAuth tokens so you don't have to babysit authentication

## Why This Matters

Component data management is one of the biggest pain points in hardware design, especially for teams. Without a centralized system:

- Engineers waste time hunting down datasheets
- BOMs end up with mismatched distributor SKUs
- The same component gets different symbols in different projects
- There's no single source of truth for part status, pricing, or lifecycle

Orcad solves this with CIS/CIP. KiCad solves the PCB design part beautifully, but the component data layer has been DIY. This tool closes that gap.

## How It Works

### 1. Authentication

You start by registering a free developer account at [Digikey's Developer Portal](https://developer.digikey.com/). Create an application, get your Client ID and Client Secret, then run:

```bash
uv run digikey_auth.py --user YOUR_CLIENT_ID --secret YOUR_CLIENT_SECRET
```

This kicks off the OAuth2 flow. You paste a URL into your browser, authorize the app, and paste the redirect URL back. The tool saves your tokens to a `.token` file with automatic refresh — you won't need to re-authenticate for months.

### 2. Database Setup

The tool creates a `components.db` SQLite file with the schema KiCad expects. Each record stores:

- Manufacturer and manufacturer product number
- Description, keywords, and category tree
- Package type and parametric data (up to 32 parameters)
- Datasheet URL and product URL
- Distributor info (Digikey product number, pricing, quantity available)
- **KiCad symbol library** and **footprint library** links

The database can live in your project's `Library/` folder, and you configure KiCad to use it via a `.kicad_dbl` ODBC configuration file.

### 3. Interactive Mode: Adding Components One at a Time

The most common workflow is searching for a part and adding it:

```bash
uv run kicad_cip.py -k "LM324N"
```

The tool searches Digikey, shows you the results, and lets you pick:

```
Total 5 products found:
  [1]: LM324N, Texas Instruments, IC OPAMP GP 4 CIRCUIT 14DIP, Qty: 2500, Active
  [2]: LM324NE4, Texas Instruments, IC OPAMP GP 4 CIRCUIT 14DIP, Qty: 1800, Active
  [3]: LM324NP, Texas Instruments, IC OPAMP GP 4 CIRCUIT 14DIP, Qty: 0, Obsolete
  ...
Choose one product, 0 to exit: 1
```

After selecting the part, it prompts you for the KiCad symbol and footprint, with **auto-completion** of existing libraries in your database:

```
Enter Kicad symbol library name: Standard:LM324
Enter Kicad footprint library name: Standard:LM324_DIP-14
```

A neat trick: you can use `?` as a placeholder for the product number:

```
Enter Kicad symbol library name: ?      -> becomes LM324N
Enter Kicad footprint library name: ?   -> becomes LM324N
```

This is a huge time-saver when you're importing dozens of parts.

### 4. Batch Mode: Importing from CSV

For setting up a whole BOM at once, batch mode is where the tool really shines:

```bash
uv run kicad_cip.py -b parts.csv
```

Your CSV file looks like this:

```csv
manufacturer_product_number,kicad_symbol_library,kicad_footprint_library
LM324N,Standard:LM324,Standard:LM324_SOIC-14
ATMega328P,Standard:ATMega328P,Standard:ATMega328P_DIP-28
STM32F103C8T6,Standard:STM32F103C8T6,Standard:STM32F103_LQFP48
```

The tool iterates each row, searches Digikey for an exact match, and auto-inserts the record with the specified symbol and footprint. It's strict — if a search returns zero or multiple results, it fails fast rather than silently picking the wrong part.

### 5. Symbol and Footprint Organization

The tool works with a **carefully organized library structure** that keeps projects maintainable at scale:

```
Library/
├── components.db                     # SQLite database
├── components_db.kicad_dbl           # KiCad ODBC config
├── Symbol/
│   ├── IC_LM324N.kicad_sym           # Per-part symbols
│   └── IC_ATMega328P.kicad_sym
├── Footprint/
│   └── Footprint.pretty/
│       ├── LM324N_SOIC-14.pretty     # Per-part footprints
│       └── ATMega328P_DIP-28.pretty
├── Step/                              # 3D models
│   ├── LM324N.step
│   └── ATMega328P.step
└── Standard/                          # Shared generic libs
    ├── Standard.kicad_sym             # Resistors, caps, power symbols
    └── Standard.pretty/
        ├── R_0402.pretty
        └── SOT23.pretty
```

The key insight: **generic components** (resistors, capacitors, standard logic) use shared symbols in `Standard.kicad_sym`, while **specific ICs** each get their own symbol file. This gives you the best of both worlds — reuse for common parts, isolation for complex ones.

## Connecting It to KiCad

Once your database is populated, you configure KiCad to use it:

1. Place `components_db.kicad_dbl` in your project root
2. In KiCad's Symbol Chooser, you'll see a new library tab for your database
3. Browse components by category, search by keyword, and place them on your schematic
4. The footprint is automatically associated — KiCad picks it up from the database record

This mirrors the Orcad CIS experience: you browse a database-driven library, place a symbol, and the footprint and metadata follow.

## What This Unlocks for Teams

With a centralized, database-backed component library, teams can:

- **Enforce part selection standards** — only approved parts make it into the database
- **Eliminate datasheet hunting** — one click from the symbol chooser opens the datasheet
- **Automate BOM generation** — export from the database with real distributor SKUs and pricing
- **Track part lifecycles** — flag obsolete components before they cause a prototype spin
- **Git-version the library** — the database, symbols, and footprints all live in version control

## Getting Started

The tool is live on [GitHub](https://github.com/your-org/kicad_lib_gen). Here's the quick start:

```bash
# Install dependencies
uv sync

# Authenticate with Digikey
uv run digikey_auth.py --user YOUR_ID --secret YOUR_SECRET

# Add a single part interactively
uv run kicad_cip.py -k "LM324N"

# Or batch import from CSV
uv run kicad_cip.py -b bom_parts.csv
```

You'll need Python 3.13+ and a free Digikey developer account, but that's it — no server, no cloud dependency, no vendor lock-in. The database is just a SQLite file on your disk.

## What's Next

This is an early but functional release. Future plans include:

- **Multi-distributor support** — Mouser, LCSC, Farnell APIs
- **Symbol auto-generation** — programmatic symbol creation from package data
- **KiCad plugin GUI** — browse and manage the database from within KiCad
- **Lifecycle alerts** — CI checks that flag NRND or obsolete parts before tape-out

If you're using KiCad in a team setting and missing the CIS/CIP workflow from Orcad, give this a try. The gap between open-source EDA and commercial tools is shrinking — one database record at a time.

---

*Have questions or ideas? Drop me a note or open an issue on GitHub. This is very much a community-driven project.*
