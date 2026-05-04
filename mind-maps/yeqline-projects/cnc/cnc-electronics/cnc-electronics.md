## CNC Electronics {#cnc-electronics}

> type: category

Root question: how do CNC electronics convert a toolpath into safe, accurate, diagnosable machine motion?

### Whole System Model {#whole-system-model}

> type: category

Understand the machine as four paths that meet at each axis: command, power, feedback, and safety.

#### Four Paths {#four-paths}

> type: concept

Most CNC electronics problems become clearer when you ask which path is carrying the wrong information, energy, measurement, or permission.

##### Command Path {#command-path}

> type: concept

G-code, jogs, probing moves, and process commands become coordinated axis motion and output states.

##### Power Path {#power-path}

> type: concept

Mains and power supplies deliver usable energy to drives, motors, spindle, pumps, valves, and process tools.

##### Feedback Path {#feedback-path}

> type: concept

Encoders, scales, probes, switches, sensors, and drive status report what actually happened.

##### Safety Path {#safety-path}

> type: concept

Safety hardware decides whether hazardous energy is allowed, even when normal control logic is wrong.

#### System Boundaries {#system-boundaries}

> type: concept

Good designs keep controller decisions, drive control, machine mechanics, and safety authority separate.

##### Controller Boundary {#controller-boundary}

> type: concept

The controller owns trajectory planning, machine state, I/O logic, homing, alarms, and command timing.

##### Drive Boundary {#drive-boundary}

> type: concept

The drive owns power electronics, current control, motor feedback handling, protection, and local faults.

##### Machine Boundary {#machine-boundary}

> type: concept

The machine structure turns motor torque into tool motion through real stiffness, backlash, friction, and thermal growth.

##### Safety Boundary {#safety-boundary}

> type: concept

Safety hardware should be able to remove or limit hazardous motion without trusting the G-code sender or firmware.

#### Example Signal Chain {#example-signal-chain}

> type: concept

The running example uses a 6-axis FluidNC controller, four 3.0 Nm NEMA 23 closed-loop steppers, and encoder drives for X, Z, and dual Y.

##### FluidNC To Drives {#fluidnc-to-drives}

> type: concept

FluidNC sends step, direction, and enable signals to four external closed-loop stepper drives.

##### Drives To Motors {#drives-to-motors}

> type: concept

Each drive converts low-power commands into controlled phase current for one motor.

##### Encoders To Drives {#encoders-to-drives}

> type: concept

Each motor encoder returns position information to its own drive, which handles the closed loop locally.

##### Alarms To Controller {#alarms-to-controller}

> type: concept

Each drive alarm should return to the controller or safety chain so one failed axis stops coordinated motion.

### Controller Layer {#controller-layer}

> type: category

The CNC controller is the machine-level decision maker. It decides what motion should happen, not how motor current is switched.

#### Controller Job {#controller-job}

> type: concept

A controller interprets commands, plans motion, schedules outputs, reads inputs, and keeps the machine in a valid operating state.

##### Program Interpretation {#program-interpretation}

> type: concept

Tracks G-code modes, units, offsets, feed modes, spindle commands, coolant, and program flow.

##### Trajectory Planning {#trajectory-planning}

> type: concept

Turns geometry into time-based motion while respecting velocity, acceleration, cornering, and axis limits.

##### Motion Timing {#motion-timing}

> type: concept

For step/dir systems, pulse timing is the controller's real-time promise to the drives.

##### Machine I/O State {#machine-io-state}

> type: concept

Connects software state to homes, limits, probes, relays, spindle commands, coolant, process outputs, and alarms.

##### Alarm State {#alarm-state}

> type: concept

Separates running, hold, homing, alarm, reset, and safety-interlocked behavior so outputs match machine condition.

#### Controller Categories {#controller-categories}

> type: concept

CNC controllers differ by where real-time motion, I/O, safety integration, and the operator interface live.

##### Embedded Firmware {#embedded-firmware}

> type: concept

FluidNC and GRBL-style controllers are compact step/dir controllers for small to mid-size machines.

##### PC Motion System {#pc-motion-system}

> type: concept

