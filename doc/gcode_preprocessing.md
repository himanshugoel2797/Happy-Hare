# G-Code Preprocessing
Happy Hare now provides a moonraker gcode preprocesser that parses uploaded gcode files prior to storage and can insert useful metadata that can then be passed into your `START_PRINT` macro to provide useful functionality.

This support is added to Moonraker configuration during installation but can be added manually by inserting the following lines into your `moonraker.conf` file and restarting Moonraker. Setting `enable_file_preprocessor: False` will disable this functionality.

```yml
[mmu_server]
enable_file_preprocessor: True
```
The functionality is similar to the "placeholder" substitution that many slicers do.  For example if you use `{total_toolchanges}` in PrusaSlicer, it will substitute this placeholder with the actual number of tool changes.  When used in conjunction with a call to your `START_PRINT` macro you can pass parameters to use at print time. E.g. by defining your "start" G-Code as:

```
START_PRINT TOOL_CHANGES={total_toolchanges} ....
```

Your `START_PRINT` macro thus gets the number of tool changes passed as an integer parameter.

<br>

The Happy Hare pre-processor implements similar functionality but runs when the file is uploaded to Moonraker. To distinguish from Slicer added tokens, this pre-processor delimits placeholders with `!` marks. Note that this does not duplicate all the Slicer placeholders but provides an extensible mechanism to implement anything missing in the slicer. At this time a single placeholder `!referenced_tools!` is implemented but this might grow over time.

> [!NOTE]  
> The `{referenced_tools}` placeholder has been submitted as a PR for PrusaSlicer but it has not yet been incorporated, so you can use `!referenced_tools!` instead!

<br>

## ![#f03c15](/doc/f03c15.png) ![#c5f015](/doc/c5f015.png) ![#1589F0](/doc/1589F0.png) Supported Placeholders

### Placeholder: !referenced_tools!
This placeholder is substituted with a comma separated list of tools used in a print.  If there are no toolchanges (non MMU print) it will be an empty string. E.g. `0,2,5,6` means that T0, T2, T5 and T6 are used in the print.

__Why is this useful?__
<br>
When combined with the `MMU_CHECK_GATES TOOLS=` functionality and placed in your `START_PRINT` macro it allows for up-front verification that all required tools are present before commencing the print!

To implement incorporate into your start g-code on your Slicer:

```yml
START_PRINT REFERENCED_TOOLS=!referenced_tools! INITIAL_TOOL={initial_tool} ...the other parameters...
```

Then in you print start macro add logic similar to:

```yml
[gcode_macro START_PRINT]
description: Called when starting print
gcode:
    {% set REFERENCED_TOOLS = params.REFERENCED_TOOLS|default("")|string %}
    {% set INITIAL_TOOL = params.INITIAL_TOOL|default(0)|int %}

    {% if REFERENCED_TOOLS == "!referenced_tools!" %}
        RESPOND MSG="Happy Hare gcode pre-processor is diabled"
        {% set REFERENCED_TOOLS = INITIAL_TOOL %}
    {% elif REFERENCED_TOOLS == "" %}
        RESPOND MSG="Happy Hare single color print"
        {% set REFERENCED_TOOLS = INITIAL_TOOL %}
    {% endif %}

    :

    MMU_CHECK_GATE TOOLS={REFERENCED_TOOLS}

    MMU_CHANGE_TOOL STANDALONE=1 TOOL={INITIAL_TOOL}    # Optional: load initial tool

    :
```

> [!NOTE]  
> * `MMU_CHECK_GATE TOOLS=` with empty string will be ignored by Happy Hare.<br>
> * Any tool that was loaded prior to calling `MMU_CHECK_GATES` will be automatically restored at the end of the checking procedure.<br>
> * In the gcode snippet above we also pass in the slicer placeholder {initial_tool} because single color prints have no tool changes and thus `REFERENCED_TOOLS` (which counts `Tx` commands) will be empty. This code will ensure that `REFERENCED_TOOLS` will always contain the initial tool.

