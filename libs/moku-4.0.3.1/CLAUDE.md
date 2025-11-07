# CLAUDE.md - Moku Python Library (v4.0.3.1)

## Project Overview

**moku** is the official first-party Python library from Liquid Instruments for controlling Moku hardware devices via HTTP API.

**Purpose**: HTTP API wrapper that handles session management, bitstream deployment, and instrument control.

**Key Distinction**:
- **This library** (`libs/moku-4.0.3.1/`): HTTP API wrapper, session management, device control
- **Our library** (`libs/moku-models/`): Pydantic models, type-safe configs, platform specifications

They complement each other - use both together for robust deployment workflows.

**Part of:** Moku-pip-dev-fork development workspace (inlined dependency)
**Integration:** Works with moku-models for type-safe deployment configurations

---

## Quick Start

```bash
# Already available in workspace (uv workspace member)
cd libs/moku-4.0.3.1/
uv pip install -e .
```

```python
from moku.instruments import MultiInstrument, Oscilloscope

# Connect to Moku:Lab (2-slot platform)
moku = MultiInstrument(ip='192.168.1.100', platform_id=2)

# Deploy Oscilloscope to slot 1
osc = moku.set_instrument(1, Oscilloscope)

# Configure routing
moku.set_connections([
    {'source': 'Input1', 'destination': 'Slot1InA'},
    {'source': 'Slot1OutA', 'destination': 'Output1'}
])

# Clean exit
moku.relinquish_ownership()
```

---

## Core Architecture

### Session Management Pattern

**Session lifecycle** (moku/__init__.py:120-291, moku/session.py:28-52):

1. **Connection** - `Moku.__init__()` creates `RequestSession` with IP address
2. **Ownership claim** - `claim_ownership()` sends POST request, receives session key
3. **Session key storage** - Key stored in `Moku-Client-Key` HTTP header (session.py:31)
4. **All API calls** - Session key carried in every request after claim
5. **Relinquish** - `relinquish_ownership()` invalidates session key

**Session key management** (session.py:46-52):
```python
def update_sk(self, response):
    key = response.headers.get(self.sk_name)  # "Moku-Client-Key"
    if key:
        self.session_key = key
        self.rs.headers.update({self.sk_name: key})
```

**Graceful Handoff Workflow**:
```python
# Session 1 (existing connection)
moku = MultiInstrument(ip='192.168.1.100', platform_id=2)
# ... do work ...
moku.relinquish_ownership()

# Session 2 (new connection, preserve state)
moku = MultiInstrument(ip='192.168.1.100', platform_id=2, persist_state=True)
# Previous instrument state is retained!
```

**persist_state behavior** (__init__.py:250-275):
- When `persist_state=True`, server attempts to retain previous instrument configuration
- Includes: deployed instruments, routing connections, instrument settings
- Does NOT include: frontend/output/DIO settings (these are device-level, not session-level)
- Server-side state tracking (not client-side)

**Context manager support** (__init__.py:292-311):
```python
with MultiInstrument(ip='192.168.1.100', platform_id=2) as moku:
    osc = moku.set_instrument(1, Oscilloscope)
    # Auto-relinquish on exit (even if exception occurs)
```

### Multi-Instrument Mode

**MultiInstrument class** (moku/instruments/_mim.py:8-239):
- Primary entry point for multi-slot platforms (Lab/Pro/Delta)
- Manages multiple instruments across slots
- Handles MCC (Matrix CrossConnect) routing between slots and physical I/O

**Platform IDs**:
- `platform_id=2` - Moku:Lab (2 slots)
- `platform_id=4` - Moku:Pro (4 slots)
- Note: Moku:Go uses standalone instrument mode (not MultiInstrument)

**Slot management** (_mim.py:46-60):
```python
# Deploy instrument to specific slot
osc = moku.set_instrument(slot=1, instrument=Oscilloscope, **kwargs)

# Empty slots automatically filled with platform bitstreams
# Ensures proper slot initialization
```

