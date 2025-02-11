
# MaxChange Tuning Recommendations

When using Klipper with MaxChange, there are several spaces, marked by cartesian co–ordinates or "grids" to configure and
align.  Each grid is described by X, Y and Z offset positions from the axes defined by your `stepper_x`, `stepper_y` and
`stepper_z` travel capability.  Some are built–in to Klipper, plus with any toolchanger, you get additional sets of grid
offsets for each toolhead's nozzle position.  Finally, when using nudge to precisely align your toolhead offsets, there is
a "corrected" grid for each toolhead.

The term "grid" or "offsets" refer to the same concept and while not totally interchangable terms, should be thought of as
referring to this same concept of shifting around the logical cartesian (0, 0, 0) position (the "origin").

In order to make macros shareable and interoperable, and to avoid a lot of confusion and time–consuming macro debugging
and staring at very small toolhead movements, it's important that macros that need to know about the differences, all
apply corrections in the same way.

Additionally, there are a set of recommendations for what to use when choosing what ranges to assign to each of the grids.

## Calibration process at a high level

All steps should be performed with the kinematic grid active (i.e., `SET_GCODE_OFFSET X=0 Y=0 Z=0`).  Note, once you have
configured `gcode_*_offset`, you will need to set this after each toolchange you do while carrying out these steps.

1. Configure the physical / kinematic ranges, which for the reasons stated below should be with 0 at the extreme end of
   movement of each axis, and with Z=0 at the lowest possible point that you could possibly ever configure it.

2. With kinematic co–ordinates active , figure out the correct values for
   `params_x_park`, `params_x_detach` as well as the Y parameters: `params_safe_toolhead_y` for the Y position where
   movement is safe for all toolheads when mounted, while other toolheads are installed.  `params_safe_shuttle_y` for the
   Y position were movement is safe for the carriage with no toolhead mounted.  `params_close_y` for a "comfortable"
   position close to the toolhead(s) where the magnets are about to, but not quite "grabbing" (causing visual movement in)
   the toolhead, and `params_attach_y` to be where the carriage is able to slide the toolheads in and out of the dock.
   Confirm toolchanging works.

3. Configure the `homing_origin`: the X, Y and Z position for all toolheads of the front–left position of the printable
   area (or potentially the center, if that makes sense for your printer), by moving the toolhead to where you want your
   origin within ~1mm or so, and then putting the X, Y and Z into `[tool TN]` (N=0, 1, ...) under `gcode_x_offset`,
   `gcode_y_offset` and `gcode_z_offset`.  These positions should again be in the kinematic grid (again, `SET_GCODE_OFFSET
   X=0 Y=0 Z=0`)

4. Pick a toolhead, any toolhead, and move it over a firmly attached nudge probe using G0/G1 movement commands and/or the
   movement buttons in Mainsail.  Get it somewhere above the tip of the nudge pin, then run
   `NUDGE_GET_PROBE_POSITION`. This will move the nozzle around and trigger the nudge probe from different directions, and
   in the process, fairly precisely locate the position of the nudge probe relative to the _currently running_ kinematic
   grid.  This position is, by default, stored as a macro variable, meaning that the next time the printer is reset, it is
   discarded (after all, the relative location is no longer known precisely).  It's also discarded by the `PRINT_END`
   macro (and M18/M84 commands?  Can that be done?).
   
5. With your nudge probe position accurately known by one toolhead, you can now run `NUDGE_FIND_TOOL_OFFSETS`.  This will
   run through each of your other toolheads (probably just one initially), and find their relative position to the nudge
   probe.  Assuming you have set the _gross correction_ (the `gcode_[xyz]_offset` under `[tool T1]`), then the nudge
   location macro should be able to find the nudge pin, even though each toolhead generally hangs a little differently to
   each other, even if you are using the same parts in the construction.  The _correction_ is stored as a _save variable_,
   which _is_ preserved between printer restarts.  This can be thought of as fine–tuning the `gcode_x_offset`,
   `gcode_y_offset`, etc that you have configured.  You should now have the ability to run aligned prints!

More details on all of this is in the below sections.  If the above is not enough, please do read below, and direct all
criticism (for now) to Sam Vilain, @daddybuiltit on Discord.

## Configuring the physical grid (`position_min` and `position_max`)

For most people with a typical printer, your minimum for your X and Y
axes is zero, and the maximum is the width and depth of the printer's
build volume.  The Z axis, on the other hand, would normally be set to
have 0 more–or–less near to where the nozzle touches the plate, but
with some room for negative movement.  When homing, you would not
touch X and Y, while Z you would probably correct by changing the "Z
offset", perhaps using the handy correction buttons in Mainsail.