PC systems use a computer for interface and planning, with real-time kernels or external timing hardware when needed.

##### Industrial CNC {#industrial-cnc}

> type: concept

Industrial CNCs integrate high-performance motion, drives, I/O, probing, diagnostics, safety, and production workflows.

##### PLC Motion {#plc-motion}

> type: concept

PLC motion systems combine automation logic, motion function blocks, fieldbus drives, distributed I/O, and safety PLCs.

##### Robot Controller {#robot-controller}

> type: concept

Robot controllers emphasize kinematics, joint coordination, safe zones, teach workflows, and coordinated process tools.

#### FluidNC Example {#fluidnc-example}

> type: concept

In this machine, FluidNC owns the axis commands while the closed-loop drives own motor feedback.

##### Axis Allocation {#axis-allocation}

> type: concept

Use one channel for X, one for Z, and two for the Y gantry, leaving spare capacity on a 6-axis controller.

##### Step-Dir Interface {#step-dir-interface}

> type: concept

Step/dir says how far and when to move; the drive decides how to energize the motor.

##### Configuration Discipline {#configuration-discipline}

> type: concept

Steps per unit, max rate, acceleration, homing direction, limit polarity, and inversion must match the real machine.

##### Dual-Y Coordination {#dual-y-coordination}

> type: concept

Two powered Y sides should home independently for squaring, then move as one mechanical axis during cutting.

##### Controller Limit {#controller-limit}

> type: warning

FluidNC is not a safety PLC or a high-end fieldbus CNC. Dangerous energy needs external rated hardware.

### Drive-Motor Layer {#drive-motor-layer}

> type: category

The drive-motor layer turns command timing into torque. This is where power electronics, motor physics, feedback, and protection meet.

#### What A Drive Does {#what-drive-does}

> type: concept

A motor drive is a controlled power amplifier between low-energy controller commands and high-energy motor windings.

##### Command Input {#command-input}

> type: concept

Drives may accept step/dir pulses, analog velocity, PWM, serial commands, or cyclic fieldbus setpoints.

##### Current Control {#current-control}

> type: concept

Motor torque comes from controlled winding current, not from the command signal itself.

##### Local Loops {#local-loops}

> type: concept

Simple drivers mostly regulate current; closed-loop stepper and servo drives also regulate measured motion.

##### Protection And Faults {#protection-and-faults}

> type: concept

Drives detect overcurrent, voltage faults, overheating, encoder faults, stalls, and following error before damage grows.

#### Drive Categories {#drive-categories}

> type: concept

Main CNC drive families follow the motor and process physics they control.

##### Open-Loop Stepper Driver {#open-loop-stepper-driver}

> type: concept
> edge.contrasts: closed-loop-stepper-drive

Regulates phase current from commanded steps but does not know whether the rotor followed.

##### Closed-Loop Stepper Drive {#closed-loop-stepper-drive}

> type: concept

Uses an encoder to detect position error, improve behavior, and raise alarms on lost motion.

##### Servo Drive {#servo-drive}

> type: concept

Controls a servo motor through feedback loops for current, velocity, and position.

##### Spindle VFD {#spindle-vfd}

> type: concept

Controls spindle speed and torque by modulating frequency, voltage, and sometimes vector current.

##### Direct And Process Drives {#direct-and-process-drives}

> type: concept

Linear motors, torque motors, plasma, laser, waterjet, EDM, and additive systems add specialized drive requirements.

#### Stepper Driver Duties {#stepper-driver-duties}

> type: concept

Stepper drivers hide significant electrical engineering behind a simple pulse input.

##### Step And Microstep {#step-and-microstep}

> type: concept

Each step pulse advances the commanded electrical angle; microstepping shapes phase currents to reduce vibration.

##### Chopper Current Control {#chopper-current-control}

> type: concept

The driver rapidly switches the supply to force winding current toward the requested value.

##### Timing And Enable {#timing-and-enable}

> type: concept

The controller must meet step width, direction setup, enable timing, input voltage, and input current requirements.

##### Alarm Reporting {#alarm-reporting}

> type: concept

Fault outputs let the controller stop the whole machine when one axis loses the ability to follow.

