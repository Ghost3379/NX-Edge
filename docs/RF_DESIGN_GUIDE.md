# NX-Edge RF Front-End Design & Impedance Analysis

Dense technical reference for the 2.4 GHz RF transmission lines, stackup parameters, impedance calculations, and matching networks on the NX-Edge module.

---

## 1. Stackup & Transmission Line Geometry

The NX-Edge utilizes a standard **6-layer stackup** manufactured to an overall thickness of **$0.80\,\text{mm} \pm 10\%$** for M.2 Key B compliance.

All RF transmission lines are routed on **Layer 1 (`F.Cu`)** directly referencing the unbroken **Layer 2 (`In1.Cu`) Ground Plane**.

```
Layer 1: F.Cu (Top Signal / RF)  ------ [ W = 0.150 mm, T = 0.035 mm (1 oz Cu) ]
Dielectric 1: Prepreg (FR4)      ====== [ H = 0.100 mm (100 µm), εr = 4.50 ]
Layer 2: In1.Cu (Continuous GND) ====== [ Reference Plane ]
```

### Physical Parameters
| Parameter | Symbol | Design Value | Notes |
| :--- | :---: | :---: | :--- |
| **Trace Width** | $W$ | $0.150\,\text{mm}$ ($150\,\mu\text{m}$) | Consistent across all RF segments |
| **Dielectric Height** | $H$ | $0.100\,\text{mm}$ ($100\,\mu\text{m}$) | Prepreg thickness (L1 to L2) |
| **Copper Thickness** | $T$ | $0.035\,\text{mm}$ ($35\,\mu\text{m}$) | $1.0\,\text{oz}$ base copper foil |
| **Relative Permittivity** | $\epsilon_r$ | $4.50$ | Standard FR4 @ 2.4 GHz |
| **Loss Tangent** | $\tan\delta$ | $0.020$ | Standard FR4 dissipation factor |

---

## 2. Characteristic Impedance Calculations

Calculated using the IPC-2141 / Hammerstad-Jensen microstrip formulation:

### 2.1 Effective Width & Permittivity
$$W_{eff} = W + \frac{T}{\pi} \left[ 1 + \ln\left( \frac{2H}{T} \right) \right] = 0.150 + \frac{0.035}{\pi} \left[ 1 + \ln\left( \frac{0.200}{0.035} \right) \right] = 0.1805\,\text{mm}$$

$$u = \frac{W_{eff}}{H} = \frac{0.1805}{0.100} = 1.805$$

$$\epsilon_{eff} = \frac{\epsilon_r + 1}{2} + \frac{\epsilon_r - 1}{2} \left( 1 + \frac{10}{u} \right)^{-a \cdot b} \approx 3.38$$

### 2.2 Calculated Characteristic Impedance ($Z_0$)
$$Z_0 = \frac{120\pi}{\sqrt{\epsilon_{eff}} \left[ u + 1.393 + 0.667 \ln(u + 1.444) \right]} = \mathbf{51.44\,\Omega}$$

* **Target System Impedance:** $Z_L = 50.00\,\Omega$
* **Impedance Error:** $+1.44\,\Omega$ ($+2.88\%$)
* *(Note: Exact $50.00\,\Omega$ theoretical width is $0.160\,\text{mm}$, giving $49.82\,\Omega$.)*

### 2.3 Transmission Line Metrics @ 2.44 GHz
* **Reflection Coefficient ($\Gamma$):**
  $$\Gamma = \frac{Z_0 - Z_L}{Z_0 + Z_L} = \frac{51.44 - 50.00}{51.44 + 50.00} = \mathbf{0.0142}$$
* **Voltage Standing Wave Ratio (VSWR):**
  $$\text{VSWR} = \frac{1 + |\Gamma|}{1 - |\Gamma|} = \frac{1 + 0.0142}{1 - 0.0142} = \mathbf{1.028 : 1}$$
