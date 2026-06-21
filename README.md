![alt text](https://drizzle.life/wp-content/uploads/front_page_RMG7123.JPG)

# Merlin400 — Open Source Firmware

Aftermarket open-source software for the discontinued Merlin400 supercritical CO₂ extractor.

This project replaces the original Drizzle firmware with a modern, documented, and maintainable system running on a Raspberry Pi (ARMv7).

## Quick Start

- **New to the system?** Start with [Setup and Operations Guide](./docs/MERLIN400.md)
- **Deploying to a machine?** Follow [Deployment and Boot Guide](./docs/SET_MERLIN400_AS_DEFAULT_BOOT.md)
- **Want the full history?** See [Development Running Log](./docs/CODEX_HANDOVER.md)

👉 **[Full Documentation Index](./docs/README.md)**

## Key Features

✅ Modern Python-based FSM (Finite State Machine) architecture  
✅ Flask REST API with real-time status monitoring  
✅ Comprehensive state logging and statistics  
✅ Hardware abstraction layer for maintainability  
✅ Complete documentation and deployment guides  
✅ Fully open source under GPL

## Project Status

- **Hardware:** Running on Raspberry Pi 2/3 (ARMv7)
- **Software:** Fully operational with recent refactoring and hardening
- **API:** `http://192.168.1.130/api/status` (on local network)
- **Service:** Systemd-managed (`merlin400-system`)

## Repository Structure

```
.
├── docs/                      # Comprehensive documentation
│   ├── README.md             # Documentation index
│   ├── MERLIN400.md          # Setup & operations guide
│   ├── SET_MERLIN400_AS_DEFAULT_BOOT.md  # Deployment guide
│   └── CODEX_HANDOVER.md     # Development log
├── merlin400-system/         # Main firmware directory
│   ├── src/
│   │   ├── startup.py        # Entry point
│   │   ├── hardware/         # Hardware control & FSM states
│   │   ├── common/           # Config, stats, utilities
│   │   └── webserver.py      # Flask API
│   └── wwwroot/              # Frontend static files
├── merlin400-3dmodels/       # 3D printable parts
├── src/                      # Legacy firmware (reference)
├── config.ini                # Machine configuration (gitignored)
└── stats.db                  # Runtime statistics database
```

## Documentation

Complete documentation is in [`/docs`](./docs/):

| Document | Purpose |
|----------|---------|
| [MERLIN400.md](./docs/MERLIN400.md) | Hardware, software architecture, API reference, operations |
| [SET_MERLIN400_AS_DEFAULT_BOOT.md](./docs/SET_MERLIN400_AS_DEFAULT_BOOT.md) | Deploying and configuring on a new machine |
| [CODEX_HANDOVER.md](./docs/CODEX_HANDOVER.md) | Full development history, commits, and verification logs |

## Development

For more details on the architecture and development process, see [CODEX_HANDOVER.md](./docs/CODEX_HANDOVER.md).

## License

See [LICENSE](./LICENSE) file for details.

## Contributing

Please see [CONTRIBUTING.md](./CONTRIBUTING.md) for contribution guidelines.
