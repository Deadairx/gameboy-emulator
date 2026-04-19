# GameBoy Emulator

A GameBoy emulator built from scratch in Rust.

## Status

**Phase:** Planning / Research

This project is in its early stages. See the [devlog](./devlog/) for progress notes and decisions.

## Resources

[GameBoy Dev Community](https://gbdev.io/)
[Emulator Development resources](https://gbdev.io/resources.html#emulator-development)
[PanDocs](https://gbdev.io/pandocs/)
[GB Complete technical reference (PDF)](https://gekkio.fi/files/gb-docs/gbctr.pdf)
[RGBDS assembly reference](https://rgbds.gbdev.io/docs/v0.9.0/gbz80.7)

## Architecture

The emulator will implement the core components of the GameBoy:

- CPU (Z80-like instruction set)
- Memory (64KB address space)
- PPU (LCD, sprites, backgrounds)
- Audio (4 channels)
- Cartridge loading (ROM header, MBC variants)
- Input (joypad)

## License

MIT