#### Open Vs Closed Loop {#open-vs-closed-loop}

> type: concept

The key difference is whether actual motion is measured and used, not whether the motor is physically a stepper.

##### Open-Loop Steppers {#open-loop-steppers}

> type: concept
> edge.contrasts: closed-loop-steppers

Open-loop steppers are simple and predictable when torque margin is high and disturbances are controlled.

##### Missed Step Failure {#missed-step-failure}

> type: warning

An open-loop missed step can remain invisible until the part, fixture, or machine position is wrong.

##### Closed-Loop Steppers {#closed-loop-steppers}

> type: concept

Closed-loop steppers add encoder-based correction, stall detection, and following-error alarms while keeping step/dir control.

##### Following Error {#following-error}

> type: concept

Following error is the gap between commanded and measured position. It is both a tuning signal and a fault signal.

#### Motor Physics {#motor-physics}

> type: concept

Motor sizing is about torque across speed, thermal limits, inertia, and the mechanical load, not holding torque alone.

##### Torque Current Link {#torque-current-link}

> type: concept

Stepper and servo torque depends on controlled winding current and magnetic alignment.

##### Inductance And Back EMF {#inductance-and-back-emf}

> type: concept

Inductance slows current rise, and back EMF at speed reduces voltage headroom for current control.

##### Thermal Budget {#thermal-budget}

> type: concept

Current setting, duty cycle, cooling, enclosure temperature, and drive losses determine safe continuous operation.

#### Example Drives {#example-drives}

> type: concept

The example machine uses four closed-loop encoder drives, one per NEMA 23 motor.

##### Per-Axis Tuning {#per-axis-tuning}

> type: concept

X, Z, and both Y sides should be current-limited and tuned for their actual mechanics.

##### Dual-Y Fault Stop {#dual-y-fault-stop}

> type: warning

If either Y drive faults while the other keeps driving, the gantry can rack and damage itself.

### Feedback And Accuracy {#feedback-and-accuracy}

> type: category

Feedback is only useful when you know what it measures: motor shaft, axis position, tool position, process state, or part quality.

#### Feedback Levels {#feedback-levels}

> type: concept

High-end systems push measurement closer to the real machining result.

##### Motor Encoder {#motor-encoder}

> type: concept

Measures motor shaft motion and helps the drive detect stalls or regulate servo loops.

##### Axis Scale {#axis-scale}

> type: concept

Measures the actual axis after some screw, belt, rack, or gearbox error has already happened.

##### Spindle Feedback {#spindle-feedback}

> type: concept

Enables rigid tapping, threading, spindle orientation, speed verification, and synchronized process control.

##### Probe And Process Feedback {#probe-and-process-feedback}

> type: concept

Probes, tool setters, arc voltage, laser height, pressure, load, vibration, and temperature connect control to the real process.

#### Loop Placement {#loop-placement}

> type: concept

Where the loop closes determines what errors electronics can correct.

##### Drive Loop {#drive-loop}

> type: concept

Closed-loop stepper drives usually close the loop at the motor, so they do not see screw error or gantry skew.

##### Controller Or Metrology Loop {#controller-or-metrology-loop}

> type: concept

Industrial controllers may close axis-scale loops or use probing and compensation to correct machine and part error.

#### Accuracy Stack {#accuracy-stack}

> type: concept

Part accuracy is the stack-up of electronics, mechanics, heat, tooling, and process force.

##### Resolution Is Not Accuracy {#resolution-is-not-accuracy}

> type: warning

Steps per unit and encoder counts set command granularity, but they do not prove the tool is where you think it is.

##### Mechanical Error {#mechanical-error}

> type: concept

Belts, screws, racks, couplers, bearings, and frames add backlash, pitch error, compliance, and lost motion.

##### Thermal And Process Error {#thermal-and-process-error}

> type: concept

Spindle growth, screw heating, frame expansion, cutting forces, and process forces move the real tool path.

#### Gantry Accuracy {#gantry-accuracy}

> type: concept

The example dual-Y gantry makes accuracy a geometry problem as much as an axis-control problem.

##### Independent Y Homes {#independent-y-homes}

> type: concept

