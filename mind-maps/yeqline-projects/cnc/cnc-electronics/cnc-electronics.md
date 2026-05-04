## CNC Electronics {#cnc-electronics}

> type: category

How electrical power, deterministic timing, feedback, and safety turn toolpath commands into controlled machine motion.

### System Mental Model {#system-mental-model}

> type: category

Think in four simultaneous paths: commands tell the machine what to do, power lets it do work, feedback says what happened, and safety limits what is allowed.

#### Command Path {#command-path}

> type: principle
> edge.related: controller-core-job, drive-core-job

G-code or a higher-level job becomes coordinated axis motion, then electrical commands to each drive.

##### Planner To Pulses {#planner-to-pulses}

> type: detail

In a step/dir system, the controller converts trajectory segments into timed step pulses and direction states.

##### Planner To Fieldbus {#planner-to-fieldbus}

> type: detail

In a fieldbus system, the controller sends cyclic position, velocity, or torque setpoints to intelligent drives.

#### Power Path {#power-path}

> type: principle
> edge.related: power-supplies, cabinet-power-chain

Energy flows from mains to protection, supplies, drives, motor windings, and finally mechanical load.

##### Logic Power {#logic-power}

> type: detail

Controllers, sensors, and opto inputs often need clean 5 V or 24 V logic power with a stable reference.

##### Motor Power {#motor-power}

> type: detail

Drives need a higher-energy supply or DC bus sized for current, acceleration, braking, and fault energy.

#### Feedback Path {#feedback-path}

> type: principle
> edge.related: feedback-hierarchy, following-error

Feedback closes the gap between commanded motion and actual motion at the drive, controller, or measurement layer.

##### Drive Feedback {#drive-feedback}

> type: detail

Closed-loop stepper and servo drives use motor encoders to regulate current, speed, position, or stall recovery.

##### Machine Feedback {#machine-feedback}

> type: detail

Industrial machines may use linear scales, probes, temperature sensors, or process sensors to measure the actual machine or process state.

#### Safety Path {#safety-path}

> type: principle
> edge.related: safety-rated-functions, emergency-stop-chain

Safety is a separate permission system that can remove torque, stop motion, or block restart even if the controller software fails.

##### Hardwired Stop {#hardwired-stop}

> type: detail

Emergency stop chains should not depend only on firmware, USB, Wi-Fi, or the G-code sender.

##### Restart Control {#restart-control}

> type: detail

After a safety trip or alarm, the machine should require intentional reset, re-enable, and often re-homing.

#### Running Example {#running-example}

> type: concept
> edge.related: fluidnc-controller, closed-loop-stepper-drive, dual-y-gantry

The example machine uses four 3.0 Nm NEMA 23 closed-loop steppers: X, Z, and two synchronized Y gantry motors driven by encoder drives from a 6-axis FluidNC controller.

### Controller Layer {#controller-layer}

> type: category

The controller owns the machine-level intent: coordinate systems, trajectory planning, timing, I/O logic, alarms, homing, probing, and job execution.

#### Controller Core Job {#controller-core-job}

> type: concept
> edge.related: command-path, io-core-job

A CNC controller interprets commands, plans coordinated motion, emits drive commands, watches inputs, and enforces machine state.

##### Interpreter {#interpreter}

> type: detail

Parses G-code, modal state, units, work offsets, feed modes, spindle commands, coolant commands, and program flow.

##### Trajectory Planner {#trajectory-planner}

> type: detail

Turns geometry into time-based motion while respecting acceleration, jerk, feed limits, corner blending, and axis constraints.

##### Real-Time Scheduler {#real-time-scheduler}

> type: detail
> edge.related: deterministic-timing

Generates motion updates at predictable times so axes stay synchronized instead of receiving uneven commands.

##### State Machine {#controller-state-machine}

> type: detail

Tracks idle, cycle, hold, homing, alarm, reset, and safety states so outputs match machine condition.

