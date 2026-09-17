# FanucIntegration_V2

Ready-made function blocks for controlling a FANUC R-30iB Plus robot over EtherCAT using the FANUC EtherCAT Slave option (R-30iB Plus, PDO3 mapping). Handles the full enable sequence, safety signal management, fault reset, homing, cycle stop, program start (RSR), and speed override through a FANUC Group Input.

---
##Installation

### Install the library

1. Open **TwinCAT XAE**.
2. Open **PLC → Library Repository**.
3. Select **Install** and choose the `FanucSimpleLib.library` file.

### Add the library to a project

1. Open the PLC project.
2. Right-click **References** and select **Add Library**.
3. Find the library under **Miscellaneous** and add it.

### Application Global Variables

The application should declare the hardware-linked variables after referencing the library:

Create a Global Variable List named GVL_Fanuc in the consuming PLC application and add the following variables:
```iecst
{attribute 'qualified_only'}
VAR_GLOBAL
	DI0 AT %I* : ARRAY[0..15] OF BYTE;
	DO0 AT %Q* : ARRAY[0..15] OF BYTE;
END_VAR
```

The application developer must then link these arrays to the corresponding FANUC EtherCAT input and output process data in the TwinCAT I/O configuration.

> **Important:** This project controls FANUC UOP signals. It does not replace the robot controller's safety functions, safety PLC, DCS configuration, or required machine risk assessment.

---

## Function Blocks

| Function block | Purpose |
|---|---|
| `FB_FanucStatus` | Converts EtherCAT input bytes into `ST_FanucStatus` |
| `FB_FanucControl` | Converts `ST_FanucControl` into EtherCAT output bytes |
| `FB_FanucEnable` | Manages remote enable, interlocks, Ready state, and START |
| `FB_FanucFaultReset` | Pulses FAULT_RESET and waits for the fault to clear |
| `FB_FanucStartProgram` | Starts RSR programs 1–8 and validates the matching ACK |
| `FB_FanucCycleStop` | Sends a configurable CSTOPI pulse |
| `FB_FanucHoming` | Starts the configured HOME macro and waits for home confirmation |
| `FB_FanucSpeedOverride` | Validates and sends a 0–100% speed override |

## Program Architecture

All command function blocks write into the same `ST_FanucControl` instance. They should be called in the following order during every PLC cycle:

1. `FB_FanucStatus`
2. `FB_FanucEnable`
3. `FB_FanucHoming`
4. `FB_FanucStartProgram`
5. `FB_FanucCycleStop`
6. `FB_FanucFaultReset`
7. `FB_FanucSpeedOverride`
8. `FB_FanucControl`

```text
EtherCAT inputs
      ↓
FB_FanucStatus
      ↓
FB_FanucEnable
      ↓
Command function blocks
      ↓
FB_FanucSpeedOverride
      ↓
FB_FanucControl
      ↓
EtherCAT outputs
```

Call all function blocks cyclically. Do not place their calls inside conditional statements because their internal timers and edge detectors must execute every PLC cycle.

## EtherCAT Input Mapping

`FB_FanucStatus` converts `GVL_Fanuc.DI0` into `ST_FanucStatus`.

| EtherCAT address | FANUC output | Status variable |
|---|---|---|
| `DI0[0].0` | UO[1] CMDENBL | `bCmd_Enabled` |
| `DI0[0].1` | UO[2] SYSRDY | `bSystem_Ready` |
| `DI0[0].2` | UO[3] PROGRUN | `bPrg_Running` |
| `DI0[0].3` | UO[4] PAUSED | `bPrg_Paused` |
| `DI0[0].4` | UO[5] HELD | `bMotion_Held` |
| `DI0[0].5` | UO[6] FAULT | `bFault` |
| `DI0[0].6` | UO[7] ATPERCH | `bAt_Home` |
| `DI0[0].7` | UO[8] TPENBL | `bTP_Enabled` |
| `DI0[1].0` | UO[9] BATALM | `bBatteryAlarm` |
| `DI0[1].1` | UO[10] BUSY | `bBusy` |
| `DI0[2].0..7` | UO[11..18] ACK1..ACK8 | `bProg_Start1ACK..bProg_Start8ACK` |

## EtherCAT Output Mapping

`FB_FanucControl` converts `ST_FanucControl` into `GVL_Fanuc.DO0`.

