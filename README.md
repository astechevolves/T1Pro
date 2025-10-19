# T1Pro
Klipper Macros for use on the T1 Pro

**** Disclaimer. These modifications are simply me sharing things that i have done to MY printer thant make my life easier. Use at your own risk. I am not responsible for damage that may occur by you putting random things off the internet into your printer. ****

These files are based off the files [provided by @camelCaseOnly here](https://www.printables.com/model/1286786-flsun-t1-pro-configuration-improvements) as a base. They were modified quite a bit for different reasons so i figured i would share my versions as well. 

In order to use these you need to copy the macros.cfg and overrides.cfg into the config folder on your T1 Pro printer. 

You will already have a printer.cfg file on your printer and it contains things like your calibrations so we generally would not overwrite it totally. I have left the entire file here as a reference if anyone were to need it. The modifications we required are detailed below. 

Printer.cfg modifications to be able to use these macros:

So we need to add `[exclude_object]` at the beginning of the file if you are interested in enabling object exclusion. This will take slicer modifications as well that we will add a bit later. 
So the beginning of the file will look something like this
```
#T1Pro

[exclude_object]

####################################################################################################
#motor part
####################################################################################################
[stepper_a]
step_pin: PE5
dir_pin: !PD7  # motor direction pin
enable_pin: !PE1
microsteps: 32
rotation_distance: 60
```

Now down lower in the file, around line 580 in my file, there will be a line called `[save_variables]`. We want to add our macros and overrides file just before that. 

That area will now look something like this. 

```
[virtual_sdcard]
path: ~/printer_data/gcodes
on_error_gcode:
    M106 S0
    TURN_OFF_HEATERS
    SET_STEPPER_ENABLE STEPPER=extruder ENABLE=0
    boxfan_off

[include macros.cfg] # added to include our custom macros.cfg file
[include overrides.cfg] # added for the downloaded macros.cfg overrides

[save_variables]
filename: ~/savedVariables1.cfg

[rotate_logger]
filename: ~/klipper_logs/mylog.txt
max_bytes: 1048576
backup_count: 3
```

Next, if you have done the Bento Mini fan mod or any other filter that uses the stock Boxfan pin we will need the below. FLSun is already overwriting the origin a M106 command with this macro. What they were doing out of the box was taking any cooling fan command from the slicer and making them all for the part cooling fan. We want to split that out and make fan P3 our boxfan which is what they call the filter fan that was in the T1 but went away in the T1 Pro.

So the M106 area of the printer.cfg wll look like this now. 

```
###########################################################################################################################
# M106
###########################################################################################################################
[gcode_macro M106]
description: fan speed control
rename_existing: M106.1
gcode:
    {% set last_fan_speed = printer.fan.speed|float %}
    {% set max_power = printer.fan.max_power|float %}
    {% set last_fan_value = last_fan_speed/max_power %}
    {% set fan_speed = params.S|default(255)|int %}
    {% set fan_selection = params.P|default(1)|int %} # added to allow for vent fan selection by slicer
    
    {% if fan_selection == 3 and fan_speed > 0 %} # use the vent fan selection if required
      RESPOND MSG="--- Turning on the Bento Box"
      boxfan_on
    {% elif fan_selection == 3 and fan_speed == 0 %}
      RESPOND MSG="--- Turning off the Bento Box"
      boxfan_off
    {% elif last_fan_value < 0.1 and fan_speed > 100 %} # or do part cooling as the original code intended if anything else is happening
        M106.1 S60
        SET_GCODE_VARIABLE MACRO=set_fan VARIABLE=fan_speed VALUE={fan_speed}
        UPDATE_DELAYED_GCODE ID=setfan DURATION=2
    {% else %}
        M106.1 S{fan_speed}
    {% endif %}
    
[gcode_macro set_fan]
variable_fan_speed: 0
gcode:
    M106.1 S{fan_speed}
```

Over to the slicer now. I am using Orca so if i put a name here it may not be the exact name of the item in your slicer. It will be similar as we aren't doing anything crazy here but a bit of research may be needed if it's not close enough to be obvious. 

Lets being with start GCode. Edit your printer setting and move over to the Machine G-Code tab. You will find the `Machine start G-code` block. Within this block there will be a bunch of G and M codes that are transferring data over to your printer and setting things up. We now have these in our start macro so you can remove everythign in that box and replace it with the below block.

Machine start G-code
```
PRINT_START BED=[first_layer_bed_temperature] EXTRUDER=[first_layer_temperature] MATERIAL={filament_type[initial_tool]}
RESPOND MSG="--- Out of macros, into print code..."
```

We also need to do the same thing for the end G-Code

Machine end G-code
```
PRINT_END
```

The rest of these can stay as they are. Now onto the filemt setting if you are using the internal fans that we set up the G-code for the M106 macro for. If you arent't using the filter or fan then this change isn't needed but it's good to know what the options do so feel free to read on. 

Clicking the edit button to the right of a filment will bring up the material settings page. In the cooling tab if you scroll down you wil come to the Exhaust Fan option. This option controls the 3rd fan on the system Aka `M106 P3`. If you recall int he macro we were taking the P3 variable and changing it to turn on the Boxfan pin. Unfortunatly this pin doesnt have any speed commands as it comes. So either we have the fan on or off. There is no middle. The speed settings for during or complete don't really change anything for us for this reason. Lets just check the activate box and put 100% in the other boxes. 

An important note here. This is a per filament settings. You can change if the fan come on for each filemtn. This way you will always have fans for things like ABS/ASA where you need it but you don't really need to have it on for thing like PLA where it's less important. You can always for it on in the START_PRINT macros and always turn off in the STOP_PRINT macro if you want it on all the time. But right now it's controlled by the slicer base don this setting. 

That's it for setup, now it's time to send a print to test. 

I will be adding details in the future as well to break down what the macros do so it can be better understood. 