Each Y side should have its own home reference so the gantry can square before cutting.

##### Hidden Gantry Error {#hidden-gantry-error}

> type: warning

Motor encoders can report perfect tracking while belts, racks, or the gantry structure still deflect.

### Cabinet And Wiring {#cabinet-and-wiring}

> type: category

The cabinet determines whether the electronics behave like a controlled instrument or a noise antenna.

#### Power Supplies {#power-supplies}

> type: concept

Separate supply roles make the system easier to size, protect, and diagnose.

##### Motor Bus Supply {#motor-bus-supply}

> type: concept

The motor supply must support speed, acceleration, current demand, heating, and drive voltage limits.

##### Control And Logic Power {#control-and-logic-power}

> type: concept

Industrial controls commonly use 24 V for sensors and relays, while controllers and some drive inputs may need compatible 5 V logic.

##### Braking Energy {#braking-energy}

> type: concept

Decelerating motors and spindles can return energy to the bus, requiring braking resistors, regeneration, or gentler decel limits.

#### Grounding And Bonding {#grounding-and-bonding}

> type: concept

Grounding is for fault safety and reference control. Shielding is for noise control. They touch, but they are not the same idea.

##### Protective Earth {#protective-earth}

> type: concept

Bond the cabinet, machine frame, motor cases, cable shields, and exposed conductive parts to protective earth.

##### Signal Common {#signal-common}

> type: concept

Logic common is a circuit reference and should be connected deliberately, not accidentally through random cables.

##### Shield Bonding Trap {#shield-bonding-trap}

> type: warning

Random shield and common connections can create noise currents through sensitive signal references.

#### Shielding And Routing {#shielding-and-routing}

> type: concept

Physical separation and shield termination matter before filters and software debounce.

##### Power Cable Routing {#power-cable-routing}

> type: concept

Motor phase and VFD cables carry fast switching current and should be separated from encoder, probe, and sensor wiring.

##### Feedback Cable Routing {#feedback-cable-routing}

> type: concept

Encoder and sensor cables need twisted pairs, shield continuity, connector quality, and distance from power wiring.

##### Cable Segregation {#cable-segregation}

> type: concept

Plan separate routes for high-power switching, low-level feedback, safety, and communications wiring.

#### Noise Immunity {#noise-immunity}

> type: concept

Noise immunity comes from reducing emissions, reducing coupling, and making receivers more tolerant.

##### Differential Signaling {#differential-signaling}

> type: concept

Differential encoder, RS-485, and fieldbus signals reject common-mode noise better than single-ended cabinet wiring.

##### Isolation And Filtering {#isolation-and-filtering}

> type: concept

Opto isolation, debounce, filters, ferrites, snubbers, and surge suppressors help only when matched to the actual noise path.

#### Example Cabinet Priorities {#example-cabinet-priorities}

> type: concept

For the FluidNC closed-loop stepper machine, cabinet basics matter more than exotic electronics.

##### Verify Drive Interfaces {#verify-drive-interfaces}

> type: concept

Confirm step, direction, enable, and alarm voltage, polarity, timing, and input current requirements.

##### Wire For Diagnosis {#wire-for-diagnosis}

> type: concept

Bring every drive alarm back, label terminals, document I/O, and make fault states visible.

### Safety And Industrial Practice {#safety-and-industrial-practice}

> type: category

Industrial best practice scales CNC electronics with rated safety, deterministic timing, rich diagnostics, servo performance, and metrology feedback.

#### Limits Homing Alarms {#limits-homing-alarms}

> type: concept

Limits, homing, and alarms define the machine's legal operating envelope.

##### Home Vs Limit {#home-vs-limit}

> type: concept

Home switches establish coordinates; limit switches protect boundaries. Mixing those jobs causes confusion.

##### Soft And Hard Limits {#soft-and-hard-limits}

> type: concept

Soft limits reject impossible moves after homing; hard limits react to physical boundary inputs.

##### Alarm Chain {#alarm-chain}

> type: concept

Drive, process, limit, and safety faults should produce clear machine states and operator messages.

#### Emergency Stop {#emergency-stop}

> type: concept

Emergency stop is a safety function design problem, not just a spare controller input.