### State Introspection APIs

**Extracting Running Configuration** (_mim.py:185-238):

```python
moku = MultiInstrument(ip='192.168.1.100', platform_id=2)

# Query instruments in each slot
instruments = moku.get_instruments()
# Returns: list[str] - instrument names per slot
# Example: ['CloudCompile', 'Oscilloscope', '', '']
# API: GET /api/mim/get_instruments

# Query MCC routing
connections = moku.get_connections()
# Returns: list[dict] - routing configuration
# Example: [{'source': 'Input1', 'destination': 'Slot1InA'}, ...]
# API: GET /api/mim/get_connections

# Query frontend settings (per channel)
frontend = moku.get_frontend(channel=1)
# Returns: dict
# Example: {'impedance': '1MOhm', 'coupling': 'DC', 'attenuation': '0dB', 'gain': None}
# API: POST /api/mim/get_frontend
# Available on: All platforms

# Query output settings
output = moku.get_output(channel=1)
# Returns: dict
# Example: {'output_gain': '0dB'}
# API: POST /api/mim/get_output
# Available on: All platforms

# Query DIO configuration
dio = moku.get_dio(port=None)
# Returns: dict
# Example: {'direction': [0, 0, 1, 1, ...]}  # 0=In, 1=Out
# API: POST /api/mim/get_dio
# Available on: Moku:Go (16 pins), Moku:Delta (32 pins), NOT Lab/Pro
```

**Configuration save/load** (_mim.py:62-87):
```python
# Save current configuration to binary .mokuconf file
moku.save_configuration('my_setup.mokuconf')
# API: GET /api/mim/save_configuration
# Returns: Binary file download

# Load configuration from file
moku.load_configuration('my_setup.mokuconf')
# API: POST /api/mim/load_configuration
# Sends: Binary file upload

# .mokuconf format: Binary format (not documented, proprietary)
# Compatible with Moku Desktop app
```

**Device-level queries** (__init__.py:313-552):
```python
# Device properties
props = moku.describe()
# Returns: {'hardware': 'MokuGo', 'mokuOS': '3.2.0', 'proxy_version': 3, 'bitstreams': {...}}

# Network/device configuration
config = moku.get_configuration()
# Returns: Device/network settings (NOT instrument configuration)

# Summary (undocumented - returns device state summary)
summary = moku.summary()
# API: GET /api/moku/summary
# Returns: dict (structure varies by device state)
```

**Mapping to moku-models**:
```python
from moku_models import MokuConfig, SlotConfig, MokuConnection, MOKU_LAB_PLATFORM
from moku.instruments import MultiInstrument

# Reconstruct MokuConfig from live device
moku = MultiInstrument(ip='192.168.1.100', platform_id=2)
instruments = moku.get_instruments()
connections = moku.get_connections()

config = MokuConfig(
    platform=MOKU_LAB_PLATFORM,
    slots={
        i+1: SlotConfig(instrument=inst)
        for i, inst in enumerate(instruments) if inst
    },
    routing=[
        MokuConnection(source=c['source'], destination=c['destination'])
        for c in connections
    ]
)

# Gaps: Frontend, output, DIO settings NOT in MokuConfig
# (See "Model Alignment" section below for details)
```

---

## Integration Patterns

### With moku-models

**Use Case 1: Deploy from MokuConfig**
```python
from moku_models import MokuConfig
from moku.instruments import MultiInstrument, Oscilloscope, CloudCompile
import yaml

# Load config (from YAML, etc.)
with open('config.yaml') as f:
    data = yaml.safe_load(f)
config = MokuConfig.from_dict(data)

# Deploy to hardware
moku = MultiInstrument(ip='192.168.1.100', platform_id=config.platform.slots)

# Deploy each instrument
for slot_num, slot_config in config.slots.items():
    instrument_class = {
        'Oscilloscope': Oscilloscope,
        'CloudCompile': CloudCompile,
        # ... map other instruments
    }[slot_config.instrument]

    # Deploy with settings
    inst = moku.set_instrument(
        slot_num,
        instrument_class,
        **slot_config.settings
    )

    # CloudCompile: Set registers and bitstream
    if slot_config.instrument == 'CloudCompile' and slot_config.control_registers:
        inst.set_controls(slot_config.control_registers)

# Apply routing
moku.set_connections([
    {'source': c.source, 'destination': c.destination}
    for c in config.routing
])
```

