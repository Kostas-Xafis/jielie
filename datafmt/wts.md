# WTS tone files

`.wts` is one of the formats used for the prompt tones in the `tone` directory of recent SDKs
(next to `.wtg`, `.sbc`, `.msbc`, `.mp3`, `.aac`, ...).

The SDK calls the codec "WTGV2" (`{"wts", AUDIO_CODING_WTGV2}`) and configures it with
`silk_fsN_enable` / `silk_fsW_enable`, and that is what it is: SILK, in the form used inside Opus.

Checked on the 21 `.wts` tones of a Xiaomi Sound Outdoor (SDK `JL701N_V1.4.2`).

## Format

```
1: Header byte

then, until the end of the file:
  1: Frame length (N)
  N: SILK frame
```

- The header byte was 0x1C in most files, but also 0x2C, 0x3C, 0x4C, 0x7C and 0xEC.
  The meaning is unknown. Keeping the original byte when replacing a file worked.
- Each frame is exactly the payload of a SILK-only Opus packet with a single 20 ms mono frame,
  that is, an Opus packet without its TOC byte.

So to decode a file, prepend a TOC byte to every frame and feed it to any Opus decoder.
All the files seen were wideband (16 kHz): TOC `0x48`.
(Presumably narrowband ones would need `0x08`, as the SDK has a separate enable for 8-12 kHz, but none were seen.)

A wrong bandwidth still "decodes", into clipping noise. With the right one the silent tone
(`null_tone.wts`) decodes into exact digital silence.

## What the stock files look like

- Every frame has the VAD flag set and no LBRR, i.e. the first byte is always within 0x80..0xBF, even for pure silence.
- No frame is longer than 78 bytes (~31 kbps).
- A silent frame is `80 02 26 14 BB 9C 84`.

## Making new ones

libopus can produce compatible frames when forced into SILK-only wideband mode
(`OPUS_SET_FORCE_MODE(MODE_SILK_ONLY)`, 16 kHz mono, 20 ms frames, no DTX, no FEC); drop the TOC byte from each packet.

What was found on the tested device when replacing tones in flash:

- Frames longer than what the stock files use break playback.
  A tone with frames of 83-85 bytes got cut off (that one also had frames with the VAD flag cleared),
  tones with frames above 100 bytes did not play at all, with or without cleared VAD flags.
  With every frame kept at 78 bytes or less the tones play fine.
- The stock encoder never clears the VAD flag, while libopus does that a lot on quiet or tonal material.
  The files that ended up working were made with a libopus patched to never mark a frame as inactive
  (the check in `silk_encode_do_VAD_FLP`). Whether frames with a cleared VAD flag are a problem on their own was not isolated.
- To reach a given file size, append silent frames rather than padding frames with extra bytes:
  an Opus decoder takes extra bytes after the SILK data for a redundancy frame. (How the device treats them was not tried.)