Perhaps you also have "overtravel" in X and Y; that is, extra distance
that you can move beyond the usable boundaries of your build
surface/plate.  When you have overtravel to the left or front of the
plate, it's possible that you set your `position_min` to be negative,
so that the (0, 0) position is at the front–left corner of your build
plate.

There's nothing wrong with these choices, and if you prefer, you can
continue to use X, Y and/or Z physical ranges which go negative.
However, this introduces another grid, which effectively has offsets
of these axes to the zero positions.  But did you know, that much like
you set the Z offset for your nozzle height using the Z offset
selection, you can set the X and Y offsets in much the same way?

The two grids are controlled in the following ways:

| **Property**  | **Physical Grid**                               | **Virtual Grid**                                       |
| Allowed Range | `position_min` & `position_max`                 | "Printable Area" Slicer setting,eg `rect_size` in Orca |
| Origin        | lowest value of `position_min` & `position_max` | "Origin" Slicer setting, eg `rect_origin` in Orca      |
| Detection     | Homing axes, eg endstop or sensorless           | Typically paper test, or probing                       |
| Correction    | `SET_KINEMATIC_POSITION` command                | `SET_GCODE_OFFSET` command                             |

Here is what this author recommends:

1. set the **Physical Grid** to have 0 at the physical end of
   movement, and the maximum should be as close as reasonably possible
   to the other extent, for _all axes_ (even Z!), and avoid adjusting
   the kinematic position, unless required for accurate toolchanging.
   If required for accurate toolchanging, a single precise alignment
   per printer restart using a nudge that is in a fixed location in
   the printer should be all that is required; this is described
   later.

2. set the **Virtual Grid** to be positioned with the (0, 0, 0)
   position at the front–left extreme of the _usable area_ of your
   build plate, even for people with X or Y overtravel on the left or
   front of the build plate.
   
To configure this:

0. if you have a printer with multiple Z steppers, run your current
   leveling command (ie, dual, tri or quad gantry leveling) so that
   you don't have to worry about this while the configuration is in
   flux.

1. if your `position_min` for the X or Y axes is currently negative,
   set it to zero, and increase the `position_max` by the same amount.
   The `position_endstop` should be at either 0, or `position_max`.
   Check that your 0 position and your `position_max` are at the true
   limits of the extent of movement; I found I had 1.5mm Y and 1mm X
   more than I thought; this can be important to get right when you
   have docks which are at or close to the extent of movement.
   
   You should be able to move the whole X axis when at the maximum
   extents of Y, and vice versa, without hitting anything (with the
   exception of docking screws hitting docks).  Even a small knock at
   the maximum extent can mess up the CoreXY position.  However, you
   may find Y sensorless homing does not work if the toolhead is all
   the way at the end of the X axis: this is normal; just move away
   from the end by 10mm or so first.

   You may find the following macro useful:
   
        [gcode_macro ADJUST_KINEMATIC_POSITION]
        gcode:
            {% set X = params.X|default(0)|float %}
            {% set Y = params.Y|default(0)|float %}
            {% set pos = printer.toolhead.position %}
            SET_KINEMATIC_POSITION X={pos.x - X} Y={pos.y - Y}
            G91
            G0 X{X} Y{Y} F600
            G92

   It allows you to adjust the physical grid and move the toolhead in
   one command.  I think you can also use `FORCE_MOVE` if you have
   that enabled, but I didn't try that.

2. for Z, remove your toolhead(s) completely, and find out exactly how
   far you could move the build plate up (for fixed gantry), gantry
   down (for flying gantry printers), or X axis down (for
   bedslingers/XZ) before you either hit a physical stop, or something
   you never remove (like the carriage) touches the plate.  This is
   your new Z=0 physical position.  Another sensible option is to
   remove your nozzle (and perhaps even your build plate), and lower
   your Z position until your heat block, toolhead shroud or whatever
   would hit first is touching or nearly touching the build plate/bed,
   and use that as your new 0 position.  Your `position_endstop`
   should either match your `position_max`, or be 0, depending on
   where your Z endstop is and the direction of homing.  This does
   mean that if you hit the "Clear" button in Mainsail, you will
   probably crash the head and dislodge it.  If that idea makes you
   uncomfortable, go ahead and make your Z=0 position a "safe"
   position by doing this with the toolhead in place, and aim for
   0.5 - 1mm gap from the nozzle to the plate at Z=0, and make your
   position_min a negative number.

3. If your `homing_override` command moves to a position close to the
   bed after homing, for safety, make sure the Z position it moves to
   after homing does not crash if you have the longest nozzle you use
   installed, and had no GCode offset configured (i.e., with
   `SET_GCODE_OFFSET Z=0`)

