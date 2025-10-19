Great — below are practical, ready-to-run SPICE/Ngspice recipes and explanations to simulate CMOS switching threshold (VM) and perform dynamic switching measurements (delays, rise/fall, short-circuit current, and how to get dynamic energy). I include a DC method to find VM and a transient method (ramp input) to extract propagation delays and waveforms. I also explain what to look for and how velocity-saturation / low VDD affect the results.

1) Quick overview — what we’ll measure

VM (switching threshold): input voltage where the inverter output equals input (Vout = Vin).

Method A: DC sweep (Vout vs Vin) and find intersection.

Method B: Transient with slow ramp and detect the instant Vout = Vin.

Dynamic metrics (transient):

tpLH (propagation delay low→high) and tpHL (high→low) measured at 50% points.

tr / tf output rise and fall times (10→90% / 90→10%).

Ishort (peak): brief current from VDD to GND while both devices conduct.

Dynamic energy: integrate supply current × VDD over a transition (or compute C·V² for capacitive loads).

2) Recommended simulation setup

Use a realistic device length (e.g., L=60n or 60nm for short-channel demo) to show velocity saturation; compare with longer L (e.g., 1µm) for contrast.

Add a load capacitor (CL) on the output to model fanout/inertia (e.g., 10fF–100fF).

Drive input with a pulse that has finite rise/fall times (or a PWL ramp) so you can measure delays cleanly.

Use .measure statements to extract delays and times automatically (ngspice supports these .measure constructs; if your SPICE variant differs slightly, you can still extract using the plot cursor or export traces).

3) Ready-to-run Ngspice netlist

Save as inverter_dynamic_vtc.cir and run ngspice inverter_dynamic_vtc.cir.

* CMOS inverter VTC + dynamic switching measurement (ngspice)
* Usage: ngspice inverter_dynamic_vtc.cir

.param VDD=1.2        ; supply
.param Wn=10u Ln=60n   ; NMOS sizing (short-channel)
.param Wp=20u Lp=60n   ; PMOS sizing (make PMOS wider to balance)
.param CL=50f          ; load capacitance on output
.param Trise=1n        ; input rise/fall time
.param Tperiod=40n     ; pulse period
.param Ton=10n         ; pulse high time (for example)

******** Sources ********
* VDD source (labeled VVDD so current probe I(VVDD) measures supply current)
VVDD vdd 0 DC {VDD}

* Input pulse (finite edges to measure delays)
Vin in 0 PULSE(0 {VDD} 1n {Trise} {Trise} {Ton} {Tperiod})

******** MOSFET models (level=3 simple demo) ********
.model nshort nmos level=3 vt0=0.45 kp=120u
.model pshort pmos level=3 vt0=-0.45 kp=40u

******** Inverter (connected bulk nodes to rails) ********
M1 out in vdd vdd pshort W={Wp} L={Lp}    ; PMOS (source to VDD)
M2 out in 0 0 nshort W={Wn} L={Ln}        ; NMOS (source to GND)

******** Load ********
Cload out 0 {CL}

******** DC sweep to find VM (Vout vs Vin) ********
* Run DC sweep to produce Vout(Vin) curve — inspect plot to find VM where Vout=Vin
.dc Vin 0 {VDD} 0.005
.save V(out) V(in)

******** Transient run to measure delays and dynamic behavior ********
*.tran <tstep> <tstop>
* We'll run transient long enough to capture several pulses
.tran 0.1n 200n uic

******** Measurements (transient) ********
* Propagation delays at 50% VDD (use param expression)
.meas tran tpLH TRIG V(in) VAL={VDD/2} RISE=1 TARG V(out) VAL={VDD/2} RISE=1
.meas tran tpHL TRIG V(in) VAL={VDD/2} FALL=1 TARG V(out) VAL={VDD/2} FALL=1

* Output rise/fall times (10%->90%)
.meas tran trise TRIG V(out) VAL={0.1*VDD} RISE=1 TARG V(out) VAL={0.9*VDD} RISE=1
.meas tran tfall TRIG V(out) VAL={0.9*VDD} FALL=1 TARG V(out) VAL={0.1*VDD} FALL=1

