# HANDOFF TO WEB WORKER

**Date**: 2025-11-07
**From**: Claude Code (CLI)
**To**: Claude Web UI Worker
**Repository**: Moku-pip-dev-fork (main branch)

---

## Mission Overview

You are tasked with **two complementary objectives**:

### P1: Session Introspection & State Extraction API Discovery
Systematically review the Moku Python library (`libs/moku-4.0.3.1/`) to identify **undocumented or underutilized APIs** that enable graceful session handoffs and state extraction from running Moku devices.

**Goal**: Understand how first-party Moku applications query a live device to extract the equivalent of our `moku-models/moku_models/moku_config.py` abstraction.

### P2: Documentation Creation (3-Tier AI-Navigable System)
Create **llms.txt** and **CLAUDE.md** files for `libs/moku-4.0.3.1/` following the 3-tier documentation pattern established in `CONTEXT_MANAGEMENT.md` and exemplified by `libs/moku-models/`.

**Goal**: Overlay AI-friendly documentation on the first-party moku library that integrates seamlessly with our existing documentation architecture.

---

## Critical Context Files (Read These First)

**Navigation Strategy**:
1. `CONTEXT_MANAGEMENT.md` - Tier 1→2→3 loading strategy, token optimization
2. `libs/moku-models/llms.txt` - Reference example (Tier 1 format)
3. `libs/moku-models/CLAUDE.md` - Reference example (Tier 2 format)
4. `libs/moku-4.0.3.1/CLAUDE.md` - Existing documentation (written for human consumption, not 3-tier)

**Key Architectural Principle**:
- `moku-models` is OUR abstraction (Pydantic models, type-safe, platform specs)
- `libs/moku-4.0.3.1` is the first-party library (HTTP API wrapper, session management)
- These should **align but not duplicate** - cross-reference where appropriate

---

## P1: Session Introspection Deep Dive

### Primary Investigation Targets

**File**: `libs/moku-4.0.3.1/moku/instruments/_mim.py` (MultiInstrument Mode)

**Key Methods Discovered** (high-value for session handoffs):
```python
class MultiInstrument:
    # State extraction
    def get_instruments(self) -> list[str]
        # Returns list of instrument names per slot (e.g., ['CloudCompile', 'Oscilloscope', '', ''])
        # Maps to: moku_config.slots dict

    def get_connections(self) -> list[dict]
        # Returns MCC routing configuration
        # Maps to: moku_config.routing (list[MokuConnection])

    def get_frontend(self, channel: int) -> dict
        # Returns input impedance, coupling, attenuation, gain
        # NOT modeled in current moku_config

    def get_output(self, channel: int) -> dict
        # Returns output gain settings
        # NOT modeled in current moku_config

    def get_dio(self, port: int | None) -> dict
        # Returns DIO direction configuration
        # NOT modeled in current moku_config

    # Configuration save/load
    def save_configuration(self, filename: str) -> None
        # Saves .mokuconf file (binary format)
        # API: GET /api/mim/save_configuration

    def load_configuration(self, filename: str) -> None
        # Loads .mokuconf file
        # API: POST /api/mim/load_configuration
```

**File**: `libs/moku-4.0.3.1/moku/__init__.py` (Base Moku class)

**Session Management** (critical for handoffs):
```python
class Moku:
    def claim_ownership(self, force_connect=True, ignore_busy=False, persist_state=False)
        # persist_state=True: Attempts to retain previous instrument state
        # This is the mechanism for graceful handoffs
        # API: POST /api/moku/claim_ownership

    def relinquish_ownership(self)
        # Clean session termination
        # API: POST /api/moku/relinquish_ownership

    def describe(self) -> dict
        # Returns hardware type, mokuOS version, proxy_version, bitstreams
        # First call made during connection

    def summary(self) -> dict
        # Undocumented - investigate return structure

    def get_configuration(self) -> dict
        # Device/network configuration (NOT instrument configuration)
        # API: GET /api/moku/get_configuration
```

