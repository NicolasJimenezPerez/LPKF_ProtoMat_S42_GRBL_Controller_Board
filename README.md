//cspell:ignore GRBL LPKF Klipper langwadt MOSFET BLDC ntchris

# LPKF ProtoMat S42 GRBL Controller Board
A project describing the design and installation of a custom GRBL replacement board for the LPKF ProtoMat S42 PCB router CNC.

## Motivation

## Why GRBL
The main reason for using GRBL over other alternatives like LinuxCNC, or even 3D printer software like Klipper, is the unusual solenoid-driven Z-axis this machine has. After some research I came across [this comment](https://github.com/gnea/grbl/issues/640#issuecomment-634253340) by @langwadt explaining how to modify GRBL to work nice with the Z-axis solenoid, so it was testing time.

After soldering together a quick and dirty controller board with two stepper controllers and a MOSFET for controlling the voltage on the solenoid it was turned on. GRBL seamed to play nice with all the component, so I continued forwards. 

The next step was the BLDC motor for the spindle. GRBL out of the box does not support the PWM protocol needed to communicate with a standard race car ESC, which was the component I had on hand to check that the motor spun; that is why the GRBL used is a modified version of the [ntchris/grbl](https://github.com/ntchris/grbl), which is a fork of GRBL adding support for ESC-compatible PWM.

Having all the components supported by GRBL it seemed stupid to try other more complex solutions, seeing as there wasn't any apparent reason to.

## Modifying the latest version of GRBL
As mentioned in the section before, base code for running this machines board is a modified version of [ntchris/grbl](https://github.com/ntchris/grbl). In this section describes the modifications made to it for future reference, as this git page may not be up to date with the latest GRBL release.

#### Deleting Z-axis homing
Knowing this is a 2D machine with an ON/OFF Z-axis the first thing is to disable the homing of the Z-axis, as there is no Z limit switch in the machine. To do this in the **config.h** file there is a section about the homing sequence similar to this:

```c
#define HOMING_CYCLE_0 (1<<Z_AXIS)                // REQUIRED: First move Z to clear workspace.
//#define HOMING_CYCLE_1 ((1<<X_AXIS)|(1<<Y_AXIS))  // OPTIONAL: Then move X,Y at the same time.
#define HOMING_CYCLE_1 (1<<X_AXIS) 
#define HOMING_CYCLE_2 (1<<Y_AXIS)
```

This should be changed to the following. Pay special attention to the change in the `HOMING_CYCLE` number from `1` to `0` in the uncommented line.

```c
//#define HOMING_CYCLE_0 (1<<Z_AXIS)                // REQUIRED: First move Z to clear workspace.
#define HOMING_CYCLE_0 ((1<<X_AXIS)|(1<<Y_AXIS))  // OPTIONAL: Then move X,Y at the same time.
//#define HOMING_CYCLE_1 (1<<X_AXIS) 
//#define HOMING_CYCLE_2 (1<<Y_AXIS)
```

#### Enable solenoid Z-axis
To enable the use of the solenoid-driven Z-axis a change must be done in **stepper.c**, as stated by [this comment](https://github.com/gnea/grbl/issues/640#issuecomment-634253340) by @langwadt. Searching for the line

```c
DIRECTION_PORT = (DIRECTION_PORT & ~DIRECTION_MASK) | (st.dir_outbits & DIRECTION_MASK);
```

and changing it to the following

```c
DIRECTION_PORT = (DIRECTION_PORT & ~((1<<X_DIRECTION_BIT)|(1<<Y_DIRECTION_BIT))) | (st.dir_outbits & ((1<<X_DIRECTION_BIT)|(1<<Y_DIRECTION_BIT)));

if(st.step_outbits & (1<<Z_STEP_BIT))  DIRECTION_PORT = (DIRECTION_PORT & ~((1<<Z_DIRECTION_BIT))) | (st.dir_outbits & ((1<<Z_DIRECTION_BIT)));
```

does the trick. Now the **Direction Z-Axis** is `HIGH` when the solenoid should be energized and `LOW` when it should be isolated from power.

## GRBL parameters

The changes to the base GRBL parameters ($$) are the following:

### Compulsory changes

1. #### X-axis direction
    The positive direction of the X-axis is reversed by default in this machine so the configuration should be compensate by sending

    ```
    $3 = 1
    ```

1. #### Maximum spindle speed
    By default the maximum spindle speed is set to `0`, so it should be increased. Looking into the documentation fot the machine we see that the max speed for the spindle was 42000 rpm, so we set it to the same value as so:
    
    ```
    $30 = 42000
    ```

1. #### Steps/mm

    The steps per millimeter on the x-axis and y-axis should be set to

    ```
    $100 = 1068.273
    $101 = 1068.273
    ```

    The Z axis does not need any modification as it just hast to be slow enough to go down before the other axes start moving.

1. #### 


## Software configuration