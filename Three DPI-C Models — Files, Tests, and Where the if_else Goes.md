UVM / UVMF // Three DPI-C Models

# Three DPI-C Models, One Testbench

Where the files go, how the tests and sequences are layered, and the one place the "which model?" if/else is allowed to live. All file and class names are examples — rename to match your project.

## 1. The Confusion, Resolved

Short answer

A test never contains "if model A … else if model B." The model is a **config value** chosen once at the top (`base_test`), and the only code that cares what the value is lives in a small `dpic/sv/` package. Common setup goes in the base classes. What differs per model is **data** (a spec sheet), not **code** (a copy of the test).

**Model names appear in one folder.** The switch on "trident / triton / picket" lives in `dpic/sv/` (two tiny case statements: which C function to call, and which numbers apply). Tests, sequences, scoreboard, and env never switch on a model name.

**Two independent axes — don't multiply them.** *Which model* (3 values) is a config knob. *What scenario* (smoke, sweep, calibration…) is the test + sequence. You maintain 3 + N things, not 3 × N classes.

**If a sequence has to branch, branch on a capability, not a name.** `if (cfg.has_cal_mode)` survives you adding a fourth model; `if (model == PICKET)` doesn't.

### Same idea, on the test floor

| Test station | Your testbench |
| --- | --- |
| Which signal-source card is installed today | `+DPIC_MODEL=…` (or the one-line `default_model()` in a thin test) |
| The card itself | The DPI-C model (`lib<model>_adc.so`) |
| The card's spec sheet (range, resolution, optional features) | `dpic_cfg` — width, rate, capability flags |
| Station power-up: load config, connect, arm | `adc_base_test` + `adc_base_seq` (common setup) |
| A test program | A scenario test + its sequence |
| A test program that checks "is this card model X?" | What to avoid — check the spec sheet (capability flag) instead |

## 2. Who Calls Whom

HVL — class world (UVM)

everything here can call C; nothing here has ports

1

TEST adc_base_test ← scenario tests and thin per-model tests extend this

- **common:** create env, create `dpic_cfg`, read `+DPIC_MODEL`, apply model defaults, publish cfg via `config_db`, timeouts
- **hook:** `default_model()` — the one thing a thin per-model test overrides
- the test starts the scenario sequence and owns the objection

cfg published through **config_db** (set once) ↓

2

SEQ adc_base_seq → scenario sequences

- **common:** fetch `dpic_cfg` in `pre_body`, `send_burst(n)` helper
- items carry **control only** — how many samples, when — never the data
- may branch on `cfg.has_cal_mode`-style flags, never on model name

3

AGENT sequencer → driver (proxy)

- gets `dpic_cfg` from config_db
- creates the wrapper and calls `init(cfg.model)` **once**, at start of simulation
- per item: `dpic.next()` N times, pack a burst, hand it to the BFM

driver proxy calls ↓

4

dpic/sv dpic_model_wrapper (in `dpic_pkg.sv`)

- **the only DPI-C import list in the testbench** — all three models' `init / next / term`
- **the if/else lives here:** `case (model)` picks which imported function runs
- holds the `chandle` state for the **selected** model only
- sibling `dpic_cfg.sv`: per-model width, rate, capability flags (second small case)

links into the sim as three shared libraries ↓ (only the selected one is ever called)

.so libtrident_adc

- `trident_adc_init/next/term`

.so libtriton_adc

- `triton_adc_init/next/term`

.so libpicket_adc

- `picket_adc_init/next/term`

sample burst crosses the HVL / HDL boundary ↓

HDL — module world (hdl_top)

pin-level only; no C code in here, so it stays emulator-friendly

5

BFM adc driver BFM

- takes the burst, drives `adc_data` / `valid` clock by clock
- no idea which model produced the samples

DUT your FPGA RTL

- wired to the interface once in `hdl_top`
- identical for all three models

test level / model-specific / HDL sequences agent shared components the one switch point

Why the DPI-C call sits in the driver proxy and not in the BFM: C code can't run inside an emulator's module side. Keeping it in the class world means the same testbench runs in simulation now and on an emulator later. If you only ever simulate, calling from the BFM works too — but you'd be giving that option up for no gain.

## 3. How One Run Picks Its Model

**command line**`+UVM_TESTNAME=test_sweep +DPIC_MODEL=picket`

→

**adc_base_test**default_model(), then plusarg overrides it

→

**dpic_cfg**model = PICKET, width, rate, flags

