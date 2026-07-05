# Known-Key MBE Samples

This repository contains known-key encrypted MBE voice samples for testing P25 and DMR decryptors.

Each sample directory contains:

- `encrypted.mbe` - encrypted vocoder frames in the SDRTrunk JSON `.mbe` export format.
- `key.txt` - the known algorithm, ALGID, optional key ID, and key material for the sample.
- `expected.wav` - the expected decoded audio after decrypting `encrypted.mbe` with the key.

The samples are intended for decryptor validation only. They are not raw RF captures and do not include receiver frequency, tuner, site frequency, or channel-frequency metadata.

## Directory Layout

Current sample families:

- `p25_adp` - P25 Phase 1 ADP / RC4, ALGID `0xAA`
- `p25_des_ofb_*` - P25 Phase 1 DES-OFB, ALGID `0x81`
- `p25_aes256_*` - P25 Phase 1 AES-256, ALGID `0x84`
- `dmr_arc4_*` - DMR ARC4 / Basic Privacy style sample, ALGID `0x21`
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

Each DMR frame object contains:

```json
{
  "time": 1782967077824,
  "hex": "..."
}
```

DMR frame fields:

- `time` - recorder timestamp in milliseconds since Unix epoch.
- `hex` - 9 bytes of encrypted DMR AMBE+2 voice frame data, hex-encoded as 18 hex characters.

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
- `key_id` - optional P25 key ID when applicable.
- `key_hex` - key bytes encoded as hex.

## Validation

A decryptor should be considered compatible with a sample when:

1. It reads `encrypted.mbe`.
2. It uses the matching `key.txt`.
3. It decrypts the frame payloads without changing their frame ordering.
4. It produces decoded audio matching the intelligible audio in `expected.wav`.

