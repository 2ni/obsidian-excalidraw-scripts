### Collection of Excalidraw scripts for Obsidian

#### Scale Properties Panel
This script allows to manage multiple coordinate systems. You can choose which one is active and you'll see x and y in px, mm or even meters if you define a scale. It's useful to represent real plans or if you need to place elements at exact locations. By using a custom coordinate system, it's often easier to place additional elements if they need a place relative to a given element.

The panel assumes we work with 96px/mm (which seems to be the case in Ecalidraw) to handle px and mm correctly. The meters are calculated based on the scale defined, eg 1:100.

1. open the scale properties panel via the command palette
2. create new coordinate systems by clicking on "New"
  * the system is defined by a line by setting 2 points
  * use command to snap the 1st point on highlighted points (eg center of circle)
  * to set the 2nd point similar to the first. You can use shift to snap at angles of 15°
  * when the 2nd point is set, the coordinate system will be visible
  * use the dropdown to switch between coordinate systems. The one in use will be highlighted
3. for better use, define a printable area first with the "Printable Layout Wizard"

For now only single elements are supported (no groups).

![Scale Properties Panel](Scale%20Properties%20Panel.png)

#### Perpendicular Line
This scripts adds perpendicular lines between the center of objects (circle, rectangle, diamond) and a given line. The color and width will be the same as the line.
![Perpendicular Line](Perpendicular%20Line.png)

#### Metakeys
WORK IN PROGRESS

This scripts shows buttons for meta keys, which is very helpful on mobile or ipads, to eg draw horizontal lines.

#### Add crosshair to origin
This script adds a crosshair at the origin (0/0) for reference
