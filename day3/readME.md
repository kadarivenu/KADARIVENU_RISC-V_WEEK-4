Nice — below I’ll give you ready-to-run SPICE/Ngspice examples to demonstrate (A) velocity-saturation effects (by comparing long- vs short-channel MOSFETs with an ID vs VDS sweep) and (B) how velocity saturation and low supply voltage affect a CMOS inverter VTC / switching (useful to see behavior at “lower nodes” / low VDD). I’ll also explain what to look for in the plots and what each simulation demonstrates.

You can copy these into files and run them in ngspice (or most SPICE-compatible tools). I keep the models simple and portable (level=3) — if you have a foundry BSIM3/BSIM4 model, substitute that .model block to get more realistic velocity-saturation behavior.

A — ID vs VDS: show velocity saturation (long vs short channel)

Save as vsat_id_vds.cir and run ngspice vsat_id_vds.cir.

* ID vs VDS: compare long vs short channel NMOS (level=3)
.options numdgt=6
.include standard.inc

***** Power / bias sources *****
VGS gate 0 DC 1.2      ; gate bias
VDS  drain 0 DC 0      ; swept source for drain

***** Devices: two transistors tied to different drains (separate circuits)
* Short-channel device (expected to show early velocity saturation)
Mshort drain_s gate 0 0 n_short W=10u L=0.05u
.model n_short nmos level=3 vt0=0.4 kp=100u

* Long-channel device (behaves more like quadratic / later saturation)
Mlong drain_l gate 0 0 n_long W=10u L=1.0u
.model n_long nmos level=3 vt0=0.4 kp=100u

***** Wiring: use independent sweep of VDS by connecting VDS to both drains in separate runs *****
* We will perform two .dc sweeps: one for each drain node
* Sweep short device
.dc VDS 0 1.5 0.01
* Map VDS source to drain_s for the short device using a behavioral source
BDS_s drain_s 0 value={V(VDS)}  ; tie drain_s net value to swept VDS
* For long device also map drain_l
BDS_l drain_l 0 value={V(VDS)}  ; tie drain_l net value to swept VDS

* Measurement: print drain currents (I(Mxxx) sign is negative for current into device)
.print dc I(Mshort) I(Mlong) V(drain_s) V(drain_l)

.end


How to run:

ngspice vsat_id_vds.cir

Use plot I(Mshort) and plot I(Mlong) or set traces in the ngspice GUI.

What to look for

Both curves increase with VDS in the linear region.

The short-channel device (L = 0.05 µm) will saturate earlier and show a flatter curve (due to velocity saturation limiting carrier velocity).

The long-channel device (L = 1 µm) will show a more pronounced quadratic rise then classical saturation at VDS ≈ VGS − Vth.

B — CMOS inverter VTC & effect of velocity saturation at low VDD

Save as inverter_vtc_vsat.cir and run ngspice inverter_vtc_vsat.cir.

* CMOS Inverter VTC: show effect at two VDD values (1.2V and 0.5V)
.options numdgt=6

***** Supply and sweep source *****
VDD vdd 0 1.2
Vin in 0 DC 0

***** PMOS & NMOS models (level=3 approximate)
.model nmos_l nmos level=3 vt0=0.4 kp=100u
.model pmos_l pmos level=3 vt0=-0.4 kp=40u

***** Inverter sized for roughly symmetric switch at 1.2V *****
M1 out in vdd vdd pmos_l W=20u L=0.06u
M2 out in 0 0 nmos_l   W=10u L=0.06u

***** DC sweep for VTC at VDD=1.2V *****
.dc Vin 0 1.2 0.01
.print dc V(out) V(in)

***** Now run at low VDD = 0.5V to see low-node behavior (you can re-run after changing VDD)
* To run low-VDD, either edit VDD line to 0.5 and re-run, or use .step param
.step param VDDval list 1.2 0.5
* Replace VDD with parameterized source:
VDD vdd 0 param=VDDval

* Tie device substrate nodes correctly (already used)
* Re-do DC sweep param-driven:
.dc Vin 0 {VDDval} 0.01
.print dc V(out) V(in) V(VDDval)

.end


How to run:

ngspice inverter_vtc_vsat.cir

Use .step or edit VDD then rerun.

Plot V(out) vs Vin for both VDD values.

What to look for

At VDD = 1.2 V, the VTC is sharp; switching threshold (VM) is near mid-rail if sizing is symmetric.

At VDD = 0.5 V (lower node / low-voltage operation), two things happen:

The overdrive (VGS − Vth) is small; both devices operate closer to threshold and velocity-saturation-limited region — gain is reduced.

VM shifts, and the transition becomes less steep → worse noise margins, slower switching.

If you substitute a short-channel model (smaller L) for the inverter transistors, you’ll see reduced gain at higher currents because velocity saturation limits ID even if VGS−Vth is larger.

Notes on modelling velocity saturation more accurately

Level=3 (or Level=2/3) includes simple high-field effects, but BSIM3/BSIM4 models (foundry) are the realistic way to simulate velocity saturation and other short-channel physics. If you have a foundry .model file, include it:

.include my_bsim4_model.lib
M1 ... modelname key=...


In BSIM-style models, parameters like vsat / vth0 / mobility degradation / u0 / a0 etc control velocity-saturation behavior. Use the foundry defaults or tune them to match measured curves.

For teaching/demonstration, the above level=3 examples are good to show qualitative behavior (earlier saturation and flattening for short L).

Suggested plotting commands (ngspice interactive)

After launching ngspice:

For the ID–VDS run:

plot -r I(Mshort) vs V(drain_s)

plot -r I(Mlong) vs V(drain_l)

For inverter VTC:

plot V(out) vs V(in)

Run twice for the two VDD values and overlay.

If using the ASCII output, you can also probe or write data to files for plotting in Python/Matlab.

Quick interpretation checklist (what confirms velocity saturation)

For the short-channel MOSFET: drain current vs VDS flattens earlier and the saturated ID scales approximately linearly with (VGS − Vth) rather than quadratically.

In the inverter at low VDD: transition slope (gain) is reduced and VM moves; with short channel devices the output high/low drive is limited by velocity saturation (lower drive current even at larger VGS).

If you swap in a BSIM4 model and sweep channel length (L), you’ll see the effect intensify as L shrinks.

If you want, I can:

(A) Produce a ready-to-run ngspice script that automatically generates PNG plots and CSV output, or

(B) Prepare a short PowerPoint slide with those example netlists, expected plots, and interpretation, or

(C) Convert these simulations into a small Matlab/Python script to plot results after running ngspice.
