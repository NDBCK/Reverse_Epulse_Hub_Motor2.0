# Reverse_Epulse_Hub_Motor2.0
 Attempt to reverse engineer the inner working of the infento 2.0 epulse hub motor unit.
========================================================================================

Motivation
----------
I'm an Infento enthusiast who likes to make rides for my children.
Like the infento epulse 1.0 motor I wanted to look into the options to let it run faster than intended.
With the infento epulse 1.0 you could change the parameters of the controller or let it run at a higher voltage. 
The infento 1.0 motor drive was (I presume) a commercially available product because it was also found in ebikes.
I wanted to investigate if a comparable procedure was possible...


A Heads up
----------
Using the epulse 2.0 motor hub in a different way than intended can void warranties and should be used wisely safety wise.
I'm not responsible if things brake, get bricked or possible injuries.

There could be mistakes in the following information because a lot is based on best guesses.


The Hardware
------------
As a starting point I tried to do some basic reverse engineering (is the motor drive in the hub wheel or in the battery holder).
I found out Infento designed a custom motor drive controller "ePulse 2.0 Unit".

Looking at the used components you can get an understanding how things work:

- STM32G431 LQFP64: ARM Cortex M4, the main brains of the controller (running on 3.3V)
- DRV8323RS: 3-phase smart gate driver to control the power stage with integrated step-down buck regulator (main logic supply?)
- IRF3205 (6x): Discrete power mosfets to drive the motor (3 rails of high-low switching)
- SN65HVD232 (Marking VP232): CAN Bus Transceiver (to talk to the motor? / master-slave config)
- TLV70233DBVR (Marking QVD SOT23-5): 3,3V Lineair regulator (Not main logic supply -> max input 5,5V)
- Connector for the USB, Power button, Motor (CAN?), Key switch, Brake
- P, S, E  indicator board leds
- Serial UART console! (Three terminal pins: PB7 USART1 RX, PB6 USART1 TX, GND)
- Header for programming the STM32 (SWD interface, TC2030-CTX connector)

![Alt text](/Epulse_Unit_PCB.jpg?raw=true "PCB overview of ePulse Unit")

The battery connector seems like a 6 pin laptop battery connector of 2.54mm pitch