#### Controller Categories {#controller-categories}

> type: concept

Main CNC controller categories differ by where motion timing, I/O logic, user interface, and safety integration live.

##### Embedded Step-Dir {#embedded-step-dir}

> type: detail
> edge.related: fluidnc-controller

Firmware controllers such as FluidNC are compact and practical for routers, lasers, plasma tables, and small machines that use step/dir drives.

##### PC Parallel Port {#pc-parallel-port}

> type: detail

Older PC-based controllers generated real-time step pulses directly from a parallel port, but timing depends heavily on the host and OS.

##### External Motion Card {#external-motion-card}

> type: detail

A PC user interface can delegate real-time motion to an Ethernet, USB, or PCI motion card that owns pulse timing or fieldbus cycles.

##### Industrial CNC {#industrial-cnc}

> type: detail

Fanuc, Siemens, Heidenhain, Mitsubishi, and similar CNCs integrate high-performance motion, drives, I/O, safety, diagnostics, and metrology workflows.

##### PLC Motion Control {#plc-motion-control}

> type: detail

Industrial automation platforms use PLCopen-style function blocks, EtherCAT or Profinet motion, and safety PLCs for machines that blend CNC and automation.

##### Robot Controller {#robot-controller}

> type: detail

Robotics controllers solve multi-axis kinematics, coordinated servo loops, safe zones, and teach-pendant workflows rather than pure Cartesian G-code motion.

#### FluidNC Controller {#fluidnc-controller}

> type: concept
> edge.related: embedded-step-dir, example-axis-mapping

In the example, FluidNC is the 6-axis motion controller that maps axes to motor outputs and sends step, direction, and enable signals to external closed-loop stepper drives.

##### Six Axis Capacity {#six-axis-capacity}

> type: detail

The six motion channels can cover X, dual Y motors, Z, and still leave channels for rotary, tangential, or future machine expansion.

##### Grbl-Style Workflow {#grbl-style-workflow}

> type: detail

FluidNC keeps the familiar sender, G-code, alarm, homing, and work-coordinate workflow while moving configuration into machine-readable files.

##### Controller Limits {#controller-limits}

> type: warning

FluidNC is not a safety PLC or a high-end fieldbus servo CNC; safety and high-power fault handling need external rated hardware.

#### Example Axis Mapping {#example-axis-mapping}

> type: concept
> edge.related: dual-y-gantry, homing-strategy

Map one motor to X, one motor to Z, and two motors to Y so the gantry can square during homing and move as one coordinated Y axis during cutting.

##### X Axis Channel {#x-axis-channel}

> type: detail

The X motor output commands the bridge or carriage left-right depending on the mechanical layout.

##### Z Axis Channel {#z-axis-channel}

> type: detail

The Z motor output commands tool height and needs conservative acceleration because crashes and gravity effects are concentrated here.

##### Y Left Channel {#y-left-channel}

> type: detail

One Y motor drives one side of the gantry and should have its own drive, encoder loop, and homing reference.

##### Y Right Channel {#y-right-channel}

> type: detail

The second Y motor drives the other side of the gantry and must be configured to match direction, steps per unit, current, and homing behavior.

### Drive Layer {#drive-layer}

> type: category

The drive is the power electronics and control layer between low-energy commands and high-energy motor windings.

#### Drive Core Job {#drive-core-job}

> type: concept
> edge.related: power-path, motor-phase-current

A motor driver or drive translates controller commands into controlled voltage and current while protecting itself, the motor, and sometimes the load.

##### Command Interface {#command-interface}

> type: detail

The drive accepts step/dir, analog velocity, PWM, serial commands, or cyclic fieldbus setpoints depending on the control architecture.

##### Power Conversion {#power-conversion}

> type: detail

Switching transistors chop the DC bus into motor phase currents with fast current regulation and thermal protection.

##### Loop Ownership {#loop-ownership}

> type: detail
> edge.related: feedback-hierarchy

