**[English](README_en.md) | [中文](README.md)**

# ACPs Agent Identity Code (AIC) Utility Module

An AIC validation and parsing utility implemented according to the **ACPs-spec-AIC-v02.02** specification.

## AIC Structure

An AIC consists of 10 levels separated by `.`:

```
1.2.156.3088.1.1.34C2.478BDF.3GF546.0JU4
└─────┬────┘ └┘ └┬┘ └─┬─┘ └──┬──┘ └──┬──┘ └─┬┘
   Prefix Version ARSP Vendor Ontology SN Instance SN CRC
   (1-4)    (5)   (6)   (7)       (8)         (9)    (10)
```

| Level | Name        | Format                | Description                                    |
| ----- | ----------- | --------------------- | ---------------------------------------------- |
| 1-4   | OID prefix  | Digits only           | `1.2.156.3088` (assigned by the national OID registration center) |
| 5     | Version     | Base36, 1 digit       | AIC version number                              |
| 6     | ARSP        | Base36, 1-6 digits    | Agent registration service provider serial number |
| 7     | Vendor      | Base36, 1-6 digits    | Agent vendor serial number                      |
| 8     | Ontology SN | Base36, 1-9 digits    | Agent ontology serial number                    |
| 9     | Instance SN | Base36, 1-9 digits    | Agent instance serial number (all 0 = ontology AIC) |
| 10    | CRC         | Base36, fixed 4 digits | CRC-16 checksum                                |

## Quick Start

### Basic Format Validation (No SALT Required)

```python
from acps_sdk.aic import validate_aic_format, is_valid_aic_format

aic = "1.2.156.3088.1.1.34C2.478BDF.3GF546.0JU4"

# Simple validation
if is_valid_aic_format(aic):
    print("Format is valid")

# Validation with error information. Note: this only validates the format, it does not verify the CRC.
valid, error = validate_aic_format(aic)
if not valid:
    print(f"Format error: {error}")
```

### Parsing an AIC

```python
from acps_sdk.aic import parse_aic

aic = "1.2.156.3088.1.1.34C2.478BDF.3GF546.0JU4"
info = parse_aic(aic)

if info:
    print(f"Raw value: {info.raw}")
    print(f"Normalized: {info.normalized}")
    print(f"Prefix: {info.prefix}")          # 1.2.156.3088
    print(f"Version: {info.version}")         # 1
    print(f"ARSP: {info.arsp}")            # 1
    print(f"Vendor: {info.vendor}")        # 34C2
    print(f"Ontology serial number: {info.ontology_serial}")  # 478BDF
    print(f"Instance serial number: {info.instance_serial}")  # 3GF546
    print(f"Checksum: {info.checksum}")      # 0JU4
    print(f"Is ontology: {info.is_ontology}") # False
    print(f"Is entity: {info.is_entity}")   # True
    print(f"Body (levels 1-9): {info.body}")    # 1.2.156.3088.1.1.34C2.478BDF.3GF546
    print(f"Ontology prefix: {info.ontology_prefix}")  # 1.2.156.3088.1.1.34C2.478BDF
```

### Ontology/Entity Determination

```python
from acps_sdk.aic import is_ontology_aic, is_entity_aic, get_ontology_prefix_from_aic

# Ontology AIC: level 9 (instance serial number) is all 0
ontology_aic = "1.2.156.3088.1.1.34C2.478BDF.000000.0SV9"

# Entity AIC: level 9 is not all 0
entity_aic = "1.2.156.3088.1.1.34C2.478BDF.3GF546.0JU4"

print(is_ontology_aic(ontology_aic))  # True
print(is_entity_aic(entity_aic))      # True

# Get the ontology prefix (used for database LIKE queries)
prefix = get_ontology_prefix_from_aic(entity_aic)
# Result: "1.2.156.3088.1.1.34C2.478BDF"
# SQL: WHERE aic LIKE '1.2.156.3088.1.1.34C2.478BDF.%'
```

### Getting the Content of a Specified Level

```python
from acps_sdk.aic import get_aic_segment

aic = "1.2.156.3088.1.1.34C2.478BDF.3GF546.0JU4"

print(get_aic_segment(aic, 5))   # "1" (Version)
print(get_aic_segment(aic, 8))   # "478BDF" (ontology serial number)
print(get_aic_segment(aic, 10))  # "0JU4" (CRC)
```

## Full Validation (SALT Required)

Validating the CRC checksum requires the SALT. The SALT is maintained internally by each ARSP (agent registration service provider) and cannot be obtained from the outside.

```python
from acps_sdk.aic import AICValidator

# Method 1: bytes format
validator = AICValidator(salt=b'\x12\x34')

# Method 2: hexadecimal string
validator = AICValidator(salt="0x1234")
validator = AICValidator(salt="1234")

# Validate
aic = "1.2.156.3088.1.1.34C2.478BDF.3GF546.0JU4"
if validator.validate(aic):
    print("AIC is fully valid (including CRC validation)")
else:
    print(f"Validation failed: {validator.last_error}")

# Require CRC validation
if validator.validate(aic, require_crc=True):
    print("CRC validation passed")

# Get detailed information
is_valid, error, info = validator.validate_with_detail(aic)
if is_valid and info:
    print(f"ARSP: {info.arsp}")
```

### Behavior Without SALT

```python
# No SALT provided
validator = AICValidator()

# Default behavior: only validate the format, not the CRC
validator.validate(aic)  # True (passes as long as the format is correct)

# Requiring the CRC: it will fail
validator.validate(aic, require_crc=True)  # False
print(validator.last_error)  # "SALT is not configured, cannot validate CRC"
```

### Calculating the Checksum

```python
validator = AICValidator(salt=b'\x12\x34')

# Use salt=0x1234 to calculate the CRC checksum of levels 1-9
body = "1.2.156.3088.1.1.34C2.478BDF.3GF546"
checksum = validator.calculate_checksum(body)
print(checksum)  # "0JU4"
```

## Base36 Utilities

```python
from acps_sdk.aic import base36_encode, base36_decode

# Encoding
print(base36_encode(255))        # "73"
print(base36_encode(255, 4))     # "0073" (fixed 4 digits, zero-padded on the left)
print(base36_encode(0x8FCF, 4))  # "0SEN"

# Decoding
print(base36_decode("0SEN"))     # 36815 (0x8FCF)
print(base36_decode("73"))       # 255
```

## Feature Reference Table

| Feature           | Function/Class                      | Requires SALT |
| ----------------- | ----------------------------------- | :-----------: |
| Format validation | `validate_aic_format()`             |      ❌       |
| Simple format validation | `is_valid_aic_format()`       |      ❌       |
| Parse AIC         | `parse_aic()`                       |      ❌       |
| Get a specified level | `get_aic_segment()`             |      ❌       |
| Determine ontology AIC | `is_ontology_aic()`            |      ❌       |
| Determine entity AIC | `is_entity_aic()`                |      ❌       |
| Get ontology prefix | `get_ontology_prefix_from_aic()`  |      ❌       |
| CRC validation    | `AICValidator.validate()` (when SALT is configured) |      ✅       |
| **Calculate checksum** | `AICValidator.calculate_checksum()` |      ✅       |
| Base36 encoding   | `base36_encode()`                   |      ❌       |
| Base36 decoding   | `base36_decode()`                   |      ❌       |

## References

- [ACPs-spec-AIC-v02.02](../../../acps-specs/02-ACPs-spec-AIC/ACPs-spec-AIC_en.md) - Agent Identity Code specification
