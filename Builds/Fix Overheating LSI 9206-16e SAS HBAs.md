# Fixing overheating LSI 9206-16e SAS HBA PCIe cards

## TL:DR

The thermal paste dried up, the card was overheating and would freeze the system or disappear from the device list. I opted to go all out, and replaced it with PTM7950. The card may still overheat after a period of time so a 120mm fan placed directly beneath it spinning between 750-1200rpm is sufficient to keep the highest temperature of the 3 ICs below 55°C. There does not appear to be any noticable increase of thermal output when the card is idle or active, even with 16 hard drives reading simultaneously.

Due to the card not supporting ASPM (blame could be 50/50 - the `mpt3sas` linux kernel module disables ASPM for MPI 2.0 cards, but there also appears to be a mismatch in supported ASPM levels between the SAS2308 controller and PLX PCIe switch chip in use), it does increase idle system power consumption by over 30 watts in my testing. I have not been able to test the card in a PCH/chipset connected PCIe slot, only ones connected directly to the CPU which is definitely a factor for the additional power consumption.

For a full `lspci -vvv` output, find the section talking about trying to forcefully enable ASPM.

---

## Story time

My dad a couple years back acquired two LSI 9206-16e HBA cards, seemingly from a Dell server of sorts, as they are marked with the Dell part number `0TFJRW` (sticker on the opposite side of the ICs). The firmware originally shipped in IR mode, but he was able to update the firmware on both ICs, both to IT mode, and the P20 release. In his use though, even after idling in a system for 5-10 minutes, the whole system would lock up, sometimes powering off and/or rebooting immediately after. It was determined that the cards were overheating, and the first idea was to attach two fans directly to the heatsink. A StarTech 60mm (10mm depth) fan was attached to cool SAS2308 controller 1 (left side) and the PLX chip, with a Noctua 40mm (10mm depth) to cool the second, right side SAS2308 controller. Both fans are 3-pin, connected to a molex to 3 pin fan adapter, in a means to attempt to keep temperatures at bay. Note the use of "attempt", since it made no difference. 