**Use Case 2: Extract MokuConfig from Running Device**
```python
from moku_models import MokuConfig, SlotConfig, MokuConnection, MOKU_LAB_PLATFORM
from moku.instruments import MultiInstrument

# Query running device
moku = MultiInstrument(ip='192.168.1.100', platform_id=2)
instruments = moku.get_instruments()
connections = moku.get_connections()

# Create MokuConfig snapshot
config = MokuConfig(
    platform=MOKU_LAB_PLATFORM,
    slots={
        i+1: SlotConfig(instrument=inst)
        for i, inst in enumerate(instruments) if inst
    },
    routing=[
        MokuConnection(source=c['source'], destination=c['destination'])
        for c in connections
    ]
)

# Save for later
import yaml
with open('snapshot.yaml', 'w') as f:
    yaml.dump(config.to_dict(), f)
```

**Use Case 3: Validate Before Deployment**
```python
from moku_models import MokuConfig

# Load user-provided config
config = MokuConfig.from_dict(yaml_data)

# Validate routing before touching hardware
errors = config.validate_routing()
if errors:
    print(f"Validation failed: {errors}")
    exit(1)

# Deploy to hardware (validation passed)
# ... use moku library ...
```

---

## File Structure

```
libs/moku-4.0.3.1/
├── moku/
│   ├── __init__.py              # Moku, MultiInstrumentSlottable base classes
│   ├── session.py               # RequestSession, HTTP API wrapper, session keys
│   ├── exceptions.py            # MokuException, InvalidParameterException, etc.
│   ├── utilities.py             # find_moku_by_serial, get_bitstream_path
│   ├── logging.py               # Logging configuration
│   ├── version.py               # Version compatibility checks
│   ├── instruments/
│   │   ├── __init__.py
│   │   ├── _mim.py              # MultiInstrument (KEY FILE)
│   │   ├── _cloudcompile.py     # CloudCompile instrument
│   │   ├── _oscilloscope.py     # Oscilloscope instrument
│   │   ├── _waveformgenerator.py
│   │   ├── _spectrumanalyzer.py
│   │   ├── _pidcontroller.py
│   │   ├── _digitalfilterbox.py
│   │   ├── _firfilter.py
│   │   ├── _laserlockbox.py
│   │   ├── _lockinamp.py
│   │   ├── _datalogger.py
│   │   ├── _logicanalyzer.py
│   │   ├── _phasemeter.py
│   │   ├── _fra.py              # Frequency Response Analyzer
│   │   ├── _tfa.py              # Time & Frequency Analyzer
│   │   ├── _awg.py              # Arbitrary Waveform Generator
│   │   └── _stream.py           # Data streaming base class
│   └── nn/
│       └── _linn.py             # LI-NN (neural network) instrument
├── pyproject.toml
├── README.md
├── llms.txt                     # Tier 1 (AI-friendly quick reference)
└── CLAUDE.md                    # This file (Tier 2)
```

**Key files**:
- `__init__.py` - Moku base class, claim/relinquish ownership, context manager
- `session.py` - RequestSession, session key management, HTTP request handling
- `instruments/_mim.py` - MultiInstrument, state introspection APIs
- `instruments/_cloudcompile.py` - CloudCompile register control

---

## Undocumented/Underutilized APIs

### Session Handoff Details

