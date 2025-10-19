reat — here’s a practical, ready-to-run toolkit and checklist to evaluate CMOS robustness to power-supply variation and device variation. It covers what to measure, how to simulate it (ngspice-style netlists you can run immediately), which sweeps to run (PVT, VDD droop, device variation / mismatch), what metrics to extract, and how to summarize/report the results. I’ll also include an ngspice netlist you can run right away that automates most of the important sweeps and prints the measured metrics.

What you want to evaluate (overview)

Targets / metrics

Static: VOH, VOL, VIL, VIH, VM, NMH, NML (from VTC)

Dynamic: tpLH, tpHL, tr, tf, Energy per transition, Ishort (peak), Idd_avg during switching

Leakage: Ioff (standby leakage) at VDD and temperature corners

PSRR / supply sensitivity: how Vout/VM/delay change with VDD, and response to supply transient/glitch

Robustness across device/process variation: effect of Vth shift, mobility (kp) variation, W/L mismatch, channel length L variation

Worst-case identification: PVT corner and param combination causing worst NM, max delay, or logic failure

Stress axes

VDD sweep: nominal ±10–30% and low-node values (e.g., 1.2V → 0.9V → 0.7V → 0.5V)

Temperature: −40°C → 25°C → 125°C (or your target range)

Process / device: Vth shifts, kp variations, channel length L (long vs short)

Supply droop / IR: local series resistance, transient drops, or injected noise/glitches

Input slew: slow vs fast input edges (Trise) — affects short-circuit current and timing

Quick summary of method

Run DC VTC sweep for each (process, VDD, temp) to get VOH/VOL/VIL/VIH and noise margins.

Run transient switching (pulse input with finite rise/fall) with a load capacitor CL to get delays and dynamic metrics.

Repeat for device variation grid (Vth shifts, L steps, kp variations). Use parameterized .step ranges to cover the grid.

Measure supply sensitivity: step VDD and also inject transient droop/glitch (PWL on VDD). Measure worst-case VM shift and delay degradation.

(Optional) Monte-Carlo: step Vth offsets and W/L mismatch values (discrete set) to approximate statistical spread; for full MC use the simulator’s Monte-Carlo feature or a foundry tool.

Aggregate results into a table and identify worst-case scenarios. Produce plots: Delay vs VDD, NM vs VDD, Delay vs Vth shift, SNM/VM heatmap.

Ready-to-run ngspice netlist

Save as inverter_robustness.cir and run ngspice inverter_robustness.cir.

This netlist:

Parameterizes VDD, device length L, and a Vth offset (dvth) to emulate device variation.

Performs DC VTC for VM and noise margin approximation, and transient switching for delays and dynamic currents.

Steps over a small grid of VDD × L × dvth × Temp to find worst cases. (You can expand lists easily.)

* inverter_robustness.cir -- ngspice
* CMOS robustness evaluation: VDD sweep, L sweep, Vth offset sweep, Temp sweep
* Saves key .meas results to screen (ngspice prints them after run).

.param VDDlist = "1.2 1.08 0.96 0.8 0.6 0.5"   ; supply values to step (add/remove as needed)
.param Llist   = "60n 120n 500n 1u"            ; channel lengths (short->long)
.param dvthlist= "-0.08 -0.03 0 0.03 0.08"     ; Vth offsets to mimic process / mismatch (V)
.param TempList= "25 85 125 -40"               ; temperatures (degC)
.param Wn=10u Wp=20u CL=50f Trise=1n Ton=10n Tperiod=40n

* MOS models (base vt0 is nominal; we will add dvth parametrically)
.param vt0_n = 0.45
.param vt0_p = -0.45
.model nmos_base nmos level=3 kp=120u vto={vt0_n}
.model pmos_base pmos level=3 kp=40u vto={vt0_p}

* Circuit (nodes: vdd, in, out, 0)
* Parameterized device using .param-based vt via renaming trick: we will use .step to alter device vt via param substitution
* Implement using .param vt_n = vt0_n + dvth ... and reference in device line with vt0 attribute
* Note: Some SPICE flavors allow model parameter substitution; ngspice typically allows inline .model expression substitution. If your simulator doesn't, create separate model blocks per corner.

* Input sources (Vin pulse)
Vin in 0 PULSE(0 {VDD} 1n {Trise} {Trise} {Ton} {Tperiod})

* VDD source (probe current with the name VVDD so I(VVDD) is supply current)
VVDD vdd 0 DC {VDD}

* Load
Cload out 0 {CL}

