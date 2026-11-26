# MaRTy+ / MaRTy3D+ CR1000X (Real-Time Heat-Strain Monitoring)

**Mobile microclimate sensing coupled with the Stolwijk-2024 thermoregulation model, running on board a Campbell Scientific CR1000X datalogger.**

This repository hosts the CRBasic datalogger program for MaRTy+, the mobile biometeorological platform introduced in Abdullah et al. (2026). The program measures six-directional shortwave and longwave radiation, air temperature, relative humidity, and wind, computes mean radiant temperature (MRT) on the logger, and feeds these measurements straight into the Stolwijk-2024 thermoregulation model. The result is a real-time estimate of core temperature, mean skin temperature, skin wettedness, sweat rate, and skin and respiratory heat losses for a specified exposure profile.

If you use this code, we kindly ask that you cite the study:

> Abdullah, M., Joshi, A., Viswanathan, S. H., Vanos, J., Middel, A., Rotella, J., & Rykaczewski, K. (2026). Linking Urban Microclimate Sensing and Thermoregulation Modeling for Real-Time Heat-Strain Assessment. *Nexus*, 100184. https://doi.org/10.1016/j.ynexs.2026.100184

## What does this repository contain?

- `MaRTyPlus_Stolwijk2024_CR1000X.CR1X`: the complete CR1000X program for MaRTy+ (sensor measurement, MRT calculation, embedded Stolwijk-2024 model, and data table).

This is the **MaRTy+** configuration, which uses a 2D sonic anemometer (Gill WindSonic) and mean wind speed. **MaRTy3D+** adds a CSAT3B 3D sonic anemometer so that turbulence intensity ($`TI`$) and integral length scale ($`L_T`$) can enter turbulence-corrected heat transfer coefficients; that version is not included here.

## What is MaRTy+ / MaRTy3D+?

MaRTy+ builds on the original MaRTy cart (Middel & Krayenhoff, 2019), named for its focus on MRT, and adds an embedded thermoregulation model so the cart reports physiological heat strain as well as the environment. The "+" denotes the integrated thermoregulation modeling; "3D" denotes three-axis airflow sensing.

| Platform | Sensor package | Capability |
|---|---|---|
| MaRTy | EE181 T/RH, three Hukseflux NR01 net radiometers, Gill 2D WindSonic, CR3000 | Environment only ($`T_{air}`$, RH, 2D wind, MRT) |
| **MaRTy+** (this code) | EE181 T/RH, three Apogee SN-500 net radiometers, Gill 2D WindSonic, CR1000X | Environment + real-time Stolwijk-2024 heat strain |
| MaRTy3D+ | EE181 T/RH, three Apogee SN-500 net radiometers, CSAT3B 3D sonic anemometer, CR1000X | Adds $`TI`$ and $`L_T`$ for turbulence-aware heat exchange |

## How is MRT calculated?

The three SN-500 radiometers (up/down, left/right, front/back) share one SDI-12 bus on port C3 (addresses `U`, `L`, `B`), which sets the sampling interval to 20 s. MRT is computed with the Integrated Radiation Measurement approach:

```math
MRT = \sqrt[4]{\frac{\sum_{i=1}^{6} W_i \left( a_k K_i + a_l L_i \right)}{a_l \, \sigma}} - 273.15
```

where $`K_i`$ and $`L_i`$ are the shortwave and longwave flux densities (W m<sup>−2</sup>) from direction $`i`$, $`a_k = 0.70`$ and $`a_l = 0.97`$ are the shortwave and longwave absorption coefficients, $`W_i`$ is 0.22 for the four lateral directions and 0.06 for up and down, and $`\sigma = 5.67 \times 10^{-8}`$ W m<sup>−2</sup> K<sup>−4</sup> is the Stefan–Boltzmann constant. MRT is in °C.

## What does the thermoregulation model do?

Stolwijk-2024 (Joshi et al., 2025) is a revised version of the classical 25-node Stolwijk model. The body is split into six segments (head, trunk, arms, hands, legs, feet), each with core, muscle, fat, and skin layers, connected to a central blood pool. Control signals drive vasodilation, vasoconstriction, sweating, and shivering, and clothing enters through intrinsic thermal ($`R_{c,cl}`$) and evaporative ($`R_{e,cl}`$) resistances.