4. If you already configured these, and updated your X or Y
   `position_min`, update all your `params_*_x` and `params_*_y`
   per–toolhead settings to correct for the new 0 position, as these
   are relative to the physical grid.  You can hold off on updating
   any other positions you have configured or hard–coded, for now.

Of course, it's entirely optional to reconfigure your physical grid
like this.  It just aligns the physical grid's origin with the
physical maximum movement, effectively eliminating one grid from the
number of grids you need to consider.

## Configuring the virtual grid (`SET_GCODE_OFFSET`)

Put your toolhead(s) back in, restart Klipper, and if you already had
this working, confirm your tool changing still works with your updated
`params_x_*` and `params_y_*` settings.

The next step for left/front X/Y overtravel printers will be to find
the rough front corner of the build plate, and then the rough Z
position, and configure these as `gcode_*_offset` in your `[tool T0]`,
`[tool T1]`, etc sections (should be in
`klipper-toolchanger/toolhead_0.cfg` and
`klipper-toolchanger/toolhead_1.cfg`).

To do this, start with 0 GCode offsets. You can tell if there is a
GCode offset in Mainsail by the presence of a smaller (font) number
above the X, Y and Z position boxes that contains a number that
differs from the position in the editable box.  If the numbers are
different, reset to the physical grid with:

     SET_GCODE_OFFSET X=0 Y=0 Z=0

(Make sure your Z position is high enough not to crash first!)

Next, if you have X or Y overtravel to the left or front, you want to
move the first toolhead so that the nozzle is over the front left of
your build plate.  If your build plate does not have anything to align
it, remove it and find the front left of the _bed_, so that you know
if you cover the bed with the plate, your origin will always be on the
plate.  You can do this with the movement buttons in Mainsail or by
entering values into the position boxes.  Don't obsess over precision
here: 1mm accuracy is fine, or 0.5mm if you're feeling fancy.  The
whole tip of the nozzle should be on the build plate, not just the
center of the nozzle over the edge of the plate.  Configure the
`gcode_x_offset` and `gcode_y_offset` in your `[tool T0]` section with
the positions you found.

While you're there, if you have any axes you have with overtravel to
the right or rear of the plate, you may as well confirm that your
printable area is correct.  Apply the GCode offset you discovered,
using:

    SET_GCODE_OFFSET X=16 Y=1.5

Move to the position that you currently consider your printable area
maximum, and check that the whole nozzle is over the bed as before.
Update the values in your _slicer_ if they are not correct.

Then, install one of your thicker build plates, and find the Z offset
by lowering Z until the nozzle nearly touches.  You should aim to get
it to within 1mm or so of the plate, but remember this is a rough
position and is only set relative to your (presumably relatively
imprecise) max Z endstop.  Live bed probing will do the rest.  Set the
value as `gcode_z_offset` in `[tool T0]`.

Once you have this Z offset set approximately, you can proceed to do
the same thing with your other toolheads.  You don't need to worry
about precise alignment between toolheads: this is what nudge is for.
All you need to ensure is that homing is accurate enough to provide
for toolchanging, or for getting close enough to a fixed position
nudge for the tool location function to work.  Or both!

## Precise alignment using Maxwell Probing

So far, the grids have been aligned using methods that are relative
reproducible, but not precise.  If you restart Klipper or your
firmware, or even just disable the steppers, then re–homing using
regular endstops or sensorless homing is not going to align your
physical grid to the same exact location each time.

In fact no alignment system is going to do this perfectly.  To clarify
the "precise" and "imprecise" alignment, we need to talk about
"tolerance", which is another term to mean "margin of error", in other
words, what the maximum difference is between the actual position of
the grids, versus the one that was specified, designed or previously
calibrated.

In order to avoid awkward instructions, when we say "imprecise", we
mean that the tolerance is about what you can "eyeball", so somewhere
in the range of 0.1mm - 1mm of precision.  Various factors will affect
the actual precision that you get when doing "quick" homing using
sensorless homing or an endstop.  The goal for this "imprecise" level
of tuning is to give you enough accuracy that your printable area is
on the build plate (for XY overtravel printers), and for your
toolchanger to be able to swap tools.  Assuming it meets this
standard, no adjustment of the physical/kinematic grid should be
necessary.