The drive may own only current control, or it may also own velocity and position loops in servo and closed-loop stepper systems.

##### Fault Boundary {#fault-boundary}

> type: detail

Drives detect overcurrent, overvoltage, undervoltage, overtemperature, encoder faults, following error, and stall conditions before damage grows.

#### Drive Families {#drive-families}

> type: concept

Motor drive categories are best understood by motor physics, feedback depth, and command interface.

##### Open-Loop Stepper Driver {#open-loop-stepper-driver}

> type: detail
> edge.contrasts: closed-loop-stepper-drive

Regulates phase current according to commanded steps but does not know whether the rotor actually followed.

##### Closed-Loop Stepper Drive {#closed-loop-stepper-drive}

> type: detail
> edge.related: closed-loop-steppers, motor-encoder

Uses an encoder to compare commanded and actual rotor position, improving stall detection, torque use, and alarm behavior.

##### Servo Drive {#servo-drive}

> type: detail
> edge.related: servo-systems, following-error

Controls a servo motor with current, velocity, and position loops, usually with encoder feedback and higher dynamic bandwidth.

##### VFD Spindle Drive {#vfd-spindle-drive}

> type: detail

Controls an induction or synchronous spindle motor by varying frequency, voltage, and sometimes vector current for speed and torque.

##### Linear Motor Drive {#linear-motor-drive}

> type: detail

Controls a direct-drive linear motor where stiffness, thermal growth, and scale feedback matter more than screw backlash.

##### Process Power Drive {#process-power-drive}

> type: detail

Plasma, laser, EDM, waterjet, and additive systems add process power supplies whose control loops interact with motion.

#### Stepper Driver Duties {#stepper-driver-duties}

> type: concept
> edge.related: open-loop-stepper-driver, closed-loop-stepper-drive

A stepper driver is responsible for current control, microstepping, timing acceptance, protection, and clear fault signaling.

##### Step Translation {#step-translation}

> type: detail

Each step pulse advances the commanded electrical angle by one microstep while direction selects sign.

##### Microstepping {#microstepping}

> type: detail

Microstepping shapes sine-like phase currents to reduce vibration and resonance, but it does not create full-step torque at every microstep.

##### Current Regulation {#current-regulation}

> type: detail
> edge.related: motor-phase-current

Chopper current control limits winding current despite changing inductance, speed, bus voltage, and back EMF.

##### Decay Mode {#decay-mode}

> type: detail

Fast, slow, and mixed decay settings affect current ripple, torque smoothness, heat, and high-speed behavior.

##### Timing Requirements {#timing-requirements}

> type: detail
> edge.related: signal-integrity

The controller must meet drive requirements for step pulse width, direction setup time, enable timing, and opto input current.

##### Protection Functions {#protection-functions}

> type: detail

Good drivers protect against overcurrent, short circuit, overtemperature, supply faults, and encoder mismatch where applicable.

##### Alarm Output {#alarm-output}

> type: detail
> edge.related: alarm-chain

Closed-loop drives should expose a fault output so the controller or safety logic can stop coordinated motion when one axis fails.

#### Motor Phase Current {#motor-phase-current}

> type: principle

Torque comes from controlled winding current, not from the step pulse itself; the pulse is only a command.

##### Inductance Limit {#inductance-limit}

> type: detail

Stepper windings resist rapid current change, so high speed requires enough bus voltage and a driver that can force current quickly.

##### Back EMF {#back-emf}

> type: detail

As speed rises, the motor generates voltage opposing the drive, reducing available current and torque margin.

##### Heat Budget {#heat-budget}

> type: detail

Motor current, drive losses, enclosure temperature, and duty cycle determine safe continuous operation more than holding torque alone.

### Motors And Feedback {#motors-and-feedback}

> type: category

Motors create force, feedback observes motion, and the chosen loop structure determines accuracy, stiffness, bandwidth, and fault behavior.