**persist_state mechanism** (__init__.py:262):
- Documented parameter: "tries to retain the previous state of the instrument(if available)"
- **What it actually does**:
  - Server-side state persistence (not client-managed)
  - Retains: instrument deployments, routing, instrument-specific settings
  - Does NOT retain: frontend impedance/coupling, output gain, DIO direction
  - State tied to device (not user session)
  - Works across different Python sessions / different client machines

**Session key lifecycle** (session.py:46-52):
- Generated by server on `claim_ownership()`
- Returned in `Moku-Client-Key` response header
- Stored in `RequestSession.session_key`
- Added to all subsequent request headers automatically
- Invalidated on `relinquish_ownership()`
- Session keys are **not** persisted (new key on each claim)

### State Extraction Methods

**Complete list of MultiInstrument getters** (_mim.py:185-238):

| Method | Returns | Maps to moku-models | Notes |
|--------|---------|---------------------|-------|
| `get_instruments()` | `list[str]` | `MokuConfig.slots` keys | ✅ Fully compatible |
| `get_connections()` | `list[dict]` | `MokuConfig.routing` | ✅ Fully compatible |
| `get_frontend(channel)` | `dict` | ❌ Not modeled | Gap: impedance, coupling, attenuation, gain |
| `get_output(channel)` | `dict` | ❌ Not modeled | Gap: output_gain |
| `get_dio(port)` | `dict` | ❌ Not modeled | Gap: DIO direction config |

**Per-instrument getters** (common across many instruments):
- `get_frontend(channel)` - Input impedance, coupling, attenuation
- `get_output_gain(channel)` - Output gain settings
- `get_output_load(channel)` - Output termination (50Ω / 1MΩ)
- `get_output_termination(channel)` - Output impedance
- `get_control_matrix(channel)` - Control path routing (PID, filters)
- `get_input_offset(channel)` / `get_output_offset(channel)` - DC offsets
- `get_samplerate()` - Sampling rate
- `get_timebase()` - Timebase settings
- `get_acquisition_mode()` - Acquisition configuration
- `get_data()` - Data acquisition (varies by instrument)

**CloudCompile-specific** (_cloudcompile.py:133-158):
- `get_control(idx)` - Read single control register (CR0-CR31)
- `get_controls()` - Read all control registers
- `get_interpolation(channel)` - Interpolation enable state

### Binary Configuration Files

**.mokuconf format** (_mim.py:62-87):
- **Format**: Proprietary binary (not documented)
- **Compatible with**: Moku Desktop app, mokucli
- **Contents**: Full device state (instruments, routing, frontend, output, DIO, settings)
- **Usage**:
  - Save: `moku.save_configuration('file.mokuconf')` → download binary file
  - Load: `moku.load_configuration('file.mokuconf')` → upload binary file
- **Reverse engineering**: Not attempted (proprietary, may change between versions)
- **Alternative**: Use state introspection APIs (`get_instruments`, `get_connections`, etc.) for portable configs

### Undocumented Device Methods

**summary()** (__init__.py:327-332):
- API: `GET /api/moku/summary`
- Returns: dict (structure not documented)
- Likely contains: device state overview, resource usage, active sessions
- **Status**: Undocumented, use with caution (may change)

**v2 API endpoints** (session.py:102-117):
- `post_to_v2(location, params)` - Alternative API version
- Example: `shutdown()` uses `post_to_v2("hwstate", {"power": "shutdown"})`
- **Status**: Sparse usage, not fully explored

**modify_hardware()** (__init__.py:523-530):
- **CAUTION**: Direct hardware state modification (bypass safety checks)
- Documented as "Never use to update the state of the Moku"
- Used internally for low-level operations
- **Status**: Not for user code

---

## Common Tasks

### Basic Device Connection