* **Return Loss ($RL$):**
  $$RL = -20 \log_{10}|\Gamma| = \mathbf{36.95\,\text{dB}}$$
* **Phase Velocity ($v_p$):**
  $$v_p = \frac{c}{\sqrt{\epsilon_{eff}}} = \frac{3.00 \times 10^8\,\text{m/s}}{\sqrt{3.38}} \approx 1.63 \times 10^8\,\text{m/s} = 0.163\,\text{mm/ps}$$
* **Guided Wavelength ($\lambda_g$):**
  $$\lambda_g = \frac{v_p}{f} = \frac{1.63 \times 10^8\,\text{m/s}}{2.44 \times 10^9\,\text{Hz}} = \mathbf{66.8\,\text{mm}}$$

---

## 3. Physical Trace Net Breakdown

Extracted from `NX-Edge.kicad_pcb`:

```
               [ L4 ] (Series)
[ ESP32-S3 ] -----------------+---------------- [ U3: BGS12WN6 ]
(LNA_IN)                      |                 (RF Switch)
                            [ C14 ]                 |
                            (Shunt)                 |
                              |                    / \
                             GND                  /   \
                                                 /     \ (Pin 1: RF2)
                                                /       \
                                  (Pin 3: RF1) /         +----> [ AE2: Hirose U.FL ]
                                              /
                                            [ C13 ] (Series)
                                              |
                                              +---------+
                                              |         |
                                            [ L3 ]    [ L2 ] (Series)
                                           (Shunt)      |
                                              |       [ AE1: Ignion NN02-201 ]
                                             GND
```

### Segment Dimensions & Electrical Lengths
| Net Name | Path | Width ($W$) | Length ($L$) | $L / \lambda_g$ Ratio | Phase Delay ($\tau$) |
| :--- | :--- | :---: | :---: | :---: | :---: |
| **`Net-(U4-LNA_IN)`** | ESP32-S3 `LNA_IN` $\rightarrow$ `L4` | $0.15\,\text{mm}$ | $0.93\,\text{mm}$ | $1.39\%$ | $5.7\,\text{ps}$ |
| **`Net-(U3-RFIN)`** | `L4`/`C14` $\rightarrow$ `U3` Pin 5 (`RFIN`) | $0.15\,\text{mm}$ | $2.27\,\text{mm}$ | $3.40\%$ | $13.9\,\text{ps}$ |
| **`Net-(U3-RF1)`** | `U3` Pin 3 (`RF1`) $\rightarrow$ `C13` | $0.15\,\text{mm}$ | $0.92\,\text{mm}$ | $1.38\%$ | $5.6\,\text{ps}$ |
| **`Net-(C13-Pad1)`** | `C13` $\rightarrow$ `L2` / `L3` node | $0.15\,\text{mm}$ | $1.64\,\text{mm}$ | $2.46\%$ | $10.1\,\text{ps}$ |
| **`Net-(AE1-Pad1)`** | `L2` $\rightarrow$ `AE1` feed pad | $0.15\,\text{mm}$ | $2.10\,\text{mm}$ | $3.14\%$ | $12.9\,\text{ps}$ |
| **`Net-(AE2-In)`** | `U3` Pin 1 (`RF2`) $\rightarrow$ `AE2` U.FL pad | $0.15\,\text{mm}$ | $2.36\,\text{mm}$ | $3.53\%$ | $14.5\,\text{ps}$ |

> [!NOTE]
> Every individual trace segment is shorter than $\lambda_g / 28$ ($< 2.4\,\text{mm}$). Distributed transmission line reflections are negligible at these lengths; trace segments behave as lumped inductive parasitics of approximately $0.4\text{–}0.9\,\text{nH}$.

---

## 4. Matching Network Component Values (BOM)

### 4.1 Transceiver Match: ESP32-S3 to Switch (`L4`, `C14`)
The ESP32-S3 `LNA_IN` port is matched internally by Espressif to $(35 + j0)\,\Omega \approx 50\,\Omega$ at 2.4 GHz.