#### Open vs Closed Loop {#open-vs-closed-loop}

> type: concept

Open-loop systems trust the command; closed-loop systems compare command to measurement and react.

##### Open-Loop Steppers {#open-loop-steppers}

> type: concept
> edge.contrasts: closed-loop-steppers

Open-loop steppers are simple and deterministic when torque margin is high, but missed steps can go unnoticed until the part or machine reveals the error.

##### Closed-Loop Steppers {#closed-loop-steppers}

> type: concept
> edge.related: closed-loop-stepper-drive, following-error

Closed-loop steppers keep stepper mechanics and step/dir control while adding encoder-based correction, stall alarms, and better low-speed behavior.

##### True Servos {#true-servos}

> type: concept
> edge.related: servo-drive, servo-systems

True servo systems are designed around continuous feedback control, higher speeds, better inertia matching, and tighter following-error management.

#### Feedback Hierarchy {#feedback-hierarchy}

> type: principle
> edge.related: drive-feedback, machine-feedback

The feedback sensor should measure the error you actually care about, as close to the tool or work as practical.

##### Motor Encoder {#motor-encoder}

> type: detail

Measures motor shaft position and helps the drive control rotor motion, but it cannot see belt stretch, screw pitch error, backlash, or frame deflection.

##### Linear Scale {#linear-scale}

> type: detail

Measures table or axis position directly and can close the loop around screw error, thermal drift, and compliance.

##### Spindle Encoder {#spindle-encoder}

> type: detail

Supports rigid tapping, threading, orientation, speed verification, and process synchronization.

##### Probe And Tool Setter {#probe-and-tool-setter}

> type: detail

Touch probes and tool setters connect electronics to metrology so work offsets, tool lengths, and inspection data become part of the control loop.

#### Following Error {#following-error}

> type: principle
> edge.related: alarm-output, deterministic-timing

Following error is the difference between commanded and actual position; high-end systems treat it as a diagnostic, tuning signal, and safety clue.

##### Error Window {#error-window}

> type: detail

The allowed error must be wide enough for acceleration transients and tight enough to detect stalls, crashes, and tuning problems.

##### Tuning Tradeoff {#tuning-tradeoff}

> type: detail

Higher gains improve stiffness but can excite resonance, amplify noise, and overload mechanical components.

##### Mechanical Compliance {#mechanical-compliance}

> type: detail

Belts, couplers, screws, bearings, frames, and tool deflection can dominate accuracy even when motor feedback looks perfect.

#### Dual Y Gantry {#dual-y-gantry}

> type: concept
> edge.related: example-axis-mapping, squaring-homing

The example machine uses two closed-loop Y motors because a wide gantry needs torque on both sides and must avoid racking.

##### Matched Drive Setup {#matched-drive-setup}

> type: detail

Both Y drives should match current limit, microstep setting, pulse timing, encoder direction, acceleration, and fault behavior.

##### Squaring Homing {#squaring-homing}

> type: detail
> edge.related: homing-strategy

Independent Y home switches allow each side to reference separately before coordinated cutting, reducing gantry skew.

##### Gantry Fault Case {#gantry-fault-case}

> type: warning

If one Y drive faults while the other continues, the gantry can rack hard; fault outputs should stop all coordinated motion.

### Power And Noise {#power-and-noise}

> type: category

Power quality, grounding, shielding, and physical wiring decide whether a CNC cabinet behaves like an instrument or like a noise antenna.

#### Power Supplies {#power-supplies}

> type: concept
> edge.related: power-path, cabinet-power-chain

Separate supply roles make troubleshooting easier: motor bus power, logic power, sensor power, brake power, and process power should be planned intentionally.

##### Motor Supply Sizing {#motor-supply-sizing}

> type: detail

Size voltage for speed and current response, current for simultaneous axis load, and reserves for acceleration and heating.

##### 24 V Controls {#twenty-four-volt-controls}

> type: detail