**Session Class**: `libs/moku-4.0.3.1/moku/session.py`
- `RequestSession.sk_name = "Moku-Client-Key"` - Session key header
- Session keys are updated via `update_sk()` from response headers
- All requests carry session key after `claim_ownership()`

### Research Questions to Answer

1. **State Reconstruction**:
   - Can you query a live MultiInstrument device and reconstruct a complete `MokuConfig` object?
   - What's missing? (Frontend settings, output gain, DIO config are NOT in our model)
   - Are there per-instrument `get_*` methods we're missing?

2. **Binary Configuration Format**:
   - What's the structure of `.mokuconf` files from `save_configuration()`?
   - Can we parse them to extract `MokuConfig`-compatible data?
   - Is this documented anywhere?

3. **persist_state Behavior**:
   - What exactly does `persist_state=True` preserve?
   - Does it work across different Python sessions?
   - How does the device track "previous state"?

4. **Undocumented Endpoints**:
   - What does `summary()` return? (line 329 in `__init__.py`)
   - Are there other `get_*` methods on individual instruments?
   - Check CloudCompile (`_cloudcompile.py`) for register introspection

5. **First-Party App Behavior**:
   - How do Moku Desktop/iPad apps query running configs?
   - Are there v2 API endpoints we're missing? (see `session.post_to_v2()`)
   - Search for any `/api/v2/` references

### Investigation Methodology

**Step 1**: Grep for all `get_` methods across instruments
```bash
grep -r "def get_" libs/moku-4.0.3.1/moku/instruments/
```

**Step 2**: Map each getter to `MokuConfig` / `SlotConfig` attributes
- Create a compatibility matrix
- Identify gaps in our model

**Step 3**: Trace session lifecycle
- `claim_ownership()` → what state is preserved?
- `persist_state=True` → server-side vs client-side state?
- Session key management → tied to ownership?

**Step 4**: Analyze CloudCompile specifics
- `control_registers` dict (CR0-CR31) - how to query current values?
- Look for `get_register()` or similar in `_cloudcompile.py`

**Step 5**: Document findings
- List all state extraction methods
- Note incompatibilities with `moku_config.py`
- Propose extensions to our model if needed

---

## P2: Three-Tier Documentation Creation

### Tier 1: llms.txt (~800-1200 tokens)

**Location**: `libs/moku-4.0.3.1/llms.txt`

**Structure** (follow `libs/moku-models/llms.txt` as template):
```markdown
# moku (Liquid Instruments Python API)

> Official Python library for controlling Moku hardware devices

## What is this?

[Brief description - this is the first-party library, not our models]

## Core Exports

### Key Classes
- Moku - Base connection class
- MultiInstrument - Multi-slot instrument mode
- Oscilloscope, WaveformGenerator, etc. - Individual instruments
- RequestSession - HTTP API wrapper

### Session Management
- claim_ownership() / relinquish_ownership()
- persist_state parameter for graceful handoffs
- Context manager support (with statement)

## Basic Usage

[Quick example - connect, deploy, query]

## Common Tasks

**Query running device state**:
- MultiInstrument.get_instruments() → slot contents
- MultiInstrument.get_connections() → MCC routing
- [Include session handoff example]

**Integration with moku-models**:
- Cross-reference: libs/moku-models/moku_config.py
- Map get_instruments() → MokuConfig.slots
- Map get_connections() → MokuConfig.routing

## Documentation

- README.md - Installation
- CLAUDE.md (this repo) - Development guide
- CLAUDE.md (Tier 2) - Complete API navigation **← CREATE THIS**
- Official docs: https://apis.liquidinstruments.com/

## Key Design Notes

- This is the HTTP API wrapper (not platform models)
- Session-based ownership model (exclusive access)
- Bitstream management is automatic
- See moku-models for type-safe platform abstractions
```