| EtherCAT address | FANUC input | Control variable |
|---|---|---|
| `DO0[0].0` | UI[1] *IMSTP | `IMSTP` |
| `DO0[0].1` | UI[2] *HOLD | `Hold` |
| `DO0[0].2` | UI[3] *SFSPD | `SFSPD` |
| `DO0[0].3` | UI[4] CSTOPI | `Cycle_stop` |
| `DO0[0].4` | UI[5] FAULT_RESET | `Fault_Reset` |
| `DO0[0].5` | UI[6] START | `Start` |
| `DO0[0].6` | UI[7] HOME | `Home` |
| `DO0[0].7` | UI[8] ENBL | `Enable` |
| `DO0[1].0..7` | UI[9..16] RSR1..RSR8 | `Start_Program1..Start_Program8` |
| `DO0[2].0` | UI[17] PNS Strobe | `PNS_Strobe` |
| `DO0[2].1` | UI[18] PROD_START | `Prod_Start` |
| `DO0[3]` | Configured Group Input | `Overwrite` |

The FANUC rack, slot, start point, byte order, and bit order must match the EtherCAT process-data configuration.

## FB_FanucStatus

Reads the raw EtherCAT input array and maps the FANUC status signals into `ST_FanucStatus`. Call it before any logic that depends on robot feedback.

```iecst
fbStatus(
	aDI := GVL_Fanuc.DI0,
	stStatus => stStatus);
```

## FB_FanucEnable

Manages the FANUC remote-enable sequence and external interlocks.

```text
Idle → ClearFault → CheckTP → WaitReady → Ready
```

The sequence starts only when `bEnable`, `bExtIMSTP`, `bExtHold`, and `bExtSFSPD` are TRUE and `bExtStart` is released.

The external interlocks use the FANUC normally-closed convention:

- `TRUE`: normal or permissive
- `FALSE`: stop, hold, or safe-speed request

Important behavior:

- `ClearFault` does not generate FAULT_RESET. Use `FB_FanucFaultReset` separately.
- `WaitReady` requires both CMDENBL and SYSRDY.
- `tReadyTimeout` defaults to `T#5S`.
- A true rising edge of `bExtStart` while already **Ready** generates a fixed 20 ms START pulse.
- A **Start** edge before Ready is ignored.
- A held **Start** does not execute automatically when **Ready** is reached.
- **Fault**, **TP enable**, **E-stop**, **safe speed**, and **hold** are monitored continuously.
- Release `bEnable` to reset the **Error state**.

## FB_FanucFaultReset

A rising edge of `bReset` sends a fixed 20 ms FAULT_RESET pulse. The function block then waits for `stStatus.bFault = FALSE`.

```text
Idle → SendPulse → WaitClear → Done → Idle
                         ↘ Error
```

- `tFaultTimeout` defaults to `T#5S` and starts after the pulse.
- `bBusy` is TRUE while pulsing or waiting for the fault to clear.
- `bDone` is TRUE for one PLC scan after the fault is confirmed clear.
- `bError` remains TRUE after timeout until `bReset` is released.

## FB_FanucStartProgram

Starts one of eight FANUC RSR programs. A rising edge of `bStart` is accepted only when `bReady = TRUE` and `nProgram` is within `1..8`.

```text
Idle → Pulsing → WaitAck → Done → Idle
  ↘                         ↘ Error
```

Important behavior:

- The selected program number is latched when the request is accepted.
- Exactly one RSR output is asserted for a fixed 200 ms pulse.
- The corresponding ACK is monitored during and after the pulse.
- ACK detection is armed only after the selected ACK has been observed LOW, preventing a stale HIGH ACK from completing a new request.
- `tAckTimeout` defaults to `T#2S` and applies after the RSR pulse.
- Loss of **Ready** returns the function block to Idle and removes the RSR output.
- `bRequestActive` is TRUE while pulsing or waiting for ACK.
- `bDone` is TRUE for one scan after a fresh matching ACK.
- Invalid program selection or ACK timeout enters Error; release `bStart` to reset it.

## FB_FanucCycleStop

A rising edge of `bStart` sends CSTOPI for `tCycleStopPulse`, which defaults to `T#20MS`.

The FANUC controller setting **CSTOPI for ABORT** determines the robot-side result:

