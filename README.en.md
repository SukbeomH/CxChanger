# [CxChanger]

[Korean](README.md) | [English](README.en.md) | [Chinese](README.zh.md)

Translated Klipper configuration files: [`klipper/en/`](klipper/en/) | [`klipper/ko/`](klipper/ko/)

[CxChanger] is a multi-material and multi-color solution for FDM 3D printing. It enables fast multi-material printing by switching between preheated hotends loaded with different filaments during a print. The goal is to provide multi-material printing at the lowest practical cost while keeping the mechanism simple and stable. For broad printer compatibility, hotend switching only requires XY motion of the toolhead. The switching mechanism is independent of the hotend itself, so other hotends can be supported by modifying the hotend mount and related parts.

<img width="3960" height="3060" alt="Toolhead" src="https://github.com/user-attachments/assets/66c70b3b-5862-4e04-8ac4-0b782e6da9f5" />

![QQ20260404-193802](https://github.com/user-attachments/assets/bb0c0aa7-5b82-490f-add7-e86a71225352)

---

## Core Features

- **Automatic preheating**: When OrcaSlicer's Ooze prevention feature is enabled, preheat commands are inserted before material changes according to the configured preheat time. During a switch, the old hotend is released and set to standby temperature, then the already-preheated hotend is picked up. If the next switch happens sooner than the heat-up time, the hotend remains at temperature instead of cooling down.
- **Underactuated switching mechanism**: The underactuated design does not require any additional active actuators. It uses the toolhead's own XY motion and mechanical interaction with the dock to switch hotends and clamp or release the filament.
- **Magnetic Maxwell kinematic coupling**: The hotend module is locked with N52 magnets and positioned by a [Maxwell kinematic coupling](https://en.wikipedia.org/wiki/Kinematic_coupling). The toolhead side uses three slots formed by locating pins, while the hotend module side uses ball-ended pins. When magnetically coupled, the hotend module is constrained in all six spatial degrees of freedom without overconstraint. This lets the module self-adjust, remove play, and position accurately even with machining, assembly, wear, or deformation errors.
- **Automatic nozzle offset calibration**: A calibrator measures each tool's XYZ offset relative to T0. Supported approaches include [ball microswitch contact nozzle offset calibration](https://github.com/Noisyfox/FoxChanger/tree/main/NozzleProbe) and [contactless eddy-current nozzle offset calibration](https://oshwhub.com/cxg01/project_lbabffjk). Contact probing requires a clean nozzle to obtain correct offsets, while contactless probing can work without cleaning the nozzle. Calibration is not required before every print; it is normally needed after initial assembly or after a nozzle crash.

## Hardware and Software Requirements

### Hardware

- **Printer**: In principle, any printer whose toolhead can move in XY is supported, such as Voron 2.4, Voron Trident, BigFish TT, and similar machines. This open-source project only provides the toolhead and base dock designs. You will need to mount the dock extrusion and make any other compatibility changes for your own printer.
- **Mainboard**: Because wired heating is used, each hotend module has one heater cartridge and one thermistor. A small number of hotend modules can be added by using spare ports on the mainboard. To add more hotend modules, an [expansion board](https://oshwhub.com/rayzark/project_qxihkkhj) is required to add heater and thermistor ports.

### Software

- **Firmware**: Overall hotend switching is implemented with Klipper macros. If nozzle offset calibration is needed, install the [toolchange](https://github.com/viesturz/klipper-toolchanger) plugin and use it together with the automatic calibration macros.
- **Slicer**: OrcaSlicer is used. Other slicers can be explored separately.

## System Overview

### Components

- Toolhead: all parts other than the hotend.
- Hotend module: includes the hotend, PTFE tube, heater, and thermistor wiring.
- Dock: stores parked hotends, blocks the parked nozzle to prevent oozing, and cools parked hotends with fans. Open-frame printers use a side centrifugal fan to cool all hotends; enclosed printers use one axial fan per dock.

### Parts

#### Printed Version

- 1.5 x 5 mm cylindrical pins (diameter x length): form the slots of the Maxwell kinematic coupling (x6)
- 3 x 6 mm ball-ended pins (diameter x length): form the balls of the Maxwell kinematic coupling (3 per hotend)
- 5 x 20 mm internally threaded ball-ended pin (diameter x length): mounted on the dock to open the extruder idler arm (1 per dock)
- 6 x 4 mm N52 magnets (diameter x thickness): attach the hotend module to the toolhead and dock (3 per hotend, 1 per dock, 3 on the toolhead)
- M3 x 6 countersunk screws
- M3 x 25 screws
- M3 x 8 screws
- M3 x 10 screws
- M3 x 12 screws
- M3 x 14 screws
- M3 x 20 screws
- M2.5 x 16 screws
- M2 x 8 screws
- M3 x 4 x 4 heat-set inserts
- HGX extruder gear set
- 4020 blower fan
- 2510 axial fan
- Omron microswitch
- 10 x 2 mm self-adhesive heat-resistant silicone strip (width x thickness)

#### CNC Version

See [`5.5-version-bom.xlsx`](5.5-version-bom.xlsx).

## Installation and Configuration Guide

### printer.cfg

You can keep the main body of your existing `printer.cfg`. Refer to the format in the provided `printer.cfg` and the notes below, or use the provided file directly and change the pins for your own mainboard.

1. Include `toolchange.cfg` and `calibration.cfg`.
2. Define the added hotends by following the `extruder1` format. Wire the heater and thermistor pins according to your mainboard. Reuse the E0 extruder; the other extruders are virtual extruders.
3. The probe uses a fixed microswitch. At the start of a print, all hotends are parked in their docks and the nozzles are blocked with high-temperature silicone pads. The selected hotend is only picked up after it reaches target temperature, which solves initial oozing without nozzle wiping. During bed leveling, no hotend is picked up, so the microswitch is the lowest point of the toolhead. Other leveling methods are not yet supported. It is recommended to reference the provided probe configuration, including bed mesh and leveling settings, mainly because they release the picked-up hotend before leveling.
4. Add dock cooling fan configuration. Add all extruders to the extruder associations for both the dock cooling fans and the heatbreak cooling fan.
5. Modify the print start G-code so the printer automatically purges the hotends that will be used before printing.
6. Modify the print end G-code. Add the `UNTOOL` command so the hotend is automatically unloaded when the print finishes, avoiding the need to remove it manually before the next print. Also add commands to turn off all hotend heaters.

### toolchange.cfg

1. Add `toolchange.cfg`. After it is added successfully, the dashboard will show `T0`, `T1`, `T2`, and `UNTOOL` commands. Do not click them before setting coordinates, or the printer will crash. Clicking `T0` picks up T0. After T0 is picked up, clicking `T1` will automatically unload T0 and pick up T1. Clicking `UNTOOL` unloads the current hotend.
2. Fix the docks and adjust their coordinates: manually attach a hotend to the toolhead, move the toolhead to the far right of the X axis, then move it forward. Fix the first dock at the position where the hotend locking screw can just pass through the large dock hole. This position is the coordinate of the first dock. Remove the hotend, home all axes, attach the hotend to the toolhead again, then slowly move the toolhead to the dock position found earlier so the locking screw passes through the large dock hole. Record the coordinate and enter it as the T0 dock position in `toolchange.cfg`. Each dock occupies 30 mm of width, so the remaining dock coordinates can be calculated as an arithmetic sequence. Fine-adjust each dock coordinate if needed.

   <img width="1539" height="1043" alt="image" src="https://github.com/user-attachments/assets/392c5682-040e-4cc8-8034-a8e724e259c7" />

3. After the coordinates are set correctly, comment out the normal speed and uncomment the test speed. Test switching at low speed so you can hit emergency stop immediately if alignment is wrong.
4. After switching works correctly at test speed, comment out the test speed and uncomment the normal speed. You can tune a more suitable speed for your own machine.

### calibration.cfg

1. Download the [toolchange](https://github.com/viesturz/klipper-toolchanger) plugin.
2. Add `calibration.cfg`.
3. Home XYZ. After leveling is complete, secure the calibrator, click `T0` to pick up the first hotend, and move the nozzle tip until it is directly above the calibrator center at about 1 mm distance. Record the coordinate and enter it as the calibrator coordinate in `calibration.cfg`. You also need to define a safe position outside the dock area to prevent collisions with other hotends in the docks during calibration.
4. After the coordinates are set, enter `CALIBRATE_TOOL TOOL=_` to calibrate a specified hotend. T0 must be calibrated first. Enter `CALIBRATE_ALL_TOOLS` to calibrate all tools after defining which hotends are present. Offsets are saved automatically after calibration completes.

### OrcaSlicer

1. Modify the printer start G-code and filament change G-code, configure more extruders according to the number of hotends, and set the filament retraction length for each extruder switch. The author's configured length is 2 mm.

- Print start G-code, modify according to your number of hotends:

```Gcode
   ; Let OrcaSlicer extract the bed temperature, initial nozzle, used nozzle list, and matching temperatures, then send them to the printer.
PRINT_START BED=[bed_temperature_initial_layer_single] INITIAL_TOOL=[initial_tool] TOOLS="{if is_extruder_used[0]}0,{endif}{if is_extruder_used[1]}1,{endif}{if is_extruder_used[2]}2,{endif}{if is_extruder_used[3]}3,{endif}{if is_extruder_used[4]}4,{endif}" TEMPS="{if is_extruder_used[0]}{nozzle_temperature_initial_layer[0]},{endif}{if is_extruder_used[1]}{nozzle_temperature_initial_layer[1]},{endif}{if is_extruder_used[2]}{nozzle_temperature_initial_layer[2]},{endif}{if is_extruder_used[3]}{nozzle_temperature_initial_layer[3]},{endif}{if is_extruder_used[4]}{nozzle_temperature_initial_layer[4]},{endif}"
```

- Filament change G-code:

```Gcode
M104 S{nozzle_temperature[next_extruder]} T{next_extruder}
T{next_extruder}
```

2. In the process settings, enable a wipe tower. If the filament and print parameters are tuned well, and the model does not involve fast tiny extrusions, you can move the model near the docks, disable the wipe tower, and try printing without one. Also enable Ooze automatic preheating. The author's settings are softening temperature -180 and preheat time 35 s; adjust them according to your hotend heating speed.

## Notes

- Print the parts in ABS with 4 walls and 60% infill.
- The extruder spring must be tightened more than on a normal extruder. Because the mechanism is compact, the lever arm is short, so the extruder clamping force is lower than that of a normal extruder at the same preload.
- To reduce heat conduction, remove the two screws that fix the heater block on the TZ2.0 hotend and rely only on the side set screw.
- Install the magnets exactly as shown in the drawings. Some are pressed all the way in; some are flush with a given surface. When assembled correctly, only the ball-ended pins on the hotend assembly contact the cylindrical-pin slots on the backplate side, with a small gap between the magnets. This is required for the Maxwell kinematic coupling to self-adjust and remove play. You can wiggle the hotend to check for looseness. If there is play, check whether assembly is correct. If print accuracy becomes worse during printing, inspect the positioning assembly; a common failure is a magnet coming loose and sliding out until it contacts the magnet on the other side, which disables the Maxwell coupling.
- Use 12.9-grade socket head cap screws for the hook screws that hang the hotend assembly in the dock holes, so they can be magnetically attracted inside the dock. 304 stainless screws will not be attracted properly.