→

**config_db**set once under `env.*`

→

**driver proxy**get cfg, `dpic.init(PICKET)`

→

**wrapper case**calls only `picket_adc_*`

→

**libpicket_adc.so**returns samples

Same run without the plusarg: `+UVM_TESTNAME=test_picket` — the thin test's `default_model()` returns PICKET and everything downstream is identical. Both ways end in the same `dpic_cfg`.

## 4. Example File Structure

```
adc_uvmf_project/
├── dpic/                                  ← YOUR code, kept outside generated UVMF folders
│   ├── matlab/
│   │   ├── trident_adc.m
│   │   ├── triton_adc.m
│   │   ├── picket_adc.m
│   │   └── build_dpi_model.m              ← one script, run once per model: build_dpi_model('picket')
│   ├── lib/                               ← build output (same function shape, unique prefix per model)
│   │   ├── libtrident_adc.so
│   │   ├── libtriton_adc.so
│   │   └── libpicket_adc.so
│   └── sv/
│       ├── dpic_pkg.sv                    ← DPI imports + dpic_model_e enum + dpic_model_wrapper  (case #1)
│       └── dpic_cfg.sv                    ← per-model width / rate / capability flags           (case #2)
│
├── verification_ip/
│   ├── interface_packages/adc_pkg/src/
│   │   ├── adc_driver_bfm.sv              ← HDL side: pin-level drive task (no C here)
│   │   ├── adc_monitor_bfm.sv
│   │   ├── adc_driver.sv                  ← proxy: small edit — create wrapper, call dpic.next(), pass burst to BFM
│   │   └── adc_txn.sv                     ← item: num_samples etc. — control only, no sample data
│   └── environment_packages/adc_env_pkg/  ← env, scoreboard (model-blind)
│
└── project_benches/adc_bench/
    ├── tb/testbench/
    │   ├── hdl_top.sv                     ← interface, clock/reset, DUT port map
    │   └── hvl_top.sv                     ← run_test(), config_db for BFM handles
    ├── tb/sequences/src/
    │   ├── adc_base_seq.sv                ← common: fetch cfg, send_burst()
    │   ├── adc_smoke_seq.sv
    │   ├── adc_sweep_seq.sv
    │   └── adc_cal_seq.sv                 ← branches on cfg.has_cal_mode, not on model name
    ├── tb/tests/src/
    │   ├── adc_base_test.sv               ← common setup + default_model() hook
    │   ├── test_trident.sv  test_triton.sv  test_picket.sv   ← optional thin tests, ~3 lines each
    │   └── test_smoke.sv  test_sweep.sv  test_cal.sv               ← scenarios: start a sequence, know nothing about models
    └── sim/
        ├── Makefile                       ← compile once, link all 3 libs, regression loop
        └── logs/
```

green = files you write and own · grey = generated or build output · UVMF folder and file names vary by version and by how your YAML is set up, so treat the `verification_ip/` and `project_benches/` branches as a map, not gospel. Check which files your UVMF version marks as safe to hand-edit before touching the driver proxy — regeneration can overwrite edits. If you build with `uvmfbuild`, point its output at `dpic/lib/` and `dpic/sv/` and the rest of this structure doesn't change.

## 5. Test and Sequence Layering — What Goes Where

### Tests (what to run)

uvm_test

adc_base_test COMMON

• create env + `dpic_cfg`\
• parse `+DPIC_MODEL`\
• publish cfg, set timeouts, report\
• HOOK `default_model()`

test_trident / test_triton / test_picket OPTIONAL

• override `default_model()` only — nothing else

test_smoke / test_sweep / test_cal SCENARIOS

• create a sequence, `start()` it, own the objection\
• zero knowledge of which model is active

one-off combos

• need a Picket-only sweep? extend `test_sweep` and override `default_model()`

### Sequences (how to stimulate)

uvm_sequence #(adc_txn)

adc_base_seq COMMON

• `pre_body`: fetch `dpic_cfg`\
• `send_burst(n)` helper\
• control-only items (counts, timing)

adc_smoke_seq

• runs on any model as-is

adc_sweep_seq

• scales ranges using `cfg.width`

adc_cal_seq CAPABILITY CHECK

• `if (!cfg.has_cal_mode)` log and skip\
• this is the legitimate "if" — keyed on a feature flag

### What a regression looks like