According to [LSI's user guide](https://docs.broadcom.com/doc/12353328), under section 5.6, "Thermal and Atmospheric Limits", the minimum airflow listed is "100 linear feet per minute at 35 °C (95 °F) bay inlet temperature". While the house never reaches those temperatures, with this card being used in a traditional ATX computer case, the effective intake temperatures may be much higher. But also, I am quite sure that both fans combined are not meeting the 100 linear feet per minute airflow rating. It was at this point my dad lost interest in utilizing these cards, and ended up placing them in a drawer.

(insert photo)

## My turn

Really the only information I had going into this was that when measuring the backside of the PCB (opposite of where the ICs are located), it got quite hot (no, neither the value or units were recorded), but no clue about the heatsinks.

My first thought was to verify thermal dissapation, so I connected the card to a system, and let it sit in the BIOS while I waited for the card to warm up. Within 2 minutes the PCB itself became almost too hot to touch with a bare finger, meanwhile the heatsinks remained cool to rest a hand on for a seemingly infinite amount of time. Mind you, both fans mentioned earlier were spinning at full speed from cold boot. From this, I immediately suspected the thermal paste.

(insert photo)

### Side note

At this point, I was unware that `lsiutil` could read the temperature for SAS2308 platform cards.

Given the paste being fully dried up, it makes sense that thermal transfer efficiency was next to nothing. While replacing the paste with quite possibly anything would have resolved the issue, I wanted to go all out for the sake of curiosity. At the time of writing, I have had good results with PTM7950 (plenty but far too many to list here frankly), but only with higher TDP computer systems. Stepping down to something more compact I thought would be an interesting case to investigate.

(insert photo)

Both the PLX and SAS2308 controller measure out to about 12mm on both sides, on the highest surface. I ended up cutting out a 10mm*10mm piece of PTM7950 since it would be a neater cut into my large sheet (sourced from Linus Tech Tips). It will spread out after enough heat cycles and should be effective. Probably? Ultimately it is still better then what was on there before.

(insert photo)

### Test time (without fans)

Before booting up the system with the card I took the time to figure out how to measure the temperature of the cards, since I figured going off burning my finger is not a very useful metric. Thank you to `Curious-Region7488` on Reddit for an in-depth explanation such as [this](https://www.reddit.com/r/homelab/comments/cqvxeb/comment/kj5mgg4/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button):

<details>
<summary>Archive copy of the comment</summary>
<br>
I know this is an old post, but I thought I'd put this here in case anyone searches for how to get the temperature of an LSI SAS HBA that's been flashed to IT mode. As many have discovered, a lot of the most popular tools (`megacli`, `storcli`, `sas2flash`, etc.) are designed for the HBA in RAID mode and are either nonfunctional or very limited when it is in IT mode. However, I found the `lsiutil` command does work.

You can get the `lsiutil` binary from [here](https://github.com/thomaslovell/LSIUtil/) and the documentation from [here](https://www.thomas-krenn.com/de/wikiDE/images/4/44/Lsi_userguide_2006_20130528.pdf) (as of January 2024).

This is the command I use:

`echo $(( 16#$( lsiutil.x86_64 -p1 -a 25,2,0,0 | grep IOCTemperature: | cut -dx -f2 ) ))`

As an explanation:

- `-p1` uses the first HBA found (run lsiutil without options to find the proper port number to use).

- `-a` tells `lsiutil` that the following numbers are commands in sequence (run `lsiutil` with a `-e` option to get the full list of available commands)

- `25,2,0,0` tells `lsiutil` to go to the Power Management menu, display the IOUnit config page, and then exit back to the command prompt

- the `grep` and `cut` commands extract the temperature from the output

- the `$(( 16#... ))` part converts the output from hexadecimal to decimal.

The result of the above command is the HBA temperature in Celsius degrees.
</details>

---

With an open air testbench system ready, I installed `lsiutil` from the AUR (I use Arch~~install~~ btw), added the card to the primary PCIe x16 slot, and ran the command above (just without the `.x86_64` in the executable name)

```bash
[sam@GigabyteZ270X ~]$ echo $(( 16#$( lsiutil -p1 -a 25,2,0,0 | grep IOCTemperature: | cut -dx -f2 ) ))
72
```

That is not too bad considering there is no active airflow, for a card intended to be used in a server chassis. How about the second SAS2308?

```bash
[sam@GigabyteZ270X ~]$ echo $(( 16#$( lsiutil -p2 -a 25,2,0,0 | grep IOCTemperature: | cut -dx -f2 ) ))
94
```

Oh.

As a sanity check, I re-ran the command. `96` now, and in that time the first controller warmed up to `73`. Out of comedic effect I could have tried huffing and puffing on the cards to see if that made a difference, but I ended up putting my finger on the heatsink to see if the measurement seemed reasonable. Ouch. It sure was. I quickly scrambled to grab the StarTech 60mm fan, connect it to a spare fan header on the motherboard, and placed it in the middle of the three heatsinks on the card, hoping it would spin fast enough to at least prevent temperatures from climbing.

While 2300rpm for a slim 60mm fan is not a lot (especially considering the fan being capable of up to around 4000rpm), it was enough to significantly reduce the temperature over the next couple minutes, down to 56 and 64 celsisus respectively. Seems like the PTM7950 is definitely doing something, especially considering the system has been powered on for more than 10 minutes, and issues would normally be noticable around this time. I opted to let this sit for an extra 10 minutes to see if the temperatures would further decrease or if the system would become unstable. Neither happened, which ultimately is acceptable to me. The temperature of the card is under control, and the system is running stable (albeit idle).

### So we need active cooling

But how much of it? Does the card need extra cooling when all 16 drives are actively transfering data? (spoiler - not really) Does the card need active cooling for the power delievery system? (probably not) What fan size should be used? How fast does the fan need to spin?

Plenty of questions, many options. My first consideration was to hot glue the fans back onto the heatsink, skipping the self-tapping screws. This ultimately failed when the hot glue would effortlessly peel off the heatsink and drop the fans when oriented heatsink side down.

It then occured to me that the card is going to be used in an ATX computer case, which means there should be plenty of PCI card slots to fill up. In addition, on a full size ATX motherboard, there is usually not more then a x1 slot directly below the primary x16 slot, so a fan should be of minimal obstruction to extra expansion cards. After some searching, I found a [120mm PCI fan mount by BridgportPCRepair on Thingiverse](https://www.thingiverse.com/thing:5756765) that should be suitable. I chose a 120mm fan since I have plenty spare to swap and test, and should be able to fit in almost any case with full height PCI slots.

One 3D print later and the fan is ready to mount.

(insert photo)

While the photo shows the fan blowing air upward onto the card, that was not the first test I did for the card. The case for this test system (Antec 300) does not have a bottom PSU intake. Due to this, the power supply must pull air from inside the case like every other component. I thought that I could orient the fan for the HBA such that it pulls air from above, blow it over the HBA, and then down into the PSU to exhaust.

### First big test - 16 drive simultaneous write

With a 72°F (22.2°C) ambient test environment, and eight idle (spinning but no I/O activity) hard drives (the other eight were in a different case), the card was idling at ~49°C on the first controller, and ~60°C on the second, with the designated fan spinning at ~1045rpm. This remained steady for over two and a half hours, until I began the write test.

The idea behind the write test was to write to all 16 hard drives simultaneously using `dd`. A separate process in a separate terminal window for each drive, would input from `/dev/urandom` and write to the respective block device of each drive, `/dev/disk/by-id/ata-...`, using a block size of `100M`. The drives were ATA secure erased before hand, less out of concerns for security of previous data, but instead to ensure the drives are as close to a factory state as possible, to avoid any performance degredation.

There ended up being a CPU bottleneck for the first portion of the test, but since hard drives typically read faster then write, this was meant to be means of "testing the waters".

(insert image)

The graphs do make it difficult to discern, but the HBA fan did ramp to a highest of ~1100rpm, while overall the HBA temperature remained largely unchanged. The first controller essentially held steady at 50°C, while the second started around ~58°C, peaking at the highest of 61°C, and then cooling down again by the end of all the drives writing.

### The final test - 16 drive simultaneous read

Since the write test was a CPU bottleneck for almost half the test, the read test should be a better maximum throughput test, in hopes of trying to stress the HBA as much as possible. Given the temperature results of the first, I was not expecting anything different (spoiler - there really was not), but still wanted to test it to rule out the possibility. Since I wanted this test to repeat, and for an infinite duration until interviened, I created a simple loop to read the entire length of each disk individually.

```bash
while true; do dd if=/dev/disk/by-id/ata-... of=/dev/null bs=100M status=progress iflag=direct; done
```

I kept the block size at 100M to minimize the amount of idle I/O time, and told `dd` to use direct input from the disk to prevent any RAM read caching that could occur (unlikely given the situation, but I wanted to prevent the possibility).

I gave up after about 18 hours since one of the Seagate 2TB drives began to mark 48 sectors as pending reallocation (how funny because it took multiple read passes for it to spontaneously decide that), and the temperature of the controller card was again, unchanged, with the fan ramping up only slightly, likely a result of the extra heat being dissipated from the hard drives.

(insert image)

So no more tests right?

### Active cooling part 2: change fan direction

As mentioned, both of the tests were performed with the fan trying to pull air over the HBA before sending it to the bottom of the case to have it exhausted out by the power supply. What if we inverse the direction of the fan, and have it pull air (from wherever), and push it onto the card? Also swap the power supply fan to the bottom because why not.

Since there seems to be no significant increase in thermal dissipation requirements for the HBA whether drives are connected or not, I opted to save the power and unplug all eight drives in the case.

#### NOTE: Yes, the Antec 300 does NOT have a bottom intake for the power supply, but NO it is not being suffocated.

The power supply does sit slightly raised up from the floor of the case, such that the fan should still be able to pull in some air. While not shown in the graph, power supply temperatures were checked in regular intervals, and as expected, there were no concerns.

(insert image)

Wait that is better, a LOT better. 48°C at the lowest on the second controller? (and a lowest of 41°C for the first controller) With the fan was only between ~750-800rpm? Quite impressive. Could it be lowered extra though?

### ASPM

Nope. Makes no difference to temperature, and only lowers power consumption measured from the wall by 1 watt. [Might even make the card unstable](https://z8.re/blog/aspm#:~:text=with%20ASPM%20support%3F-,LSI%209211%2D8i,chipsets%20listed%20in%20mpi2_cnfg.h%20ASPM%20will%20explicitly%20be%20disabled.,-So%20unless%20I) it seems, at least in Linux. Without restating much of what is already known, ASPM was disabled for all MPI 2.0 cards in the Linux upstream `mpt3sas` driver since "back in 2013 someone filed a bug report stating that their system would constantly lock up during high read workloads.' There were a lot of reports, and after enough testing, just as many people realized disabling ASPM would entirely resolve the issue. Throwing caution into the wind, what about forcefully enabling ASPM?

If you read the first two sentences of the previous paragraph, you have your answer. I did also verify this on a system with "Native ASPM" (OS-managed ASPM rather then BIOS negotiated), to see if that was an influencial factor. Nope.

Just for the sake of it, 

<details>
<summary>here is the output of `lspci -vvv`.</summary>
<br>

```
00:01.0 PCI bridge: Intel Corporation 6th-10th Gen Core Processor PCIe Controller (x16) (rev 07) (prog-if 00 [Normal decode])
        Subsystem: Gigabyte Technology Co., Ltd Device 5000
        Control: I/O+ Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx+
        Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
        Latency: 0, Cache Line Size: 64 bytes
        Interrupt: pin A routed to IRQ 120
        Bus: primary=00, secondary=01, subordinate=05, sec-latency=0
        I/O behind bridge: d000-efff [size=8K] [16-bit]
        Memory behind bridge: ef000000-ef4fffff [size=5M] [32-bit]
        Prefetchable memory behind bridge: c0000000-c05fffff [size=6M] [32-bit]
        Secondary status: 66MHz- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- <SERR- <PERR-
        BridgeCtl: Parity- SERR+ NoISA- VGA- VGA16+ MAbort- >Reset- FastB2B-
                PriDiscTmr- SecDiscTmr- DiscTmrStat- DiscTmrSERREn-
        Capabilities: [88] Subsystem: Gigabyte Technology Co., Ltd Device 5000
        Capabilities: [80] Power Management version 3
                Flags: PMEClk- DSI- D1- D2- AuxCurrent=0mA PME(D0+,D1-,D2-,D3hot+,D3cold+)
                Status: D0 NoSoftRst+ PME-Enable- DSel=0 DScale=0 PME-
        Capabilities: [90] MSI: Enable+ Count=1/1 Maskable- 64bit-
                Address: fee00238  Data: 0000
        Capabilities: [a0] Express (v2) Root Port (Slot+), IntMsgNum 0
                DevCap: MaxPayload 256 bytes, PhantFunc 0
                        ExtTag- RBE+ TEE-IO-
                DevCtl: CorrErr- NonFatalErr- FatalErr- UnsupReq-
                        RlxdOrd- ExtTag- PhantFunc- AuxPwr- NoSnoop-
                        MaxPayload 256 bytes, MaxReadReq 128 bytes
                DevSta: CorrErr- NonFatalErr- FatalErr- UnsupReq- AuxPwr- TransPend-
                LnkCap: Port #2, Speed 8GT/s, Width x16, ASPM L0s L1, Exit Latency L0s <1us, L1 <8us
                        ClockPM- Surprise- LLActRep- BwNot+ ASPMOptComp+
                LnkCtl: ASPM Disabled; RCB 64 bytes, LnkDisable- CommClk-
                        ExtSynch- ClockPM- AutWidDis- BWInt+ AutBWInt+
                LnkSta: Speed 8GT/s, Width x8
                        TrErr- Train- SlotClk+ DLActive- BWMgmt- ABWMgmt-
                SltCap: AttnBtn- PwrCtrl- MRL- AttnInd- PwrInd- HotPlug- Surprise-
                        Slot #1, PowerLimit 75W; Interlock- NoCompl+
                SltCtl: Enable: AttnBtn- PwrFlt- MRL- PresDet- CmdCplt- HPIrq- LinkChg-
                        Control: AttnInd Unknown, PwrInd Unknown, Power- Interlock-
                SltSta: Status: AttnBtn- PowerFlt- MRL- CmdCplt- PresDet+ Interlock-
                        Changed: MRL- PresDet+ LinkState-
                RootCap: CRSVisible-
                RootCtl: ErrCorrectable- ErrNon-Fatal- ErrFatal- PMEIntEna- CRSVisible-
                RootSta: PME ReqID 0000, PMEStatus- PMEPending-
                DevCap2: Completion Timeout: Not Supported, TimeoutDis- NROPrPrP- LTR+
                         10BitTagComp- 10BitTagReq- OBFF Via WAKE#, ExtFmt- EETLPPrefix-
                         EmergencyPowerReduction Not Supported, EmergencyPowerReductionInit-
                         FRS- LN System CLS Not Supported, TPHComp- ExtTPHComp- ARIFwd-
                         AtomicOpsCap: Routing- 32bit+ 64bit+ 128bitCAS+
                DevCtl2: Completion Timeout: 50us to 50ms, TimeoutDis- ARIFwd-
                         AtomicOpsCtl: ReqEn- EgressBlck-
                         IDOReq- IDOCompl- LTR+ EmergencyPowerReductionReq-
                         10BitTagReq- OBFF Via WAKE#, EETLPPrefixBlk-
                LnkCap2: Supported Link Speeds: 2.5-8GT/s, Crosslink- Retimer- 2Retimers- DRS-
                LnkCtl2: Target Link Speed: 8GT/s, EnterCompliance- SpeedDis-
                         Transmit Margin: Normal Operating Range, EnterModifiedCompliance- ComplianceSOS-
                         Compliance Preset/De-emphasis: -6dB de-emphasis, 0dB preshoot
                LnkSta2: Current De-emphasis Level: -6dB, EqualizationComplete+ EqualizationPhase1+
                         EqualizationPhase2+ EqualizationPhase3+ LinkEqualizationRequest-
                         Retimer- 2Retimers- CrosslinkRes: unsupported
        Capabilities: [100 v1] Virtual Channel
                Caps:   LPEVC=0 RefClk=100ns PATEntryBits=1
                Arb:    Fixed- WRR32- WRR64- WRR128-
                Ctrl:   ArbSelect=Fixed
                Status: InProgress-
                VC0:    Caps:   PATOffset=00 MaxTimeSlots=1 RejSnoopTrans-
                        Arb:    Fixed+ WRR32- WRR64- WRR128- TWRR128- WRR256-
                        Ctrl:   Enable+ ID=0 ArbSelect=Fixed TC/VC=ff
                        Status: NegoPending- InProgress-
        Capabilities: [140 v1] Root Complex Link
                Desc:   PortNumber=02 ComponentID=01 EltType=Config
                Link0:  Desc:   TargetPort=00 TargetComponent=01 AssocRCRB- LinkType=MemMapped LinkValid+
                        Addr:   00000000fed19000
        Capabilities: [d94 v1] Secondary PCI Express
                LnkCtl3: LnkEquIntrruptEn- PerformEqu-
                LaneErrStat: 0
        Kernel driver in use: pcieport

01:00.0 PCI bridge: PLX Technology, Inc. PEX 8724 24-Lane, 6-Port PCI Express Gen 3 (8 GT/s) Switch, 19 x 19mm FCBGA (rev ca) (prog-if 00 [Normal decode])
        Subsystem: PLX Technology, Inc. PEX 8724 24-Lane, 6-Port PCI Express Gen 3 (8 GT/s) Switch, 19 x 19mm FCBGA
        Control: I/O+ Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx-
        Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
        Latency: 0, Cache Line Size: 64 bytes
        Interrupt: pin A routed to IRQ 16
        Region 0: Memory at ef400000 (32-bit, non-prefetchable) [size=256K]
        Bus: primary=01, secondary=02, subordinate=05, sec-latency=0
        I/O behind bridge: d000-efff [size=8K] [16-bit]
        Memory behind bridge: ef000000-ef3fffff [size=4M] [32-bit]
        Prefetchable memory behind bridge: c0000000-c05fffff [size=6M] [32-bit]
        Secondary status: 66MHz- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- <SERR- <PERR-
        BridgeCtl: Parity- SERR+ NoISA- VGA- VGA16+ MAbort- >Reset- FastB2B-
                PriDiscTmr- SecDiscTmr- DiscTmrStat- DiscTmrSERREn-
        Capabilities: [40] Power Management version 3
                Flags: PMEClk- DSI- D1- D2- AuxCurrent=0mA PME(D0+,D1-,D2-,D3hot+,D3cold+)
                Status: D0 NoSoftRst+ PME-Enable- DSel=0 DScale=0 PME-
        Capabilities: [48] MSI: Enable- Count=1/8 Maskable+ 64bit+
                Address: 0000000000000000  Data: 0000
                Masking: 00000000  Pending: 00000000
        Capabilities: [68] Express (v2) Upstream Port, IntMsgNum 0
                DevCap: MaxPayload 2048 bytes, PhantFunc 0
                        ExtTag- AttnBtn- AttnInd- PwrInd- RBE+ SlotPowerLimit 75W TEE-IO-
                DevCtl: CorrErr- NonFatalErr- FatalErr- UnsupReq-
                        RlxdOrd+ ExtTag- PhantFunc- AuxPwr- NoSnoop+
                        MaxPayload 256 bytes, MaxReadReq 128 bytes
                DevSta: CorrErr+ NonFatalErr- FatalErr- UnsupReq+ AuxPwr- TransPend-
                LnkCap: Port #0, Speed 8GT/s, Width x8, ASPM L1, Exit Latency L1 <4us
                        ClockPM- Surprise- LLActRep- BwNot- ASPMOptComp+
                LnkCtl: ASPM Disabled; LnkDisable- CommClk-
                        ExtSynch- ClockPM- AutWidDis- BWInt- AutBWInt-
                LnkSta: Speed 8GT/s, Width x8
                        TrErr- Train- SlotClk- DLActive- BWMgmt- ABWMgmt-
                DevCap2: Completion Timeout: Not Supported, TimeoutDis- NROPrPrP- LTR+
                         10BitTagComp- 10BitTagReq- OBFF Via message, ExtFmt- EETLPPrefix-
                         EmergencyPowerReduction Not Supported, EmergencyPowerReductionInit-
                         FRS-
                         AtomicOpsCap: Routing+ 32bit- 64bit- 128bitCAS-
                DevCtl2: Completion Timeout: 50us to 50ms, TimeoutDis-
                         AtomicOpsCtl: EgressBlck-
                         IDOReq- IDOCompl- LTR+ EmergencyPowerReductionReq-
                         10BitTagReq- OBFF Disabled, EETLPPrefixBlk-
                LnkCap2: Supported Link Speeds: 2.5-8GT/s, Crosslink+ Retimer- 2Retimers- DRS-
                LnkCtl2: Target Link Speed: 8GT/s, EnterCompliance- SpeedDis-
                         Transmit Margin: Normal Operating Range, EnterModifiedCompliance- ComplianceSOS-
                         Compliance Preset/De-emphasis: -6dB de-emphasis, 0dB preshoot
                LnkSta2: Current De-emphasis Level: -6dB, EqualizationComplete+ EqualizationPhase1+
                         EqualizationPhase2+ EqualizationPhase3+ LinkEqualizationRequest-
                         Retimer- 2Retimers- CrosslinkRes: unsupported
        Capabilities: [a4] Subsystem: PLX Technology, Inc. PEX 8724 24-Lane, 6-Port PCI Express Gen 3 (8 GT/s) Switch, 19 x 19mm FCBGA
        Capabilities: [100 v1] Device Serial Number ca-87-00-10-b5-df-0e-00
        Capabilities: [fb4 v1] Advanced Error Reporting
                UESta:  DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP-
                        ECRC- UnsupReq- ACSViol- UncorrIntErr- BlockedTLP- AtomicOpBlocked- TLPBlockedErr-
                        PoisonTLPBlocked- DMWrReqBlocked- IDECheck- MisIDETLP- PCRC_CHECK- TLPXlatBlocked-
                UEMsk:  DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP-
                        ECRC- UnsupReq- ACSViol- UncorrIntErr+ BlockedTLP- AtomicOpBlocked- TLPBlockedErr-
                        PoisonTLPBlocked- DMWrReqBlocked- IDECheck- MisIDETLP- PCRC_CHECK- TLPXlatBlocked-
                UESvrt: DLP+ SDES+ TLP- FCP+ CmpltTO- CmpltAbrt- UnxCmplt- RxOF+ MalfTLP+
                        ECRC- UnsupReq- ACSViol- UncorrIntErr+ BlockedTLP- AtomicOpBlocked- TLPBlockedErr-
                        PoisonTLPBlocked- DMWrReqBlocked- IDECheck- MisIDETLP- PCRC_CHECK- TLPXlatBlocked-
                CESta:  RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr+ CorrIntErr- HeaderOF-
                CEMsk:  RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr+ CorrIntErr+ HeaderOF+
                AERCap: First Error Pointer: 1f, ECRCGenCap+ ECRCGenEn- ECRCChkCap+ ECRCChkEn-
                        MultHdrRecCap- MultHdrRecEn- TLPPfxPres- HdrLogCap-
                HeaderLog: 00000000 00000000 00000000 00000000
        Capabilities: [138 v1] Power Budgeting <?>
        Capabilities: [10c v1] Secondary PCI Express
                LnkCtl3: LnkEquIntrruptEn- PerformEqu-
                LaneErrStat: 0
        Capabilities: [148 v1] Virtual Channel
                Caps:   LPEVC=0 RefClk=100ns PATEntryBits=8
                Arb:    Fixed- WRR32- WRR64- WRR128-
                Ctrl:   ArbSelect=Fixed
                Status: InProgress-
                VC0:    Caps:   PATOffset=03 MaxTimeSlots=1 RejSnoopTrans-
                        Arb:    Fixed- WRR32- WRR64+ WRR128- TWRR128- WRR256-
                        Ctrl:   Enable+ ID=0 ArbSelect=WRR64 TC/VC=ff
                        Status: NegoPending- InProgress-
                        Port Arbitration Table <?>
        Capabilities: [e00 v1] Multicast
                McastCap: MaxGroups 64, ECRCRegen+
                McastCtl: NumGroups 1, Enable-
                McastBAR: IndexPos 0, BaseAddr 0000000000000000
                McastReceiveVec:      0000000000000000
                McastBlockAllVec:     0000000000000000
                McastBlockUntransVec: 0000000000000000
                McastOverlayBAR: OverlaySize 0 (disabled), BaseAddr 0000000000000000
        Capabilities: [b00 v1] Latency Tolerance Reporting
                Max snoop latency: 71680ns
                Max no snoop latency: 71680ns
        Capabilities: [b70 v1] Vendor Specific Information: ID=0001 Rev=0 Len=010 <?>
        Kernel driver in use: pcieport

02:01.0 PCI bridge: PLX Technology, Inc. PEX 8724 24-Lane, 6-Port PCI Express Gen 3 (8 GT/s) Switch, 19 x 19mm FCBGA (rev ca) (prog-if 00 [Normal decode])
        Subsystem: PLX Technology, Inc. PEX 8724 24-Lane, 6-Port PCI Express Gen 3 (8 GT/s) Switch, 19 x 19mm FCBGA
        Control: I/O+ Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx+
        Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
        Latency: 0, Cache Line Size: 64 bytes
        Interrupt: pin A routed to IRQ 126
        Bus: primary=02, secondary=03, subordinate=03, sec-latency=0
        I/O behind bridge: e000-efff [size=4K] [16-bit]
        Memory behind bridge: ef200000-ef3fffff [size=2M] [32-bit]
        Prefetchable memory behind bridge: c0000000-c01fffff [size=2M] [32-bit]
        Secondary status: 66MHz- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- <SERR- <PERR-
        BridgeCtl: Parity- SERR+ NoISA- VGA- VGA16+ MAbort- >Reset- FastB2B-
                PriDiscTmr- SecDiscTmr- DiscTmrStat- DiscTmrSERREn-
        Capabilities: [40] Power Management version 3
                Flags: PMEClk- DSI- D1- D2- AuxCurrent=0mA PME(D0+,D1-,D2-,D3hot+,D3cold+)
                Status: D0 NoSoftRst+ PME-Enable- DSel=0 DScale=0 PME-
        Capabilities: [48] MSI: Enable+ Count=1/8 Maskable+ 64bit+
                Address: 00000000fee00318  Data: 0000
                Masking: 000000fe  Pending: 00000000
        Capabilities: [68] Express (v2) Downstream Port (Slot+), IntMsgNum 0
                DevCap: MaxPayload 2048 bytes, PhantFunc 0
                        ExtTag- RBE+ TEE-IO-
                DevCtl: CorrErr- NonFatalErr- FatalErr- UnsupReq-
                        RlxdOrd+ ExtTag- PhantFunc- AuxPwr- NoSnoop+
                        MaxPayload 256 bytes, MaxReadReq 128 bytes
                DevSta: CorrErr+ NonFatalErr- FatalErr- UnsupReq+ AuxPwr- TransPend-
                LnkCap: Port #1, Speed 8GT/s, Width x8, ASPM L1, Exit Latency L1 <4us
                        ClockPM- Surprise+ LLActRep+ BwNot+ ASPMOptComp+
                LnkCtl: ASPM Disabled; LnkDisable- CommClk-
                        ExtSynch- ClockPM- AutWidDis- BWInt+ AutBWInt+
                LnkSta: Speed 8GT/s, Width x8
                        TrErr- Train- SlotClk- DLActive+ BWMgmt- ABWMgmt-
                SltCap: AttnBtn+ PwrCtrl+ MRL+ AttnInd+ PwrInd+ HotPlug+ Surprise-
                        Slot #65, PowerLimit 25W; Interlock- NoCompl-
                SltCtl: Enable: AttnBtn- PwrFlt- MRL- PresDet- CmdCplt- HPIrq- LinkChg-
                        Control: AttnInd Off, PwrInd Off, Power+ Interlock-
                SltSta: Status: AttnBtn- PowerFlt- MRL+ CmdCplt- PresDet+ Interlock-
                        Changed: MRL- PresDet+ LinkState+
                DevCap2: Completion Timeout: Not Supported, TimeoutDis- NROPrPrP- LTR+
                         10BitTagComp- 10BitTagReq- OBFF Via message, ExtFmt- EETLPPrefix-
                         EmergencyPowerReduction Not Supported, EmergencyPowerReductionInit-
                         FRS- ARIFwd+
                         AtomicOpsCap: Routing+
                DevCtl2: Completion Timeout: 50us to 50ms, TimeoutDis- ARIFwd+
                         AtomicOpsCtl: EgressBlck-
                         IDOReq- IDOCompl- LTR+ EmergencyPowerReductionReq-
                         10BitTagReq- OBFF Disabled, EETLPPrefixBlk-
                LnkCap2: Supported Link Speeds: 2.5-8GT/s, Crosslink+ Retimer- 2Retimers- DRS-
                LnkCtl2: Target Link Speed: 8GT/s, EnterCompliance- SpeedDis-, Selectable De-emphasis: -6dB
                         Transmit Margin: Normal Operating Range, EnterModifiedCompliance- ComplianceSOS-
                         Compliance Preset/De-emphasis: -6dB de-emphasis, 0dB preshoot
                LnkSta2: Current De-emphasis Level: -6dB, EqualizationComplete+ EqualizationPhase1+
                         EqualizationPhase2+ EqualizationPhase3+ LinkEqualizationRequest-
                         Retimer- 2Retimers- CrosslinkRes: unsupported
        Capabilities: [a4] Subsystem: PLX Technology, Inc. PEX 8724 24-Lane, 6-Port PCI Express Gen 3 (8 GT/s) Switch, 19 x 19mm FCBGA
        Capabilities: [100 v1] Device Serial Number ca-87-00-10-b5-df-0e-00
        Capabilities: [fb4 v1] Advanced Error Reporting
                UESta:  DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP-
                        ECRC- UnsupReq- ACSViol- UncorrIntErr- BlockedTLP- AtomicOpBlocked- TLPBlockedErr-
                        PoisonTLPBlocked- DMWrReqBlocked- IDECheck- MisIDETLP- PCRC_CHECK- TLPXlatBlocked-
                UEMsk:  DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP-
                        ECRC- UnsupReq- ACSViol- UncorrIntErr- BlockedTLP- AtomicOpBlocked- TLPBlockedErr-
                        PoisonTLPBlocked- DMWrReqBlocked- IDECheck- MisIDETLP- PCRC_CHECK- TLPXlatBlocked-
                UESvrt: DLP+ SDES+ TLP- FCP+ CmpltTO- CmpltAbrt- UnxCmplt- RxOF+ MalfTLP+
                        ECRC- UnsupReq- ACSViol- UncorrIntErr+ BlockedTLP- AtomicOpBlocked- TLPBlockedErr-
                        PoisonTLPBlocked- DMWrReqBlocked- IDECheck- MisIDETLP- PCRC_CHECK- TLPXlatBlocked-
                CESta:  RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr+ CorrIntErr- HeaderOF-
                CEMsk:  RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr+ CorrIntErr- HeaderOF+
                AERCap: First Error Pointer: 1f, ECRCGenCap+ ECRCGenEn- ECRCChkCap+ ECRCChkEn-
                        MultHdrRecCap- MultHdrRecEn- TLPPfxPres- HdrLogCap-
                HeaderLog: 00000000 00000000 00000000 00000000
        Capabilities: [138 v1] Power Budgeting <?>
        Capabilities: [10c v1] Secondary PCI Express
                LnkCtl3: LnkEquIntrruptEn- PerformEqu-
                LaneErrStat: 0
        Capabilities: [148 v1] Virtual Channel
                Caps:   LPEVC=0 RefClk=100ns PATEntryBits=8
                Arb:    Fixed- WRR32- WRR64- WRR128-
                Ctrl:   ArbSelect=Fixed
                Status: InProgress-
                VC0:    Caps:   PATOffset=03 MaxTimeSlots=1 RejSnoopTrans-
                        Arb:    Fixed- WRR32- WRR64+ WRR128- TWRR128- WRR256-
                        Ctrl:   Enable+ ID=0 ArbSelect=WRR64 TC/VC=ff
                        Status: NegoPending- InProgress-
                        Port Arbitration Table <?>
        Capabilities: [e00 v1] Multicast
                McastCap: MaxGroups 64, ECRCRegen+
                McastCtl: NumGroups 1, Enable-
                McastBAR: IndexPos 0, BaseAddr 0000000000000000
                McastReceiveVec:      0000000000000000
                McastBlockAllVec:     0000000000000000
                McastBlockUntransVec: 0000000000000000
                McastOverlayBAR: OverlaySize 0 (disabled), BaseAddr 0000000000000000
        Capabilities: [f24 v1] Access Control Services
                ACSCap: SrcValid+ TransBlk+ ReqRedir+ CmpltRedir+ UpstreamFwd+ EgressCtrl+ DirectTrans+
                ACSCtl: SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
        Capabilities: [b70 v1] Vendor Specific Information: ID=0001 Rev=0 Len=010 <?>
        Kernel driver in use: pcieport

02:08.0 PCI bridge: PLX Technology, Inc. PEX 8724 24-Lane, 6-Port PCI Express Gen 3 (8 GT/s) Switch, 19 x 19mm FCBGA (rev ca) (prog-if 00 [Normal decode])
        Subsystem: PLX Technology, Inc. PEX 8724 24-Lane, 6-Port PCI Express Gen 3 (8 GT/s) Switch, 19 x 19mm FCBGA
        Control: I/O+ Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx+
        Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
        Latency: 0, Cache Line Size: 64 bytes
        Interrupt: pin A routed to IRQ 127
        Bus: primary=02, secondary=04, subordinate=04, sec-latency=0
        I/O behind bridge: 0000f000-00000fff [disabled] [32-bit]
        Memory behind bridge: fff00000-000fffff [disabled] [32-bit]
        Prefetchable memory behind bridge: 00000000fff00000-00000000000fffff [disabled] [64-bit]
        Secondary status: 66MHz- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- <SERR- <PERR-
        BridgeCtl: Parity- SERR+ NoISA- VGA- VGA16+ MAbort- >Reset- FastB2B-
                PriDiscTmr- SecDiscTmr- DiscTmrStat- DiscTmrSERREn-
        Capabilities: [40] Power Management version 3
                Flags: PMEClk- DSI- D1- D2- AuxCurrent=0mA PME(D0+,D1-,D2-,D3hot+,D3cold+)
                Status: D3 NoSoftRst+ PME-Enable+ DSel=0 DScale=0 PME-
        Capabilities: [48] MSI: Enable+ Count=1/8 Maskable+ 64bit+
                Address: 00000000fee00338  Data: 0000
                Masking: 000000fe  Pending: 00000000
        Capabilities: [68] Express (v2) Downstream Port (Slot+), IntMsgNum 0
                DevCap: MaxPayload 2048 bytes, PhantFunc 0
                        ExtTag- RBE+ TEE-IO-
                DevCtl: CorrErr- NonFatalErr- FatalErr- UnsupReq-
                        RlxdOrd+ ExtTag- PhantFunc- AuxPwr- NoSnoop+
                        MaxPayload 256 bytes, MaxReadReq 128 bytes
                DevSta: CorrErr+ NonFatalErr- FatalErr- UnsupReq+ AuxPwr- TransPend-
                LnkCap: Port #8, Speed 8GT/s, Width x8, ASPM L1, Exit Latency L1 <4us
                        ClockPM- Surprise+ LLActRep+ BwNot+ ASPMOptComp+
                LnkCtl: ASPM Disabled; LnkDisable- CommClk-
                        ExtSynch- ClockPM- AutWidDis- BWInt+ AutBWInt+
                LnkSta: Speed 2.5GT/s, Width x0
                        TrErr- Train- SlotClk- DLActive- BWMgmt- ABWMgmt-
                SltCap: AttnBtn- PwrCtrl- MRL- AttnInd- PwrInd- HotPlug- Surprise-
                        Slot #72, PowerLimit 25W; Interlock- NoCompl-
                SltCtl: Enable: AttnBtn- PwrFlt- MRL- PresDet- CmdCplt- HPIrq- LinkChg-
                        Control: AttnInd Unknown, PwrInd Unknown, Power- Interlock-
                SltSta: Status: AttnBtn- PowerFlt- MRL- CmdCplt- PresDet- Interlock-
                        Changed: MRL- PresDet- LinkState-
                DevCap2: Completion Timeout: Not Supported, TimeoutDis- NROPrPrP- LTR+
                         10BitTagComp- 10BitTagReq- OBFF Via message, ExtFmt- EETLPPrefix-
                         EmergencyPowerReduction Not Supported, EmergencyPowerReductionInit-
                         FRS- ARIFwd+
                         AtomicOpsCap: Routing+
                DevCtl2: Completion Timeout: 50us to 50ms, TimeoutDis- ARIFwd-
                         AtomicOpsCtl: EgressBlck-
                         IDOReq- IDOCompl- LTR- EmergencyPowerReductionReq-
                         10BitTagReq- OBFF Disabled, EETLPPrefixBlk-
                LnkCap2: Supported Link Speeds: 2.5-8GT/s, Crosslink+ Retimer- 2Retimers- DRS-
                LnkCtl2: Target Link Speed: 8GT/s, EnterCompliance- SpeedDis-, Selectable De-emphasis: -6dB
                         Transmit Margin: Normal Operating Range, EnterModifiedCompliance- ComplianceSOS-
                         Compliance Preset/De-emphasis: -6dB de-emphasis, 0dB preshoot
                LnkSta2: Current De-emphasis Level: -3.5dB, EqualizationComplete- EqualizationPhase1-
                         EqualizationPhase2- EqualizationPhase3- LinkEqualizationRequest-
                         Retimer- 2Retimers- CrosslinkRes: unsupported
        Capabilities: [a4] Subsystem: PLX Technology, Inc. PEX 8724 24-Lane, 6-Port PCI Express Gen 3 (8 GT/s) Switch, 19 x 19mm FCBGA
        Capabilities: [100 v1] Device Serial Number ca-87-00-10-b5-df-0e-00
        Capabilities: [fb4 v1] Advanced Error Reporting
                UESta:  DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP-
                        ECRC- UnsupReq- ACSViol- UncorrIntErr- BlockedTLP- AtomicOpBlocked- TLPBlockedErr-
                        PoisonTLPBlocked- DMWrReqBlocked- IDECheck- MisIDETLP- PCRC_CHECK- TLPXlatBlocked-
                UEMsk:  DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP-
                        ECRC- UnsupReq- ACSViol- UncorrIntErr+ BlockedTLP- AtomicOpBlocked- TLPBlockedErr-
                        PoisonTLPBlocked- DMWrReqBlocked- IDECheck- MisIDETLP- PCRC_CHECK- TLPXlatBlocked-
                UESvrt: DLP+ SDES+ TLP- FCP+ CmpltTO- CmpltAbrt- UnxCmplt- RxOF+ MalfTLP+
                        ECRC- UnsupReq- ACSViol- UncorrIntErr+ BlockedTLP- AtomicOpBlocked- TLPBlockedErr-
                        PoisonTLPBlocked- DMWrReqBlocked- IDECheck- MisIDETLP- PCRC_CHECK- TLPXlatBlocked-
                CESta:  RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr+ CorrIntErr- HeaderOF-
                CEMsk:  RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr+ CorrIntErr+ HeaderOF+
                AERCap: First Error Pointer: 1f, ECRCGenCap+ ECRCGenEn- ECRCChkCap+ ECRCChkEn-
                        MultHdrRecCap- MultHdrRecEn- TLPPfxPres- HdrLogCap-
                HeaderLog: 00000000 00000000 00000000 00000000
        Capabilities: [138 v1] Power Budgeting <?>
        Capabilities: [10c v1] Secondary PCI Express
                LnkCtl3: LnkEquIntrruptEn- PerformEqu-
                LaneErrStat: 0
        Capabilities: [148 v1] Virtual Channel
                Caps:   LPEVC=0 RefClk=100ns PATEntryBits=8
                Arb:    Fixed- WRR32- WRR64- WRR128-
                Ctrl:   ArbSelect=Fixed
                Status: InProgress-
                VC0:    Caps:   PATOffset=03 MaxTimeSlots=1 RejSnoopTrans-
                        Arb:    Fixed- WRR32- WRR64+ WRR128- TWRR128- WRR256-
                        Ctrl:   Enable+ ID=0 ArbSelect=WRR64 TC/VC=ff
                        Status: NegoPending+ InProgress-
                        Port Arbitration Table <?>
        Capabilities: [e00 v1] Multicast
                McastCap: MaxGroups 64, ECRCRegen+
                McastCtl: NumGroups 1, Enable-
                McastBAR: IndexPos 0, BaseAddr 0000000000000000
                McastReceiveVec:      0000000000000000
                McastBlockAllVec:     0000000000000000
                McastBlockUntransVec: 0000000000000000
                McastOverlayBAR: OverlaySize 0 (disabled), BaseAddr 0000000000000000
        Capabilities: [f24 v1] Access Control Services
                ACSCap: SrcValid+ TransBlk+ ReqRedir+ CmpltRedir+ UpstreamFwd+ EgressCtrl+ DirectTrans+
                ACSCtl: SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
        Capabilities: [b70 v1] Vendor Specific Information: ID=0001 Rev=0 Len=010 <?>
        Kernel driver in use: pcieport

02:09.0 PCI bridge: PLX Technology, Inc. PEX 8724 24-Lane, 6-Port PCI Express Gen 3 (8 GT/s) Switch, 19 x 19mm FCBGA (rev ca) (prog-if 00 [Normal decode])
        Subsystem: PLX Technology, Inc. PEX 8724 24-Lane, 6-Port PCI Express Gen 3 (8 GT/s) Switch, 19 x 19mm FCBGA
        Control: I/O+ Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx+
        Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
        Latency: 0, Cache Line Size: 64 bytes
        Interrupt: pin A routed to IRQ 128
        Bus: primary=02, secondary=05, subordinate=05, sec-latency=0
        I/O behind bridge: d000-dfff [size=4K] [16-bit]
        Memory behind bridge: ef000000-ef1fffff [size=2M] [32-bit]
        Prefetchable memory behind bridge: c0200000-c03fffff [size=2M] [32-bit]
        Secondary status: 66MHz- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- <SERR- <PERR-
        BridgeCtl: Parity- SERR+ NoISA- VGA- VGA16+ MAbort- >Reset- FastB2B-
                PriDiscTmr- SecDiscTmr- DiscTmrStat- DiscTmrSERREn-
        Capabilities: [40] Power Management version 3
                Flags: PMEClk- DSI- D1- D2- AuxCurrent=0mA PME(D0+,D1-,D2-,D3hot+,D3cold+)
                Status: D0 NoSoftRst+ PME-Enable- DSel=0 DScale=0 PME-
        Capabilities: [48] MSI: Enable+ Count=1/8 Maskable+ 64bit+
                Address: 00000000fee00358  Data: 0000
                Masking: 000000fe  Pending: 00000000
        Capabilities: [68] Express (v2) Downstream Port (Slot+), IntMsgNum 0
                DevCap: MaxPayload 2048 bytes, PhantFunc 0
                        ExtTag- RBE+ TEE-IO-
                DevCtl: CorrErr- NonFatalErr- FatalErr- UnsupReq-
                        RlxdOrd+ ExtTag- PhantFunc- AuxPwr- NoSnoop+
                        MaxPayload 256 bytes, MaxReadReq 128 bytes
                DevSta: CorrErr+ NonFatalErr- FatalErr- UnsupReq+ AuxPwr- TransPend-
                LnkCap: Port #9, Speed 8GT/s, Width x8, ASPM L1, Exit Latency L1 <4us
                        ClockPM- Surprise+ LLActRep+ BwNot+ ASPMOptComp+
                LnkCtl: ASPM Disabled; LnkDisable- CommClk-
                        ExtSynch- ClockPM- AutWidDis- BWInt+ AutBWInt+
                LnkSta: Speed 8GT/s, Width x8
                        TrErr- Train- SlotClk- DLActive+ BWMgmt- ABWMgmt-
                SltCap: AttnBtn+ PwrCtrl+ MRL+ AttnInd+ PwrInd+ HotPlug+ Surprise-
                        Slot #73, PowerLimit 25W; Interlock- NoCompl-
                SltCtl: Enable: AttnBtn- PwrFlt- MRL- PresDet- CmdCplt- HPIrq- LinkChg-
                        Control: AttnInd Off, PwrInd Off, Power+ Interlock-
                SltSta: Status: AttnBtn- PowerFlt- MRL+ CmdCplt- PresDet+ Interlock-
                        Changed: MRL- PresDet+ LinkState+
                DevCap2: Completion Timeout: Not Supported, TimeoutDis- NROPrPrP- LTR+
                         10BitTagComp- 10BitTagReq- OBFF Via message, ExtFmt- EETLPPrefix-
                         EmergencyPowerReduction Not Supported, EmergencyPowerReductionInit-
                         FRS- ARIFwd+
                         AtomicOpsCap: Routing+
                DevCtl2: Completion Timeout: 50us to 50ms, TimeoutDis- ARIFwd+
                         AtomicOpsCtl: EgressBlck-
                         IDOReq- IDOCompl- LTR+ EmergencyPowerReductionReq-
                         10BitTagReq- OBFF Disabled, EETLPPrefixBlk-
                LnkCap2: Supported Link Speeds: 2.5-8GT/s, Crosslink+ Retimer- 2Retimers- DRS-
                LnkCtl2: Target Link Speed: 8GT/s, EnterCompliance- SpeedDis-, Selectable De-emphasis: -6dB
                         Transmit Margin: Normal Operating Range, EnterModifiedCompliance- ComplianceSOS-
                         Compliance Preset/De-emphasis: -6dB de-emphasis, 0dB preshoot
                LnkSta2: Current De-emphasis Level: -6dB, EqualizationComplete+ EqualizationPhase1+
                         EqualizationPhase2+ EqualizationPhase3+ LinkEqualizationRequest-
                         Retimer- 2Retimers- CrosslinkRes: unsupported
        Capabilities: [a4] Subsystem: PLX Technology, Inc. PEX 8724 24-Lane, 6-Port PCI Express Gen 3 (8 GT/s) Switch, 19 x 19mm FCBGA
        Capabilities: [100 v1] Device Serial Number ca-87-00-10-b5-df-0e-00
        Capabilities: [fb4 v1] Advanced Error Reporting
                UESta:  DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP-
                        ECRC- UnsupReq- ACSViol- UncorrIntErr- BlockedTLP- AtomicOpBlocked- TLPBlockedErr-
                        PoisonTLPBlocked- DMWrReqBlocked- IDECheck- MisIDETLP- PCRC_CHECK- TLPXlatBlocked-
                UEMsk:  DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP-
                        ECRC- UnsupReq- ACSViol- UncorrIntErr- BlockedTLP- AtomicOpBlocked- TLPBlockedErr-
                        PoisonTLPBlocked- DMWrReqBlocked- IDECheck- MisIDETLP- PCRC_CHECK- TLPXlatBlocked-
                UESvrt: DLP+ SDES+ TLP- FCP+ CmpltTO- CmpltAbrt- UnxCmplt- RxOF+ MalfTLP+
                        ECRC- UnsupReq- ACSViol- UncorrIntErr+ BlockedTLP- AtomicOpBlocked- TLPBlockedErr-
                        PoisonTLPBlocked- DMWrReqBlocked- IDECheck- MisIDETLP- PCRC_CHECK- TLPXlatBlocked-
                CESta:  RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr+ CorrIntErr- HeaderOF-
                CEMsk:  RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr+ CorrIntErr- HeaderOF+
                AERCap: First Error Pointer: 1f, ECRCGenCap+ ECRCGenEn- ECRCChkCap+ ECRCChkEn-
                        MultHdrRecCap- MultHdrRecEn- TLPPfxPres- HdrLogCap-
                HeaderLog: 00000000 00000000 00000000 00000000
        Capabilities: [138 v1] Power Budgeting <?>
        Capabilities: [10c v1] Secondary PCI Express
                LnkCtl3: LnkEquIntrruptEn- PerformEqu-
                LaneErrStat: 0
        Capabilities: [148 v1] Virtual Channel
                Caps:   LPEVC=0 RefClk=100ns PATEntryBits=8
                Arb:    Fixed- WRR32- WRR64- WRR128-
                Ctrl:   ArbSelect=Fixed
                Status: InProgress-
                VC0:    Caps:   PATOffset=03 MaxTimeSlots=1 RejSnoopTrans-
                        Arb:    Fixed- WRR32- WRR64+ WRR128- TWRR128- WRR256-
                        Ctrl:   Enable+ ID=0 ArbSelect=WRR64 TC/VC=ff
                        Status: NegoPending- InProgress-
                        Port Arbitration Table <?>
        Capabilities: [e00 v1] Multicast
                McastCap: MaxGroups 64, ECRCRegen+
                McastCtl: NumGroups 1, Enable-
                McastBAR: IndexPos 0, BaseAddr 0000000000000000
                McastReceiveVec:      0000000000000000
                McastBlockAllVec:     0000000000000000
                McastBlockUntransVec: 0000000000000000
                McastOverlayBAR: OverlaySize 0 (disabled), BaseAddr 0000000000000000
        Capabilities: [f24 v1] Access Control Services
                ACSCap: SrcValid+ TransBlk+ ReqRedir+ CmpltRedir+ UpstreamFwd+ EgressCtrl+ DirectTrans+
                ACSCtl: SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
        Capabilities: [b70 v1] Vendor Specific Information: ID=0001 Rev=0 Len=010 <?>
        Kernel driver in use: pcieport

03:00.0 Serial Attached SCSI controller: Broadcom / LSI SAS2308 PCI-Express Fusion-MPT SAS-2 (rev 05)
        Subsystem: Broadcom / LSI Device 3070
        Control: I/O+ Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx+
        Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
        Latency: 0, Cache Line Size: 64 bytes
        Interrupt: pin A routed to IRQ 17
        Region 0: I/O ports at e000 [size=256]
        Region 1: Memory at ef340000 (64-bit, non-prefetchable) [size=64K]
        Region 3: Memory at ef300000 (64-bit, non-prefetchable) [size=256K]
        Expansion ROM at ef200000 [disabled] [size=1M]
        Capabilities: [50] Power Management version 3
                Flags: PMEClk- DSI- D1+ D2+ AuxCurrent=0mA PME(D0-,D1-,D2-,D3hot-,D3cold-)
                Status: D0 NoSoftRst+ PME-Enable- DSel=0 DScale=0 PME-
        Capabilities: [68] Express (v2) Endpoint, IntMsgNum 0
                DevCap: MaxPayload 4096 bytes, PhantFunc 0, Latency L0s <64ns, L1 <1us
                        ExtTag+ AttnBtn- AttnInd- PwrInd- RBE+ FLReset+ SlotPowerLimit 0W TEE-IO-
                DevCtl: CorrErr- NonFatalErr- FatalErr- UnsupReq-
                        RlxdOrd+ ExtTag+ PhantFunc- AuxPwr- NoSnoop+ FLReset-
                        MaxPayload 256 bytes, MaxReadReq 512 bytes
                DevSta: CorrErr+ NonFatalErr- FatalErr- UnsupReq+ AuxPwr- TransPend-
                LnkCap: Port #0, Speed 8GT/s, Width x8, ASPM L0s, Exit Latency L0s <64ns
                        ClockPM- Surprise- LLActRep- BwNot- ASPMOptComp+
                LnkCtl: ASPM Disabled; RCB 64 bytes, LnkDisable- CommClk-
                        ExtSynch- ClockPM- AutWidDis- BWInt- AutBWInt-
                LnkSta: Speed 8GT/s, Width x8
                        TrErr- Train- SlotClk+ DLActive- BWMgmt- ABWMgmt-
                DevCap2: Completion Timeout: Range BC, TimeoutDis+ NROPrPrP- LTR-
                         10BitTagComp- 10BitTagReq- OBFF Not Supported, ExtFmt- EETLPPrefix-
                         EmergencyPowerReduction Not Supported, EmergencyPowerReductionInit-
                         FRS- TPHComp- ExtTPHComp-
                         AtomicOpsCap: 32bit- 64bit- 128bitCAS-
                DevCtl2: Completion Timeout: 50us to 50ms, TimeoutDis-
                         AtomicOpsCtl: ReqEn-
                         IDOReq- IDOCompl- LTR- EmergencyPowerReductionReq-
                         10BitTagReq- OBFF Disabled, EETLPPrefixBlk-
                LnkCap2: Supported Link Speeds: 2.5-8GT/s, Crosslink- Retimer- 2Retimers- DRS-
                LnkCtl2: Target Link Speed: 8GT/s, EnterCompliance- SpeedDis-
                         Transmit Margin: Normal Operating Range, EnterModifiedCompliance- ComplianceSOS-
                         Compliance Preset/De-emphasis: -6dB de-emphasis, 0dB preshoot
                LnkSta2: Current De-emphasis Level: -6dB, EqualizationComplete+ EqualizationPhase1+
                         EqualizationPhase2+ EqualizationPhase3+ LinkEqualizationRequest-
                         Retimer- 2Retimers- CrosslinkRes: unsupported
        Capabilities: [d0] Vital Product Data
pcilib: sysfs_read_vpd: read failed: No such device
                Not readable
        Capabilities: [a8] MSI: Enable- Count=1/1 Maskable- 64bit+
                Address: 0000000000000000  Data: 0000
        Capabilities: [c0] MSI-X: Enable+ Count=16 Masked-
                Vector table: BAR=1 offset=0000e000
                PBA: BAR=1 offset=0000f000
        Capabilities: [100 v2] Advanced Error Reporting
                UESta:  DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP-
                        ECRC- UnsupReq- ACSViol- UncorrIntErr- BlockedTLP- AtomicOpBlocked- TLPBlockedErr-
                        PoisonTLPBlocked- DMWrReqBlocked- IDECheck- MisIDETLP- PCRC_CHECK- TLPXlatBlocked-
                UEMsk:  DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP-
                        ECRC- UnsupReq- ACSViol- UncorrIntErr- BlockedTLP- AtomicOpBlocked- TLPBlockedErr-
                        PoisonTLPBlocked- DMWrReqBlocked- IDECheck- MisIDETLP- PCRC_CHECK- TLPXlatBlocked-
                UESvrt: DLP+ SDES+ TLP- FCP+ CmpltTO- CmpltAbrt- UnxCmplt- RxOF+ MalfTLP+
                        ECRC- UnsupReq- ACSViol- UncorrIntErr+ BlockedTLP- AtomicOpBlocked- TLPBlockedErr-
                        PoisonTLPBlocked- DMWrReqBlocked- IDECheck- MisIDETLP- PCRC_CHECK- TLPXlatBlocked-
                CESta:  RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr+ CorrIntErr- HeaderOF-
                CEMsk:  RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr+ CorrIntErr- HeaderOF-
                AERCap: First Error Pointer: 00, ECRCGenCap- ECRCGenEn- ECRCChkCap- ECRCChkEn-
                        MultHdrRecCap- MultHdrRecEn- TLPPfxPres- HdrLogCap-
                HeaderLog: 04000001 00000003 03010000 0dd6a3fa
        Capabilities: [1e0 v1] Secondary PCI Express
                LnkCtl3: LnkEquIntrruptEn- PerformEqu-
                LaneErrStat: 0
        Capabilities: [1c0 v1] Power Budgeting <?>
        Capabilities: [190 v1] Dynamic Power Allocation <?>
        Capabilities: [148 v1] Alternative Routing-ID Interpretation (ARI)
                ARICap: MFVC- ACS-, Next Function: 0
                ARICtl: MFVC- ACS-, Function Group: 0
        Kernel driver in use: mpt3sas
        Kernel modules: mpt3sas

05:00.0 Serial Attached SCSI controller: Broadcom / LSI SAS2308 PCI-Express Fusion-MPT SAS-2 (rev 05)
        Subsystem: Broadcom / LSI Device 3070
        Control: I/O+ Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx+
        Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
        Latency: 0, Cache Line Size: 64 bytes
        Interrupt: pin A routed to IRQ 17
        Region 0: I/O ports at d000 [size=256]
        Region 1: Memory at ef140000 (64-bit, non-prefetchable) [size=64K]
        Region 3: Memory at ef100000 (64-bit, non-prefetchable) [size=256K]
        Expansion ROM at ef000000 [disabled] [size=1M]
        Capabilities: [50] Power Management version 3
                Flags: PMEClk- DSI- D1+ D2+ AuxCurrent=0mA PME(D0-,D1-,D2-,D3hot-,D3cold-)
                Status: D0 NoSoftRst+ PME-Enable- DSel=0 DScale=0 PME-
        Capabilities: [68] Express (v2) Endpoint, IntMsgNum 0
                DevCap: MaxPayload 4096 bytes, PhantFunc 0, Latency L0s <64ns, L1 <1us
                        ExtTag+ AttnBtn- AttnInd- PwrInd- RBE+ FLReset+ SlotPowerLimit 0W TEE-IO-
                DevCtl: CorrErr- NonFatalErr- FatalErr- UnsupReq-
                        RlxdOrd+ ExtTag+ PhantFunc- AuxPwr- NoSnoop+ FLReset-
                        MaxPayload 256 bytes, MaxReadReq 512 bytes
                DevSta: CorrErr+ NonFatalErr- FatalErr- UnsupReq+ AuxPwr- TransPend-
                LnkCap: Port #0, Speed 8GT/s, Width x8, ASPM L0s, Exit Latency L0s <64ns
                        ClockPM- Surprise- LLActRep- BwNot- ASPMOptComp+
                LnkCtl: ASPM Disabled; RCB 64 bytes, LnkDisable- CommClk-
                        ExtSynch- ClockPM- AutWidDis- BWInt- AutBWInt-
                LnkSta: Speed 8GT/s, Width x8
                        TrErr- Train- SlotClk+ DLActive- BWMgmt- ABWMgmt-
                DevCap2: Completion Timeout: Range BC, TimeoutDis+ NROPrPrP- LTR-
                         10BitTagComp- 10BitTagReq- OBFF Not Supported, ExtFmt- EETLPPrefix-
                         EmergencyPowerReduction Not Supported, EmergencyPowerReductionInit-
                         FRS- TPHComp- ExtTPHComp-
                         AtomicOpsCap: 32bit- 64bit- 128bitCAS-
                DevCtl2: Completion Timeout: 50us to 50ms, TimeoutDis-
                         AtomicOpsCtl: ReqEn-
                         IDOReq- IDOCompl- LTR- EmergencyPowerReductionReq-
                         10BitTagReq- OBFF Disabled, EETLPPrefixBlk-
                LnkCap2: Supported Link Speeds: 2.5-8GT/s, Crosslink- Retimer- 2Retimers- DRS-
                LnkCtl2: Target Link Speed: 8GT/s, EnterCompliance- SpeedDis-
                         Transmit Margin: Normal Operating Range, EnterModifiedCompliance- ComplianceSOS-
                         Compliance Preset/De-emphasis: -6dB de-emphasis, 0dB preshoot
                LnkSta2: Current De-emphasis Level: -6dB, EqualizationComplete+ EqualizationPhase1+
                         EqualizationPhase2+ EqualizationPhase3+ LinkEqualizationRequest-
                         Retimer- 2Retimers- CrosslinkRes: unsupported
        Capabilities: [d0] Vital Product Data
pcilib: sysfs_read_vpd: read failed: No such device
                Not readable
        Capabilities: [a8] MSI: Enable- Count=1/1 Maskable- 64bit+
                Address: 0000000000000000  Data: 0000
        Capabilities: [c0] MSI-X: Enable+ Count=16 Masked-
                Vector table: BAR=1 offset=0000e000
                PBA: BAR=1 offset=0000f000
        Capabilities: [100 v2] Advanced Error Reporting
                UESta:  DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP-
                        ECRC- UnsupReq- ACSViol- UncorrIntErr- BlockedTLP- AtomicOpBlocked- TLPBlockedErr-
                        PoisonTLPBlocked- DMWrReqBlocked- IDECheck- MisIDETLP- PCRC_CHECK- TLPXlatBlocked-
                UEMsk:  DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP-
                        ECRC- UnsupReq- ACSViol- UncorrIntErr- BlockedTLP- AtomicOpBlocked- TLPBlockedErr-
                        PoisonTLPBlocked- DMWrReqBlocked- IDECheck- MisIDETLP- PCRC_CHECK- TLPXlatBlocked-
                UESvrt: DLP+ SDES+ TLP- FCP+ CmpltTO- CmpltAbrt- UnxCmplt- RxOF+ MalfTLP+
                        ECRC- UnsupReq- ACSViol- UncorrIntErr+ BlockedTLP- AtomicOpBlocked- TLPBlockedErr-
                        PoisonTLPBlocked- DMWrReqBlocked- IDECheck- MisIDETLP- PCRC_CHECK- TLPXlatBlocked-
                CESta:  RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr+ CorrIntErr- HeaderOF-
                CEMsk:  RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr+ CorrIntErr- HeaderOF-
                AERCap: First Error Pointer: 00, ECRCGenCap- ECRCGenEn- ECRCChkCap- ECRCChkEn-
                        MultHdrRecCap- MultHdrRecEn- TLPPfxPres- HdrLogCap-
                HeaderLog: 04000001 00000003 05010000 3fd43416
        Capabilities: [1e0 v1] Secondary PCI Express
                LnkCtl3: LnkEquIntrruptEn- PerformEqu-
                LaneErrStat: 0
        Capabilities: [1c0 v1] Power Budgeting <?>
        Capabilities: [190 v1] Dynamic Power Allocation <?>
        Capabilities: [148 v1] Alternative Routing-ID Interpretation (ARI)
                ARICap: MFVC- ACS-, Next Function: 0
                ARICtl: MFVC- ACS-, Function Group: 0
        Kernel driver in use: mpt3sas
        Kernel modules: mpt3sas
```
</details>

### Active cooling part 3: All bets off

Since using this card adds roughly an extra 30 watts of power consumption, why not a couple extra? While at it, why not also up the noise profile? All of the current tests have been performed with a Corsair AF120 led series quiet edition fan. Rated at up to 1500rpm and 25dBA, this is a quite respectable fan, still delivering adequate air flow when used appropiately, and staying out of our ears doing so. Helpful in a homelab setup where the system's accoustic profile is something to be noticed by anyone. But why not throw all considerations out? Who cares how loud the system gets if the card gets to stay as cold as possible? Enter the Cooler Master JetFlo 120mm fan.

(insert image)

I will not bore you with the details of this fan. To keep things brief, this fan is known for delievering high air flow (~95 CFM), ramping to a quick-for-a-120mm-fan 2000rpm, all with a rated noise level of 36 dBA.

Now before I talk about temperatures, why did I chose a JetFlo (or an air flow focused fan for that matter), compared to a static pressure optimized fan? First off, the fin density of the heatsink is quite low compared to most desktop tower coolers and radiators, so there should be less restriction for airflow. Second, the card is not 120mm tall, so only half the fan aims directly at the heatsink fins. If curiousity does get the best of me in the future to try a static pressure optimized fan, I will be sure to update back.

(insert image)

36°C on controller 1, 40°C on controller 2. That is impressive. Power consumption did increase by 3 watts however, which is understandable. My understanding is that PTM7950 does work most efficiently starting at 50°C, so it's possible that anything lower then this is unlikely. Perhaps regular paste would have behaved differently? Possibly not though if the paste cannot transfer sink heat away as fast.

Additionally, I did notice the fan spinning 300rpm higher then it's rated. It's a weird quirk of this fan. My source? Trust me and look it up, other people notice it too.

### Active cooling part 4: ~~Do we really need it?~~ YES YOU DO

As a last test, I unplugged the fan, leaving only the CPU fan, top exhaust fan, and front intake fans plugged in, on regular CPU temperature fan curves to see if that could keep the temperatures at bay. It did not. It took less than 5 minutes for the card to reach 101°C on the second controller. Interestingly the first controller only reached 70°C but at the rate of progression I imagine it would have been only a matter of time. Plugging the fan back in though quickly cooled down the HBA back into reasonable temperature ranges within one minute. Safe to say the PTM7950 is doing its job.

(insert image)

### So what now? (conclusion, closing notes, extra specifics for the detail oriented folks)

Well at minimum, there are now 2 working LSI 9216-16e HBA cards ready to be used in a system. That is it really. I already know PTM7950 works and it would not appropiate to advocate for it purely from this, since I never tested with regular paste. I now know that the SAS2308 based cards need a style of airflow to prevent them from essentially melting, and had an excuse to shirk other duties to experiment with this. Worth it I would say.

#### How did I automatically control the fan speed for the HBA?

Just because of how it is, the HBA does not provide temperature information displayable with `lm-sensors`. Could a custom driver be written? Yes, of course. I however am not resourceful in that regard, at least in this day and age. Instead, in my quest for achieving this goal, I came across [`afancontrol`](https://afancontrol.readthedocs.io/en/latest/). Like the name suggests, it is similar to the known `fancontrol` program, but written in Python, and supports a variety of means for pulling in temperature data (storage device temperatures, `lm-sensors`, `exec`utable commands, and even reading from a `file`, to name a few), and of course being able to read fan RPM and control various PWM outputs based on temperature inputs. A cool idea I might look into as well is the ability to control extra fans by an Arduino connected over USB.

At least for now, I have no guide on setting up `afancontrol`. I do plan on it in the future - seems to be a very nice tool. I will however provide 

<details>
<summary>my configuration file.</summary>
<br>

```text
[daemon]
pidfile = /run/afancontrol.pid
logfile = /var/log/afancontrol.log
interval = 5

[actions]

[filter: moving_quantile]
type = moving_quantile
window_size = 5
quantile=0.75

[temp:hba1]
type = exec
command = echo $(( 16#$( sudo lsiutil -p1 -a 25,2,0,0 | grep IOCTemperature: | cut -dx -f2 ) ))
filter = moving_quantile
min = 30
max = 65
panic = 80

[temp:hba2]
type = exec
command = echo $(( 16#$( sudo lsiutil -p2 -a 25,2,0,0 | grep IOCTemperature: | cut -dx -f2 ) ))
filter = moving_median_p3
min = 30
max = 65
panic = 80

[fan:hba]
type = linux
pwm = /sys/class/hwmon/hwmon1/pwm4
fan_input = /sys/class/hwmon/hwmon1/fan4_input
pwm_line_start = 95
pwm_line_end = 155
never_stop = yes

[mapping:1]
fans = hba
temps = hba1, hba2
```

</details>

#### How am I controlling the fan headers of my Gigabyte motherboard?

"I have (insert model) Gigabyte motherboard and I cannot see any sensors even running `sensors-detect`!"

Sounds like you should try to manually install a different version of the [it87 kernel module](https://gist.github.com/bakman2/e801f342aaa7cade62d7bd54fd3eabd8).

Worked for me on my Gigabyte Z270X Ultra Gaming board. I haven't figured out what temperature and voltages relates to what, but fan 1 is `CPU`, fan 2 is `SYS_FAN1`, fan 3 is `SYS_FAN2`, fan 4 is `SYS_FAN3_PUMP`, and fan 5 is `CPU_OPT`.

<details>
<summary>Archive copy of the gist</summary>
<br>

# PWM fan control in Linux with a Gigabyte Aorus motherboard

- install lm-sensors with your package manager

`sensors`

If it won't show any fan/speed, continue

`sensor-detect` 

Say YES to at least "Super I/O sensors"

Expected output:
```
Trying family `ITE'...                                      Yes
Found unknown chip with ID 0x8688
```
If similar, continue

```
git clone https://github.com/frankcrawford/it87

cd it87
sudo make clean
sudo make install
sudo modprobe it87 ignore_resource_conflict=1 force_id=0x8622

sensors
```


The fans should show up now, if yes, continue to make them available at boot:

```
echo options it87 ignore_resource_conflict=1 force_id=0x8622 > /etc/modprobe.d/it87.conf
echo it87 >> /etc/modules
```
</details>

---

#### How did you make those graphs and get the realtime date for all of that?

Telegraf and Home Assistant send data to InfluxDB. Grafana reads from InfluxDB. This was my first time with any of this (except for Home Assistant). Maybe someday I will make my own write up of this, but for now, do your own research, and experiment.

I will however, share the configurations for everything.

<details>
<summary>InfluxDB Docker Compose</summary>
<br>

```yaml
services:
  influxdb:
    image: influxdb:2-alpine
    container_name: influxdb2
    restart: unless-stopped
    ports:
      - 8086:8086
    environment:
      DOCKER_INFLUXDB_INIT_MODE: setup
      DOCKER_INFLUXDB_INIT_USERNAME_FILE: /run/secrets/influxdb-admin-username
      DOCKER_INFLUXDB_INIT_PASSWORD_FILE: /run/secrets/influxdb-admin-password
      DOCKER_INFLUXDB_INIT_ADMIN_TOKEN_FILE: /run/secrets/influxdb-admin-token
      DOCKER_INFLUXDB_INIT_ORG: docs
      DOCKER_INFLUXDB_INIT_BUCKET: home
    secrets:
      - influxdb-admin-username
      - influxdb-admin-password
      - influxdb-admin-token
    volumes:
      - /opt/influxdb/data:/var/lib/influxdb2
      - /opt/influxdb/config:/etc/influxdb2
secrets:
  influxdb-admin-username:
    file: ./.influxdb-admin-username
  influxdb-admin-password:
    file: ./.influxdb-admin-password
  influxdb-admin-token:
    file: ./.influxdb-admin-token
```

</details>

(This is largly based from [InfluxDB's own Docker Compose documentation](https://docs.influxdata.com/influxdb/v2/install/use-docker-compose/))

<details>
<summary>Home Assistant Docker Compose</summary>
<br>

```yaml
services:
  home-assistant:
    container_name: home-assistant
    image: ghcr.io/home-assistant/home-assistant:2025.4
    volumes:
      - /opt/home-assistant/config:/config
      - /usr/bin/hddtemp:/usr/bin/hddtemp
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8123"]
      interval: 30s
      timeout: 10s
      retries: 6
    restart: unless-stopped
    privileged: true
    network_mode: host
```

</details>

"Oh but why would you run Home Assistant to pull data?" Because the power measuring device of my choice (Sonoff S31) runs ESPHome, since I actually use it in my main Home Assistant instance. This was the easiest (and simpliest to me) option, since I do not have MQTT configured for my ESPHome devices, and I am not interested in reflashing a device purely for the sole purpose of adding MQTT (even though I do already have a fully configured, operational, and active MQTT broker, but that is getting off topic).

<details>
<summary>Send data from Home Assistant to InfluxDB</summary>
<br>

```yaml
automation:
config:
frontend:
history:
homeassistant:
http:
  use_x_forwarded_for: true
  trusted_proxies:
  ip_ban_enabled: true  # use this to enable auto IP ban
  login_attempts_threshold: 3  # set the number of allowed login attempts
influxdb:
  api_version: 2
  ssl: false
  host: localhost
  port: 8086
  token: homeassistant
  organization: 0c6e520e416d5f51
  bucket: home
  tags:
    source: HA
logbook:
recorder:
  purge_keep_days: 30
  commit_interval: 30
  exclude:
    domains:
      - camera
      - image
      - media_player
      - update
scene:
script:
```

</details>

(I removed `default_config` since I already have a primary Home Assistant instance. I did not want any of my schenanigans here to take it offline for everyone else, so I made a minimal (but not minimal as it could have been I realize) configuration and added only my ESPHome device)

<details>
<summary>Telegraf to send system metrics to InfluxDB</summary>
<br>

```
[global_tags]
[agent]
  interval = "10s"
  round_interval = true
  metric_batch_size = 1000
  metric_buffer_limit = 10000
  collection_jitter = "0s"
  flush_interval = "10s"
  flush_jitter = "0s"
  precision = "0s"

[[outputs.influxdb]]
  urls = ["http://127.0.0.1:8086"]
  database = "home"
  username = "homeassistant"
  password = "homeassistant"

[[inputs.cpu]]
  percpu = true
  totalcpu = true
  collect_cpu_time = false
  report_active = false
  core_tags = false

[[inputs.diskio]]
  devices = ["*"]
  skip_serial_number = false

# Read temperature of P1 from LSI 2616-16e
[[inputs.exec]]
  commands = ["sh /etc/telegraf/lsiutil-p1-temp-c.sh"]
  timeout = "7s"
  data_format = "influx"

# Read temperature of P2 from LSI 2616-16e
[[inputs.exec]]
  commands = ["sh /etc/telegraf/lsiutil-p2-temp-c.sh"]
  timeout = "7s"
  data_format = "influx"

# Read iostat of each hard drive
[[inputs.exec]]
  commands = ["sh /etc/telegraf/iostat.sh"]
  timeout = "7s"
  data_format = "influx"

[[inputs.mem]]

[[inputs.system]]

[[inputs.linux_cpu]]
    host_sys = "/sys"
    metrics = ["cpufreq", "thermal"]

[[inputs.sensors]]
    remove_numbers = false
    timeout = "7s"

[[inputs.smartctl]]
    use_sudo = true

[[inputs.temp]]
    metric_format = "v2"
    add_device_tag = true
```

</details>

Is the username and password still in the config? Yes! Read what it is set to and you will understand why it does not matter given the situation. Also, you will need to configure `sudoers` to allow the `telegraf` user to execute `smartctl` and `lsiutil` as root. In my case, this is to allow Telegraf to read the temperature of my storage devices, and also of the LSI HBA.

<details>
<summary>/etc/telegraf/iostat.sh</summary>
<br>

```sh
iostat -my -j ID -d 1 1 | grep . | tail -n +3 | awk '{print $8 " " $2 " " $3}' | tr "," "." | while read device MB_reads MB_wrtns
do
    echo "iostat,device=$device MB_reads=$MB_reads"
    echo "iostat,device=$device MB_wrtns=$MB_wrtns"
done
```

</details>

Credit to the [custom IOStat Telegraf plugin script](https://github.com/monitoring-tools/telegraf-plugins/blob/master/iostat/iostat.sh) for guiding me to an idea for how to achieve the exact metrics I was looking for.

<details>
<summary>/etc/telegraf/lsiutil-p1-temp-c.sh</summary>
<br>

```sh
#!/bin/sh
echo lsiutil,device=p1 p1=$(( 16#$( sudo lsiutil -p1 -a 25,2,0,0 | grep IOCTemperature: | cut -dx -f2 ) ))i
```

</details>

Credit to Reddit user `Curious-Region7488` that came up with this formula. Change `-p1` to `-p2` to read the second SAS2308 controller. Could I have made this a single script? Yes. Do it yourself if you are so inclined. See the [InfluxDB Line protocol documentation](https://docs.influxdb.com/influxdb/v2/reference/syntax/line-protocol/) to read more in-depth as to why the command changed to what it did.

<details>
<summary>Grafana Docker Compose</summary>
<br>

```yaml
services:
  grafana:
    image: grafana/grafana:12.0.0
    user: '0'
    container_name: grafana
    restart: unless-stopped
    ports:
      - 3000:3000
    volumes:
      - /opt/grafana/data:/var/lib/grafana
```

</details>

(This is largly based from [Grafana's own Docker Compose documentation](https://grafana.com/docs/grafana/latest/setup-grafana/installation/docker/#run-grafana-via-docker-compose))

<details>
<summary>Grafana Dashboard</summary>
<br>

```json
{
  "__inputs": [
    {
      "name": "DS_INFLUXDB",
      "label": "influxdb",
      "description": "",
      "type": "datasource",
      "pluginId": "influxdb",
      "pluginName": "InfluxDB"
    }
  ],
  "__elements": {},
  "__requires": [
    {
      "type": "grafana",
      "id": "grafana",
      "name": "Grafana",
      "version": "12.0.0"
    },
    {
      "type": "datasource",
      "id": "influxdb",
      "name": "InfluxDB",
      "version": "1.0.0"
    },
    {
      "type": "panel",
      "id": "timeseries",
      "name": "Time series",
      "version": ""
    }
  ],
  "annotations": {
    "list": [
      {
        "builtIn": 1,
        "datasource": {
          "type": "grafana",
          "uid": "-- Grafana --"
        },
        "enable": true,
        "hide": true,
        "iconColor": "rgba(0, 211, 255, 1)",
        "name": "Annotations & Alerts",
        "type": "dashboard"
      }
    ]
  },
  "description": "idk this is my first time ever",
  "editable": true,
  "fiscalYearStartMonth": 0,
  "graphTooltip": 0,
  "id": null,
  "links": [],
  "panels": [
    {
      "datasource": {
        "type": "influxdb",
        "uid": "${DS_INFLUXDB}"
      },
      "description": "Fan speed measured by on-board it87 chip",
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds",
            "seriesBy": "min"
          },
          "custom": {
            "axisBorderShow": false,
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisGridShow": true,
            "axisLabel": "",
            "axisPlacement": "auto",
            "axisSoftMin": 500,
            "barAlignment": 1,
            "barWidthFactor": 0.6,
            "drawStyle": "line",
            "fillOpacity": 50,
            "gradientMode": "opacity",
            "hideFrom": {
              "legend": false,
              "tooltip": false,
              "viz": false
            },
            "insertNulls": false,
            "lineInterpolation": "linear",
            "lineStyle": {
              "fill": "solid"
            },
            "lineWidth": 3,
            "pointSize": 5,
            "scaleDistribution": {
              "type": "linear"
            },
            "showPoints": "auto",
            "spanNulls": true,
            "stacking": {
              "group": "A",
              "mode": "none"
            },
            "thresholdsStyle": {
              "mode": "off"
            }
          },
          "fieldMinMax": false,
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "red"
              },
              {
                "color": "green",
                "value": 200
              }
            ]
          },
          "unit": "rotrpm"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 10,
        "w": 4,
        "x": 0,
        "y": 0
      },
      "id": 2,
      "options": {
        "legend": {
          "calcs": [],
          "displayMode": "list",
          "placement": "bottom",
          "showLegend": true
        },
        "timezone": [
          "browser"
        ],
        "tooltip": {
          "hideZeros": false,
          "mode": "single",
          "sort": "none"
        }
      },
      "pluginVersion": "12.0.0",
      "targets": [
        {
          "alias": "Fan 1 (CPU)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "sensors",
          "orderByTime": "ASC",
          "policy": "default",
          "refId": "A",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "fan1_input"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "mean"
              }
            ]
          ],
          "tags": []
        },
        {
          "alias": "Fan 2 (SYS_FAN1)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "sensors",
          "orderByTime": "ASC",
          "policy": "default",
          "refId": "B",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "fan2_input"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "mean"
              }
            ]
          ],
          "tags": []
        },
        {
          "alias": "Fan 3 (SYS_FAN2)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "sensors",
          "orderByTime": "ASC",
          "policy": "default",
          "refId": "C",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "fan3_input"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "mean"
              }
            ]
          ],
          "tags": []
        },
        {
          "alias": "Fan 4 (SYS_FAN3_PUMP)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "sensors",
          "orderByTime": "ASC",
          "policy": "default",
          "refId": "D",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "fan4_input"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "mean"
              }
            ]
          ],
          "tags": []
        },
        {
          "alias": "Fan 5 (CPU_OPT)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "sensors",
          "orderByTime": "ASC",
          "policy": "default",
          "refId": "E",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "fan5_input"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "mean"
              }
            ]
          ],
          "tags": []
        }
      ],
      "title": "Fans",
      "type": "timeseries"
    },
    {
      "datasource": {
        "type": "influxdb",
        "uid": "${DS_INFLUXDB}"
      },
      "description": "",
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic"
          },
          "custom": {
            "axisBorderShow": false,
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisGridShow": true,
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "barWidthFactor": 0.6,
            "drawStyle": "line",
            "fillOpacity": 50,
            "gradientMode": "none",
            "hideFrom": {
              "legend": false,
              "tooltip": false,
              "viz": false
            },
            "insertNulls": false,
            "lineInterpolation": "linear",
            "lineWidth": 3,
            "pointSize": 5,
            "scaleDistribution": {
              "type": "linear"
            },
            "showPoints": "auto",
            "spanNulls": false,
            "stacking": {
              "group": "A",
              "mode": "none"
            },
            "thresholdsStyle": {
              "mode": "off"
            }
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green"
              }
            ]
          }
        },
        "overrides": []
      },
      "gridPos": {
        "h": 5,
        "w": 4,
        "x": 4,
        "y": 0
      },
      "id": 4,
      "options": {
        "legend": {
          "calcs": [],
          "displayMode": "list",
          "placement": "bottom",
          "showLegend": true
        },
        "timezone": [
          "browser"
        ],
        "tooltip": {
          "hideZeros": false,
          "mode": "single",
          "sort": "none"
        }
      },
      "pluginVersion": "12.0.0",
      "targets": [
        {
          "alias": "15-Minute",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "system",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "C",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "load15"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": []
        },
        {
          "alias": "5-Minute",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "system",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "B",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "load5"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": []
        },
        {
          "alias": "1-Minute",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "measurement": "system",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "A",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "load1"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": []
        }
      ],
      "title": "System Load",
      "type": "timeseries"
    },
    {
      "datasource": {
        "type": "influxdb",
        "uid": "${DS_INFLUXDB}"
      },
      "description": "",
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds",
            "seriesBy": "min"
          },
          "custom": {
            "axisBorderShow": false,
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisGridShow": true,
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 1,
            "barWidthFactor": 0.6,
            "drawStyle": "line",
            "fillOpacity": 50,
            "gradientMode": "opacity",
            "hideFrom": {
              "legend": false,
              "tooltip": false,
              "viz": false
            },
            "insertNulls": false,
            "lineInterpolation": "linear",
            "lineStyle": {
              "fill": "solid"
            },
            "lineWidth": 3,
            "pointSize": 5,
            "scaleDistribution": {
              "type": "linear"
            },
            "showPoints": "auto",
            "spanNulls": true,
            "stacking": {
              "group": "A",
              "mode": "none"
            },
            "thresholdsStyle": {
              "mode": "area"
            }
          },
          "fieldMinMax": false,
          "mappings": [],
          "max": 20,
          "min": 110,
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green"
              },
              {
                "color": "#EAB839",
                "value": 45
              },
              {
                "color": "orange",
                "value": 60
              },
              {
                "color": "red",
                "value": 80
              }
            ]
          },
          "unit": "celsius"
        },
        "overrides": [
          {
            "__systemRef": "hideSeriesFrom",
            "matcher": {
              "id": "byNames",
              "options": {
                "mode": "exclude",
                "names": [
                  "LSI 9216-16e P2",
                  "LSI 9216-16e P1"
                ],
                "prefix": "All except:",
                "readOnly": true
              }
            },
            "properties": [
              {
                "id": "custom.hideFrom",
                "value": {
                  "legend": false,
                  "tooltip": false,
                  "viz": true
                }
              }
            ]
          }
        ]
      },
      "gridPos": {
        "h": 10,
        "w": 8,
        "x": 8,
        "y": 0
      },
      "id": 3,
      "options": {
        "legend": {
          "calcs": [],
          "displayMode": "list",
          "placement": "bottom",
          "showLegend": true
        },
        "timezone": [
          "browser"
        ],
        "tooltip": {
          "hideZeros": false,
          "mode": "single",
          "sort": "none"
        }
      },
      "pluginVersion": "12.0.0",
      "targets": [
        {
          "alias": "CPU Package",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "temp",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "A",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "temp"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "sensor::tag",
              "operator": "=",
              "value": "coretemp_package_id_0"
            }
          ]
        },
        {
          "alias": "Apple SSD SM0512F",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "smartctl",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "B",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "temperature"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "model::tag",
              "operator": "=",
              "value": "APPLE SSD SM0512F"
            }
          ]
        },
        {
          "alias": "Seagate 500GB (1)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "smartctl",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "C",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "temperature"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "model::tag",
              "operator": "=",
              "value": "ST500DM002-1BD142"
            }
          ]
        },
        {
          "alias": "Seagate 500GB (2)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "smartctl",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "D",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "temperature"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "model::tag",
              "operator": "=",
              "value": "ST3500410AS"
            }
          ]
        },
        {
          "alias": "Seagate 1TB (3)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "smartctl",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "E",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "temperature"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "model::tag",
              "operator": "=",
              "value": "ST1000DM003-1ER162"
            }
          ]
        },
        {
          "alias": "Samsung 1TB (4)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "smartctl",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "G",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "temperature"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "model::tag",
              "operator": "=",
              "value": "SAMSUNG HD103SI"
            }
          ]
        },
        {
          "alias": "Seagate 1TB (5)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "smartctl",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "H",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "temperature"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "model::tag",
              "operator": "=",
              "value": "ST31000524AS"
            }
          ]
        },
        {
          "alias": "Seagate 1TB (6)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "smartctl",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "I",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "temperature"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "model::tag",
              "operator": "=",
              "value": "ST31000340AS"
            },
            {
              "condition": "AND",
              "key": "serial::tag",
              "operator": "=",
              "value": "5QJ0NMM1"
            }
          ]
        },
        {
          "alias": "Seagate 1TB (7)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "smartctl",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "J",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "temperature"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "model::tag",
              "operator": "=",
              "value": "ST31000340AS"
            },
            {
              "condition": "AND",
              "key": "serial::tag",
              "operator": "=",
              "value": "6QJ04MKS"
            }
          ]
        },
        {
          "alias": "Seagate 1TB (8)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "smartctl",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "K",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "temperature"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "model::tag",
              "operator": "=",
              "value": "ST31000333AS"
            }
          ]
        },
        {
          "alias": "Seagate 2TB (9)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "smartctl",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "F",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "temperature"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "model::tag",
              "operator": "=",
              "value": "ST32000641AS"
            },
            {
              "condition": "AND",
              "key": "serial::tag",
              "operator": "=",
              "value": "9WM6V6ZZ"
            }
          ]
        },
        {
          "alias": "Seagate 2TB (10)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "smartctl",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "L",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "temperature"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "model::tag",
              "operator": "=",
              "value": "ST2000DM001-1CH164"
            }
          ]
        },
        {
          "alias": "Seagate 2TB (11)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "smartctl",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "M",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "temperature"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "model::tag",
              "operator": "=",
              "value": "ST2000DM001-9YN164"
            },
            {
              "condition": "AND",
              "key": "serial::tag",
              "operator": "=",
              "value": "S1E0146Q"
            }
          ]
        },
        {
          "alias": "Seagate 2TB (12)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "smartctl",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "N",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "temperature"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "model::tag",
              "operator": "=",
              "value": "ST2000DL003-9VT166"
            },
            {
              "condition": "AND",
              "key": "serial::tag",
              "operator": "=",
              "value": "6YD0J8GT"
            }
          ]
        },
        {
          "alias": "Seagate 2TB (13)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "smartctl",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "O",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "temperature"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "model::tag",
              "operator": "=",
              "value": "ST2000DL003-9VT166"
            },
            {
              "condition": "AND",
              "key": "serial::tag",
              "operator": "=",
              "value": "6YD0LKS5"
            }
          ]
        },
        {
          "alias": "Seagate 2TB (14)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "smartctl",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "P",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "temperature"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "model::tag",
              "operator": "=",
              "value": "ST2000DM001-9YN164"
            },
            {
              "condition": "AND",
              "key": "serial::tag",
              "operator": "=",
              "value": "Z3405GDW"
            }
          ]
        },
        {
          "alias": "Seagate 2TB (15)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "smartctl",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "Q",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "temperature"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "model::tag",
              "operator": "=",
              "value": "ST32000641AS"
            },
            {
              "condition": "AND",
              "key": "serial::tag",
              "operator": "=",
              "value": "9WM6LPZE"
            }
          ]
        },
        {
          "alias": "Seagate 2TB (16)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "smartctl",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "R",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "temperature"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "model::tag",
              "operator": "=",
              "value": "ST32000641AS"
            },
            {
              "condition": "AND",
              "key": "serial::tag",
              "operator": "=",
              "value": "9WM3TBH3"
            }
          ]
        },
        {
          "alias": "LSI 9216-16e P1",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "lsiutil",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "S",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "p1"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": []
        },
        {
          "alias": "LSI 9216-16e P2",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "lsiutil",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "T",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "p2"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": []
        }
      ],
      "title": "Temperature",
      "type": "timeseries"
    },
    {
      "datasource": {
        "type": "influxdb",
        "uid": "${DS_INFLUXDB}"
      },
      "description": "",
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic-by-name",
            "seriesBy": "min"
          },
          "custom": {
            "axisBorderShow": false,
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisGridShow": true,
            "axisLabel": "",
            "axisPlacement": "auto",
            "axisSoftMax": 50,
            "axisSoftMin": 0,
            "barAlignment": 1,
            "barWidthFactor": 0.6,
            "drawStyle": "line",
            "fillOpacity": 50,
            "gradientMode": "opacity",
            "hideFrom": {
              "legend": false,
              "tooltip": false,
              "viz": false
            },
            "insertNulls": false,
            "lineInterpolation": "linear",
            "lineStyle": {
              "fill": "solid"
            },
            "lineWidth": 3,
            "pointSize": 5,
            "scaleDistribution": {
              "type": "linear"
            },
            "showPoints": "auto",
            "spanNulls": true,
            "stacking": {
              "group": "A",
              "mode": "none"
            },
            "thresholdsStyle": {
              "mode": "off"
            }
          },
          "fieldMinMax": false,
          "mappings": [],
          "thresholds": {
            "mode": "percentage",
            "steps": [
              {
                "color": "green"
              }
            ]
          },
          "unit": "MiBs"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 10,
        "w": 8,
        "x": 16,
        "y": 0
      },
      "id": 10,
      "options": {
        "legend": {
          "calcs": [],
          "displayMode": "list",
          "placement": "bottom",
          "showLegend": true
        },
        "timezone": [
          "browser"
        ],
        "tooltip": {
          "hideZeros": false,
          "mode": "single",
          "sort": "none"
        }
      },
      "pluginVersion": "12.0.0",
      "targets": [
        {
          "alias": "Apple SSD SM0512F",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "iostat",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "B",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "MB_reads"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "device::tag",
              "operator": "=",
              "value": "ata-APPLE_SSD_SM0512F_S1K5NYBDC74074"
            }
          ]
        },
        {
          "alias": "Seagate 500GB (1)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "iostat",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "C",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "MB_reads"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "device::tag",
              "operator": "=",
              "value": "ata-ST500DM002"
            }
          ]
        },
        {
          "alias": "Seagate 500GB (2)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "iostat",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "D",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "MB_reads"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "device::tag",
              "operator": "=",
              "value": "ata-ST3500410AS"
            }
          ]
        },
        {
          "alias": "Seagate 1TB (3)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "iostat",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "E",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "MB_reads"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "device::tag",
              "operator": "=",
              "value": "ata-ST1000DM003"
            }
          ]
        },
        {
          "alias": "Samsung 1TB (4)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "iostat",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "G",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "MB_reads"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "device::tag",
              "operator": "=",
              "value": "ata-SAMSUNG_HD103SI"
            }
          ]
        },
        {
          "alias": "Seagate 1TB (5)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "iostat",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "H",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "MB_reads"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "device::tag",
              "operator": "=",
              "value": "ata-ST31000524AS"
            }
          ]
        },
        {
          "alias": "Seagate 1TB (6)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "iostat",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "I",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "MB_reads"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "device::tag",
              "operator": "=",
              "value": "ata-ST31000340AS"
            }
          ]
        },
        {
          "alias": "Seagate 1TB (7)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "iostat",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "J",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "MB_reads"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "device::tag",
              "operator": "=",
              "value": "ata-ST31000340AS"
            }
          ]
        },
        {
          "alias": "Seagate 1TB (8)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "iostat",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "K",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "MB_reads"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "device::tag",
              "operator": "=",
              "value": "ata-ST31000333AS"
            }
          ]
        },
        {
          "alias": "Seagate 2TB (9)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "iostat",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "F",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "MB_reads"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "device::tag",
              "operator": "=",
              "value": "ata-ST32000641AS"
            }
          ]
        },
        {
          "alias": "Seagate 2TB (10)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "iostat",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "L",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "MB_reads"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "device::tag",
              "operator": "=",
              "value": "ata-ST2000DM001"
            }
          ]
        },
        {
          "alias": "Seagate 2TB (11)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "iostat",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "M",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "MB_reads"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "device::tag",
              "operator": "=",
              "value": "ata-ST2000DM001"
            }
          ]
        },
        {
          "alias": "Seagate 2TB (12)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "iostat",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "N",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "MB_reads"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "device::tag",
              "operator": "=",
              "value": "ata-ST2000DL003"
            }
          ]
        },
        {
          "alias": "Seagate 2TB (13)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "iostat",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "O",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "MB_reads"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "condition": "AND",
              "key": "device::tag",
              "operator": "=",
              "value": "ata-ST2000DL003"
            }
          ]
        },
        {
          "alias": "Seagate 2TB (14)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "iostat",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "P",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "MB_reads"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "condition": "AND",
              "key": "device::tag",
              "operator": "=",
              "value": "ata-ST2000DM001"
            }
          ]
        },
        {
          "alias": "Seagate 2TB (15)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "iostat",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "Q",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "MB_reads"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "condition": "AND",
              "key": "device::tag",
              "operator": "=",
              "value": "ata-ST32000641AS"
            }
          ]
        },
        {
          "alias": "Seagate 2TB (16)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "iostat",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "R",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "MB_reads"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "condition": "AND",
              "key": "device::tag",
              "operator": "=",
              "value": "ata-ST32000641AS"
            }
          ]
        }
      ],
      "title": "Disk Read Speed",
      "type": "timeseries"
    },
    {
      "datasource": {
        "type": "influxdb",
        "uid": "${DS_INFLUXDB}"
      },
      "description": "",
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic"
          },
          "custom": {
            "axisBorderShow": false,
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisGridShow": true,
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "barWidthFactor": 0.6,
            "drawStyle": "line",
            "fillOpacity": 50,
            "gradientMode": "none",
            "hideFrom": {
              "legend": false,
              "tooltip": false,
              "viz": false
            },
            "insertNulls": false,
            "lineInterpolation": "linear",
            "lineWidth": 3,
            "pointSize": 5,
            "scaleDistribution": {
              "type": "linear"
            },
            "showPoints": "auto",
            "spanNulls": false,
            "stacking": {
              "group": "A",
              "mode": "none"
            },
            "thresholdsStyle": {
              "mode": "off"
            }
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green"
              }
            ]
          },
          "unit": "decbytes"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 5,
        "w": 4,
        "x": 4,
        "y": 5
      },
      "id": 9,
      "options": {
        "legend": {
          "calcs": [],
          "displayMode": "list",
          "placement": "bottom",
          "showLegend": true
        },
        "timezone": [
          "browser"
        ],
        "tooltip": {
          "hideZeros": false,
          "mode": "single",
          "sort": "none"
        }
      },
      "pluginVersion": "12.0.0",
      "targets": [
        {
          "alias": "Used",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "mem",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "A",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "used"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": []
        }
      ],
      "title": "Memory",
      "type": "timeseries"
    },
    {
      "datasource": {
        "type": "influxdb",
        "uid": "${DS_INFLUXDB}"
      },
      "description": "Power usage measured from the wall using a Sonoff S31. This is inclusive of the AP also powered off the surge protector.",
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic-by-name"
          },
          "custom": {
            "axisBorderShow": false,
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisGridShow": true,
            "axisLabel": "",
            "axisPlacement": "auto",
            "axisSoftMax": 200,
            "axisSoftMin": 5,
            "barAlignment": 0,
            "barWidthFactor": 0.6,
            "drawStyle": "line",
            "fillOpacity": 50,
            "gradientMode": "none",
            "hideFrom": {
              "legend": false,
              "tooltip": false,
              "viz": false
            },
            "insertNulls": false,
            "lineInterpolation": "linear",
            "lineWidth": 3,
            "pointSize": 5,
            "scaleDistribution": {
              "type": "linear"
            },
            "showPoints": "auto",
            "spanNulls": false,
            "stacking": {
              "group": "A",
              "mode": "none"
            },
            "thresholdsStyle": {
              "mode": "off"
            }
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green"
              }
            ]
          },
          "unit": "watt"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 5,
        "w": 4,
        "x": 0,
        "y": 10
      },
      "id": 1,
      "options": {
        "legend": {
          "calcs": [],
          "displayMode": "list",
          "placement": "bottom",
          "showLegend": true
        },
        "timezone": [
          "browser"
        ],
        "tooltip": {
          "hideZeros": false,
          "mode": "single",
          "sort": "none"
        }
      },
      "pluginVersion": "12.0.0",
      "targets": [
        {
          "alias": "Sonoff S31",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "measurement": "W",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "A",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "value"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": []
        }
      ],
      "title": "Watts",
      "type": "timeseries"
    },
    {
      "datasource": {
        "type": "influxdb",
        "uid": "${DS_INFLUXDB}"
      },
      "description": "",
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic"
          },
          "custom": {
            "axisBorderShow": false,
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisGridShow": true,
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "barWidthFactor": 0.6,
            "drawStyle": "line",
            "fillOpacity": 50,
            "gradientMode": "none",
            "hideFrom": {
              "legend": false,
              "tooltip": false,
              "viz": false
            },
            "insertNulls": false,
            "lineInterpolation": "linear",
            "lineWidth": 3,
            "pointSize": 5,
            "scaleDistribution": {
              "type": "linear"
            },
            "showPoints": "auto",
            "spanNulls": false,
            "stacking": {
              "group": "A",
              "mode": "none"
            },
            "thresholdsStyle": {
              "mode": "off"
            }
          },
          "mappings": [],
          "max": 4200000,
          "min": 800000,
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green"
              }
            ]
          },
          "unit": "rotkhz"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 5,
        "w": 4,
        "x": 4,
        "y": 10
      },
      "id": 6,
      "options": {
        "legend": {
          "calcs": [],
          "displayMode": "list",
          "placement": "bottom",
          "showLegend": true
        },
        "timezone": [
          "browser"
        ],
        "tooltip": {
          "hideZeros": false,
          "mode": "single",
          "sort": "none"
        }
      },
      "pluginVersion": "12.0.0",
      "targets": [
        {
          "alias": "Core 0",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "linux_cpu",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "A",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "scaling_cur_freq"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "cpu::tag",
              "operator": "=",
              "value": "0"
            }
          ]
        },
        {
          "alias": "Core 1",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "linux_cpu",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "B",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "scaling_cur_freq"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "cpu::tag",
              "operator": "=",
              "value": "1"
            }
          ]
        },
        {
          "alias": "Core 2",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "linux_cpu",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "C",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "scaling_cur_freq"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "cpu::tag",
              "operator": "=",
              "value": "2"
            }
          ]
        },
        {
          "alias": "Core 3",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "linux_cpu",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "D",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "scaling_cur_freq"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "cpu::tag",
              "operator": "=",
              "value": "3"
            }
          ]
        },
        {
          "alias": "Core 4",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "linux_cpu",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "E",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "scaling_cur_freq"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "cpu::tag",
              "operator": "=",
              "value": "4"
            }
          ]
        },
        {
          "alias": "Core 5",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "linux_cpu",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "F",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "scaling_cur_freq"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "cpu::tag",
              "operator": "=",
              "value": "5"
            }
          ]
        },
        {
          "alias": "Core 6",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "linux_cpu",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "G",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "scaling_cur_freq"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "cpu::tag",
              "operator": "=",
              "value": "6"
            }
          ]
        },
        {
          "alias": "Core 7",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "linux_cpu",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "H",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "scaling_cur_freq"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "cpu::tag",
              "operator": "=",
              "value": "7"
            }
          ]
        }
      ],
      "title": "CPU Frequency",
      "type": "timeseries"
    },
    {
      "datasource": {
        "type": "influxdb",
        "uid": "${DS_INFLUXDB}"
      },
      "description": "",
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "continuous-GrYlRd",
            "seriesBy": "min"
          },
          "custom": {
            "axisBorderShow": false,
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisGridShow": true,
            "axisLabel": "",
            "axisPlacement": "auto",
            "axisSoftMax": 50,
            "axisSoftMin": 0,
            "barAlignment": 1,
            "barWidthFactor": 0.6,
            "drawStyle": "line",
            "fillOpacity": 50,
            "gradientMode": "opacity",
            "hideFrom": {
              "legend": false,
              "tooltip": false,
              "viz": false
            },
            "insertNulls": false,
            "lineInterpolation": "linear",
            "lineStyle": {
              "fill": "solid"
            },
            "lineWidth": 3,
            "pointSize": 5,
            "scaleDistribution": {
              "type": "linear"
            },
            "showPoints": "auto",
            "spanNulls": true,
            "stacking": {
              "group": "A",
              "mode": "none"
            },
            "thresholdsStyle": {
              "mode": "off"
            }
          },
          "fieldMinMax": false,
          "mappings": [],
          "max": 0,
          "min": 100,
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green"
              },
              {
                "color": "#EAB839",
                "value": 50
              },
              {
                "color": "orange",
                "value": 65
              },
              {
                "color": "red",
                "value": 95
              }
            ]
          },
          "unit": "percent"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 10,
        "w": 8,
        "x": 8,
        "y": 10
      },
      "id": 7,
      "options": {
        "legend": {
          "calcs": [],
          "displayMode": "list",
          "placement": "bottom",
          "showLegend": true
        },
        "timezone": [
          "browser"
        ],
        "tooltip": {
          "hideZeros": false,
          "mode": "single",
          "sort": "none"
        }
      },
      "pluginVersion": "12.0.0",
      "targets": [
        {
          "alias": "Apple SSD SM0512F",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "diskio",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "B",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "io_util"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "serial::tag",
              "operator": "=",
              "value": "APPLE_SSD_SM0512F_S1K5NYBDC74074"
            }
          ]
        },
        {
          "alias": "Seagate 500GB (1)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "diskio",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "C",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "io_util"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "serial::tag",
              "operator": "=",
              "value": "ST500DM002-1BD142_Z6E7QCFV"
            }
          ]
        },
        {
          "alias": "Seagate 500GB (2)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "diskio",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "D",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "io_util"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "serial::tag",
              "operator": "=",
              "value": "ST3500410AS_5VM0E238"
            }
          ]
        },
        {
          "alias": "Seagate 1TB (3)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "diskio",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "E",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "io_util"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "serial::tag",
              "operator": "=",
              "value": "ST1000DM003-1ER162_Z4Y1SVLR"
            }
          ]
        },
        {
          "alias": "Samsung 1TB (4)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "diskio",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "G",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "io_util"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "serial::tag",
              "operator": "=",
              "value": "SAMSUNG_HD103SI_S1XHJ9KB705901"
            }
          ]
        },
        {
          "alias": "Seagate 1TB (5)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "diskio",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "H",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "io_util"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "serial::tag",
              "operator": "=",
              "value": "ST31000524AS_6VPE5N0A"
            }
          ]
        },
        {
          "alias": "Seagate 1TB (6)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "diskio",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "I",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "io_util"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "serial::tag",
              "operator": "=",
              "value": "ST31000340AS_5QJ0NMM1"
            }
          ]
        },
        {
          "alias": "Seagate 1TB (7)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "diskio",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "J",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "io_util"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "serial::tag",
              "operator": "=",
              "value": "ST31000340AS_6QJ04MKS"
            }
          ]
        },
        {
          "alias": "Seagate 1TB (8)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "diskio",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "K",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "io_util"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "serial::tag",
              "operator": "=",
              "value": "ST31000333AS_6TE0HRYH"
            }
          ]
        },
        {
          "alias": "Seagate 2TB (9)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "diskio",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "F",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "io_util"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "serial::tag",
              "operator": "=",
              "value": "ST32000641AS_9WM6V6ZZ"
            }
          ]
        },
        {
          "alias": "Seagate 2TB (10)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "diskio",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "L",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "io_util"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "serial::tag",
              "operator": "=",
              "value": "ST2000DM001-1CH164_Z1E3BJ1R"
            }
          ]
        },
        {
          "alias": "Seagate 2TB (11)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "diskio",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "M",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "io_util"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "serial::tag",
              "operator": "=",
              "value": "ST2000DM001-9YN164_S1E0146Q"
            }
          ]
        },
        {
          "alias": "Seagate 2TB (12)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "diskio",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "N",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "io_util"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "serial::tag",
              "operator": "=",
              "value": "ST2000DL003-9VT166_6YD0J8GT"
            }
          ]
        },
        {
          "alias": "Seagate 2TB (13)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "diskio",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "O",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "io_util"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "condition": "AND",
              "key": "serial::tag",
              "operator": "=",
              "value": "ST2000DL003-9VT166_6YD0LKS5"
            }
          ]
        },
        {
          "alias": "Seagate 2TB (14)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "diskio",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "P",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "io_util"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "condition": "AND",
              "key": "serial::tag",
              "operator": "=",
              "value": "ST2000DM001-9YN164_Z3405GDW"
            }
          ]
        },
        {
          "alias": "Seagate 2TB (15)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "diskio",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "Q",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "io_util"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "condition": "AND",
              "key": "serial::tag",
              "operator": "=",
              "value": "ST32000641AS_9WM6LPZE"
            }
          ]
        },
        {
          "alias": "Seagate 2TB (16)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "diskio",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "R",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "io_util"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "condition": "AND",
              "key": "serial::tag",
              "operator": "=",
              "value": "ST32000641AS_9WM3TBH3"
            }
          ]
        }
      ],
      "title": "Disk Utilization",
      "type": "timeseries"
    },
    {
      "datasource": {
        "type": "influxdb",
        "uid": "${DS_INFLUXDB}"
      },
      "description": "",
      "fieldConfig": {
        "defaults": {
          "color": {
            "fixedColor": "orange",
            "mode": "palette-classic",
            "seriesBy": "last"
          },
          "custom": {
            "axisBorderShow": false,
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisGridShow": true,
            "axisLabel": "",
            "axisPlacement": "auto",
            "axisSoftMax": 50,
            "axisSoftMin": 0,
            "barAlignment": 1,
            "barWidthFactor": 0.6,
            "drawStyle": "line",
            "fillOpacity": 50,
            "gradientMode": "opacity",
            "hideFrom": {
              "legend": false,
              "tooltip": false,
              "viz": false
            },
            "insertNulls": false,
            "lineInterpolation": "linear",
            "lineStyle": {
              "fill": "solid"
            },
            "lineWidth": 3,
            "pointSize": 5,
            "scaleDistribution": {
              "type": "linear"
            },
            "showPoints": "auto",
            "spanNulls": true,
            "stacking": {
              "group": "A",
              "mode": "none"
            },
            "thresholdsStyle": {
              "mode": "off"
            }
          },
          "fieldMinMax": false,
          "mappings": [],
          "thresholds": {
            "mode": "percentage",
            "steps": [
              {
                "color": "green"
              }
            ]
          },
          "unit": "MiBs"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 10,
        "w": 8,
        "x": 16,
        "y": 10
      },
      "id": 11,
      "options": {
        "legend": {
          "calcs": [],
          "displayMode": "list",
          "placement": "bottom",
          "showLegend": true
        },
        "timezone": [
          "browser"
        ],
        "tooltip": {
          "hideZeros": false,
          "mode": "single",
          "sort": "none"
        }
      },
      "pluginVersion": "12.0.0",
      "targets": [
        {
          "alias": "Apple SSD SM0512F",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "iostat",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "B",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "MB_wrtns"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "device::tag",
              "operator": "=",
              "value": "ata-APPLE_SSD_SM0512F_S1K5NYBDC74074"
            }
          ]
        },
        {
          "alias": "Seagate 500GB (1)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "iostat",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "C",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "MB_wrtns"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "device::tag",
              "operator": "=",
              "value": "ata-ST500DM002"
            }
          ]
        },
        {
          "alias": "Seagate 500GB (2)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "iostat",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "D",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "MB_wrtns"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "device::tag",
              "operator": "=",
              "value": "ata-ST3500410AS"
            }
          ]
        },
        {
          "alias": "Seagate 1TB (3)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "iostat",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "E",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "MB_wrtns"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "device::tag",
              "operator": "=",
              "value": "ata-ST1000DM003"
            }
          ]
        },
        {
          "alias": "Samsung 1TB (4)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "iostat",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "G",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "MB_wrtns"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "device::tag",
              "operator": "=",
              "value": "ata-SAMSUNG_HD103SI"
            }
          ]
        },
        {
          "alias": "Seagate 1TB (5)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "iostat",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "H",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "MB_wrtns"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "device::tag",
              "operator": "=",
              "value": "ata-ST31000524AS"
            }
          ]
        },
        {
          "alias": "Seagate 1TB (6)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "iostat",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "I",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "MB_wrtns"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "device::tag",
              "operator": "=",
              "value": "ata-ST31000340AS"
            }
          ]
        },
        {
          "alias": "Seagate 1TB (7)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "iostat",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "J",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "MB_wrtns"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "device::tag",
              "operator": "=",
              "value": "ata-ST31000340AS"
            }
          ]
        },
        {
          "alias": "Seagate 1TB (8)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "iostat",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "K",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "MB_wrtns"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "device::tag",
              "operator": "=",
              "value": "ata-ST31000333AS"
            }
          ]
        },
        {
          "alias": "Seagate 2TB (9)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "iostat",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "F",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "MB_wrtns"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "device::tag",
              "operator": "=",
              "value": "ata-ST32000641AS"
            }
          ]
        },
        {
          "alias": "Seagate 2TB (10)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "iostat",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "L",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "MB_wrtns"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "device::tag",
              "operator": "=",
              "value": "ata-ST2000DM001"
            }
          ]
        },
        {
          "alias": "Seagate 2TB (11)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "iostat",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "M",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "MB_wrtns"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "device::tag",
              "operator": "=",
              "value": "ata-ST2000DM001"
            }
          ]
        },
        {
          "alias": "Seagate 2TB (12)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "iostat",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "N",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "MB_wrtns"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "key": "device::tag",
              "operator": "=",
              "value": "ata-ST2000DL003"
            }
          ]
        },
        {
          "alias": "Seagate 2TB (13)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "iostat",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "O",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "MB_wrtns"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "condition": "AND",
              "key": "device::tag",
              "operator": "=",
              "value": "ata-ST2000DL003"
            }
          ]
        },
        {
          "alias": "Seagate 2TB (14)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "iostat",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "P",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "MB_wrtns"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "condition": "AND",
              "key": "device::tag",
              "operator": "=",
              "value": "ata-ST2000DM001"
            }
          ]
        },
        {
          "alias": "Seagate 2TB (15)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "iostat",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "Q",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "MB_wrtns"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "condition": "AND",
              "key": "device::tag",
              "operator": "=",
              "value": "ata-ST32000641AS"
            }
          ]
        },
        {
          "alias": "Seagate 2TB (16)",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "iostat",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "R",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "MB_wrtns"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": [
            {
              "condition": "AND",
              "key": "device::tag",
              "operator": "=",
              "value": "ata-ST32000641AS"
            }
          ]
        }
      ],
      "title": "Disk Write Speed",
      "type": "timeseries"
    },
    {
      "datasource": {
        "type": "influxdb",
        "uid": "${DS_INFLUXDB}"
      },
      "description": "Power usage measured from the wall using a Sonoff S31. This is inclusive of the AP also powered off the surge protector.",
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds",
            "seriesBy": "last"
          },
          "custom": {
            "axisBorderShow": false,
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisGridShow": true,
            "axisLabel": "",
            "axisPlacement": "auto",
            "axisSoftMax": 1,
            "axisSoftMin": 0.9,
            "barAlignment": 0,
            "barWidthFactor": 0.6,
            "drawStyle": "line",
            "fillOpacity": 50,
            "gradientMode": "opacity",
            "hideFrom": {
              "legend": false,
              "tooltip": false,
              "viz": false
            },
            "insertNulls": false,
            "lineInterpolation": "linear",
            "lineWidth": 3,
            "pointSize": 5,
            "scaleDistribution": {
              "type": "linear"
            },
            "showPoints": "auto",
            "spanNulls": false,
            "stacking": {
              "group": "A",
              "mode": "none"
            },
            "thresholdsStyle": {
              "mode": "off"
            }
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "red"
              },
              {
                "color": "yellow",
                "value": 0.8
              },
              {
                "color": "#EAB839",
                "value": 0.9
              },
              {
                "color": "green",
                "value": 0.95
              }
            ]
          }
        },
        "overrides": [
          {
            "__systemRef": "hideSeriesFrom",
            "matcher": {
              "id": "byNames",
              "options": {
                "mode": "exclude",
                "names": [
                  "Power Factor"
                ],
                "prefix": "All except:",
                "readOnly": true
              }
            },
            "properties": [
              {
                "id": "custom.hideFrom",
                "value": {
                  "legend": false,
                  "tooltip": false,
                  "viz": true
                }
              }
            ]
          }
        ]
      },
      "gridPos": {
        "h": 5,
        "w": 4,
        "x": 0,
        "y": 15
      },
      "id": 8,
      "options": {
        "legend": {
          "calcs": [],
          "displayMode": "list",
          "placement": "bottom",
          "showLegend": true
        },
        "timezone": [
          "browser"
        ],
        "tooltip": {
          "hideZeros": false,
          "mode": "single",
          "sort": "none"
        }
      },
      "pluginVersion": "12.0.0",
      "targets": [
        {
          "alias": "Power Factor",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "measurement": "sensor.sonoff_s31_power_factor",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "A",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "value"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": []
        }
      ],
      "title": "Sonoff S31",
      "type": "timeseries"
    },
    {
      "datasource": {
        "type": "influxdb",
        "uid": "${DS_INFLUXDB}"
      },
      "description": "",
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic"
          },
          "custom": {
            "axisBorderShow": false,
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisGridShow": true,
            "axisLabel": "",
            "axisPlacement": "auto",
            "axisSoftMax": 100,
            "axisSoftMin": 0,
            "barAlignment": 0,
            "barWidthFactor": 0.6,
            "drawStyle": "line",
            "fillOpacity": 50,
            "gradientMode": "none",
            "hideFrom": {
              "legend": false,
              "tooltip": false,
              "viz": false
            },
            "insertNulls": false,
            "lineInterpolation": "linear",
            "lineWidth": 3,
            "pointSize": 5,
            "scaleDistribution": {
              "type": "linear"
            },
            "showPoints": "auto",
            "spanNulls": false,
            "stacking": {
              "group": "A",
              "mode": "normal"
            },
            "thresholdsStyle": {
              "mode": "off"
            }
          },
          "mappings": [],
          "max": 100,
          "min": 0,
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green"
              }
            ]
          },
          "unit": "percent"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 5,
        "w": 4,
        "x": 4,
        "y": 15
      },
      "id": 5,
      "options": {
        "legend": {
          "calcs": [],
          "displayMode": "list",
          "placement": "bottom",
          "showLegend": true
        },
        "timezone": [
          "browser"
        ],
        "tooltip": {
          "hideZeros": false,
          "mode": "single",
          "sort": "none"
        }
      },
      "pluginVersion": "12.0.0",
      "targets": [
        {
          "alias": "Guest",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "measurement": "cpu",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "A",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "usage_guest"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": []
        },
        {
          "alias": "Guest nice",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "cpu",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "B",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "usage_guest_nice"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": []
        },
        {
          "alias": "Interrupt",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "cpu",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "F",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "usage_irq"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": []
        },
        {
          "alias": "Nice",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "cpu",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "E",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "usage_nice"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": []
        },
        {
          "alias": "Soft Interrupt",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "cpu",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "G",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "usage_softirq"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": []
        },
        {
          "alias": "Steal",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "cpu",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "H",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "usage_steal"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": []
        },
        {
          "alias": "System",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "cpu",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "I",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "usage_system"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": []
        },
        {
          "alias": "User",
          "datasource": {
            "type": "influxdb",
            "uid": "${DS_INFLUXDB}"
          },
          "groupBy": [
            {
              "params": [
                "$__interval"
              ],
              "type": "time"
            },
            {
              "params": [
                "null"
              ],
              "type": "fill"
            }
          ],
          "hide": false,
          "measurement": "cpu",
          "orderByTime": "ASC",
          "policy": "autogen",
          "refId": "J",
          "resultFormat": "time_series",
          "select": [
            [
              {
                "params": [
                  "usage_user"
                ],
                "type": "field"
              },
              {
                "params": [],
                "type": "distinct"
              }
            ]
          ],
          "tags": []
        }
      ],
      "title": "CPU Time",
      "type": "timeseries"
    }
  ],
  "refresh": "10s",
  "schemaVersion": 41,
  "tags": [],
  "templating": {
    "list": []
  },
  "time": {
    "from": "now-30m",
    "to": "now"
  },
  "timepicker": {},
  "timezone": "browser",
  "title": "Main",
  "uid": "196d909a-d8c2-4b24-b8e1-9922157d89d8",
  "version": 50,
  "weekStart": ""
}
```

</details>

You will need to update any of the configured sources based on whatever the query should be for your InfluxDB.

I also will not be attaching how to connect your InfluxDB to Grafana. I could not find a way to express it as a configuration file, and there are plenty of guides available anyway.

Here are a couple other links I found helpful to puruse through when trying to become more familar with this.

[Linuxserver.io](https://www.linuxserver.io/blog/2017-11-25-how-to-monitor-your-server-using-grafana-influxdb-and-telegraf) has a good guide for setting up not InfluxDB, Grafana and Telegraf.

[Jeff Davis](https://jeffdavis.dev/add-unraid-to-your-grafana-dashboard/) talks about how they added Unraid metrics to Grafana.

[Ronald from vd Brink](https://vdbrink.github.io/homeassistant/homeassistant_dashboard_grafana.html) talks about how to embed Grafana graphs into Home Assistant.

[Paolo Tagliaferri](https://www.paolotagliaferri.com/home-assistant-data-persistence-and-visualization-with-grafana-and-influxdb/) talks about how to persist database values longer then the default SQL database for Home Assistant, by sending it to an external database such as InfluxDB, and then display long-term metrics using Grafana.

There are plenty other links and guides out there.

#### What are the exact specifications of your test system?

**CPU**: Intel i7-6700K, stock values everything (voltage, core ratio, etc)

**CPU Cooler**: DeepCool GAMMAXX 400, version 1 with the basic blue LED fan


**RAM**: 32GB (4x8GB) Corsair 8GB DDR4-3000 (XMP profile enabled), it is actually two separate kits but of the same model number combined together. Seems to POST stable on this board.

**Motherboard**: Gigabyte Z270X-Ultra-Gaming Rev1.0

**Power Supply**: Antec Earthwatts EA-450 Green, 80+ Bronze, works sufficiently

**Case**: Antec 300, version 1, (not the "two" model with USB 3.0 front header ports)

**Intake fan (x2)**: Stock fans from Cooler Master Hyper 212 LED Turbo (both lead into a fan splitter)

**Exhaust fan**: Rosewill 140mm, no idea what it exactly came out of

**Main HBA fan**: Corsair AF LED series, AF120, blue LED, 3-pin

**Second HBA fan**: Cooler Master JetFlo 120mm, white LED, 4-pin

**HBA**: LSI 9216-16e, Dell P/N `0TFJRW`

For cables, any **SFF-8644 to SFF-8088** is sufficient, and then since my test hard drive set physically extended into a second case, any **SFF-8088 to SFF-8087 PCI slot adapter** (and of course SFF-8087 to SATA breakout cables) will also be sufficient.

**Power monitoring**: Sonoff S31 flashed with ESPHome (it is just an ESP8266 with a relay, inline current measuring device, a button, and couple LED indicator lights, along with a serial header)

**Hard drives**:

***2x*** Seagate Barracuda Green 2TB, ST2000DL003-9VT166

***1x*** Seagate Barracuda 7200.14 500GB, ST500DM002-1BD142

***3x*** Seagate Barracuda XT 2TB, ST32000641AS

***3x*** Seagate Barracuda 7200.14 2TB, 2x ST2000DM001-9YN164, 1x ST2000DM001-1CH164

***1x*** Seagate Barracuda 7200.14 1TB, ST1000DM003-1ER162

***1x*** Samsung SpinPoint F2 EG 1TB, HD103SI

***3x*** Seagate Barracuda 7200.11 1TB, 2x ST31000340AS, 1x ST31000333AS

***1x*** Seagate Barracuda 7200.12 1TB, ST31000524AS

***1x*** Seagate Barracuda 7200.12 500GB, ST3500410AS

***1x*** Apple SSD 512GB, Samsung SSUAX label ([confusing, I know](https://beetstech.com/blog/apple-proprietary-ssd-ultimate-guide-to-specs-and-upgrades))