The Motor cable connector looks like a HIGO connector: [HG-F.M-Z1115A](https://www.higocon.com/motor-connector/hg-fm-z1115a.html)
 
The USB-C connector
-------------------
The USB connector had only GND, VCC, D+ and D- connected (so no USB3.0)
But it seems that the datalines are not connected to a USB controller or the STM32 (directly).

The D+ and D- lines are each connected to a 60 Ohm resistor, whith the other end connected to eachother.
This looks more like a typical CAN bus split termination (120 Ohms in series).

So probably that a master and slave ePulse unit communicate over the CAN protocol (when using two motor drivers and two motors).
 
The UART serial console
-----------------------
For testing purposes I fitted 10KOhm resistors in series with RX and TX using an USB to UART converter of 3,3V.

The pinout is: Infento RX, Infento TX, GND (see picture) -> to be clear the epulse controller sends data over the middle pin.
The connection settings I used are: 115200bps, 8 data bits, no parity, 1 stop bit, no flow control, LF (linefeed after a command)

At boot following messages are received:
```
Booting ePulse Unit: 
Unique ID: XXXXXXXXXXXXXXXXXXXXXXXXXX
CPU Frequency: 170000000
UNIT: Load settings from EEPROM
IO: Board LEDS - ON
IO: Power Switch leds - Red
IO: Power Switch leds - green
IO: Power Switch leds - Blue
IO: Board Voltage - 25.09
IO: Board Temperature - 27.87
IO: Motor Temperature - -34.85
IO: Speed 1 Voltage - 0.00
IO: Speed 2 Voltage - 0.00
IO: Forward/Reverse Switch - 0
IO: Brake Switch - 0
IO: Speed Select Switch - 0
IO: Power Switch - 0
DRV8323RS: REG_DC: 33:32
DRV8323RS: REG_GDHS: 819:819
DRV8323RS: REG_GDLS: 1843:1843
DRV8323RS: REG_OCPC: 859:859
DRV8323RS: REG_CSAC: 579:579
UNIT: DRV8323RS OK
MOT: Monitor enabled!
MOT: Init
MOT: Enable driver.
STM32-CS: Using ADC: 1
STM32-DRV: Stopping timer 1
STM32-DRV: Stopping timer 1
STM32-DRV: Stopping timer 1
STM32-DRV: Syncronising timers! Timer no. 1
STM32-DRV: Restarting timer 1
MOT: Align sensor.
MOT: Skip dir calib.
MOT: Skip offset calib.
MOT: Align current sense.
MOT: Success: 1
MOT: Ready.
FOC init success
Unit in Slave Mode
CAN: Bit Rate prescaler: 5
CAN: Arbitration Phase segment 1: 101
CAN: Arbitration Phase segment 2: 34
CAN: Arbitration SJW: 34
CAN: Actual Arbitration Bit Rate: 250000 bit/s
CAN: Arbitration sample point: 75.00%
CAN: Exact Arbitration Bit Rate ? yes
CAN: Data Phase segment 1: 24
CAN: Data Phase segment 2: 9
CAN: Data SJW: 9
CAN: Actual Data Bit Rate: 1000000 bit/s
CAN: Data sample point: 73.53%
CAN: Exact Data Bit Rate ? yes
CAN: Init success
```

Commands 
--------
These commands are found by fuzzying (and are thus not a complete list)

Almost all command that get a parameter value can be used to set the parameter by adding a number.

For example: 
command 'MLC' gets the current limit
command 'MLC12.345' sets the current limit to 12.345

```
Sent: '?' | Received: 'M:motor'
Basic help functionality

Sent: '@' | Received: 'Verb:on!'
Verbose mode?

Sent: 'M0.123' | Received: 'Target: 0.123'
Set target 

Sent: 'M-98.7654' | Received: 'Target: -98.765'
Set target 

Sent: 'MAD' | Received: 'PID angle| D: 0.000'
Get PID Angle D

Sent: 'MAD-1.234' | Received: 'PID angle| D: -1.234'
Set PID Angle

Sent: 'MAF' | Received: 'PID angle| Tf: 0.000'
Get  PID angle Tf

Sent: 'MAI' | Received: 'PID angle| I: 0.000'
Get PID angle I

Sent: 'MAL' | Received: 'PID angle| limit: 50.000'
Get PID angle limit

Sent: 'MAP' | Received: 'PID angle| P: 20.000'
Get PID angle P

Sent: 'MAR' | Received: 'PID angle| ramp: 0.000'
Get PID angle ramp

Sent: 'MK' | Received: 'Motor KV: 32.200'
Get Motor KV

Sent: 'MK33' | Received: 'Motor KV: 33.000'
Set Motor KV

Sent: 'MLC' | Received: 'Limits| curr: 15.000'
Get motor current limit

Sent: 'MLU' | Received: 'Limits| volt: 6.645'
Get motor volt limit?

Sent: 'MLV' | Received: 'Limits| vel: 50.000'
Get motor velocity limit


Sent: 'MDD' | Received: 'PID curr d| D: 0.000'
Sent: 'MDF' | Received: 'PID curr d| Tf: 1.000'
Sent: 'MDI' | Received: 'PID curr d| I: 0.300'
Sent: 'MDL' | Received: 'PID curr d| limit: 25.200'
Sent: 'MDP' | Received: 'PID curr d| P: 0.100'
Sent: 'MDR' | Received: 'PID curr d| ramp: 0.000'
Sent: 'MQD' | Received: 'PID curr q| D: 0.000'
Sent: 'MQF' | Received: 'PID curr q| Tf: 0.300'
Sent: 'MQI' | Received: 'PID curr q| I: 0.300'
Sent: 'MQL' | Received: 'PID curr q| limit: 25.200'
Sent: 'MQP' | Received: 'PID curr q| P: 0.100'
Sent: 'MQR' | Received: 'PID curr q| ramp: 0.000'
Read or change current (I) PID parameters

Sent: 'MVD' | Received: 'PID vel| D: 0.000'
Sent: 'MVF' | Received: 'PID vel| Tf: 0.100'
Sent: 'MVI' | Received: 'PID vel| I: 0.000'
Sent: 'MVL' | Received: 'PID vel| limit: 0.000'
Sent: 'MVP' | Received: 'PID vel| P: 10.000'
Sent: 'MVR' | Received: 'PID vel| ramp: 0.000'
Read or change velocity PID parameters

Sent: 'MWT0' | Received: 'PWM Mod | type: SinePWM'
Sent: 'MWT1' | Received: 'PWM Mod | type: SVPWM'
Sent: 'MWT2' | Received: 'PWM Mod | type: Trap 120'
Sent: 'MWT3' | Received: 'PWM Mod | type: Trap 150'
Sent: 'MWC' | Received: 'PWM Mod | center: 1'
Change modulation type
```


Unknown or unclear commands
---------------------------
```
Sent: '#' | Received: 'Decimal:3'
Sent: 'MMC' | Received: 'Monitor | clear'
Sent: 'MMD' | Received: 'Monitor | downsample: 0'
Sent: 'MMG' | Received: 'Monitor | target: 0.000'
Sent: 'MMG0' | Received: 'Monitor | target: 0.000'
Sent: 'MMG1' | Received: 'Monitor | Vq: 0.000'
Sent: 'MMG2' | Received: 'Monitor | Vd: 0.000'
Sent: 'MMG3' | Received: 'Monitor | Cq: 0.000'
Sent: 'MMG4' | Received: 'Monitor | Cd: 0.000'
Sent: 'MMG5' | Received: 'Monitor | vel: 0.000'
Sent: 'MMG6' | Received: 'Monitor | angle: -0.065'
Sent: 'MMG7' | Received: 'Monitor | all: 0.000;0.000;0.000;0.000;0.000;0.000;-0.065'
Sent: 'MMS' | Received: 'Monitor | 0000000'
Sent: 'ME0' | Received: 'Status: 0'
Sent: 'ME1' | Received: 'Status: 1'
Sent: 'ME2' | Received: 'Status: 1'
Sent: 'ME3' | Received: 'Status: 1'
Sent: 'ME4' | Received: 'Status: 1'
Sent: 'ME5' | Received: 'Status: 1'
Sent: 'ME6' | Received: 'Status: 1'
Sent: 'ME7' | Received: 'Status: 1'
Sent: 'ME8' | Received: 'Status: 1'
Sent: 'ME9' | Received: 'Status: 1'
...
Sent: 'MSE' | Received: 'Sensor | el. offset: 5.240'
Sent: 'MSM' | Received: 'Sensor | offset: 0.000'
Sent: '@1' | Received: 'on!'
Sent: '@2' | Received: 'Verb:on!'
Sent: '@3' | Received: '@machine'
Sent: 'MCD' | Received: 'Motion: downsample: 12'
Sent: 'M+' | Received: 'Target: 0.000'
Sent: 'M-' | Received: 'Target: 0.000'
Sent: 'MR' | Received: 'R phase: 0'
Sent: 'MI' | Received: 'L phase: 0'

Sent: 'MT0' | Received: 'Torque: volt'
Sent: 'MT1' | Received: 'Torque: dc curr'
Sent: 'MT2' | Received: 'Torque: foc curr'
Sent: 'MC0' | Received: 'Motion:torque'
Sent: 'MC1' | Received: 'Motion:vel'
Sent: 'MC2' | Received: 'Motion:angle'
Sent: 'MC3' | Received: 'Motion:vel open'
Sent: 'MC4' | Received: 'Motion:angle open'
Sent: 'ME' | Received: 'Status: 0'	
```

40V Possibilities?
------------------
The first clues for using a 40V battery are hopefull, albeit there is not a lot of headroom for the mosfets.
40V batteries (presume 10 lithium cells in series) have a max voltage of 42V

- The Vds,max is 55V for the IRF3205, if the motor brakes electronically, the voltage over the mosfets will rise (brake energy -> electrical energy).
- DC bus caps are 80V
- DRV8323RS can handle up to 65V (and probably makes the low voltage power rail)

More detailed investigation is needed


Some extra info
---------------
- All commands are based on the firmware that was released on 12/2024 (first batch).
- Changes in parameters are lost after power loss.
- Testing the serial console works with lower voltages than the battery voltage (tested 9V, probably everything above 3.3V will work).
- The firmware has no ROP
- Combining @ and some other ASCII characters sometimes results in an unresponsive epulse drive (for example '@M'), repowering fixes this.
- I have no clue why there are 5 mosfets with the same markings and one oddball (older markings?). But they are all the same type.
- I cannot feel like the serial console is left (put) there by infento for enthusiasts to thinker with ;)