**Token Budget**: ~800-1000 tokens (Tier 1 should be skimmable)

**Critical**: Include clear pointers to Tier 2 and cross-references to `moku-models`

---

### Tier 2: CLAUDE.md (~3500-5000 tokens)

**Location**: `libs/moku-4.0.3.1/CLAUDE.md`

**Goal**: Replace or heavily modify the existing `CLAUDE.md` to follow the 3-tier pattern.

**Structure** (follow `libs/moku-models/CLAUDE.md` as template):

```markdown
# CLAUDE.md - Moku Python Library (v4.0.3.1)

## Project Overview

[This is the first-party library, inlined in our dev workspace]

**Key Distinction**:
- **This library** (`libs/moku-4.0.3.1/`): HTTP API wrapper, session management, instrument control
- **Our library** (`libs/moku-models/`): Pydantic models, type-safe configs, platform specs

They complement each other - use both together.

---

## Quick Start

[Installation, basic connection example]

---

## Core Architecture

### Session Management Pattern

[Explain claim_ownership, persist_state, session keys]

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

### Multi-Instrument Mode

[Deep dive on MultiInstrument class, slot management, routing]

### State Introspection APIs

**Extracting Running Configuration**:
```python
moku = MultiInstrument(ip='192.168.1.100', platform_id=2)

# Query instruments in each slot
instruments = moku.get_instruments()  # ['CloudCompile', 'Oscilloscope', '', '']

# Query MCC routing
connections = moku.get_connections()  # [{'source': 'Input1', 'destination': 'Slot1InA'}, ...]

# Query frontend settings (per channel)
frontend = moku.get_frontend(channel=1)  # {'impedance': '1MOhm', 'coupling': 'DC', ...}

# Query output settings
output = moku.get_output(channel=1)  # {'output_gain': '0dB'}

# Query DIO configuration
dio = moku.get_dio(port=None)  # All DIO pins or specific port
```

**Mapping to moku-models**:
```python
from moku_models import MokuConfig, SlotConfig, MokuConnection, MOKU_LAB_PLATFORM

# Reconstruct MokuConfig from live device
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
```

---

## Integration Patterns

### With moku-models

[Cross-library integration patterns]

**Use Case 1: Deploy from MokuConfig**
```python
from moku_models import MokuConfig
from moku.instruments import MultiInstrument, Oscilloscope

# Load config (from YAML, etc.)
config = MokuConfig.from_dict(yaml.safe_load(open('config.yaml')))

# Deploy to hardware
moku = MultiInstrument(ip='192.168.1.100', platform_id=config.platform.slots)
for slot_num, slot_config in config.slots.items():
    instrument_class = getattr(instruments, slot_config.instrument)
    moku.set_instrument(slot_num, instrument_class, **slot_config.settings)
moku.set_connections([c.model_dump() for c in config.routing])
```

**Use Case 2: Extract MokuConfig from Running Device**
```python
# Query running device
moku = MultiInstrument(ip='192.168.1.100', platform_id=2)
instruments = moku.get_instruments()
connections = moku.get_connections()

# Create MokuConfig snapshot
config = MokuConfig(...)  # As shown above

# Save for later
config.to_dict() → YAML → disk
```

---

## File Structure

[Map of moku package - __init__.py, session.py, instruments/, etc.]

---

## Undocumented/Underutilized APIs

[Document findings from P1 investigation here]

**Session Handoff Methods**:
- persist_state behavior details
- Session key lifecycle

**State Extraction Methods**:
- Complete list of get_* methods
- What each returns
- Gaps in moku-models coverage

**Binary Configuration Files**:
- .mokuconf format (if you can reverse engineer it)
- save_configuration() / load_configuration() deep dive

---

## Common Tasks

[Practical examples for frequent operations]

---

## Development Workflow

[How to modify the inlined library, testing, etc.]

---

## API Reference Pointers

[Links to official docs, grouped by category]

---

**Last Updated**: 2025-11-07
**Part of**: Moku-pip-dev-fork (development workspace)
**Complements**: libs/moku-models/ (type-safe platform models)
```

