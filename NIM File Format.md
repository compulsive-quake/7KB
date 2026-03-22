# NIM File Format

**Name Index Map** — binary format mapping block IDs (from TTS) to block name strings.

**Source**: `index.html` lines ~4548–4569

## File Structure

| Offset | Type | Description |
|--------|------|-------------|
| 0 | uint32 | Version |
| 4 | uint32 | Entry count |
| 8+ | entries[] | Repeated entries |

### Entry Structure

| Type | Description |
|------|-------------|
| uint32 | Block ID |
| uint8 | Name length |
| char[] | Block name string (UTF-8) |

All values are **little-endian**.

## Parsed Output

Returns a `Map<number, string>` mapping block IDs to block names.

Example entries:
- `1 → "woodShapes"`
- `42 → "steelDoor01"`
- `100 → "concreteShapes:wedge"`

## Usage

The NIM map is the bridge between the numeric block IDs stored in [[TTS File Format]] and the named block definitions in `blocks.xml`. See [[Block Definition Resolution]].
