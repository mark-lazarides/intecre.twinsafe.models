# Intecre.TwinSAFE.Models

Functional models of the Beckhoff TwinSAFE logic function blocks and the TwinSAFE group, written in
TwinCAT 3 for a **standard** PLC. They let you build and trial a safety application, and the
standard-PLC code that talks to it (Run/Stop, ErrAck, Restart, EDM, state and diag info data), without
any TwinSAFE hardware.

These are simulation models only. They are not safety rated and must never be used to implement a
safety function.

Behaviour is taken from the Beckhoff manual *TwinSAFE Logic FB* (version 4.3.0): pin names, parameters,
state codes and diagnostic bits follow that document so PLC code written against the models carries
over to the real thing.

## Solution layout

```
Intecre.TwinSAFE.Models.sln
Intecre.TwinSAFE.Models/                 TwinCAT solution project (tsproj)
  TwinSafeModels/                        PLC project
    Core/                                group, connection, terminal models, FB base class, helpers
    DUTs/                                state / diag enums and parameter structs
    FunctionBlocks/                      one T_Safe* FB per TwinSAFE FB
    Tests/                               TcUnit suites in ST (T_*_Tests) and FBD (T_*_Tests_FBD) + PRG_TestRunner
    Examples/                            PRG_EStopExample (ST) and PRG_EStopExample_CFC (same wiring in CFC)
    PlcTask.TcTTO                        runs PRG_TestRunner and both examples
```

Requirements: TwinCAT 3.1 4026, TcUnit (referenced as `TcUnit, * (www.tcunit.org)`), Tc2_Standard.

The project compiles cleanly in TwinCAT 3.1 4026 (verified with *Check all objects*). The TcUnit suites
have not yet been executed on a runtime; activate the configuration on a local or remote target and log
in to run them.

## Function block mapping

| TwinSAFE FB | Model | Parameters | Notes |
|---|---|---|---|
| AND | `T_SafeAnd` | `ST_SafeLogicParameters` | AndIn1 may be a standard signal |
| OR | `T_SafeOr` | `ST_SafeLogicParameters` | |
| XOR | `T_SafeXor` | none | EL6910 |
| DECOUPLE | `T_SafeDecouple` | none | |
| OPMODE | `T_SafeOpMode` | `ST_SafeOpModeParameters` | manual restart when Restart is wired, discrepancy time |
| ESTOP | `T_SafeEStop` | `ST_SafeEStopParameters` | 4 input pairs, EDM1/2, delayed output, EDM reset time |
| MON | `T_SafeMon` | `ST_SafeMonParameters` | Secure pair, manual restart, function test, ERRORDELAY |
| TWOHAND | `T_SafeTwoHand` | `ST_SafeTwoHandParameters` | 1BUTTON / 2BUTTON / RELEASE states |
| MUTING | `T_SafeMuting` | `ST_SafeMutingParameters` | sequential and parallel sensors, filter, max time, backwards, OSSD optional |
| EDM | `T_SafeEdm` | `ST_SafeEdmParameters` | switch-on / switch-off monitoring |
| RS | `T_SafeRs` | `ST_SafeLatchParameters` | reset dominant |
| SR | `T_SafeSr` | `ST_SafeLatchParameters` | set dominant |
| TON | `T_SafeTon` | `ST_SafeTimerParameters` | |
| TOF | `T_SafeTof` | `ST_SafeTimerParameters` | |
| CONNECTION SHUTDOWN | `T_SafeConnShutdown` | `ST_SafeConnShutdownParameters` | wire to a `T_SafeConnection` |
| TwinSAFE group | `T_SafeGroup` | none | EL/EJ6910 inputs and outputs, wired to the FBs and connections |
| TwinSAFE connection | `T_SafeConnection` | inputs | FSoE state codes 100..106 |
| EL1904 | `T_SafeInputTerminal` | inputs | fail-safe inputs unless in Data |
| EL2904 | `T_SafeOutputTerminal` | inputs | fail-safe outputs unless in Data |

Not modelled: TON2 (remaining-time persistence), the analogue and drive blocks (ADD, SUB, MUL, DIV,
COMPARE, LIMIT, COUNTER, SCALE, SPEED, LOADSENSING, CAMMONITOR, SLI, SLP, SBT, ADVPOSMON, Envelope,
ViolationCounter, safeCAM, safeLookUpTable, safeSequence), and group customising (Deactivate / Passivate).

## How the pieces fit together