```python
from moku.instruments import MultiInstrument

# IP address connection
moku = MultiInstrument(ip='192.168.1.100', platform_id=2)

# Serial number lookup
moku = MultiInstrument(serial='MG12345', platform_id=2)

# Force disconnect existing users
moku = MultiInstrument(ip='192.168.1.100', platform_id=2, force_connect=True)

# Preserve previous state
moku = MultiInstrument(ip='192.168.1.100', platform_id=2, persist_state=True)

# Context manager (auto-cleanup)
with MultiInstrument(ip='192.168.1.100', platform_id=2) as moku:
    # Work here
    pass  # Auto-relinquish on exit
```

### Deploy CloudCompile with Custom Bitstream

```python
from moku.instruments import CloudCompile

moku = MultiInstrument(ip='192.168.1.100', platform_id=2)

# Deploy with custom bitstream
cc = moku.set_instrument(
    slot=1,
    instrument=CloudCompile,
    bs_path='path/to/custom.tar'
)

# Initialize control registers
cc.set_controls({
    0: 0xE0000000,  # CR0: Enable probe
    1: 0x12345678,  # CR1: Configuration
    2: 0x00000001   # CR2: Trigger
})

# Query current register values
cr0_value = cc.get_control(idx=0)
all_registers = cc.get_controls()
```

### Configure MCC Routing

```python
# Simple routing example
moku.set_connections([
    {'source': 'Input1', 'destination': 'Slot1InA'},
    {'source': 'Slot1OutA', 'destination': 'Output1'}
])

# Complex routing (slot-to-slot + output)
moku.set_connections([
    {'source': 'Input1', 'destination': 'Slot1InA'},
    {'source': 'Input2', 'destination': 'Slot2InA'},
    {'source': 'Slot1OutA', 'destination': 'Slot2InB'},
    {'source': 'Slot2OutA', 'destination': 'Output1'},
    {'source': 'Slot2OutB', 'destination': 'Output2'}
])

# Query current routing
current_routing = moku.get_connections()
```

### Configure Frontend and Output

```python
# Set input frontend
moku.set_frontend(
    channel=1,
    impedance='1MOhm',  # or '50Ohm'
    coupling='DC',       # or 'AC'
    attenuation='0dB'    # or '-20dB', '14dB', '20dB', '32dB', '40dB'
)

# Set output gain
moku.set_output(
    channel=1,
    output_gain='0dB'    # or '14dB'
)

# Query current settings
frontend = moku.get_frontend(channel=1)
output = moku.get_output(channel=1)
```

### Configure DIO (Moku:Go and Moku:Delta only)

```python
# Set DIO direction (0=In, 1=Out)
moku.set_dio(direction=[0, 0, 1, 1, 0, 0, 1, 1] + [0]*8)
# First 16 pins: 0=In, 1=Out (for Moku:Go)

# Query current DIO configuration
dio_config = moku.get_dio()
```

### Save and Load Configurations

```python
# Save current configuration
moku.save_configuration('my_setup.mokuconf')

# Load saved configuration
moku.load_configuration('my_setup.mokuconf')

# Share with Moku Desktop app
# .mokuconf files are compatible across Python API and Desktop app
```

---

## Development Workflow

### Modifying the Inlined Library

```bash
# Edit source files
vim libs/moku-4.0.3.1/moku/__init__.py

# Reinstall in dev mode (uv workspace handles this automatically)
cd libs/moku-4.0.3.1/
uv pip install -e .

# Test changes
python test_script.py
```

### Testing with Real Hardware

```python
# Always use try/finally or context manager for cleanup
try:
    moku = MultiInstrument(ip='192.168.1.100', platform_id=2)
    # ... test operations ...
finally:
    moku.relinquish_ownership()

# Or use context manager
with MultiInstrument(ip='192.168.1.100', platform_id=2) as moku:
    # ... test operations ...
    # Auto-cleanup even if exception
```

### Debugging Session Issues