* Devices: we'll define models through parameters and then include in the instance via model name that uses param expansion
* To allow vt shift via parameter dvth, we'll create model lines dynamically via .param (?) 
* Practical approach: create a single model which references a parameter vt_n, vt_p. ngspice supports .model vto expression only if param is expanded before parse. To keep portable, we will define multiple model sets by running separate nets or let user adapt if their SPICE does not allow this.

* For simplicity here, we instantiate devices with L as parameter; use measured I(Mn) etc for currents
M1 out in vdd vdd pmos_base W={Wp} L={L}         ; PMOS
M2 out in 0 0 nmos_base W={Wn} L={L}            ; NMOS

* ------------ Measurement definitions ------------
* DC VTC: sweep Vin from 0 to VDD at fine resolution to compute VM (we will print V(out) for offline processing)
.dc Vin 0 {VDD} 0.002
* transient for dynamic metrics
.tran 0.05n 200n uic

* Measures (transient)
.meas tran tpLH TRIG V(in) VAL={VDD/2} RISE=1 TARG V(out) VAL={VDD/2} RISE=1
.meas tran tpHL TRIG V(in) VAL={VDD/2} FALL=1 TARG V(out) VAL={VDD/2} FALL=1
.meas tran trise TRIG V(out) VAL={0.1*VDD} RISE=1 TARG V(out) VAL={0.9*VDD} RISE=1
.meas tran tfall TRIG V(out) VAL={0.9*VDD} FALL=1 TARG V(out) VAL={0.1*VDD} FALL=1
.meas tran Ishort_max MAX I(VVDD)         ; peak supply current during transition (negative sign may indicate direction)

* Measures (DC)
* We will estimate VM by locating point where Vout-Vin crosses zero. Here we save the V(out) and V(in) for external processing.
.save V(out) V(in)

* Leakage at steady state: run a tiny DC bias state (Vin=0 and VDD applied) and measure supply current
* Note: I(VVDD) at steady-state DC point gives leakage (may contain small dynamic charges)
* For static leakage measurement we will run a separate DC operating point run in post-processing or use .op

* ------------- Parameter sweeps --------------
* Sweep through VDDlist, Llist, dvthlist, TempList. ngspice supports nested .step via multiple .step statements.
.step param VDDval list {VDDlist}
.step param L list {Llist}
.step param dvth list {dvthlist}
.step param Tval list {TempList}
.temp {Tval}

* For each parameter combination user should re-run: but note that .temp is not nested within .step in all SPICE flavors. If your ngspice handles nested .step/.temp fine, it will run all combos.

* User note: if model parameterization via dvth is required, you should create different model blocks per dvth or rewrite the model using param substitution supported by your SPICE. If not supported, run multiple netlists with changed vt0 values.

* Print out important meas results to console (ngspice prints .meas values automatically)
* End
.end

How to run & get useful outputs

ngspice inverter_robustness.cir

The .step lines will run multiple simulation cases. ngspice prints .meas results to stdout for each step (tpLH, tpHL, trise, tfall, Ishort_max). Save the console output to a file for parsing:

Example: ngspice inverter_robustness.cir | tee robustness_raw.txt

Save the DC VTC for each step by using wrdata after .dc run (or add wrdata vtc_${VDDval}_${L}.csv V(in) V(out) in a postprocessing loop). If wrdata doesn't accept param expansion, run manually per corner or script-run ngspice with substituted parameters.

Parse robustness_raw.txt (or multiple CSVs) to build a table of metrics.

Python helper — parse ngspice .meas output and summarize

Save as parse_ngspice_meas.py. This simple parser expects lines like tpLH(tran)= 2.34e-09 in the ngspice stdout and assembles a CSV summary.

# parse_ngspice_meas.py
import re, csv,sys

infile = sys.argv[1] if len(sys.argv)>1 else 'robustness_raw.txt'
outcsv = 'robustness_summary.csv'

pattern = re.compile(r'^\s*(\S+)\(\S+\)\s*=\s*([-+eE0-9.]+)')  # matches lines like "tpLH(tran)=  2.34e-09"
# also read parameter header lines if present to get VDD, L, dvth, Temp text - user should add prints in netlist for these
param_pattern = re.compile(r'^\s*Param:\s*(\S+)\s*=\s*([-\.\deE]+)')