On the logger, each 20 s scan reads the sensors and then advances the model in twenty 1 s steps, so the model state carries forward continuously between scans.

## What does the program output?

`Table1` (written every 20 s to the CR1000X and to `CRD:OtherData` on the memory card) records:

- **Physiology:** core temperature, mean skin temperature, mean skin wettedness, cardiac output
- **Heat exchange:** skin latent and sensible heat loss, respiratory heat loss
- **Environment:** air temperature, RH, MRT, wind speed, and all shortwave/longwave components, net radiation, and albedo from the three radiometers
- **GPS:** position, speed, course, altitude, fix quality, and satellite count (GPS16X-HVS on ComC1; the logger clock syncs to GPS time)

Core temperature can be read against the heat-strain classes used in the paper:

| $`T_{core}`$ (°C) | Heat strain level |
|---|---|
| < 37.5 | No risk |
| 37.5 – 38.5 | Mild strain |
| 38.5 – 39.5 | Moderate strain |
| 39.5 – 40.5 | Severe strain |
| > 40.5 | Critical risk |

## How do I set up a subject or scenario?

The person being modeled is defined in the program under `Define Subject & Clothing Parameters`:

- `met`: metabolic rate in METs (default 1.3)
- `Rc_cl()` and `Re_cl()`: clothing thermal and evaporative resistance for each segment (defaults 0.0775 m<sup>2</sup> K W<sup>−1</sup> and 12.3 m<sup>2</sup> Pa W<sup>−1</sup>)
- `posture`: posture used for the low-wind (< 0.2 m/s) heat transfer coefficients

Anthropometry and tissue properties (`SA`, `C`, `Q_bm`, `BFB`, `TC`) and set points (`T_set`) are for the Stolwijk-2024 reference person. Outputs are therefore **profile-specific, not person-specific**.

## What should I keep in mind?

- Clothing parameters are static: the pumping effect and wet-conduction changes in insulation are neglected, which suits stationary or slow-moving subjects but not high-activity use.
- The radiometers are read one after another on a shared SDI-12 channel, which can add short-lived MRT variability during abrupt sun–shade transitions.
- MaRTy+/3D+ are comparative diagnostic tools for evaluating locations and heat-mitigation designs; they do not replace physiological monitoring of individuals.
- Stolwijk-2024 has been evaluated against human-subject data for air temperatures of 28–47 °C, MRT of 28–57 °C, air speeds of 0.1–3.3 m s<sup>−1</sup>, metabolic rates of 0.8–12.1 MET, and clothing from nude to 1.7 clo (Joshi et al., 2025; Abdullah et al., 2026). Results outside these conditions should be interpreted with care.

**NOTE:** *Compile the program in CRBasic Editor and check sensor wiring, SDI-12 addresses, and serial ports against your own cart before deploying.*

## Bibliography

- Abdullah, M., Joshi, A., Viswanathan, S. H., Vanos, J., Middel, A., Rotella, J., & Rykaczewski, K. (2026). Linking Urban Microclimate Sensing and Thermoregulation Modeling for Real-Time Heat-Strain Assessment. *Nexus*, 100184. https://doi.org/10.1016/j.ynexs.2026.100184
- Abdullah, M., Viswanathan, S. H., Jose, C. T., Vanos, J., Middel, A., Rykaczewski, K., & Joshi, A. (2026). Turbulence of airflow matters in human thermophysiological response in the heat. *Temperature*. https://doi.org/10.1080/23328940.2026.2620979
- Joshi, A., Twidwell, B., Park, M., & Rykaczewski, K. (2025). Comparative analysis of thermoregulation models to assess heat strain in moderate to extreme heat. *Journal of Thermal Biology*. https://pubmed.ncbi.nlm.nih.gov/39740271/
- Middel, A., & Krayenhoff, E. S. (2019). Micrometeorological determinants of pedestrian thermal exposure during record-breaking heat in Tempe, Arizona: Introducing the MaRTy observational platform. *Science of the Total Environment*, 687, 137–151.