Every block is a plain function block with `VAR_INPUT` / `VAR_OUTPUT` pins named as in the manual and
called like any TwinCAT FB, so it can be placed and wired in FBD exactly as in the Safety Editor. There
are no public methods.

* **Ports are active when connected.** Each signal input carries an `{attribute 'is_connected'}` flag, so
  a pair is two channel when both pins are wired, `Restart` connected means manual restart, `EDMx`
  connected activates the feedback loop, and so on. Unwired pins are ignored, as inactive ports are in
  TwinSAFE.
* **Parameters** that live in the Safety Editor property grid (contact type, discrepancy and delay times,
  options) are one `Parameters` struct input with defaults, normally set in the declaration:
  `eStop : T_SafeEStop := (Parameters := (DiscrepancyTime1 := T#500MS, DelayTime := T#1S));`
* **Group membership.** Every FB and connection takes its group by reference at initialisation
  (`eStop : T_SafeEStop(group := group);`) and registers itself, as a block placed in a TwinSAFE group
  belongs to it. Members read the group's `InRun` and `ErrAck` themselves, and the group scans its
  members' `Error`, `IsRunning`, `ComError` and `InStartup` outputs each cycle, so there is no group
  wiring to draw. Call the group before its members; declare it before them (the group is passed by
  reference, so it must exist when the members initialise). `T_SafeConnShutdown` takes the connection it
  shuts down the same way: `(group := group, connection := outputTerminal)`.

```pascal
VAR
    group          : T_SafeGroup;
    inputTerminal  : T_SafeInputTerminal(group := group);
    outputTerminal : T_SafeOutputTerminal(group := group);
    eStop          : T_SafeEStop(group := group) := (Parameters := (DiscrepancyTime1 := T#500MS, DelayTime := T#1S));
END_VAR

group(Run := plcSafetyRun, ErrAck := plcSafetyErrAck);
inputTerminal(Channel1 := fieldEStopNc1, Channel2 := fieldEStopNc2);
eStop(Restart := hmiRestart, EDM1 := contactorFeedback,          // standard PLC signals
      EStopIn1 := inputTerminal.SafeIn1, EStopIn2 := inputTerminal.SafeIn2);
outputTerminal(SafeOut1 := eStop.EStopOut);
```

`PRG_EStopExample` is a runnable version of this with a simulated contactor feedback loop, and
`PRG_EStopExample_CFC` is the same program drawn in CFC, the closest standard-PLC equivalent of the Safety
Editor: one box per block, wired pin to pin, with the group standing alone because membership is fixed at
declaration.

### Standard PLC interface

What the standard PLC normally links to, and where it is on the models:

| TwinSAFE | Model |
|---|---|
| Group Run/Stop, ErrAck, ModuleFault | `T_SafeGroup.Run`, `.ErrAck`, `.ModuleFault` |
| Group ComErr, FbErr, OutErr, OtherErr, ComStartup, FbDeactive, FbRun, InRun | outputs of the same name |
| Group Map State / Map Diag | `T_SafeGroup.State`, `.Diag` (`E_SafeGroupState`, `E_SafeGroupDiag`) |
| FB Standard-In ports (Restart, EDMx, AndIn1, MutingEnable, Mon1/2) | the FB `VAR_INPUT` of that name |
| FB Standard-Out ports (Error, xxxOut) | the FB `VAR_OUTPUT` of that name |
| FB Map State / Map Diag | `State : BYTE`, `Diag : WORD` on every FB |
| Connection Map State / Map Diag, ComErrAck | `State`, `Diag`, `ComErrAck` on `T_SafeConnection` and the terminals |

### Input configuration

A port's *Active* setting is its wiring (`is_connected`). *Contact type* is the `<Port>MakeContact` field
of the `Parameters` struct (FALSE = break contact, logic 0 requests the safe state; TRUE = make contact,
the signal is negated). A pair is two channel when both ports are wired; a `DiscrepancyTime` of `T#0S`
means two channel without discrepancy monitoring.

`T_SafeInputPair` implements the discrepancy monitoring and the *Safe Inputs after Disc Error* rule
(both channels must have been in the safe state together before the error can be acknowledged; diag bit
8+ is set while that is outstanding).

### Test hooks

* `InjectComError` (rising edge) on `T_SafeConnection` and the terminals simulates an FSoE error with
  `InjectErrorCode` (default watchdog). `InjectModuleError` on the terminals simulates a module error with
  *Module error is ComError*.