**Token Budget**: ~3500-5000 tokens (comprehensive but not exhaustive)

**Critical**:
- Clear cross-references to `moku-models`
- Document the session handoff APIs prominently
- Integration patterns showing both libraries working together

---

## P3: Model Alignment Assessment

After completing P1 & P2, create a section in your response:

### MokuConfig Compatibility Assessment

**Current moku_config.py Coverage**:
- ✅ Platform selection
- ✅ Slot → instrument mapping
- ✅ MCC routing (connections)
- ✅ CloudCompile bitstream + control_registers
- ✅ Instrument settings (generic dict)

**Gaps Identified** (from first-party API):
- ❌ Frontend settings (impedance, coupling, attenuation, gain)
- ❌ Output gain settings
- ❌ DIO direction configuration
- ❌ Platform-specific constraints (validation)
- ❌ Binary .mokuconf import/export

**Recommendations**:
1. **Extend SlotConfig**?
   - Add `frontend_settings: dict[int, FrontendConfig] | None`
   - Add `output_settings: dict[int, OutputConfig] | None`

2. **Extend MokuConfig**?
   - Add `dio_config: DIOConfig | None`

3. **Backward Compatibility**?
   - Make all new fields Optional with sensible defaults
   - Existing code continues to work

4. **Alternative**: Keep moku_config.py minimal
   - Only model instrument deployment (current scope)
   - Don't replicate full device configuration
   - Document gaps in CLAUDE.md

**Provide your recommendation** based on what you discover.

---

## Deliverables

### 1. Investigation Report (in your response)

**Format**:
```markdown
## P1: Session Introspection Findings

### State Extraction APIs Discovered
[Comprehensive list with examples]

### Session Handoff Mechanics
[How persist_state works, session key lifecycle]

### Undocumented Features
[Anything not in official docs]

### Integration with moku-models
[Mapping table: first-party API → our abstractions]

### Gaps in Our Model
[What we're missing, what we should add]
```

### 2. Documentation Files (as file edits/writes)

**Create**:
- `libs/moku-4.0.3.1/llms.txt` (NEW, ~800-1000 tokens)
- `libs/moku-4.0.3.1/CLAUDE.md` (REPLACE or heavily modify existing)

**Requirements**:
- Follow 3-tier pattern exactly (use moku-models as reference)
- Token counts within guidelines (Tier 1: <1k, Tier 2: <5k)
- Clear cross-references between moku and moku-models
- Session handoff APIs prominently featured
- Progressive disclosure: llms.txt → CLAUDE.md → source code

### 3. Model Enhancement Proposal (optional)

If you identify critical gaps in `moku_config.py`, propose:
- New Pydantic model fields
- Backward-compatible changes
- Migration strategy

**We are willing to re-sculpt moku_config.py** if alignment is poor.

---

## Success Criteria

✅ **P1 Complete**:
- All `get_*` methods in MultiInstrument documented
- Session handoff workflow fully explained
- Map every API to moku_config attributes (or note gaps)
- Identify at least 3 undocumented/underutilized features

✅ **P2 Complete**:
- llms.txt follows Tier 1 format (skimmable, pointers to Tier 2)
- CLAUDE.md follows Tier 2 format (comprehensive, integration patterns)
- Token budgets respected (~1k Tier 1, ~4k Tier 2)
- Cross-references to moku-models are clear and accurate

✅ **P3 Complete**:
- Compatibility assessment delivered
- Clear recommendation on extending moku_config.py
- Backward compatibility considered

---

## Investigation Tips

### Finding Hidden APIs