Industrial sensors, relays, contactors, and safety devices usually prefer 24 V control power because it is robust over cabinet distances.

##### 5 V Interfaces {#five-volt-interfaces}

> type: detail

Many stepper drive opto inputs expect 5 V signaling, so verify that the controller output and drive input current are compatible.

##### Braking Energy {#braking-energy}

> type: detail

Decelerating axes and spindles return energy to the DC bus, which may require braking resistors, regenerative drives, or conservative decel limits.

#### Cabinet Power Chain {#cabinet-power-chain}

> type: concept

The cabinet should have a legible chain from mains entry to disconnect, fusing, filtering, contactors, supplies, drives, and loads.

##### Mains Disconnect {#mains-disconnect}

> type: detail

A lockable disconnect and branch protection define the maintenance boundary before internal power distribution begins.

##### Circuit Protection {#circuit-protection}

> type: detail

Fuses, breakers, and electronic protection should be sized for wire ampacity, supply limits, and fault-clearing behavior.

##### Contactor Strategy {#contactor-strategy}

> type: detail

Contactors can remove drive or spindle power, but drive-integrated safety such as STO is often a cleaner motion-safety mechanism.

#### Grounding And Bonding {#grounding-and-bonding}

> type: concept
> edge.related: noise-immunity

Grounding is for fault current and reference stability; shielding is for noise control; mixing those mental models causes cabinet problems.

##### Protective Earth {#protective-earth}

> type: detail

Bond metal enclosures, machine frame, motor cases, cable shields, and exposed conductive parts to protective earth for fault safety.

##### Signal Reference {#signal-reference}

> type: detail

Logic common is a circuit reference, not a substitute for protective earth; decide where references connect and avoid accidental loops.

##### Star Vs Plane {#star-vs-plane}

> type: detail

Small machines often use star-like returns, while industrial cabinets use bonded metal backplanes and short low-impedance connections.

#### Shielding And Routing {#shielding-and-routing}

> type: concept
> edge.related: signal-integrity

Route high-energy switching cables away from low-level signals, and terminate shields deliberately instead of leaving them decorative.

##### Motor Cables {#motor-cables}

> type: detail

Motor phase cables carry fast switching currents and should be shielded, mechanically protected, and separated from encoder and sensor wiring.

##### Encoder Cables {#encoder-cables}

> type: detail

Encoder wiring needs twisted pairs, shield continuity, correct termination, and separation from VFD, plasma, laser, and motor power wiring.

##### VFD Cables {#vfd-cables}

> type: detail

Spindle VFD output cables are a major EMI source and should use VFD-rated shielded cable, proper shield termination, and routing discipline.

##### Ferrites And Filters {#ferrites-and-filters}

> type: detail

Filters, ferrites, snubbers, and surge suppressors solve specific noise paths; they do not replace good layout and bonding.

#### Signal Integrity {#signal-integrity}

> type: concept
> edge.related: timing-requirements, differential-signals

Signal integrity means the receiver sees the intended timing, voltage, polarity, and edge quality after the real cable, connectors, noise, and grounding are included.

##### Pulse Shape {#pulse-shape}

> type: detail

Long cables, weak drivers, opto inputs, and capacitance can stretch edges enough to create missed or extra steps.

##### Common-Mode Range {#common-mode-range}

> type: detail

Differential interfaces still fail if the voltage between device grounds exceeds the receiver common-mode limit.

#### Noise Immunity {#noise-immunity}

> type: principle

Noise immunity is engineered by lowering source emissions, reducing coupling paths, and increasing receiver tolerance.

##### Differential Signals {#differential-signals}

> type: detail

RS-485, encoder differential pairs, and industrial fieldbuses reject common-mode noise better than single-ended cabinet wiring.

##### Opto Isolation {#opto-isolation}

> type: detail

Opto-isolated inputs reduce damage and ground-noise coupling, but still need enough input current and correct polarity.

##### Debounce And Filtering {#debounce-filtering}