| scenario \\ model | trident | triton | picket |
| --- | --- | --- | --- |
| `test_smoke` | runs | runs | runs |
| `test_sweep` | runs | runs | runs |
| `test_cal` | skips (no cal mode) | skips (no cal mode) | runs |

Example flags — set them from your real ADC specs. Nine runs from three scenario classes plus the optional thin tests. The "wrong" way — one test class per cell — would be nine classes, and a fourth model would add three more.

## 6. The Skeleton Code

Shape only — enough to show where each piece sits. Imports, includes, and your real transaction fields are left out. The numbers in `dpic_cfg` are placeholders.

dpic/sv/dpic_pkg.sv — the DPI imports and the first switch

```
package dpic_pkg;
  import uvm_pkg::*;  `include "uvm_macros.svh"

  typedef enum {TRIDENT, TRITON, PICKET} dpic_model_e;

  // one init / next / term trio per model — same shape, unique prefix
  import "DPI-C" function chandle trident_adc_init();
  import "DPI-C" function real    trident_adc_next(input chandle s);
  import "DPI-C" function void    trident_adc_term(input chandle s);
  // ... triton_adc_*, picket_adc_* identical in shape ...

  class dpic_model_wrapper extends uvm_object;
    `uvm_object_utils(dpic_model_wrapper)
    dpic_model_e model;
    chandle      state;
    function new(string name = "dpic_model_wrapper"); super.new(name); endfunction

    function void init(dpic_model_e m);
      model = m;
      case (model)                         // ← THE if/else
        TRIDENT: state = trident_adc_init();
        TRITON : state = triton_adc_init();
        PICKET : state = picket_adc_init();
      endcase
    endfunction

    function real next();
      case (model)
        TRIDENT: return trident_adc_next(state);
        TRITON : return triton_adc_next(state);
        PICKET : return picket_adc_next(state);
        default : begin `uvm_fatal("DPIC", "model not initialised") return 0.0; end
      endcase
    endfunction
  endclass
endpackage
```

dpic/sv/dpic_cfg.sv — the "spec sheet" (second switch; data, not behaviour)

```
class dpic_cfg extends uvm_object;
  `uvm_object_utils(dpic_cfg)
  dpic_model_e model;
  int unsigned width;          // placeholder values — use your real ADC specs
  bit          has_cal_mode;   // capability flag — sequences check this, not the model name

  function new(string name = "dpic_cfg"); super.new(name); endfunction

  function void set_model_by_name(string s);
    case (s.tolower())
      "trident": model = TRIDENT;
      "triton" : model = TRITON;
      "picket" : model = PICKET;
      default  : `uvm_fatal("DPIC_CFG", {"unknown DPIC_MODEL: ", s})
    endcase
  endfunction

  function void apply_model_defaults();
    case (model)
      TRIDENT: begin width = 14; has_cal_mode = 0; end
      TRITON : begin width = 12; has_cal_mode = 0; end
      PICKET : begin width = 16; has_cal_mode = 1; end
    endcase
  endfunction
endclass
```

tb/tests/src/adc_base_test.sv, test_picket.sv, test_cal.sv — common setup, thin test, scenario test

```
class adc_base_test extends uvm_test;
  `uvm_component_utils(adc_base_test)
  adc_env  env;
  dpic_cfg cfg;
  function new(string name, uvm_component parent); super.new(name, parent); endfunction

  virtual function dpic_model_e default_model();   // HOOK
    return TRIDENT;
  endfunction

  virtual function void build_phase(uvm_phase phase);
    string m;
    super.build_phase(phase);
    cfg       = dpic_cfg::type_id::create("cfg");
    cfg.model = default_model();
    if ($value$plusargs("DPIC_MODEL=%s", m)) cfg.set_model_by_name(m);   // plusarg wins
    cfg.apply_model_defaults();
    uvm_config_db #(dpic_cfg)::set(this, "env.*", "dpic_cfg", cfg);
    env = adc_env::type_id::create("env", this);
  endfunction
endclass

// ---- thin per-model test: three lines of substance ----
class test_picket extends adc_base_test;
  `uvm_component_utils(test_picket)
  function new(string name = "test_picket", uvm_component parent = null); super.new(name, parent); endfunction
  virtual function dpic_model_e default_model(); return PICKET; endfunction
endclass