```python
import logging
from moku.logging import get_logger

# Enable debug logging
logging.basicConfig(level=logging.DEBUG)

# Library uses get_logger(__name__) throughout
# Debug logs show:
# - Session key updates
# - API requests (GET/POST with URLs)
# - Response status codes
# - Ownership claim/relinquish

moku = MultiInstrument(ip='192.168.1.100', platform_id=2)
# DEBUG logs show full session lifecycle
```

---

## API Reference Pointers

**Official Documentation**:
- **API docs**: https://apis.liquidinstruments.com/
- **Python examples**: https://liquidinstruments.com/software/python-api/
- **Moku Desktop**: https://liquidinstruments.com/software/
- **mokucli**: https://liquidinstruments.com/software/utilities/

**Instrument Families**:
- **Signal Analysis**: Oscilloscope, SpectrumAnalyzer, Phasemeter, TFA (Time & Frequency Analyzer)
- **Signal Generation**: WaveformGenerator, AWG (Arbitrary Waveform Generator)
- **Control Systems**: PIDController, LaserLockBox, DigitalFilterBox, FIRFilter
- **Data Acquisition**: DataLogger, LockInAmp
- **Digital Tools**: LogicAnalyzer (Moku:Go only), PatternGenerator
- **Custom FPGA**: CloudCompile (programmable instrument with 32 control registers)
- **Advanced**: FRA (Frequency Response Analyzer), LI-NN (neural network instrument)

**HTTP API Structure** (session.py):
- Base URL: `http://{ip}/api/{group}/{operation}`
- Groups: `moku`, `mim`, `slot1`, `slot2`, etc., `bitstreams`, `ssd`, `logs`, `persist`
- Session key header: `Moku-Client-Key`
- Response format: JSON with `success`, `data`, `code`, `messages` fields

---

## Model Alignment Assessment

### Current moku-models Coverage

**MokuConfig** (moku_models/moku_config.py:40-150):
- ✅ Platform selection (`platform: MokuGoPlatform | MokuLabPlatform | ...`)
- ✅ Slot → instrument mapping (`slots: dict[int, SlotConfig]`)
- ✅ MCC routing (`routing: list[MokuConnection]`)
- ✅ CloudCompile control registers (`SlotConfig.control_registers: dict[int, int]`)
- ✅ CloudCompile bitstream path (`SlotConfig.bitstream: str`)
- ✅ Instrument settings (`SlotConfig.settings: dict[str, Any]`)

**SlotConfig** (moku_models/moku_config.py:16-38):
- ✅ Instrument type name (`instrument: str`)
- ✅ Generic settings dict (`settings: dict[str, Any]`)
- ✅ CloudCompile specifics (`control_registers`, `bitstream`)

### Gaps Identified

**NOT modeled in moku-models** (but available in first-party API):

1. **Frontend settings** (_mim.py:199-211):
   - Input impedance (`'1MOhm'` / `'50Ohm'`)
   - Input coupling (`'AC'` / `'DC'`)
   - Input attenuation (`'-20dB'`, `'0dB'`, `'14dB'`, `'20dB'`, `'32dB'`, `'40dB'`)
   - Input gain (optional)
   - **Per channel** (1-indexed)

2. **Output settings** (_mim.py:213-225):
   - Output gain (`'0dB'` / `'14dB'`)
   - **Per channel** (1-indexed)

3. **DIO configuration** (_mim.py:227-238):
   - Direction per pin (`list[int]` - 0=In, 1=Out)
   - Port mapping
   - **Platform-specific**: Moku:Go (16 pins), Moku:Delta (32 pins), NOT Lab/Pro

4. **Per-instrument frontend/output** (various _*.py files):
   - Many instruments have their own `get_frontend()`, `get_output_load()`, etc.
   - These are instrument-specific variations
   - Currently stored in generic `SlotConfig.settings` dict

5. **Platform-specific constraints**:
   - Validation of valid attenuation values per platform
   - DIO availability checking
   - Output gain options (vary by instrument/platform)

### Recommendations

**Option 1: Extend MokuConfig (Recommended if full device state needed)**