> type: detail

Limit switches, probes, and E-stop feedback need hardware and software filtering chosen so noise is ignored but real events are not delayed dangerously.

#### Example Wiring Priorities {#example-wiring-priorities}

> type: concept
> edge.related: running-example, five-volt-interfaces

For the FluidNC closed-loop stepper example, verify 5 V step/dir compatibility, bring every drive alarm into the controller or safety chain, and keep encoder and motor cables separated.

### I/O And Safety {#io-and-safety}

> type: category

Machine I/O tells the controller what the machine is allowed to do, while safety-rated hardware decides what the machine is allowed to energize.

#### I/O Core Job {#io-core-job}

> type: concept
> edge.related: controller-core-job

I/O connects abstract controller states to real devices: switches, probes, relays, valves, spindles, pumps, lasers, torches, brakes, and alarms.

##### Inputs {#inputs}

> type: detail

Inputs include limits, homes, probes, door interlocks, pressure switches, arc-ok, coolant flow, drive-ready, and drive-alarm signals.

##### Outputs {#outputs}

> type: detail

Outputs include enables, spindle commands, coolant, mist, air blast, solenoids, brakes, lamps, process start, and alarm indicators.

##### Input Polarity {#input-polarity}

> type: detail

Normally-closed circuits are often preferred for critical signals because broken wires become visible faults.

#### Limits And Homing {#limits-and-homing}

> type: concept

Limits prevent travel beyond safe boundaries; homing establishes a repeatable machine coordinate reference.

##### Limit Switches {#limit-switches}

> type: detail

Limit switches are last-resort travel boundaries and should stop motion before mechanics hit hard stops.

##### Home Switches {#home-switches}

> type: detail

Home switches are reference sensors and should be repeatable, protected, and placed where homing motion is mechanically safe.

##### Soft Limits {#soft-limits}

> type: detail

Soft limits use the known machine coordinate system to prevent commands outside the valid work envelope.

##### Hard Limits {#hard-limits}

> type: detail

Hard limits watch physical inputs during motion and put the controller into alarm when a boundary switch trips.

#### Homing Strategy {#homing-strategy}

> type: concept
> edge.related: example-axis-mapping, squaring-homing

The example should home X, Z, and both Y sides in a sequence that avoids collisions, squares the gantry, and leaves a known machine zero.

##### Pull-Off Distance {#pull-off-distance}

> type: detail

After switch contact, the axis backs off enough to clear the sensor and establish a stable post-home state.

##### Search And Latch {#search-and-latch}

> type: detail

A faster search move finds the switch, then a slower latch move improves repeatability.

##### Dual-Y Switches {#dual-y-switches}

> type: detail

Two independent Y home switches let the controller correct gantry skew before treating both Y motors as a synchronized axis.

#### Alarm Chain {#alarm-chain}

> type: concept
> edge.related: alarm-output, gantry-fault-case

Alarms should propagate from drives and process devices to the controller state machine and, for hazardous cases, to safety hardware.

##### Drive Alarm Input {#drive-alarm-input}

> type: detail

Each closed-loop stepper drive should expose alarm or fault status so a stall or encoder fault cannot be hidden.

##### Controller Alarm State {#controller-alarm-state}

> type: detail

Controller alarm state blocks further motion until the cause is understood and the operator intentionally clears it.

##### Diagnostic Labels {#diagnostic-labels}

> type: detail

Wire labels, terminal numbers, LED indicators, and documented I/O names make faults diagnosable instead of mysterious.

#### Emergency Stop Chain {#emergency-stop-chain}

> type: concept
> edge.related: safety-path, safety-rated-functions

E-stop is a safety function design problem, not merely an input pin that asks firmware to pause.

##### Stop Category 0 {#stop-category-zero}

> type: detail

Immediate torque removal is simple but may allow uncontrolled coast, lost position, or falling vertical loads.

##### Stop Category 1 {#stop-category-one}