##### Feed Hold Is Not E-Stop {#feed-hold-is-not-e-stop}

> type: warning

Feed hold, pause, reset, or sender disconnect are operational controls, not rated emergency stop functions.

##### Stop And Reset Behavior {#stop-and-reset-behavior}

> type: concept

Stop category 0 removes torque immediately; stop category 1 decelerates first; reset must not automatically restart hazards.

#### Safety-Rated Motion {#safety-rated-motion}

> type: concept

Modern drives and safety controllers can limit hazardous motion with defined safety functions.

##### STO And SS1 {#sto-and-ss1}

> type: concept

Safe Torque Off prevents torque-producing output, while Safe Stop 1 stops first and then removes torque.

##### SLS And SOS {#sls-and-sos}

> type: concept

Safely Limited Speed permits guarded slow motion; Safe Operating Stop monitors standstill while holding position.

##### Safety Validation {#safety-validation}

> type: concept

Rated safety requires risk assessment, wiring discipline, stop-time proof, reset testing, and maintenance evidence.

#### Servo Practice {#servo-practice}

> type: concept

High-end CNC machines use servos when dynamic stiffness, speed range, duty cycle, and fault observability matter.

##### Cascaded Loops {#cascaded-loops}

> type: concept

Current, velocity, and position loops are tuned as a nested system, not as independent knobs.

##### Feedforward And Inertia {#feedforward-and-inertia}

> type: concept

Feedforward, friction compensation, gravity compensation, and inertia matching reduce error before feedback reacts.

#### Fieldbus Motion {#fieldbus-motion}

> type: concept

Fieldbus motion replaces loose pulse wiring with cyclic command, feedback, state, and diagnostics data.

##### Motion Networks {#motion-networks}

> type: concept

EtherCAT, Profinet IRT, Sercos, and MECHATROLINK are used for synchronized industrial motion.

##### CiA 402 Drive Profile {#cia-402-drive-profile}

> type: concept

CiA 402 standardizes drive state, control word, status word, operation modes, and cyclic process data.

##### Deterministic Timing {#deterministic-timing}

> type: concept

Cycle time, jitter, and synchronized feedback determine whether axes receive time-aligned commands and measurements.

#### Diagnostics And Metrology {#diagnostics-and-metrology}

> type: concept

Diagnostics explain what failed; metrology explains what the machine or part actually did.

##### Drive Trace And Alarms {#drive-trace-and-alarms}

> type: concept

Industrial drives can record current, speed, position, following error, bus voltage, and fault bits around an event.

##### Commissioning Record {#commissioning-record}

> type: concept

Keep motor parameters, tuning, I/O maps, firmware versions, homing settings, and safety validation records.

##### Metrology Feedback {#metrology-feedback}

> type: concept

Linear scales, probing, tool setters, and thermal compensation close the loop around machine truth and part truth.

#### Industry Variations {#industry-variations}

> type: concept

Different CNC industries reuse the same electronics ideas but emphasize different hazards and feedback.

##### Milling Turning Grinding {#milling-turning-grinding}

> type: concept

Milling, turning, and grinding emphasize spindle synchronization, stiffness, coolant, thermal stability, dressing, and precision feedback.

##### Plasma Laser Waterjet {#plasma-laser-waterjet}

> type: concept

Plasma, laser, and waterjet machines emphasize process enable safety, height control, EMI, pressure, and fire or arc hazards.

##### Additive Robotics {#additive-robotics}

> type: concept

Additive and robotic systems emphasize coordinated process energy, kinematics, collision zones, sensor fusion, and human interaction safety.

#### Upgrade Path {#upgrade-path}

> type: concept

The FluidNC closed-loop stepper machine is a good learning platform; industrial machines move more functions into rated hardware and deterministic networks.

##### Clean Stepper Cabinet {#clean-stepper-cabinet}

> type: concept

First improve power distribution, shielding, alarm wiring, homing, documentation, and fault tests.

##### Servo Fieldbus Safety {#servo-fieldbus-safety}

> type: concept

Industrial upgrades usually combine servo sizing, fieldbus diagnostics, drive safety functions, and formal validation.
