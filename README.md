# Known-Key MBE Samples

This repository contains known-key encrypted MBE voice samples for testing P25 and DMR decryptors.

Each sample directory contains:

- `encrypted.mbe` - encrypted vocoder frames in the SDRTrunk JSON `.mbe` export format.
- `encrypted_legacy.mbe` - present for DMR samples; the original metadata-free export for testing forced-algorithm and
  late-entry recovery paths.
- `key.txt` - the known algorithm, ALGID, optional key ID, and key material for the sample.
- `expected.wav` - the expected decoded audio after decrypting `encrypted.mbe` with the key.

The samples are intended for decryptor validation only. They are not raw RF captures and do not include receiver frequency, tuner, site frequency, or channel-frequency metadata.

Sample radio provenance:

- DMR ARC4, AES-128, and AES-256: Baofeng DM-32.
- P25 ADP / RC4: Motorola APX1000.
- P25 DES-OFB and AES-256: Harris XG-100P.

## Directory Layout

Current sample families:

- `p25_adp` - P25 Phase 1 ADP / RC4, ALGID `0xAA`
- `p25_des_ofb_*` - P25 Phase 1 DES-OFB, ALGID `0x81`
- `p25_aes256_*` - P25 Phase 1 AES-256, ALGID `0x84`
- `dmr_arc4_*` - DMRA ARC4 / Enhanced Privacy sample, ALGID `0x21`
- `dmr_aes128_*` - DMR AES-128 sample, ALGID `0x24`
- `dmr_aes256_*` - DMR AES-256 sample, ALGID `0x25`

## `.mbe` File Encoding

The `.mbe` files in this repository are UTF-8 JSON files. They are not raw MBELib binary files.

Top-level fields:

```json
{
  "protocol": "APCO25-PHASE1",
  "version": 2,
  "call_type": "GROUP",
  "from": "1",
  "to": "10",
  "encrypted": true,
  "frames": []
}
```

Common fields:

- `protocol` - `APCO25-PHASE1` or `DMR`.
- `version` - JSON container/schema version. This is not a vocoder version.
- `call_type` - call type when known, usually `GROUP`.
- `from` / `to` - source and destination IDs as recorder/exporter strings.
- `encrypted` - whether the captured voice frames were encrypted.
- `system` / `site` - optional sanitized labels from the recorder/exporter.
- `frames` - ordered array of encrypted vocoder frames.

Frame timestamps are recorder timestamps in milliseconds since Unix epoch when present.

## P25 Phase 1 Frame Encoding

P25 samples use one JSON frame per P25 Phase 1 IMBE voice frame.

Each P25 frame object contains:

```json
{
  "encryption_algorithm": 170,
  "encryption_key_id": 26985,
  "encryption_mi": "A3FD5550D287CD8E00",
  "time": 1782959607692,
  "hex": "..."
}
```

P25 frame fields:

- `encryption_algorithm` - decimal ALGID. For example, `170` is `0xAA` ADP / RC4.
- `encryption_key_id` - decimal key ID / KID.
- `encryption_mi` - P25 message indicator / MI as a hex string.
- `hex` - 18 bytes of encrypted P25 Phase 1 IMBE frame data, hex-encoded as 36 hex characters.

The P25 `hex` value is the SDRTrunk `.mbe` voice-frame payload shape expected by SDRTrunk/JMBE tooling. Consumers should decode the hex string into bytes, apply the appropriate decryptor for the sample, then pass the resulting frame bytes through the same P25 IMBE decode path used for unencrypted `.mbe` playback.

## DMR Frame Encoding

DMR samples use one JSON frame per DMR AMBE+2 voice frame.

Most DMR frame objects contain only the timestamp and encoded voice frame:

```json
{
  "time": 1782967077824,
  "hex": "..."
}
```

When a valid DMRA late-entry message indicator has been recovered, the first frame of the following voice superframe
also carries an encryption-context marker:

```json
{
  "encryption_algorithm": 36,
  "encryption_key_id": 2,
  "encryption_mi": "AA6A4E7F",
  "time": 1782967078522,
  "hex": "..."
}
```

The DMR `encrypted.mbe` files were enriched after capture from their preserved `encrypted_legacy.mbe` originals. Their
message indicators were recovered from the AMBE+2 late-entry fragments and accepted only after Golay correction and a
passing CRC. No unavailable PI-header value was guessed.

DMR frame fields:

- `encryption_algorithm` - optional decimal DMRA ALGID (`33` / `0x21` ARC4, `36` / `0x24` AES-128, or
  `37` / `0x25` AES-256).
- `encryption_key_id` - optional key selector. The DMR sample files use fixture key IDs `3` for ARC4, `2` for
  AES-128, and `1` for AES-256, matching `key.txt` and the SDRTrunk VCE test configuration. The original metadata-free
  files did not preserve the transmitted key ID.
- `encryption_mi` - optional 32-bit DMRA message indicator / IV recovered from the AMBE+2 late-entry fragments after
  Golay correction and CRC validation.
- `time` - recorder timestamp in milliseconds since Unix epoch.
- `hex` - 9 bytes of encrypted DMR AMBE+2 voice frame data, hex-encoded as 18 hex characters.

An encryption-context marker applies to that frame and subsequent frames until another marker appears. Metadata is not
repeated on every frame. Frames before the first recoverable late-entry marker remain encrypted without a serialized
context because the original `.mbe` files did not retain the PI header.

The DMR `hex` value is the SDRTrunk `.mbe` voice-frame payload shape expected by SDRTrunk/JMBE tooling. Consumers should decode the hex string into bytes, apply the appropriate decryptor for the sample, then pass the resulting 9-byte frames through the same DMR AMBE+2 decode path used for unencrypted `.mbe` playback.

## Key Files

Each `key.txt` uses simple `name=value` lines:

```text
protocol=P25
algorithm=ADP_RC4
algid=0xAA
key_id=0x4099
key_hex=deadbeef69
```

Fields:

- `protocol` - `P25` or `DMR`.
- `algorithm` - human-readable algorithm name.
- `algid` - algorithm ID in hex.
- `key_id` - optional key selector used to resolve the matching key. For the DMR fixtures, see the qualification in the
  DMR frame-field description above.
- `key_hex` - key bytes encoded as hex.

## Validation

A decryptor should be considered compatible with a sample when:

1. It reads `encrypted.mbe`.
2. It uses the matching `key.txt`.
3. It decrypts the frame payloads without changing their frame ordering.
4. It produces decoded audio matching the intelligible audio in `expected.wav`.