> type: detail

Controlled deceleration followed by torque removal is often better for high-inertia spindles, gantries, and vertical axes.

##### Reset Discipline {#reset-discipline}

> type: detail

Reset should restore permission to restart, not automatically restart motion or re-energize hazards.

##### Software Stop Trap {#software-stop-trap}

> type: warning

A feed hold, pause button, or sender disconnect is operational control; it is not a substitute for a rated emergency stop chain.

#### Safety-Rated Functions {#safety-rated-functions}

> type: concept
> edge.related: industrial-safety, emergency-stop-chain

Drive-integrated safety functions remove or limit hazardous motion with defined behavior and safety performance.

##### STO {#sto}

> type: detail

Safe Torque Off removes drive output torque without necessarily removing mains power or providing electrical isolation for maintenance.

##### SS1 {#ss1}

> type: detail

Safe Stop 1 monitors or commands deceleration, then transitions to STO after the stop condition is met.

##### SLS {#sls}

> type: detail

Safely Limited Speed allows controlled motion only below a verified speed limit, useful for setup, jog, and guarded access modes.

##### SOS {#sos}

> type: detail

Safe Operating Stop monitors standstill while the drive remains energized, useful where holding position matters.

### Industrial Practices {#industrial-practices}

> type: category

High-end machines scale the same ideas with tighter timing, richer feedback, rated safety, diagnostics, and metrology tied into production quality.

#### Industrial Safety {#industrial-safety}

> type: concept
> edge.related: safety-rated-functions, emergency-stop-chain

Industrial safety starts with risk assessment, then assigns rated hardware and validation evidence to each hazard instead of treating safety as normal control logic.

##### Risk Assessment {#risk-assessment}

> type: detail

Identify severity, exposure, avoidance possibility, and required safety performance before choosing relays, safety PLCs, guard locks, or drive safety functions.

##### Validation Evidence {#validation-evidence}

> type: detail

Document wiring, fault reaction, reset behavior, stop time, safety function tests, and maintenance intervals so the machine remains safe after commissioning.

#### Servo Systems {#servo-systems}

> type: concept
> edge.related: servo-drive, true-servos

Industrial CNCs favor servos when speed range, acceleration, stiffness, duty cycle, and following-error control matter more than simplicity.

##### Current Loop {#current-loop}

> type: detail

The inner loop regulates motor torque-producing current and usually runs fastest.

##### Velocity Loop {#velocity-loop}

> type: detail

The velocity loop damps motion and rejects load disturbances before they become large position errors.

##### Position Loop {#position-loop}

> type: detail

The position loop tracks commanded trajectory and manages following error at the axis level.

##### Feedforward {#feedforward}

> type: detail

Velocity, acceleration, friction, and gravity feedforward reduce error before feedback has to correct it.

#### Fieldbus Motion {#fieldbus-motion}

> type: concept
> edge.related: deterministic-timing, cia-402-profile

Industrial fieldbus motion moves from pulse trains to synchronized cyclic data with drive state, commands, feedback, and diagnostics in one network.

##### EtherCAT {#ethercat}

> type: detail

EtherCAT is widely used for high-axis-count synchronized motion because distributed clocks can align device timing with sub-microsecond class precision.

##### Profinet IRT {#profinet-irt}

> type: detail

Profinet IRT is common in Siemens-centered automation where deterministic Ethernet integrates PLC, drives, I/O, and safety.

##### Sercos And Mechatrolink {#sercos-mechatrolink}

> type: detail

Sercos and MECHATROLINK are established motion networks in machine-tool and servo automation ecosystems.

##### CiA 402 Profile {#cia-402-profile}

> type: detail

CiA 402 defines common drive behavior such as state machine, control word, status word, operation modes, and process data for servo, inverter, and stepper drives.

#### Deterministic Timing {#deterministic-timing}

> type: principle
> edge.related: real-time-scheduler, ethercat