### Placeholder: !colors!
This placeholder is substituted with a comma separated list of extruder colors as defined in the slicer. This could be used to setup the filament colors in the MMU gate map.  Although the colors defined in the slicer have nothing to do with the actual filaments loaded in the MMU it might be convenient (if not using spoolman) to transfer over the colors from the slicer gcode file, light LEDs on the MMU and perform a visual match on the whether the correct filaments are loaded

To implement incorporate into your start g-code on your Slicer:

```yml
START_PRINT COLORS=!colors! ...the other parameters...
```

Then in you print start macro add logic similar to:

```yml
[gcode_macro START_PRINT]
description: Called when starting print
gcode:
    {% set COLORS = params.COLORS|default("")|string %}
    {% set colors = COLORS.split(",") %}
    {% set ttg_map = printer.mmu.ttg_map %}

    {% for tool, color in enumerate(colors) %}
        {% set gate = ttg_map[tool] %}                  # Make sure map to correct gate in case of TTG map
        MMU_GATE_MAP GATE={gate} COLOR={color}          # Register the filament color against correct gate in gate map
    {% endfor %}
```

To see the colors displayed you need to enable LED support and use the `filament_color` effect. E.g. to display both on exit led and the color of current filament on the status led:
```
MMU_LED EXIT_EFFECT=filament_color STATUS_EFFECT=filament_color
```

Another, perhaps more useful idea, is to set the `custom_colors` array and combine with referenced tools to light up the colors of just the filaments used and defined in your slicer for the current print. That way you can visually check that the colors loaded in your MMU match the print design. Here is the complete code of how to accomplish that:

```yml
[gcode_macro START_PRINT]
description: Called when starting print
gcode:
    {% set REFERENCED_TOOLS = params.REFERENCED_TOOLS|default("")|string %}
    {% set INITIAL_TOOL = params.INITIAL_TOOL|default(0)|int %}
    {% set COLORS = params.COLORS|default("")|string %}
    {% set referenced_tools = REFERENCED_TOOLS.split(",") %}
    {% set colors = COLORS.split(",") %}
    {% set ttg_map = printer.mmu.ttg_map %}

    {% if REFERENCED_TOOLS == "!referenced_tools!" %}
        RESPOND MSG="Happy Hare gcode pre-processor is diabled"
        {% set REFERENCED_TOOLS = INITIAL_TOOL %}
    {% elif REFERENCED_TOOLS == "" %}
        RESPOND MSG="Happy Hare single color print"
        {% set REFERENCED_TOOLS = INITIAL_TOOL %}
    {% endif %}

    {% for _ in ttg_map %}                              # Reset custom colors for all gates to black
        MMU_LED GATE={loop.index0} COLOR=black
    {% endfor %}

    {% for color in colors %}
        {% set gate = ttg_map[loop.index0] %}           # Map to the correct gate in case of TTG map
        {% if loop.index0|string in referenced_tools %}
            MMU_LED GATE={gate} COLOR={color}           # Register the filament color against the correct gate
        {% endif %}
    {% endfor %}

    MMU_LED EXIT_EFFECT=custom_color QUIET=1            # Turn on the custom color effect on gate exit
    MMU_CHECK_GATE TOOLS={REFERENCED_TOOLS}             # Verify all necessary tools are loaded

    MMU_CHANGE_TOOL STANDALONE=1 TOOL={INITIAL_TOOL}    # Optional: load initial tool
```

Alternatively you can retrieve the RGB colors necessary to directly drive other leds by accessing the printer variable `printer.mmu.gate_color_rbg` which contains a list of truples contains the 0-1.0 value for each of the R,G,B pixels.  See led doc for more details.

### Placeholder: !temperatures!
This placeholder is substituted with a comma separated list of filament temperatures as defined in the slicer.

