# Overview
**ToolTime** lets you build Cartesian CNC machines with Intel Edison.

Rather than taking the traditional microcontroller approach, which assumes
a smaller memory, and run-time interpretation, **ToolTime** compiles SVG
or G-Code into a realtime bitstream format suitable for the Quark MCU, allowing
full program optimizations.

# Components

##Host
Components on the host are implemented in node.js as libraries with command
line wrappers. Everything is treated in a reactive fashion, allowing streams
of input and streams of events to work together.

### SVG -> G-Code Generator
Particularly useful for laser cutting or sheet goods, turning a vector image
into G-Code is effectively drawing with the toolhead on your machine.

### G-Code Parser
A stream of G-Code as text transformed into an observable sequence of parsed
intermediate structures. G-Code is a flat, modal language, so there is not
much of a parse tree, far more imperative and linear than tree shaped.

### Machine Configurator
Extract parsed instructions representing other-than-motion commands, such as:
* Machine configuration
* Coordinate systems
* Workspace coordinates
* Spindle control
* Tool selection

The transformation style map each instruction to an action and state, so
that an immutable observable of states is generated. In this fashion the state
of the machine is not *maintained*, but rather *precomputed*.

### G-Code Vectorizer
Given an observable sequence of parsed G-Code, mixed with machine states
from the Machine Configurator, map movement instructions to a vector and state.
This vectorization will expand nonlinear instructions such as arcs into linear
vector segments.

This results in a mixed observable of (action, state) and (vector, state).

Post vectorization all units will be metric.

### Vector Velocity Constraints
Before executing the actual velocity planning, set up a series of Constraints
on each vector. The *state* associated with each vector is extended with:
* entry max velocity
* max velocity
* exit max velocity

These velocities are in terms of feed rate, velocities along the vector not
component wise velocities.

The exit velocities are modeled based on the junction relationships
between sequential motion vectors **A** and **B** and their feed rate target
velocities.
* *cos*(**A**, **B**) <=  *junction*, and the target velocity is equal,
  keep a constant velocity by using the same segment time
* *cos*(**A**, **B**) > *junction* and *cos*(**A**, **B**) < 0,
  this is a change in direction slow to 0 on the exit from A
* Other changes in velocity, accelerate or decelerate on an s-curve into
  the next segment target velocity

Entry velocities are modeled on the exit velocity of the prior vector.


This results in a mixed observable of (action, state) and (vector, state).

### Vector Step Rasterizer
Given a observable of vectorized G-Code, map to an observable rasterized
sequence of vectors by translating metric distances into step distance
equivalents. This results in a mixed observable of (action, state) and
(rasterized, state).

This runs a cartesian robot inverse kinematics, making the mapping linear.

### Vector Velocity Planner
Given an observable sequence of (action, state) and (rasterized, state), emit
an observable sequence of (action, state) and (rasterized, time, state).

Velocity planning is about setting the time for a vector of [x, y, z] steps
considering:
* the component wise exit velocity of the previous vector
* the component wise target velocity of this vector
* the component wise target velocity of the next vector
* the component wise velocity limits
* the component wise jerk limits

This is a bit of a different approach than trying multiple iterations looking
at maximum attainable velocities on fixed time windows.

Start by recursive subdivision of the rasterized vector, down to the minimum
resolution step limit of the shortest vector in motion into *segments*. Motion
planning is then assigning times to these segments, and thus a velocity.

(jerk_equations.pdf) provides a good reference on the math. The difference in
the approach here, we will be solving for time in each segment, and the inflection
points on acceleration curves are not determined by time, but instead by reaching
a halfway point of velocity in a velocity transition.

Acceleration is a constant jerk velocity S curve, transitioning through increasing and
then decreasing acceleration around the midpoint of delta-velocity. Deceleration
is a similar S curve, but with negative acceleration.





### Bitstream Encoder

#### Configuration Encoder

#### Vector Encoder

#### Event Registration

### Realtime Bitstream Decoder

#### Digital Pin Signals

#### PWM Pin Signals

#### Interrupts