```bash
# Search for all getters
grep -rn "def get_" libs/moku-4.0.3.1/moku/

# Search for all API operations
grep -rn "operation = " libs/moku-4.0.3.1/moku/

# Find v2 API endpoints
grep -rn "post_to_v2\|url_for_v2" libs/moku-4.0.3.1/moku/

# Session-related methods
grep -rn "session\." libs/moku-4.0.3.1/moku/ | grep -E "(get|post)"
```

### Understanding API Calls

Every instrument method ultimately calls:
```python
self.session.get(group, operation)      # Query
self.session.post(group, operation, params)  # Command
```

**Map these to HTTP**:
- `session.get("mim", "get_instruments")` → `GET /api/mim/get_instruments`
- `session.post("moku", "claim_ownership", {...})` → `POST /api/moku/claim_ownership`

### Tracing Session Lifecycle

1. `Moku.__init__()` → creates `RequestSession`
2. `claim_ownership()` → receives session key in response header (`Moku-Client-Key`)
3. `RequestSession.update_sk()` → stores key, adds to all future requests
4. All subsequent API calls carry session key
5. `relinquish_ownership()` → invalidates session key

### Reading moku-models for Reference

**Don't duplicate - cross-reference**:
- If moku-models already defines something (platform specs, port naming), point to it
- Your docs should complement, not replace
- Think: "How do these two libraries work together?"

---

## Questions to Resolve

If you encounter ambiguities, document them clearly:

1. **Can we fully reconstruct a MokuConfig from a live device?**
   - What's available via APIs?
   - What requires .mokuconf parsing?

2. **Should our model expand to match first-party scope?**
   - Frontend/output/DIO settings?
   - Or keep it focused on instrument deployment only?

3. **Are there CloudCompile-specific APIs for register introspection?**
   - Can we query CR0-CR31 values from a running device?

4. **What does persist_state actually persist?**
   - Instrument settings?
   - Routing configuration?
   - Frontend/output/DIO state?

---

## Repository Context

**You will be working from main branch** with this structure:
```
/private/tmp/Moku-pip-dev-fork/
├── libs/
│   ├── moku-4.0.3.1/          # First-party library (YOUR FOCUS)
│   │   ├── moku/
│   │   │   ├── __init__.py
│   │   │   ├── session.py
│   │   │   ├── instruments/
│   │   │   │   ├── _mim.py    # MultiInstrument - KEY FILE
│   │   │   │   ├── _oscilloscope.py
│   │   │   │   ├── _cloudcompile.py
│   │   │   │   └── ...
│   │   │   └── ...
│   │   ├── pyproject.toml
│   │   └── CLAUDE.md          # EXISTS - needs restructuring
│   │
│   └── moku-models/           # Our abstraction (REFERENCE)
│       ├── moku_models/
│       │   ├── moku_config.py # KEY FILE - may need extension
│       │   └── ...
│       ├── llms.txt           # TIER 1 REFERENCE
│       └── CLAUDE.md          # TIER 2 REFERENCE
│
├── CONTEXT_MANAGEMENT.md      # TIER STRATEGY - READ THIS
├── CLAUDE.md                  # Monorepo guide
└── HANDOFF_TO_WEB_WORKER.md   # THIS FILE
```

**All paths are relative to repository root** (`/private/tmp/Moku-pip-dev-fork/`)

---

## Final Notes

**Philosophy**: We're building a bridge between:
- First-party library (HTTP wrapper, session management)
- Our models (type-safe, deployment-focused, simulation-ready)

**Your documentation should make this bridge obvious**.

**Token Efficiency**: The whole point of 3-tier docs is that an AI agent:
- Loads Tier 1 (~1k tokens) for orientation
- Loads Tier 2 (~4k tokens) only when designing/integrating
- Loads source code only when debugging

**Maintain this efficiency!**

---

**Good luck! This is high-value work that will significantly improve the usability of both libraries.**

---

**Handoff created by**: Claude Code (CLI)
**Ready for**: Claude Web UI Worker
**Repository**: https://github.com/[your-org]/Moku-pip-dev-fork (main branch)