- Disabled: queued RSR programs are cleared and the running program continues.
- Enabled: queued RSR programs are cleared and the running program is aborted.

This setting is configured on the FANUC controller and cannot be selected through the function block.

CSTOPI provides no acknowledgement. Therefore, `bDone` confirms only that the configured CSTOPI pulse was transmitted—it does not confirm that robot motion has stopped.

## FB_FanucHoming

A rising edge of `bStart` is accepted only while `bReady = TRUE`. The function block sends a fixed 200 ms HOME pulse and then waits for `stStatus.bAt_Home`.

```text
Idle → RequestHome → WaitComplete → Done → Idle
                                  ↘ Error
```

- `tHomingTimeout` defaults to `T#30S`.
- `bBusy` is TRUE while pulsing HOME or waiting for completion.
- `bDone` is TRUE for one scan after at-home feedback.
- Timeout enters Error; release `bStart` to reset it.
- Loss of Ready returns the function block to Idle.
- Returning to Idle removes the PLC command but does not necessarily cancel a FANUC macro already running.

UI[7] must be configured to invoke the required homing macro, and UO[7] must represent the application's home confirmation.

## FB_FanucSpeedOverride

Continuously writes a validated percentage to `stControl.Overwrite`.

- Valid input: `0..100`
- `0` = 0% programmed speed
- `100` = 100% programmed speed
- Values above 100 set `bError = TRUE` and force `nActual = 0`
- This is a continuous setpoint, not a pulse command

The connected FANUC Group Input must be configured consistently with the byte mapping.

## Example Usage

```iecst
// 1. Read current FANUC feedback
fbStatus(
	aDI := GVL_Fanuc.DI0,
	stStatus => stStatus);

// 2. Enable and interlock management
fbEnable(
	bEnable			:= bEnableRobot,
	stStatus		:= stStatus,
	bExtIMSTP		:= bExtIMSTP,
	bExtHold		:= bExtHold,
	bExtSFSPD		:= bExtSFSPD,
	bExtStart		:= bExtStart,
	tReadyTimeout	:= T#5S,
	stControl		=> stControl);

// 3. Homing
fbHoming(
	bStart				:= bStartHome,
	bReady				:= fbEnable.bReady,
	tHomingTimeout	:= T#30S,
	stStatus			:= stStatus,
	stControl			:= stControl);

// 4. Program start
fbStartProgram(
	bStart			:= bStartProgram,
	nProgram		:= nProgram,
	bReady			:= fbEnable.bReady,
	tAckTimeout	:= T#2S,
	stStatus		:= stStatus,
	stControl		:= stControl);

// 5. Cycle Stop
fbCycleStop(
	bStart				:= bCycleStop,
	tCycleStopPulse	:= T#20MS,
	stControl			:= stControl);

// 6. Fault Reset
fbFaultReset(
	bReset			:= bFaultReset,
	tFaultTimeout	:= T#5S,
	stStatus		:= stStatus,
	stControl		:= stControl);

// 7. Speed Override
fbSpeedOverride(
	nOverwrite	:= byOverwrite,
	stControl	:= stControl);

// 8. Write the completed output image
fbControl(
	aDO			:= GVL_Fanuc.DO0,
	stControl	:= stControl);
```

## FANUC Configuration Requirements

Before commissioning:

1. Configure FANUC UOP for remote operation.
2. Map the required UI and UO signals to the EtherCAT connection.
3. Verify that the FANUC and TwinCAT byte and bit ordering match.
4. Configure RSR1 through RSR8 and the corresponding ACK/SNO outputs.
5. Configure HOME UI[7] to execute the required homing macro.
6. Configure UO[7] as the required home-position confirmation.
7. Configure the Group Input used for speed override.
8. Verify that **CSTOPI for ABORT** matches the required application behavior.
9. Test all commands at reduced speed in a controlled commissioning environment.

## Project Structure

```text
FanucIntegrationV2/
├── DUTs/    Status/control structures and state enumerations
├── GVLs/    EtherCAT process-data arrays. `GVL_Fanuc` is included in the reference project but excluded from the compiled library.
└── POUs/    FANUC interface and command function blocks
```

## Software

- TwinCAT 3 PLC
- IEC 61131-3 Structured Text
- Current project object versions: TwinCAT 3.1.4026.x