* Peak instantaneous short-circuit current from VDD (during transition)
* I(VVDD) is the current through the voltage source VVDD (positive = from source into circuit)
.meas tran Ishort_max MAX I(VVDD)

* Average supply current over the transient (gives approximation of dynamic power when multiplied by VDD)
.meas tran Idd_avg AVG I(VVDD) FROM=0 TO=200n

* Save waveforms for post-processing/export
.save V(out) V(in) I(VVDD) I(M1) I(M2)

******** Notes ********
* To get VM precisely: run the .dc sweep and either:
*  - plot V(out) vs V(in) and visually find the intersection Vout=Vin
*  - or export the DC sweep data to CSV and compute numerically the Vin where Vout-Vin crosses zero.
*
* For dynamic energy of one transition:
*  - integrate I(VVDD)*VDD over the transition time (use waveform viewer integral or export I(VVDD) vs time and integrate externally),
*  - or approximate by 0.5*CL*(VDD)^2 for a full rail-to-rail charge of CL (valid if transitions charge/discharge CL).
*
.end

4) How to run & extract results (step-by-step)

ngspice inverter_dynamic_vtc.cir.

DC VM: after the .dc sweep finishes, in ngspice type plot V(out) vs V(in) and visually find the voltage where curve crosses the y=x line. (Some GUIs let you draw y=x; otherwise export .raw to CSV and compute the crossing numerically — see next bullet.)

To export sweep data: use wrdata vtc.csv V(out) V(in) in ngspice (or .save already included).

Transient: after .tran, open the waveform viewer (or use plot V(out) V(in) I(VVDD)).

Check .measure results: ngspice prints the .meas results to the console after the run. Look for tpLH, tpHL, trise, tfall, Ishort_max, and Idd_avg.

Dynamic energy:

Simple formula: E = 0.5 * CL * VDD^2 per full rail-to-rail transition (good first approximation).

Accurate: integrate P(t)=VDD * I(VVDD,t) over the transition time in waveform viewer or export I(VVDD) as CSV and numerically integrate.

5) What to expect & interpretation tips

VM (switching point):

With symmetric transistor sizing (Wp ≈ β ratio to balance drive) VM ≈ VDD/2.

If PMOS weaker → VM shifts lower (closer to VDD) and vice versa.

tpLH vs tpHL:

If PMOS is wider (bigger Wp), tpLH (rising output) reduces; if NMOS stronger, tpHL reduces.

Rise/fall times:

Increase with larger CL and reduced drive current (shorter L or velocity saturation limits peak current).

Ishort_max:

Peaks near VM because both transistors conduct. Higher input edge slope (fast input edges) → larger short-circuit current peak.

Effect of velocity saturation and short-channel:

In short-channel devices with velocity saturation, increasing VGS beyond a point does not proportionally raise ID; this reduces available drive current, which:

Increases tp (slower switching)

Reduces slope (gain) of VTC around VM

Can change VM slightly if PMOS/NMOS are affected differently

Low VDD / near-threshold operation:

Smaller VGS–Vth → both devices operate near threshold or subthreshold — gain is reduced, VM may shift, delays increase, energy per operation may reduce or increase depending on leakage vs dynamic tradeoffs.

6) Suggestions to extend the experiment

Compare device lengths: duplicate the inverter with Ln=1u Lp=1u and re-run to see classical (long-channel) vs velocity-saturated (short-channel) behavior.

Sweep CL: see how delay scales with capacitive load (tp ∝ CL / Ion).

Sweep Trise (input edge): short-circuit current depends strongly on input slope — quantify energy lost to short-circuit as input edges get faster.

Use a BSIM/Foundry model: include .include bsim4.lib to capture accurate velocity-saturation and DIBL/series resistance effects.

Automate param sweeps:

.step param L list 60n 180n 1u to step channel length.

.step param VDD list 1.2 0.9 0.5 to see low-node behavior.

7) If you want, next steps I can do right now (pick one)

Provide a second netlist that runs automated parameter sweeps (L, VDD, CL) and writes CSV results for delay vs parameter.

Make a PNG example plot (I can produce an example plot if you want me to run code here — tell me which plot: VTC, transient Vout/Vin, or Ion vs time).

Turn this into a PowerPoint slide that documents the netlist, typical plots, and interpretation points.