// ---- scenario test: knows nothing about models ----
class test_cal extends adc_base_test;
  `uvm_component_utils(test_cal)
  function new(string name = "test_cal", uvm_component parent = null); super.new(name, parent); endfunction
  virtual task run_phase(uvm_phase phase);
    adc_cal_seq seq = adc_cal_seq::type_id::create("seq");
    phase.raise_objection(this);
    seq.start(env.agent.sequencer);          // use your env's real path
    phase.drop_objection(this);
  endtask
endclass
```

tb/sequences/src/adc_base_seq.sv and adc_cal_seq.sv — common helpers, capability check

```
class adc_base_seq extends uvm_sequence #(adc_txn);
  `uvm_object_utils(adc_base_seq)
  dpic_cfg cfg;
  function new(string name = "adc_base_seq"); super.new(name); endfunction

  virtual task pre_body();
    if (!uvm_config_db #(dpic_cfg)::get(m_sequencer, "", "dpic_cfg", cfg))
      `uvm_fatal("SEQ", "dpic_cfg not found")
  endtask

  task send_burst(int unsigned n);                  // shared helper
    adc_txn t = adc_txn::type_id::create("t");
    start_item(t);
    t.num_samples = n;                           // control only — driver fetches the data
    finish_item(t);
  endtask
endclass

class adc_cal_seq extends adc_base_seq;
  `uvm_object_utils(adc_cal_seq)
  function new(string name = "adc_cal_seq"); super.new(name); endfunction
  virtual task body();
    if (!cfg.has_cal_mode) begin              // capability, not model name
      `uvm_info("CAL", "skipped: selected model has no cal mode", UVM_LOW)
      return;
    end
    send_burst(256);
  endtask
endclass
```

verification_ip/.../adc_driver.sv — the proxy edit (fragment; your generated class and field names will differ)

```
dpic_model_wrapper dpic;
dpic_cfg           cfg;

virtual function void start_of_simulation_phase(uvm_phase phase);
  super.start_of_simulation_phase(phase);
  if (!uvm_config_db #(dpic_cfg)::get(this, "", "dpic_cfg", cfg))
    `uvm_fatal("DRV", "dpic_cfg not found")
  dpic = dpic_model_wrapper::type_id::create("dpic");
  dpic.init(cfg.model);                          // once per run — NOT per item, or the model's state resets
endfunction

// ...inside the per-item handling:
for (int i = 0; i < txn.num_samples; i++) burst[i] = dpic.next();
bfm.drive_burst(burst);                          // pin-level work stays in the HDL-side BFM
```

project_benches/adc_bench/sim/Makefile — compile once, run the matrix

```
MODELS = trident triton picket
TESTS  = test_smoke test_sweep test_cal
# link ALL three libraries even though each run uses one (flag spelling varies by simulator)
SV_LIBS = -sv_lib ../../../dpic/lib/libtrident_adc \
          -sv_lib ../../../dpic/lib/libtriton_adc  \
          -sv_lib ../../../dpic/lib/libpicket_adc

regress:
	for t in $(TESTS); do for m in $(MODELS); do \
	  $(SIM) $(SV_LIBS) +UVM_TESTNAME=$${t} +DPIC_MODEL=$${m} -l logs/$${t}_$${m}.log; \
	done; done
```

## 7. Ways This Goes Wrong

| Mistake | What you'll see | Fix |
| --- | --- | --- |
| `init()` called per item | ADC signal restarts from phase zero every burst | Init once in `start_of_simulation_phase`; keep one `chandle` per run |
| `if (model == PICKET)` in a sequence or scoreboard | Every new model means hunting through test code | Add a capability flag to `dpic_cfg` and branch on that |
| One test class per model × scenario | 3 × N files that drift apart | Scenario = test; model = `+DPIC_MODEL` or `default_model()` |
| Only the selected library linked | Unresolved DPI import at elaboration for the unused models | Link all three in every run |
| Identically named C helper functions across the three libraries | Duplicate-symbol warnings, or one model calling another's helper | Unique prefix per model at build time; check the link log |
| Models with different output widths feeding the driver directly | Driver grows per-model branches | Normalize inside the wrapper or from `cfg.width` |
| DPI-C call placed inside the BFM | Works in simulation, blocks later emulation | Keep the call in the driver proxy; pass samples down |
| Hand-edited generated UVMF files | Edits vanish on the next regeneration | Keep edits to the proxy small and documented; keep real logic in `dpic/sv/` |

Mental model: the model is the card you slot into the station, the scenario is the test program you run on it, and the only code allowed to ask "which card?" is the one small package that talks to the card's connector.