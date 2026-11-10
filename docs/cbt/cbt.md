# CDCM based transceiver (CBT)

The CBT is a physical layer of the MIKUMARI system. About the CBT, see also [Ref](https://ieeexplore.ieee.org/document/10098185). The roles of the CBT are as follows.

- Encode and decode the CDCM modulated waveform pattern
- Defines three types of CBT character using sign extension from 8-bit to 10-bit
- Initialize IOSERDES when detecting the cable (fiber) connection

Currently, the CBT supports the CDCM-10-2.5, CDCM-10-1.5, CDCM-8-2.5, and CDCM-8-1.5. Then, the frequency ratios between the serial and parallel clock signals are 5 for CDCM-10-XX and 4 for CDCM-8-XX with the double-data-rate (DDR) mode, respectively. The SDR mode is not supported. In addition, only the differential signal is supported.
**If you want to use the Local Area Common Clock Protocol (LACCP), you need to select CDCM-8-XX modes. The serdesOffset port does not work with CDCM-10-XX modes.**

## Interface of CBT

**CbtLane** is the top-level CBT module. Types and width constants are defined in `defCDCM.vhd`.

### Generics

Generics without a default must be specified when instantiating `CbtLane`.

| Generic | Description |
| --- | --- |
| `kFamily` | FPGA family: `"7S"` for 7-series; `"US"` for UltraScale/UltraScale+. |
| `kCdcmModWidth` | CDCM time slices: `8` for CDCM-8; `10` for CDCM-10. |
| `kIoStandardTx` | TX differential output buffer I/O standard. |
| `kTxPolarity` | Invert TX polarity when `true`. |
| `genIDELAYCTRL` | Instantiate IDELAYCTRL when `true`. |
| `kDiffTerm` | Enable differential input termination when `true`. |
| `kRxPolarity` | Invert RX polarity when `true`. |
| `kIoStandardRx` | RX differential input buffer I/O standard. |
| `kIoDelayGroup` | IODELAY_GROUP for IDELAYCTRL and delay primitives. |
| `kFixIdelayTap` | Use `tapValueIn` for initialization instead of automatic tap adjustment when `true`. |
| `kFreqFastClk` | SERDES serial clock frequency in MHz. |
| `kFreqRefClk` | IDELAYCTRL reference clock frequency in MHz. |
| `kBitslice0` | Set `true` when RX uses BITSLICE_0 on UltraScale devices. |
| `kNumEncodeBits` | CDCM payload bits per `clkPar` cycle: `1` for 1.5 modes; `2` for 2.5 modes. |
| `kCbtMode` | CBT operating mode: `"Master"` or `"Slave"`. |
| `enDebug` | Enable the preset `mark_debug` attributes. |

### Clock/Reset

| Port | Direction | Description |
| --- | --- | --- |
| `srst` | In | Active-high reset for `clkPar` logic and the transceiver. |
| `pwrOnRst` | In | Active-high reset for logic driven by `clkIndep` and `clkIdelayRef`. |
| `clkSerTx` | In | Serial TX clock from BUFG: 5 x `clkPar` for CDCM-10; 4 x `clkPar` for CDCM-8. |
| `clkSerRx` | In | Serial RX clock from BUFG: 5 x `clkPar` for CDCM-10; 4 x `clkPar` for CDCM-8. |
| `clkPar` | In | Parallel clock from BUFG. |
| `clkIndep` | In | Independent clock for the clock monitor. |
| `clkIdelayRef` | In | IDELAYCTRL reference clock, independent of `clkPar`. |

### Initialization/Status

| Port | Direction | Description |
| --- | --- | --- |
| `initIn` | In | Restart initialization. Synchronous to `clkPar`. |
| `tapValueIn` | In | IDELAY tap setting used when `kFixIdelayTap` is `true`. |
| `reqIdelayShift` | In | `"00"`: no adjustment; `"01"`: increase delay by one step of `tapValueOut`; `"10"`: decrease by one step; `"11"`: reserved. |
| `reqShutOffOut` | Out | Request the upper-layer protocol to suspend communication for readjustment. |
| `shutOffAckIn` | In | Upper-layer acknowledgement of `reqShutOffOut`. |
| `delayPerTap` | Out | Delay per tap in picoseconds. |
| `cbtLaneUp` | Out | High when the CBT is ready for communication. |
| `tapValueOut` | Out | Current IDELAY tap setting. |
| `bitslipNum` | Out | Current bitslip position; also updated during CDCM-8 dynamic adjustment. |
| `serdesOffset` | Out | Signed phase offset between the input bit pattern and `clkPar` for LACCP (CDCM-8 only). |
| `firstBitPatt` | Out | ISERDES pattern captured after IDELAY adjustment, before bitslip. |
| `cntValueOutInit` | Out | Initial master delay counter value (`CNTVALUEOUT`) for UltraScale CDCM-8; zero on 7-series. |
| `cntValueOutSlaveInit` | Out | Initial slave delay counter value (`CNTVALUEOUT`) for UltraScale CDCM-8; zero on 7-series. |
| `patternErr` | Out | A received waveform violates the CDCM pattern. |
| `idelayErr` | Out | IDELAY adjustment failed. |
| `bitslipErr` | Out | SERDES bitslip failed to find the expected pattern. |
| `watchDogErr` | Out | Watchdog timeout; the other endpoint appears to be down. |

### Data I/F

| Port | Direction | Description |
| --- | --- | --- |
| `isKTypeTx` | In | TX character type: `1` for K-type; `0` for D-type. |
| `dataInTx` | In | 8-bit TX user data. |
| `validInTx` | In | High when `dataInTx` is valid; low requests an idle pattern. |
| `txBeat` | Out | Indicates the encoder character cycle. |
| `txAck` | Out | Acknowledges acceptance of valid TX data. |
| `isIdleRx` | Out | High when the received character is idle. |
| `isKTypeRx` | Out | RX character type: `1` for K-type; `0` for D-type. |
| `dataOutRx` | Out | 8-bit RX user data. |
| `validOutRx` | Out | High when `dataOutRx` is valid. |

### Physical I/F

| Port | Direction | Description |
| --- | --- | --- |
| `cdcmTxp` | Out | Positive differential TX output. Connect to the top-level port. |
| `cdcmTxn` | Out | Negative differential TX output. Connect to the top-level port. |
| `cdcmRxp` | In | Positive differential RX input. Connect to the top-level port. |
| `cdcmRxn` | In | Negative differential RX input. Connect to the top-level port. |
| `modClock` | Out | Received modulated clock. For UltraScale CDCM-8 with `kBitslice0=true`, outputs a virtual clock. |

## CBT characters

The CBT character is a 10-bit internal data format in the CBT and is generated by simply adding 2-bit type header to a 8-bit data. There are K-, D-, and T-types characters. In addition, there is a IDLE character, which consists of the modulated signal with the duty ration of 50%. The T-type characters are used to control the CBT, and they are hidden inside the CBT. K-type characters are used to control the link protocol, and their bit pattern are defined in the CBT level. D-type characters are user data. The transmission request for D- and K-type characters are exclusive because it is determined by isKTypeTx signal. However, T-type character transmission can conflict with D- and K-type characters. The transmission priority among characters is defined as **D < T < K** characters. During the T-type character transmission, D-type character transmission is held up. The link protocol needs to keep the current dataInTx until the txAck is returned.

## CDCM encode and decode

Currently, the CBT supports CDCM-10-1.5 (CDCM-8-1.5) and CDCM-10-2.5; they can transmit 1-bit binary + idle pattern and 2-bit binary + idle pattern per clkPar cycle, respectively. For details, see [Ref](https://ieeexplore.ieee.org/document/9131833) and [Ref](https://ieeexplore.ieee.org/document/10098185). Since the CBT character has 10-bit data width, 10 and 5 clock cycles are necessary to send a CBT character by CDCM-10-1.5 (CDCM-8-1.5) and CDCM-10-2.5, respectively. CDCM-10-1.5 (CDCM-8-1.5) has longer latency while it provides the better jitter performance because the duty cycle change range is narrower than that of CDCM-10-2.5.

## Initialization process

`CbtLane` initializes the receive sampling delay, the SERDES bit position, and the CBT character boundary before completing the Master/Slave handshake. The sequence depends on `kCbtMode`.

### Reset and restart conditions

`srst` resets the transceiver and the `clkPar` logic. `pwrOnRst` resets the independent-domain logic and IDELAY initialization logic. Initialization proceeds after reset and internal initialization requests are released; deasserting `initIn` alone does not immediately start the sequence because its request is stretched internally for more than `2 * kMaxWDT` cycles of `clkPar`.

| Mode | Initialization requests generated by `CbtLane` |
| --- | --- |
| `"Master"` | The stretched `initIn` request, RX clock loss detected by `ClockMonitor`, or an internal reset interval triggered by the RX quality monitor or `watchDogErr`. `ClockMonitor` uses `clkIndep`; clock loss holds the receiver in initialization until the RX clock is detected again. |
| `"Slave"` | The stretched `initIn` request or the RX quality monitor request. No `ClockMonitor` is instantiated in this mode. |

In both modes, `CbtRx` also requests reinitialization on a watchdog timeout or a handshake timeout. On 7-series devices with `genIDELAYCTRL=true`, SERDES initialization waits for IDELAYCTRL `RDY`. For UltraScale CDCM-8 automatic adjustment, it waits for the initial master and slave IDELAY counter values to be captured.

### Sampling delay and SERDES alignment

With `kFixIdelayTap=false`, the receiver scans the IDELAY settings, checks for an unchanged sampled idle pattern, and selects the center of a sufficiently wide stable region. With `kFixIdelayTap=true`, it loads `tapValueIn` and still checks sample stability before proceeding. The selected setting is reported by `tapValueOut`.

After IDELAY adjustment, the receiver captures `firstBitPatt` and performs bitslip until the sampled pattern matches `kIdleCDCM` for 16 consecutive checks (`kPattOkThreshold`). In the RTL's bit-vector order, `kIdleCDCM` is `00000_11111` for CDCM-10. CDCM-8 uses `0000_1111`, padded with a leading `0` and a trailing `1` for the same internal 10-bit comparison. Completion of IDELAY adjustment and bitslip sets the internal status to `kInitFinish`; this does not yet assert `cbtLaneUp`.

### Master/Slave handshake

The receiver state machine controls its local transmitter through the internal back channel. `SendZero` disables transmission by sending an all-zero pattern, `SendIdle` sends the idle pattern, and `SendInitPattern` alternates `kInitPCDCM` and `kInitMCDCM`. The initialization T-type characters I1 and I2 are selected by `GetInit1Char(kNumEncodeBits)` and `GetInit2Char(kNumEncodeBits)`.

| Stage | Master | Slave |
| --- | --- | --- |
| Local SERDES initialization | Sends idle until its local status reaches `kInitFinish`, then sends the initialization waveform. | Sends zero until its local status reaches `kInitFinish`, then sends idle. |
| Initialization waveform detection | Counts 16 received `kInitPCDCM` or `kInitMCDCM` patterns after `kInitFinish`, then sends I1. | Counts the same 16 patterns, then sends the initialization waveform. |
| CBT character alignment | Sends I1 and enables decoder alignment. Once aligned, sends I2. | Sends the initialization waveform and enables decoder alignment. Once aligned, sends I1. |
| Readiness confirmation | Counts eight valid received I2 characters, then enters `StateCbtRxUp`. | On the first valid received I2, starts sending I2. It then counts eight valid received characters other than I2 before entering `StateCbtRxUp`. |

`CdcmRxDecoder` aligns the CBT character boundary to I1 separately from the SERDES bitslip. The initialization waveform and readiness counters accumulate matches; they do not require consecutive matches. Initialization T-type characters are handled internally, and user TX data is not acknowledged during these handshake stages.

`CbtRx` asserts its internal RX-up status in `StateCbtRxUp`; `CbtTx` asserts TX-up on an encoder beat in that state. `cbtLaneUp` is asserted when both are high and their conjunction has propagated through the 128-stage `clkPar` delay pipeline (`kDelayLaneUp`).

If IDELAY adjustment or bitslip fails, `idelayErr` or `bitslipErr` is asserted and the corresponding adjustment state machine retries. A handshake timeout causes `CbtRx` to restart initialization.

## Error detection

When cbtLaneUp is high, the CBT checks whether the sampled bit pattern is matched with the CDCM encoding rule or not. If a broken pattern is detected, the patternErr signal is asserted, but at this moment, the CBT is not reset. If **more than 1%** of received bit pattern are broken, the RX quality check monitor requests to reset the CBT.

When cbtLaneUp is high, the CBT transmits the T-type character, dogfood character, periodically. The dogfood character resets the watch dog timer in other side. If the watch dog timer can't eat dogfood within specified time, the watchDogErr goes high, and the watchdog timer requests to reset the CBT.
