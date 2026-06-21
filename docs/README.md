# Merlin400 Documentation

This directory contains comprehensive documentation for the Merlin400 custom firmware and operations.

## Documentation Files

### 📋 [MERLIN400.md](./MERLIN400.md)
**Setup and Operations Guide**

Overview of the aftermarket open-source software for the Merlin400 extractor. Covers:
- Hardware specifications (Raspberry Pi, OS, storage)
- SSH access and deployment procedures
- Software architecture and file structure
- Systemd services and common commands
- API endpoints and program parameters
- Runtime statistics and recent work performed

**Start here** if you're new to this system or need operational information.

---

### 🚀 [SET_MERLIN400_AS_DEFAULT_BOOT.md](./SET_MERLIN400_AS_DEFAULT_BOOT.md)
**Deployment and Boot Configuration Guide**

Step-by-step instructions for making the aftermarket `merlin400-system` the default boot software on the Raspberry Pi:
- Prerequisites and service checks
- Stopping/starting the original vs. aftermarket software
- Making changes permanent across reboots
- Verification procedures
- Troubleshooting common issues
- How to revert to original software if needed

**Use this** when deploying to a new machine or switching boot configurations.

---

### 📝 [CODEX_HANDOVER.md](./CODEX_HANDOVER.md)
**Development Running Log and Handover**

Complete running log of all changes, deployments, and verification steps. Includes:
- SSH access details and deployment patterns
- System overview and FSM state flow
- API endpoints documentation
- Detailed work completed (bug fixes, hardening, features, refactoring)
- Git history and commit references
- Open work and recommendations
- Deployment procedures

**Reference this** for:
- Understanding what's been done and why
- Tracking deployment history
- Finding specific commit references
- Deployment best practices

---

## Quick Links

- **Want to get started?** → Read [MERLIN400.md](./MERLIN400.md)
- **Setting up a new machine?** → Follow [SET_MERLIN400_AS_DEFAULT_BOOT.md](./SET_MERLIN400_AS_DEFAULT_BOOT.md)
- **Need to understand what's been done?** → Check [CODEX_HANDOVER.md](./CODEX_HANDOVER.md)

---

## Key Information at a Glance

**Hardware:** Raspberry Pi 2/3 (ARMv7) at `192.168.1.130`  
**SSH:** `ssh merlin400` (key-based) or `ssh pi@192.168.1.130` (password: `dubqengm`)  
**API:** `http://192.168.1.130/api/status`  
**Service:** `merlin400-system` (systemd)  
**Repos:**
- Public upstream: https://github.com/64bandil/merlin400
- Private fork: https://github.com/chandz102/merlin400-system

---

## Support

For issues or questions:
1. Check the relevant documentation file above
2. Review the troubleshooting section in [SET_MERLIN400_AS_DEFAULT_BOOT.md](./SET_MERLIN400_AS_DEFAULT_BOOT.md#troubleshooting)
3. Check recent commits and logs in [CODEX_HANDOVER.md](./CODEX_HANDOVER.md)