On the other hand, when we use "precise" alignment, this refers to
having your grid known against some reference point (more on this
later) that the position is known to within 0.025mm or better: a
"babystep" of 1/40th of a mm.  As opposed to merely "homed", the
position is referred to as "probed" to reflect the typical technology
required to achieve it.  Many MaxChange owners in fact report 0.01mm
accuracy or even better than that for their probes.  This matters for
two main reasons: first layer accuracy, and alignment between
toolheads.  Perhaps it also affects your toolchanging stability.  In
turn, first layer accuracy matters because the amount of material
extruded needs to fill the initial gap between the nozzle and the
build plate, so that you get as much of the extruded material to touch
the bed as possible, giving you good adhesion.  "Landing the first
layer" is often rightfully referred to as the most important factor to
a successful print, and the easiest to use commercial printers have
sensors and approaches like bed meshing in place which allow for an
accurate, even first layer.  After that first layer, each successive
layer needs to be aligned to the one before it, and printer kinematics
usually allow for this to be much more reliable than the first layer.
Alignment between toolheads matters for a toolchanger, because now as
well as your first layer being a failure point, every point where
materials from different toolheads meet in X, Y and even Z can be a
failure point.  You will end up with errors such as gaps, bulges and
discoloration if there are tiny gaps or overlaps in the relative
positioning.  Single nozzle MMUs are generally immune to this issue,
as after a material change, the nozzle can be accurately returned to
lay the next line of material as well aligned to the previous line as
two lines that were printed one right after the other.

One advantage of the Maxwell coupling is that the combination of a
steady force of attraction from the magnet, versus the three plate
"pins" and matching holes, allows for very reproducible position each
time after a toolhead is attached to the carriage, or very slightly
dislodged during Z probing.  This trick is also used to ensure the
toolhead returns to the same place relative to the dock when parked,
so that a subsequent attempt to pick the toolhead up can return to the
same kinematic position again, and successfully attach the toolhead.

So, the core kinematics of the printer should mean that the carriage
position remains accurate relative to prior positions, and a given
toolhead will have its nozzle tip in the same precise location
relative to the carriage, so long as:

* steppers are holding position, and not disabled using M18 or a
  Klipper restart

* the Maxwell magnets and pins are well secured, and the contact
  surfaces inside the holes for the Maxwell pins also have no "give"
  to them.  MaxChange parts have tight tolerances in these areas for
  this reason.
  
* There are no collisions that cause missed steps, or that "shake up"
  and loosen fasteners that make up the Maxwell pins.

* Nothing is changed in the toolhead (such as changing a nozzle) that
  affects the effective position of the nozzle tip.
  
Put together, this all means that we have three sets of calibrations:

1. "homing" calibrations

2. "full" positional calibration that determines the relative carriage
   offset positions between nozzles across all the toolheads, and if
   required, the position from the fixed nudge probe to the first
   toolhead.

3. a limited alignment that must happen every time the printer is
   restarted, stepper motors are disabled (which typically happens
   at the end of a print), or axes are re–homed.

This also means that we have to pay special attention with all precise
alignment values / offsets, what they are relative to, and where they
are stored (which implies their invalidation policy).

The first set of calibrations are described in the previous sections
of this document, and only need to be performed if something is
changed such that nudge probing no longer works.

The second set, proving relative toolhead positions, can be performed
with a nudge that is permanently or temporarily installed.  The
position of the nudge is first detected with one toolhead, and then
the other toolhead(s) are switched to, and their relative offsets
measured.

Finally, the third group would be normally set up to conditionally run
in your `PRINT_START` macro, with their state and values invalidated
by the `PRINT_END` macro, or by homing and motors off commands.  This
group also includes accurate Z height probing.

Bed meshing via full multi–point probing (for printers with beds
sturdy enough for this to be an option) can fit into either group 2 or
group 3.  You might find it needs to go into group 3 if your build
plates have significant differences in how similarly to each other
they can provide a working build surface.

You can tell for sure in which group a calibration value is by
considering the accuracy for he thing that is it relative to, and how
precisely it is measured.

1. in general, the kinematic position is considered "imprecise" after
   homing and only subject to corrections, upgrading it to "precise"
   if absolutely required for toolchanging or nudge sensor location

2. bed probing for the global Z offset returns accurate values,
   relative to the imprecise kinematic grid, and is stored as a
   Klipper macro variable and in group 3

3. the nudge probe location _relative to the kinematic grid_ is
   precise, but the kinematic grid is imprecise, so it is also stored
   as a Klipper macro variable.  However, this probe location is
   itself only required for relative toolhead nozzle probing, so does
   not need to be in group 3.  The exception to this is if the nudge
   probe is used to upgrade the kinematic bed grid from imprecise to
   precise in order to make toolchanges reliable if your docks end up
   being fussy.

4. the _relative_ nozzle position between is precise, and can be
   stored between restarts in "save variables".

### Precision calibration, step–by–step

...