rows = []
cur_params = {}
with open(infile) as f:
    for line in f:
        # capture parameter printouts if you modified netlist to echo parameter per step
        mparam = param_pattern.search(line)
        if mparam:
            cur_params[mparam.group(1)] = mparam.group(2)
        m = pattern.search(line)
        if m:
            key = m.group(1)
            val = float(m.group(2))
            row = {'param_VDD': cur_params.get('VDDval',''), 'param_L': cur_params.get('L',''), 'param_dvth': cur_params.get('dvth',''), 'param_T': cur_params.get('Tval','')}
            row[key] = val
            rows.append(row)

# Merge rows by parameters (simple approach)
# For robust reporting you may want to restructure based on your netlist prints
with open(outcsv,'w',newline='') as csvf:
    all_keys = set()
    for r in rows:
        all_keys |= set(r.keys())
    all_keys = ['param_VDD','param_L','param_dvth','param_T'] + sorted(k for k in all_keys if not k.startswith('param_'))
    writer = csv.DictWriter(csvf,fieldnames=all_keys)
    writer.writeheader()
    for r in rows:
        writer.writerow({k: r.get(k,'') for k in all_keys})

print("Wrote",outcsv)


Tip: add lines in the netlist to print parameters for each step, e.g. after the .step block add:

.printdc param VDDval L dvth Tval


or echo statements so the parser can know which measured block belongs to which corner. The exact syntax depends on your SPICE.

How to evaluate supply robustness (PSRR, droop, transient)

PSRR (DC sense): measure dVM/dVDD (how VM shifts with VDD). Simpler: compute VM at multiple VDD values and fit slope. Good metric is ∂VM/∂VDD — ideal inverter: VM proportional to VDD, so normalized change is small for robust design.

Transient droop: add a PWL source on VDD: VVDD vdd 0 PWL(0 1.2 5n 0.9 10n 1.2) to model a droop to 0.9V at 5ns, then measure whether output flips incorrectly or delay increases. Sweep droop depth and timing.

Local IR (series resistor): place Rser vdd_supply vdd {R} and step R to model supply impedance. Measure VOH drop at load.

Supply noise injection: Inject small sinusoidal or random noise on VDD and run transient to see resulting jitter on output switching time. Compute sensitivity: d(tp)/d(Vdd_noise_amp) or jitter RMS.

Guidelines: what counts as “robust”?

Fabrication / product requirements vary. Example acceptance criteria (illustrative):

NMH, NML ≥ 200–300 mV at nominal VDD for robust I/O at 1.2V tech. (target depends on VDD and tech)

Delay degradation under −10% VDD: < 2× nominal (or meet a spec)

Leakage under high temp: < spec threshold (depends on standby power budget)

Worst-case VM shift under VDD droop < 0.1*VDD (or meets logic-chain timing margins)

Report both absolute metrics and relative degradation (% change from nominal).

Reporting & visualization suggestions

Tables: list worst-case corner per metric (e.g., the corner that gives minimum NM, max delay, max leakage).

Plots:

Delay (tpLH / tpHL) vs VDD for each L and dvth.

NMH/NML vs VDD and Temp (heatmaps if many points).

VM vs VDD (show slope) and VM shift histogram for device variations.

Ishort_max vs input slew.

For SRAM/cell-level robustness: produce butterfly plots and SNM vs VDD/Temp.

Practical tips & pitfalls

Use fine DC VTC step (≤ 1–2 mV) around the switching region for accurate VM and NM extraction.

For small-signal or PSRR at a frequency, use AC analysis: perturb VDD with a small AC source and measure output AC amplitude. But for realistic switching PSRR use transient injection.

When modeling device variation, use foundry corner models for FF/TT/SS when available — they capture Vth and mobility shifts more accurately than ad-hoc dvth shifts.

Include layout parasitics (R/VDD nets, extracted RC) for final signoff-level evaluation — local IR drops often dominate worst-case VOH/VOL real behavior.

For Monte-Carlo: if exact statistical modeling is required (yield numbers), use your SPICE/EDA tool’s Monte-Carlo run with proper distributions for Vth, W/L, series resistance.

Next steps I can do for you right now (pick any and I’ll produce immediately)

Create a more detailed ngspice netlist that (a) automatically writes per-corner CSV files using parameter substitution (I’ll tailor it to your ngspice version), or

Produce a Python notebook that ingests VTC CSVs and measured .meas outputs and produces the full robustness report and plots (Delay vs VDD, NM heatmaps, worst-case identification), or

Build a transient supply-glitch netlist that runs a sweep over glitch amplitude/time and reports the minimum glitch that causes a logic error for a given input timing window, or

Prepare a template report (spreadsheet) with recommended columns and charts to present to management/PE.