Coordinated motion depends on bounded latency and jitter because each axis command must arrive at the right time relative to all others.

##### Cycle Time {#cycle-time}

> type: detail

Servo and fieldbus cycles are chosen to match machine dynamics, drive bandwidth, and required contour accuracy.

##### Jitter Budget {#jitter-budget}

> type: detail

Timing jitter becomes path error, velocity noise, or surface finish error when axes do not update evenly.

##### Timestamped Feedback {#timestamped-feedback}

> type: detail

High-end systems timestamp I/O and feedback so measurements can be correlated with motion instead of guessed afterward.

#### Diagnostics {#diagnostics}

> type: concept

Diagnostics turn faults into evidence: what failed, when it failed, under what command, and what the machine did next.

##### Drive Trace {#drive-trace}

> type: detail

Servo drives can record current, speed, position, following error, bus voltage, and fault bits around an event.

##### Alarm Taxonomy {#alarm-taxonomy}

> type: detail

Good alarm systems distinguish limit hit, following error, encoder loss, overtravel, overcurrent, overtemperature, and safety circuit open.

##### Commissioning Data {#commissioning-data}

> type: detail

Capture motor parameters, I/O maps, homing results, tune files, firmware versions, and safety validation evidence before production use.

#### Metrology Feedback {#metrology-feedback}

> type: concept
> edge.related: machine-feedback, probe-and-tool-setter

Metrology closes the loop around machine truth, not just motor truth.

##### Glass Scales {#glass-scales}

> type: detail

Linear glass scales can correct screw pitch error, thermal drift, and mechanical compliance when integrated into the control loop.

##### Thermal Compensation {#thermal-compensation}

> type: detail

Machine tools compensate spindle growth, ball screw heating, frame expansion, and process temperature when precision demands it.

##### Probe Compensation {#probe-compensation}

> type: detail

Probing can update work offsets, detect fixture error, inspect features, and adapt machining to real stock position.

#### Cross-Industry Patterns {#cross-industry-patterns}

> type: concept

Different CNC industries reuse the same electronics patterns with different dominant risks.

##### Milling And Turning {#milling-turning}

> type: detail

Prioritize spindle synchronization, tool changer interlocks, rigid tapping, coolant control, and high stiffness servo axes.

##### Grinding {#grinding}

> type: detail

Prioritizes thermal stability, high-resolution feedback, wheel dressing, vibration control, and extremely repeatable infeed.

##### Plasma And Laser {#plasma-laser}

> type: detail

Prioritize torch or laser enable safety, arc-ok or beam-ready signals, height control, EMI management, and fire risk controls.

##### Waterjet {#waterjet}

> type: detail

Prioritizes pump pressure, abrasive feed, pierce sequencing, water management, and safety around high-pressure energy.

##### Additive And Robotics {#additive-robotics}

> type: detail

Prioritize coordinated process energy, multi-axis kinematics, collision zones, sensor fusion, and safe human interaction.

#### Upgrade Path {#upgrade-path}

> type: concept
> edge.related: running-example, fieldbus-motion

The example FluidNC machine can teach core CNC electronics, while the industrial upgrade path replaces step/dir timing with fieldbus drives, rated safety, and richer feedback.

##### Better Stepper Cabinet {#better-stepper-cabinet}

> type: detail

First improve wiring, shielding, alarm capture, homing, documentation, and power distribution before changing the control architecture.

##### Servo Conversion {#servo-conversion}

> type: detail

A servo upgrade changes motor sizing, inertia matching, tuning, braking, fault handling, and often mechanical expectations.

##### Fieldbus Conversion {#fieldbus-conversion}

> type: detail

A fieldbus upgrade moves command, feedback, diagnostics, and drive state into a deterministic network and usually changes the controller class.

##### Safety Upgrade {#safety-upgrade}

> type: detail

A safety upgrade adds risk assessment, safety relay or safety PLC, STO wiring, guard interlocks, validation, and documented reset behavior.