```python
from pydantic import BaseModel, Field
from typing import Literal

class FrontendConfig(BaseModel):
    """Frontend configuration per channel."""
    impedance: Literal['1MOhm', '50Ohm']
    coupling: Literal['AC', 'DC']
    attenuation: Literal['-20dB', '0dB', '14dB', '20dB', '32dB', '40dB'] | None = None
    gain: float | None = None

class OutputConfig(BaseModel):
    """Output configuration per channel."""
    output_gain: Literal['0dB', '14dB']

class DIOConfig(BaseModel):
    """DIO configuration."""
    direction: list[int]  # 0=In, 1=Out
    port_map: dict[int, str] | None = None

class MokuConfig(BaseModel):
    platform: MokuGoPlatform | MokuLabPlatform | MokuProPlatform
    slots: dict[int, SlotConfig]
    routing: list[MokuConnection]
    frontend: dict[int, FrontendConfig] | None = None  # NEW
    output: dict[int, OutputConfig] | None = None      # NEW
    dio: DIOConfig | None = None                       # NEW
    metadata: dict[str, Any] = Field(default_factory=dict)
```

**Pros**:
- Complete device state representation
- Can fully reconstruct device configuration from `MokuConfig`
- Enables true "configuration snapshots"

**Cons**:
- Increases model complexity
- Most users only care about instrument deployment (not frontend/output/DIO)
- Frontend/output settings often adjusted manually after deployment
- Backward compatibility requires all new fields to be Optional

**Option 2: Keep MokuConfig Minimal (Recommended for current use case)**

**Rationale**:
- `MokuConfig` is for instrument **deployment**, not full device configuration
- Frontend/output/DIO are device-level settings (pre/post instrument deployment)
- Users can call `moku.set_frontend()`, `moku.set_output()`, `moku.set_dio()` directly
- Document gaps clearly in CLAUDE.md (this file) and llms.txt

**Pros**:
- Maintains simplicity
- Clear separation: moku-models = deployment, moku = device control
- No backward compatibility concerns
- Aligns with current usage patterns

**Cons**:
- Cannot represent full device state in `MokuConfig`
- Snapshot extraction incomplete (missing frontend/output/DIO)

**Option 3: Hybrid Approach (Document Workaround)**

Keep `MokuConfig` minimal but document how to capture full state:

```python
from dataclasses import dataclass
from moku_models import MokuConfig

@dataclass
class FullDeviceState:
    """Complete device state (not part of moku-models)."""
    config: MokuConfig  # Instrument deployment
    frontend: dict[int, dict]  # Frontend per channel
    output: dict[int, dict]    # Output per channel
    dio: dict | None           # DIO configuration

def capture_full_state(moku):
    """Capture complete device state."""
    # Deployment config
    instruments = moku.get_instruments()
    connections = moku.get_connections()
    config = MokuConfig(...)  # As before

    # Device-level settings
    frontend = {
        i: moku.get_frontend(i)
        for i in range(1, moku.platform_id + 1)
    }
    output = {
        i: moku.get_output(i)
        for i in range(1, moku.platform_id + 1)
    }
    dio = moku.get_dio() if hasattr(moku, 'get_dio') else None

    return FullDeviceState(config, frontend, output, dio)
```

**Pros**:
- No changes to moku-models
- Users can opt-in to full state capture
- Pattern documented clearly

**Cons**:
- Not standardized (user must implement)
- No validation

**Final Recommendation**: **Option 2** (Keep MokuConfig minimal)
- Document gaps clearly
- Provide examples of capturing frontend/output/DIO separately
- Focus moku-models on deployment (its core strength)
- Let moku library handle device-level settings directly

---

**Last Updated**: 2025-11-07
**Part of**: Moku-pip-dev-fork (development workspace)
**Complements**: libs/moku-models/ (type-safe platform models)
**Version**: 4.0.3.1 (inlined from PyPI)
