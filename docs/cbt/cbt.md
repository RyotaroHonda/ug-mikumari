# CDCM based transceiver (CBT)

The CBT is a physical layer of the MIKUMARI system. About the CBT, see also [Ref](https://ieeexplore.ieee.org/document/10098185). The roles of the CBT are as follows.

- Encode and decode the CDCM modulated waveform pattern
- Defines three types of CBT character using sign extension from 8-bit to 10-bit
- Initialize IOSERDES when detecting the cable (fiber) connection

Currently, the CBT supports the CDCM-10-2.5, CDCM-10-1.5, CDCM-8-2.5, and CDCM-8-1.5. Then, the frequency ratios between the serial and parallel clock signals are 5 for CDCM-10-XX and 4 for CDCM-8-XX with the double-data-rate (DDR) mode, respectively. The SDR mode is not supported. In addition, only the differential signal is supported.
**If you want to use the Local Area Common Clock Protocol (LACCP), you need to select CDCM-8-XX modes. The serdesOffset port does not work with CDCM-10-XX modes.**

## Interface of CBT

The top-level block of the CBT is **CbtLane**. The global parameters for the CBT are defined in ``defCDCM.vhd``. The CbtLane's entity port structure is as follows.

```VHDL
entity CbtLane is
  generic
    (
      kFamily          : string;  -- US: UltraScale devices, 7S: 7-Series devices
      -- CDCM-Mod-Pattern --
      kCdcmModWidth    : integer; -- # of time slices of the CDCM signal
      -- CDCM-TX --
      kIoStandardTx    : string;  -- IO standard of OBUFDS
      kTxPolarity      : boolean; -- If true, inverse Tx polarity
      -- CDCM-RX --
      genIDELAYCTRL    : boolean; -- If TRUE, IDELAYCTRL is instantiated.
      kDiffTerm        : boolean; -- IBUF DIFF_TERM
      kRxPolarity      : boolean; -- If true, inverts Rx polarity
      kIoStandardRx    : string;  -- IOSTANDARD of IBUFDS
      kIoDelayGroup    : string;  -- IODELAY_GROUP for IDELAYCTRL and IDELAY
      kFixIdelayTap    : boolean; -- If TRUE, value on tapValueIn is set to IDELAY
      kFreqFastClk     : real;    -- Frequency of SERDES fast clock (MHz).
      kFreqRefClk      : real;    -- Frequency of refclk for IDELAYCTRL (MHz).
      kBitslice0       : boolean; -- If the RX side is routed to the IO pad on the Bitslice0, set TRUE.
      -- Encoder/Decoder
      kNumEncodeBits   : integer:= 2;  -- 1:CDCM-10-1.5 or 2:CDCM-10-2.5
      -- Master/Slave
      kCbtMode         : string;
      -- DEBUG --
      enDebug          : boolean:= false
    );
  port
    (
      -- SYSTEM port --
      srst          : in std_logic; -- Reset logics driven by clkPar. Transceiver function reset. (active high)
      pwrOnRst      : in std_logic; -- Reset logics driven by clkIndep and clkIdelayRef. (active high)
      clkSerTx      : in std_logic; -- From BUFG (driving serial data line)
      clkSerRx      : in std_logic; -- From BUFG (driving serial data line)
      clkPar        : in std_logic; -- From BUFG
      clkIndep      : in std_logic; -- Independent clock for monitor
      clkIdelayRef  : in std_logic; -- REFCLK input for IDELAYCTRL. Must be independent from clkPar.
      initIn        : in std_logic; -- Re-do the initialization process. Sync with clkPar.
      tapValueIn    : in std_logic_vector(kWidthTap-1 downto 0); -- IDELAY TAP value input (active when kFixIdelayTap is true)

      -- Dynamic Control --
      reqIdelayShift : in std_logic_vector(kReqIdelayShiftBits-1 downto 0); -- "00": No shift. "01": Shift plus by 1 bit. "10": Shift minus by 1 bit. "11": Reserved.
      reqShutOffOut : out std_logic; -- Request signal to the upper-layer protocol to shutoff communication.
      shutOffAckIn  : in std_logic; -- Acknowledge signal from the upper-layer protocol for the shutoff request.

      -- Status --
      cbtLaneUp     : out std_logic; -- Indicates that CBT is ready for communication
      tapValueOut   : out std_logic_vector(kWidthTap-1 downto 0); -- IDELAY TAP value output
      bitslipNum    : out std_logic_vector(kWidthBitSlipNum-1 downto 0); -- Number of bitslip made
      serdesOffset  : out signed(kWidthSerdesOffset-1 downto 0);
      firstBitPatt  : out CdcmPatternType; -- ISERDES output pattern after finishing the idelay adjustment
      cntValueOutInit : out std_logic_vector(kCNTVALUEbit-1 downto 0);       -- Initial value of IDELAYE3 of the master side
      cntValueOutSlaveInit : out std_logic_vector(kCNTVALUEbit-1 downto 0);  -- Initial value of IDELAYE3 of the slave side
      dataOutRxShift : out std_logic_vector(kDataOutRxShiftBits-1 downto 0);
      delayPerTap   : out std_logic_vector(kBitDelayPerTap-1 downto 0);

      -- Error --
      patternErr    : out std_logic; -- Indicates CDCM waveform pattern is collapsed.
      idelayErr     : out std_logic; -- Attempted bitslip but the expected pattern was not found.
      bitslipErr    : out std_logic; -- Bit pattern which does not match the CDCM rule is detected.
      watchDogErr   : out std_logic; -- Watch dog can't eat dogfood within specified time. The other side seems to be down.

      -- Data I/F --
      isKTypeTx     : in std_logic; -- 1: Generate a K type character. 0: D type character.
      dataInTx      : in CbtUDataType;
      validInTx     : in std_logic; -- 1: charIn is valid. Encode and send it to CDCM-TX.
                                    -- 0: Send idle pattern;
      txBeat        : out std_logic; -- Indicates encoder cycle.
      txAck         : out std_logic; -- Acknowledge to validInTx.

      isIdleRx      : out std_logic; -- Indicates present character is idle.
      isKTypeRx     : out std_logic; -- 1: K type character. 0: D type character.
      dataOutRx     : out CbtUDataType;
      validOutRx    : out std_logic; -- 1: charOut is valid.

      -- CDCM ports --
      cdcmTxp       : out std_logic; -- Connect to TOPLEVEL port
      cdcmTxn       : out std_logic; -- Connect to TOPLEVEL port
      cdcmRxp       : in std_logic;  -- Connect to TOPLEVEL port
      cdcmRxn       : in std_logic;  -- Connect to TOPLEVEL port
      modClock      : out std_logic  -- CDCM modulated clock.

    );
end CbtLane;
```

<table class="vmgr-table">
  <thead><tr>
    <th class="nowrap"><span class="mgr-10">Port </span></th>
    <th class="nowrap"><span class="mgr-10">In/Out</span></th>
    <th class="nowrap"><span class="mgr-10">Comment</span></th>
  </tr></thead>
  <tbody>
  <tr><td class="tcenter" colspan=4><b>Generic port</b></td></tr>
  <tr>
    <td>kFamily</td>
    <td class="tcenter">-</td>
    <td>Select US or 7S for UltraScale devices and 7-Series devices, respectively.</td>
  </tr>
  <tr>
    <td>kCdcmModWidth</td>
    <td class="tcenter">-</td>
    <td>Select 8 or 10 for CDCM-8-XX and CDCM-10-XX, respectively.</td>
  </tr>
  <tr>
    <td>kIoStandardTx</td>
    <td class="tcenter">-</td>
    <td>Tx port IO standard, e.g., LVDS.</td>
  </tr>
  <tr>
    <td>kTxPolarity</td>
    <td class="tcenter">-</td>
    <td>If it's true, the TX signal polarity is reversed. Use it when the differential signal p/n connection is inverse on FEE.</td>
  </tr>
  <tr>
    <td>genIDELAYCTRL</td>
    <td class="tcenter">-</td>
    <td>If it's true, the IDELAYCTRL primitive is instantiated in the CBT.</td>
  </tr>
  <tr>
    <td>kDiffTerm</td>
    <td class="tcenter">-</td>
    <td>Enable the internal 100-ohm termination register in FPGA.</td>
  </tr>
  <tr>
    <td>kRxPolarity</td>
    <td class="tcenter">-</td>
    <td>If it's true, the RX signal polarity is reversed. Use it when the differential signal p/n connection is inverse on FEE.</td>
  </tr>
  <tr>
    <td>kIoDelayGroup</td>
    <td class="tcenter">-</td>
    <td>Set the IODELAY_GROUP constraint to the IDELAYCTRL and IDELAYE2 primitives in the CBT.</td>
  </tr>
  <tr>
    <td>kFixIdelayTap</td>
    <td class="tcenter">-</td>
    <td>If it is true, the automatic IDELAY tap adjustment function is disabled. Instead, the value on tapValueIn is set to IDELAY.</td>
  </tr>
  <tr>
    <td>kFreqFastClk</td>
    <td class="tcenter">-</td>
    <td>The frequency value of the serial clock. It is used to determine the tap number for adjusting IDELAYE2</td>
  </tr>
  <tr>
    <td>kFreqRefClk</td>
    <td class="tcenter">-</td>
    <td>The frequency value for IDELAYCTRL. It is used to determine the tap number for adjusting IDELAYE2</td>
  </tr>
  <tr>
    <td>kBitslice0</td>
    <td class="tcenter">-</td>
    <td>Valid only on UltraScale devices. Set to TRUE if the RX is connected to an I/O on bitslice 0. Always set to FALSE for 7-series devices.</td>
  </tr>
  <tr>
    <td>kNumEncodeBits</td>
    <td class="tcenter">-</td>
    <td>Set the payload size of the CDCM signal. Set 1 or 2. 1: CDCM-10-1.5. 2: CDCM-10-2.5. Currently, CDCM-8-2.5 is not supported.</td>
  </tr>
    <td>kCbtMode</td>
    <td class="tcenter">-</td>
    <td>Set Master or Slave. The CBT runs with the designated mode.</td>
  </tr>
  </tr>
    <td>enDebug</td>
    <td class="tcenter">-</td>
    <td>Enable preset mark_debug constraints. The debug core will be implemented.</td>
  </tr>
  <tr><td class="tcenter" colspan=4><b>IO port</b></td></tr>
  <tr><td colspan=4><b>System port</b></td></tr>
  <tr>
    <td>srst</td>
    <td class="tcenter">In</td>
    <td>Asynchronous assert, synchronous de-assert reset. Reset logics driven by clkPar. It is expected that this reset comes from the PLL lock signal generating clkPar and clkSer signals. (active high) </td>
  </tr>
  <tr>
    <td>pwrOnRst</td>
    <td class="tcenter">In</td>
    <td>This signal resets logics driven by clkIndep and clkIdelayRef. It is expected that this signal goes high once after power on, and thus it is generated by the PLL lock signal generating clkIndep and clkIdelayRef. (active high) </td>
  </tr>
  <tr>
    <td>clkSerTX</td>
    <td class="tcenter">In</td>
    <td>Serial clock input for TX. The clock skew must be adjusted between clkSer and clkPar. For 7-series devices, this signal should be identical with clkSerRX. </td>
  </tr>
  <tr>
    <td>clkSerRX</td>
    <td class="tcenter">In</td>
    <td>Serial clock input for RX. The clock skew must be adjusted between clkSer and clkPar. For 7-series devices, this signal should be identical with clkSerTX. </td>
  </tr>
  <tr>
    <td>clkPar</td>
    <td class="tcenter">In</td>
    <td>Parallel clock input. The clock skew must be adjusted between clkSer and clkPar. Status, error, and Data I/F ports are synchronized with this clock.</td>
  </tr>
  <tr>
    <td>clkIndep</td>
    <td class="tcenter">In</td>
    <td>The independent clock from the clkPar. Its clock frequency should be clkPar < clkIndep < 2*clkPar. If the frequency is exactly twice of clkPar, you could get a trouble.</td>
  </tr>
  <tr>
    <td>clkIdelayRef</td>
    <td class="tcenter">In</td>
    <td>REFCLK input for IDELAYCTRL. It must be independent from clkPar.</td>
  </tr>x
  <tr>
    <td>initIn</td>
    <td class="tcenter">In</td>
    <td>If it is high, the CBT re-do the initialization process. This signal must be synchronized with clkPar.</td>
  </tr>
  <tr>
    <td>tapValueIn</td>
    <td class="tcenter">In</td>
    <td>IDELAY TAP value input. This port is active if kFixIdelayTap is true.</td>
  </tr>
  <tr><td colspan=4><b>Dynamic Control port</b></td></tr>
  <tr>
    <td>cbtLaneUp</td>
    <td class="tcenter">Out</td>
    <td>This goes high when the CBT becomes ready for communication after finishing the initialization process. </td>
  </tr>
  <tr>
    <td>tapValueOut</td>
    <td class="tcenter">Out</td>
    <td>The IDELAY tap value currently used. </td>
  </tr>
  <tr>
    <td>bitslipNum</td>
    <td class="tcenter">Out</td>
    <td>The number indicating how many bitslip is made in the initialization process.</td>
  </tr>
  <tr>
    <td>serdesOffset</td>
    <td class="tcenter">Out</td>
    <td>The port provides the value indicating the phase difference between the input bit patter and the ClkPar signal. This is the special port for LACCP, not for users. This port works only with CDCM-8-XX modes. </td>
  </tr>
  <tr>
    <td>firstBitPatt</td>
    <td class="tcenter">Out</td>
    <td>This port provides the bit pattern before starting the bit-slip process. This is for debug, not for users. </td>
  </tr>
  <tr>
    <td>patternErr</td>
    <td class="tcenter">Out</td>
    <td>This goes high when the waveform, which is not matched with the CDCM modulation pattern, is detected. Data is broken. </td>
  </tr>
  <tr>
    <td>idelayErr</td>
    <td class="tcenter">Out</td>
    <td>This goes high if the appropriate tap value, which provides the stable communication, was not found. </td>
  </tr>
  <tr>
    <td>bitslipErr</td>
    <td class="tcenter">Out</td>
    <td>This goes high when the reference bit pattern cannot be detected during the initialization process. Re-initialization is necessary.</td>
  </tr>
  <tr>
    <td>watchDotErr</td>
    <td class="tcenter">Out</td>
    <td>This goes high when the watch dog timer can't eat dogfood within specified time. The other side link seems to be down.</td>
  </tr>
  <tr>
    <td>isKTypeTx</td>
    <td class="tcenter">In</td>
    <td>If this is high, the current TX data is translated to a K-type character. If low, the TX data becomes a D-type character.</td>
  </tr>
  <tr>
    <td>dataInTx</td>
    <td class="tcenter">In</td>
    <td>8-bit TX data.</td>
  </tr>
  <tr>
    <td>validInTx</td>
    <td class="tcenter">In</td>
    <td>It denotes that the current dataInTx is valid. It is the request for the CBT to transmit it.</td>
  </tr>
  <tr>
    <td>txBeat</td>
    <td class="tcenter">Out</td>
    <td>This signal goes high once per a CBT character transfer cycle for one clock cycle. It indicates a boundary of the transfer cycle.</td>
  </tr>
  <tr>
    <td>txAck</td>
    <td class="tcenter">Out</td>
    <td>The acknowledge signal respect to validInTx. This goes high when the dataInTx is latched at the same timing of the txBeat.</td>
  </tr>
  <tr>
    <td>isIdleRx</td>
    <td class="tcenter">Out</td>
    <td>This goes high when the current dataOutRx is a idle data.</td>
  </tr>
  <tr>
    <td>isKTypeRx</td>
    <td class="tcenter">Out</td>
    <td>This goes high when the current dataOutRx is a K-Type data.</td>
  </tr>
  <tr>
    <td>dataOutRx</td>
    <td class="tcenter">Out</td>
    <td>8-bit RX data.</td>
  </tr>
  <tr>
    <td>validOutRx</td>
    <td class="tcenter">Out</td>
    <td>This goes high when the current dataOutRX valid both for D- and K-types data.</td>
  </tr>
  <tr>
    <td>cdcmTxp</td>
    <td class="tcenter">Out</td>
    <td>Transmission line positive. Connect to the toplevel port.</td>
  </tr>
  <tr>
    <td>cdcmTxn</td>
    <td class="tcenter">Out</td>
    <td>Transmission line negative. Connect to the toplevel port.</td>
  </tr>
  <tr>
    <td>cdcmRxp</td>
    <td class="tcenter">In</td>
    <td>Receive line positive. Connect to the toplevel port.</td>
  </tr>
  <tr>
    <td>cdcmRxn</td>
    <td class="tcenter">In</td>
    <td>Receive line negative. Connect to the toplevel port.</td>
  </tr>
  <tr>
    <td>modClock</td>
    <td class="tcenter">Out</td>
    <td>The CDCM modulated clock output. It is valid in the secondary mode.</td>
  </tr>
</tbody>
</table>

## CBT characters

The CBT character is a 10-bit internal data format in the CBT and is generated by simply adding 2-bit type header to a 8-bit data. There are K-, D-, and T-types characters. In addition, there is a IDLE character, which consists of the modulated signal with the duty ration of 50%. The T-type characters are used to control the CBT, and they are hidden inside the CBT. K-type characters are used to control the link protocol, and their bit pattern are defined in the CBT level. D-type characters are user data. The transmission request for D- and K-type characters are exclusive because it is determined by isKTypeTx signal. However, T-type character transmission can conflict with D- and K-type characters. The transmission priority among characters is defined as **D < T < K** characters. During the T-type character transmission, D-type character transmission is held up. The link protocol needs to keep the current dataInTx until the txAck is returned.

## CDCM encode and decode

Currently, the CBT supports CDCM-10-1.5 (CDCM-8-1.5) and CDCM-10-2.5; they can transmit 1-bit binary + idle pattern and 2-bit binary + idle pattern per clkPar cycle, respectively. For details, see [Ref](https://ieeexplore.ieee.org/document/9131833) and [Ref](https://ieeexplore.ieee.org/document/10098185). Since the CBT character has 10-bit data width, 10 and 5 clock cycles are necessary to send a CBT character by CDCM-10-1.5 (CDCM-8-1.5) and CDCM-10-2.5, respectively. CDCM-10-1.5 (CDCM-8-1.5) has longer latency while it provides the better jitter performance because the duty cycle change range is narrower than that of CDCM-10-2.5.

## Initialization process

The CBT starts the initialization process when the following conditions are met.

- srst and initIn are low.
- The clock monitor detects a clock like signal in the RX signal.

The IDELAYE2 tap number is adjusted so as to stabilize the sampled data using the idle pattern, and bitslip is performed so as to reproduce the bit pattern of ``0b11111_00000``. If kFixIdealyTap is true, the value on tapValueIn is set to IDEALY. After initializing IOSERDES, some T-type characters are exchanged to confirm that both end points are actually ready for communication each other. Then, the cbtLaneUp is asserted.

## Error detection

When cbtLaneUp is high, the CBT checks whether the sampled bit pattern is matched with the CDCM encoding rule or not. If a broken pattern is detected, the patternErr signal is asserted, but at this moment, the CBT is not reset. If **more than 1%** of received bit pattern are broken, the RX quality check monitor requests to reset the CBT.

When cbtLaneUp is high, the CBT transmits the T-type character, dogfood character, periodically. The dogfood character resets the watch dog timer in other side. If the watch dog timer can't eat dogfood within specified time, the watchDogErr goes high, and the watchdog timer requests to reset the CBT.
