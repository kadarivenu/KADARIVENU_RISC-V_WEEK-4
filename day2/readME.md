1. Velocity Saturation in MOSFETs (like NMOS)
🔹 Basic Idea

In a MOSFET, the drain current (ID) increases with drain-to-source voltage (VDS) until it saturates (normal saturation region).
However, when electric field in the channel becomes very high (especially in modern short-channel devices), carrier velocity stops increasing linearly — this is called velocity saturation.

📘 Explanation

At low electric fields:

𝑣
=
𝜇
𝐸
v=μE

where

𝑣
v = carrier velocity,

𝜇
μ = mobility,

𝐸
E = electric field.

At high electric fields, velocity reaches a maximum limit:

𝑣
𝑠
𝑎
𝑡
≈
10
7
 cm/s for electrons in Si
v
sat
	​

≈10
7
 cm/s for electrons in Si

and no longer increases with field.

Therefore, even if VDS increases, ID no longer rises significantly — the current saturates earlier.

📉 Effect on Drain Current

In velocity saturation region:

𝐼
𝐷
=
𝑊
𝐶
𝑜
𝑥
𝑣
𝑠
𝑎
𝑡
(
𝑉
𝐺
𝑆
−
𝑉
𝑡
ℎ
)
I
D
	​

=WC
ox
	​

v
sat
	​

(V
GS
	​

−V
th
	​

)

➡ Current becomes linearly dependent on (VGS − Vth) instead of quadratic.
This happens in short-channel MOSFETs (used in modern CMOS).

⚙️ Key Point

Long-channel device → quadratic ID–VGS relation.

Short-channel device (with velocity saturation) → linear ID–VGS relation.

This limits current drive and switching speed.

🧩 2. CMOS Inverter & Voltage Transfer Characteristic (VTC)
🔹 Structure

A CMOS inverter combines:

PMOS (connected to VDD)

NMOS (connected to GND)

Both gates are tied together to form the input, and drains are connected together to form the output.

 VDD
  │
 PMOS
  │
  ├── Output (Vout)
  │
 NMOS
  │
 GND

🔹 Operation Regions
Input Voltage (Vin)	PMOS	NMOS	Output (Vout)	State
0 (low)	ON	OFF	≈ VDD	Logic 1
High (VDD)	OFF	ON	≈ 0	Logic 0
Mid (≈ Vth)	Partially ON	Partially ON	Transition	Switching
📈 Voltage Transfer Characteristic (VTC)

Plots Vout vs. Vin

Has an S-shaped curve showing sharp transition between logic levels.

Vout |\
     | \
     |  \
     |   \
     |    \
     |_____\_______ Vin

⚙️ Important Points on VTC

Noise Margins: Flat regions indicate high noise immunity.

Switching Threshold (VM): Input voltage where Vin = Vout.

Inverter Gain (−dVout/dVin): Very high near VM → fast switching.

Symmetrical VTC: Achieved when PMOS and NMOS are sized properly.

🔍 Summary
Parameter	Meaning
VM	Switching threshold
NMH / NML	Noise margins for logic high & low
Gain	Sharpness of transition
Power	Very low (no static current in logic states)