| RefDes | Package | Function | Recommended Value | Alternative / Tuning | Murata Part Number |
| :---: | :---: | :--- | :---: | :---: | :--- |
| **`L4`** | `0402` | Series Matching / Harmonic Filter | **$0\,\Omega$ Jumper** | $1.2\,\text{nH} \pm 0.1\,\text{nH}$ | `LQW15AN1N2C00D` |
| **`C14`** | `0402` | Shunt Capacitance to GND | **DNP** *(Do Not Populate)* | $0.5\,\text{pF} \pm 0.05\,\text{pF}$ | `GJM1555C1HR50WB01D` |

### 4.2 Antenna Booster Match: Switch to Ignion NN02-201 (`C13`, `L3`, `L2`)
The Ignion `NN02-201` is a non-resonant virtual antenna booster requiring lumped reactive elements to tune resonance into the $2.400\text{–}2.500\,\text{GHz}$ band for the $22\times 60\,\text{mm}$ ground plane.

| RefDes | Package | Function | Recommended Value | Tuning Range | Recommended Manufacturer Part |
| :---: | :---: | :--- | :---: | :---: | :--- |
| **`C13`** | `0402` | Series DC Block / Low-band Tuning | **$1.5\,\text{pF}$** ($\pm 0.05\,\text{pF}$) | $1.0\,\text{pF} \text{ -- } 2.4\,\text{pF}$ | Murata `GJM1555C1H1R5WB01D` (C0G/NP0, High-Q) |
| **`L3`** | `0402` | Shunt Matching to GND | **$2.7\,\text{nH}$** ($\pm 0.1\,\text{nH}$) | $2.2\,\text{nH} \text{ -- } 3.3\,\text{nH}$ | Murata `LQW15AN2N7B00D` (Wirewound, High-Q, SRF > 6 GHz) |
| **`L2`** | `0402` | Series Antenna Feed Inductor | **$1.8\,\text{nH}$** ($\pm 0.1\,\text{nH}$) | $0\,\Omega \text{ -- } 2.2\,\text{nH}$ | Murata `LQW15AN1N8B00D` (Wirewound, High-Q) |

> [!IMPORTANT]
> **Component Quality Requirement:** At 2.4 GHz, inductors MUST be high-Q wirewound types (e.g. Murata LQW15 or Johanson L-07W) and capacitors MUST be RF-grade NP0/C0G (Murata GJM or Johanson S-series). Standard multilayer general-purpose 0402 passives have low Q and will introduce $1.0\text{–}2.5\,\text{dB}$ of insertion loss.

### 4.3 External U.FL Receptacle Path (`AE2`)
* Direct $50\,\Omega$ coplanar line from `U3` Pin 1 (`RF2`) to `AE2` Hirose U.FL pad ($L = 2.36\,\text{mm}$).
* **Required passives:** None ($0$ components). Standard U.FL antennas and 50Ω RF coax cables connect directly.

---

## 5. Verification & Tuning Procedure

1. **U.FL Path Baseline:**
   * Set `GPIO10` (`CTRL_RF`) = `HIGH` (selects U.FL path).
   * Measure return loss ($S_{11}$) at the U.FL connector using a calibrated Vector Network Analyzer (VNA) with a 50Ω load. Target: $S_{11} < -20\,\text{dB}$ across 2400–2484 MHz.
2. **Onboard Antenna Tuning:**
   * Set `GPIO10` (`CTRL_RF`) = `LOW` (selects `AE1` ceramic booster).
   * Populate baseline values: `C13 = 1.5pF`, `L3 = 2.7nH`, `L2 = 1.8nH`.
   * Measure $S_{11}$ in final operating enclosure (or free space) with battery connected.
   * If resonance center frequency is low: Reduce `C13` or replace `L2` with a $0\,\Omega$ jumper.
   * If resonance center frequency is high: Increase `L3` to $3.0\text{–}3.3\,\text{nH}$.