* `PartnerShutdown` on `T_SafeConnection` simulates a shutdown command from the FSoE partner.
* `StartupTime` sets how long a connection takes to reach the Data state.

## Behaviour notes and assumptions

Where the manual is explicit the models follow it. Where it is silent the following choices were made.
Each is a candidate to revisit once compared against real hardware.

**Group**

* FB errors are sticky: an FB in ERROR stays in ERROR when the group stops (group shows STOPERROR)
  and only leaves via ErrAck. The manual's FB tables say STOP for `FbRun = FALSE` without exception,
  but STOPERROR would not exist otherwise.
* The manual says an EL6910 group is in STOPERROR after initialisation when connections are assigned.
  The model starts in STOP and goes to START while connections come up, without needing an ErrAck.
* The group scans its members' outputs when it is called, so it sees FB and connection states from the
  previous cycle when the group is called first, as recommended.
* Connections start up once the group `Run` input is TRUE and do not restart when it goes FALSE.
* SAFE (3), DEACTIVE (16) and WAITCOMERROR (17) group states exist in the enum but are never entered.

**ESTOP / MON**

* While Restart is TRUE in SAFE the EDM inputs are checked immediately (restart behaviour sample 3),
  so a held restart during an e-stop event produces an EDM error before the contactors drop out.
* An EDM error is cleared by ErrAck alone. Discrepancy errors additionally need the *safe inputs* rule.
* MON without manual restart never raises an EDM error; it stays in SAFE while EDM is FALSE (as documented).
* MON manual function test is required once after STOP → running, not again after an ErrAck.
* The DELAYOUT delay runs to completion even if the inputs return to normal during the delay.

**OPMODE**

* The discrepancy timer only runs after a mode had been selected at least once since the FB started,
  so a selector left between positions at power-up does not raise an error.

**TWOHAND**

* From SAFE, both hands actuated in the same cycle go via 1BUTTON and reach RUN a cycle later.
* Diag bit 2 (discrepancy between the hands) waits for both hands released before acknowledgement when
  *Safe Inputs after Disc Error* is set (bit 10).

**MUTING**

* Sensor order (from the state table): four sequential sensors
  `M1↑ M2↑ OSSD↓ M3↑ M1↓ M4↑ M2↓ OSSD↑ M3↓ M4↓`; two sensors (parallel mode, or sequential with
  MutingIn3/4 unwired) `S1↑ S2↑ OSSD↓ S1↓ OSSD↑ S2↓`. The two-sensor order is an assumption.
* Any sensor or OSSD change that is not the next expected event violates the sequence (diag bit 6).
* A sensor actuated out of order while `MutingEnable` is TRUE and no sequence is running is a violation.
* OSSD interrupted with no sequence running gives SAFE (automatic restart when clear), never an error.
* Withdrawing `MutingEnable` mid-sequence abandons the sequence; the OSSD then decides RUN / SAFE.
* EDM inputs gate the SAFE → RUN transition without raising an error (there is no Restart input);
  the EL6910 *Reset Time* option does raise diag bits 4/5.

**Timers**

* TON / TOF use the PLC task time. TwinSAFE timebase quantisation (1 / 10 / 100 ms) is not modelled.

## Tests

`PRG_TestRunner` runs 36 TcUnit suites: 18 in structured text (`T_*_Tests`) and their FBD twins
(`T_*_Tests_FBD`), 48 tests each. The FBD suites exercise the blocks the way a TwinSAFE project is
authored: each test is a sequence of networks calling the FB with its pins wired. Timed tests in FBD
use chained `TON` timers and `R_TRIG` one-shots on the `EN` of the assert boxes instead of the ST
`CASE` state machines, so the two versions of a timed test are equivalent rather than identical.

Activate the configuration, log in, and read the results in the TwinCAT error list or through the
TcUnit xUnit output. Timed tests use 50 to 900 ms waits, so a full run takes a few seconds. All suites
compile; they have not yet been executed on a runtime. One thing to confirm on a runtime first is the
`is_connected` semantics in ST calls: the ST suites assume a pin passed in the call `fb(Pin := x)` reads
as connected and an omitted pin as unconnected.

## Conventions

Blocks use the `T_` prefix, private members `_camelCase`, and state machines use `CASE` with a
`StateChanged()` entry pattern. Deliberately unlike the house class style, the blocks expose only
`VAR_INPUT` / `VAR_OUTPUT` pins and no public methods, so they behave like their TwinSAFE equivalents and
wire up in FBD.
