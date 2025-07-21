!!! example "Try out PyCO2SYS v2!"
    PyCO2SYS v2 is currently in beta testing.  The v2 docs and installation instructions are at [mvdh.xyz/PyCO2SYS](https://mvdh.xyz/PyCO2SYS/).  For now, installing via `pip` or `conda` still gets you v1.8.3, matching the docs here ([PyCO2SYS.readthedocs.io](https://pyco2sys.readthedocs.io/en/latest/)).

    Please try v2 out and report any issues you encounter via [the GitHub repo](https://github.com/mvdh7/PyCO2SYS/issues)!
    
    The API has been kept as similar as possible to v1, but there are some breaking changes.  The v2 code runs significantly faster with lower memory overhead and is designed to be more intuitive to use.

# Calculate everything with `pyco2.sys`

## Syntax

From v1.6.0, the recommended way to run PyCO2SYS is to calculate everything you need at once with the top-level `pyco2.sys` function.  The syntax is:

```python
import PyCO2SYS as pyco2
results = pyco2.sys(
    par1=None, par2=None, par1_type=None, par2_type=None, **kwargs
)
```

The simplest possible syntax above only requires values for two carbonate system parameters (`par1` and `par2`) and the types of these parameters (`par1_type` and `par2_type`).  Everything else is assigned default values.  To override the default values, add in the relevant `kwargs` from below.

!!! warning "`sys == CO2SYS_nd`"

    `pyco2.sys` is and will remain identical to `pyco2.CO2SYS_nd`, which was introduced in v1.5.0 (and which still works in exactly the same way).

From v1.7.0, it is possible to run `pyco2.sys` without providing any carbonate system parameters or with just one parameter (see below).  All of the other optional arguments can still be used.  In this case, the `results` dict contains all the equilibrium constants and total salt contents under the specified conditions, e.g.:

```python
# Convert a pH value to all pH scales, but don't solve the whole system:
results = pyco2.sys(par1=8.1, par1_type=3, **kwargs)

# Convert an fCO2 value to another temperature, but don't solve:
results = pyco2.sys(
    par1=400, par1_type=5, temperature=12, temperature_out=25, **kwargs
)

# Evaluate all equilibrium constants and total salts at default conditions:
results = pyco2.sys()
```

You can also use `pyco2.sys` to [propagate uncertainties](../uncertainty) through your marine carbonate system calculations.

## Arguments

Each argument to `pyco2.sys` described on this page can either be a single scalar value, or a [NumPy array](https://docs.scipy.org/doc/numpy/reference/generated/numpy.array.html) containing a series of values.  A combination of different multidimensional array shapes and sizes is allowed as long as they can all be [broadcasted](https://numpy.org/doc/stable/user/basics.broadcasting.html) with each other.

!!! inputs "`pyco2.sys` arguments"

    For all arguments and results in μmol·kg<sup>−1</sup>, the "kg" refers to the total solution, not H<sub>2</sub>O.  These are therefore most accurately termed *substance content* or *molinity* values (as opposed to *concentration* or *molality*).

    #### Carbonate system parameters

    Either two, one or no carbonate system parameters can be provided.

    * `par1` and `par2`: values of two different carbonate system parameters.
    * `par1_type` and `par2_type`: which types of parameter `par1` and `par2` are.

    If two parameters are provided, these can be any pair of:

    * **Total alkalinity** (type `1`) in μmol·kg<sup>−1</sup>.
    * **Dissolved inorganic carbon** (type `2`) in μmol·kg<sup>−1</sup>.
    * **pH** (type `3`) on the Total, Seawater, Free or NBS scale[^1].  Which scale is given by the argument `opt_pH_scale`.
    * Any one of:
        * **Partial pressure of CO<sub>2</sub>** (type `4`) in μatm,
        * **Fugacity of CO<sub>2</sub>** (type `5`) in μatm,
        * **Aqueous CO<sub>2</sub>** (type `8`) in μmol·kg<sup>−1</sup>, or
        * **Dry mole fraction of CO<sub>2</sub>** (type `9`) in ppm.
    * Any one of:
        * **Carbonate ion** (type `6`) in μmol·kg<sup>−1</sup>,
        * **Saturation state w.r.t. calcite** (type `10`), or
        * **Saturation state w.r.t. aragonite** (type `11`).
    * **Bicarbonate ion** (type `7`) in μmol·kg<sup>−1</sup>.

    If one parameter is provided, then the full marine carbonate system cannot be solved, but some results can be calculated.  The single parameter must be given as `par1` plus the relevant `par1_type`, and it can be any of:

    * **pH** (type `3`) on any scale, as above.  In this case, the pH values are converted to all the other pH scales at the input conditions only.
    * Any one of types `4` (<i>p</i>CO<sub>2</sub>), `5` (<i>ƒ</i>CO<sub>2</sub>), `8` ([CO<sub>2</sub>(aq)]) or `9` (<i>x</i>CO<sub>2</sub>) above.  In this case, the others in this group of parameters are all calculated at the input conditions.  If output temperatures are provided, then they are also all calculated at this condition by converting <i>p</i>CO<sub>2</sub> to the new temperature following [TSW09](../refs/#t).

    If no carbonate system parameters are provided, then all the equilibrium constants and total salt contents are returned, under the given conditions.

    #### Hydrographic conditions

    * `salinity`: **practical salinity** (default 35).
    * `temperature`: **temperature** at which `par1` and `par2` arguments are provided in °C (default 25 °C).
    * `pressure`: **water pressure** at which `par1` and `par2` arguments are provided in dbar (default 0 dbar).

    If you also want to calculate outputs at a different temperature and pressure from the original measurements, then you can also use:

    * `temperature_out`: **temperature** at which results will be calculated in °C ("output conditions").
    * `pressure_out`: **water pressure** at which results will be calculated in dbar ("output conditions").

    For example, if a sample was collected at 1000 dbar pressure (~1 km depth) at an in situ water temperature of 2.5 °C and subsequently measured in a lab at 25 °C, then the correct values would be `temperature=25`, `temperature_out=2.5`, `pressure=0`, and `pressure_out=1000`.

    If neither `temperature_out` nor `pressure_out` is provided, then calculations will only be performed at the conditions specified by `temperature` and `pressure`, and none of the results with keys ending with `_out` will be returned in the `CO2_results` dict.  If only one of `temperature_out` or `pressure_out` is provided, then we assume that the other one has the same values for the input and output calculations.

    #### Nutrients and other solutes

    Some default to zero if not provided:

    * `total_silicate`: **total silicate** in μmol·kg<sup>−1</sup>.
    * `total_phosphate`: **total phosphate** in μmol·kg<sup>−1</sup>.
    * `total_ammonia`: **total ammonia** in μmol·kg<sup>−1</sup>.
    * `total_sulfide`: **total hydrogen sulfide** in μmol·kg<sup>−1</sup>.
    * `total_alpha`: **total Hα** (a user-defined extra contributor to alkalinity) in μmol·kg<sup>−1</sup>.
    * `total_beta`: **total Hβ** (a user-defined extra contributor to alkalinity) in μmol·kg<sup>−1</sup>.

    If using non-zero `total_alpha` and/or `total_beta`, then you should also provide the corresponding equilibrium constant values `k_alpha` and/or `k_beta`.

    Others, PyCO2SYS estimates from salinity if not provided:

    * `total_borate`: **total borate** in μmol·kg<sup>−1</sup>.
    * `total_calcium`: **total calcium** in μmol·kg<sup>−1</sup>.
    * `total_fluoride`: **total fluoride** in μmol·kg<sup>−1</sup>.
    * `total_sulfate`: **total sulfate** in μmol·kg<sup>−1</sup>.

    If `total_borate` is provided, then the `opt_total_borate` argument is ignored.

    Throughout, the kg in μmol·kg<sup>−1</sup> refers to the total solution, not H<sub>2</sub>O.

    #### Atmospheric pressure

    * `pressure_atmosphere`/`pressure_atmosphere_out`: atmospheric pressure in atm.

    This is used for conversions between *p*CO<sub>2</sub>, *f*CO<sub>2</sub> and *x*CO<sub>2</sub>.  If no value is provided, then 1 atm is assumed.

    #### Settings

    * `opt_pH_scale`: which **pH scale** was used for any pH entries in `par1` or `par2`, as defined by [ZW01](../refs/#z):
        * `1`: Total, i.e. $\mathrm{pH} = -\log_{10} ([\mathrm{H}^+] + [\mathrm{HSO}_4^-])$ **(default)**.
        * `2`: Seawater, i.e. $\mathrm{pH} = -\log_{10} ([\mathrm{H}^+] + [\mathrm{HSO}_4^-] + [\mathrm{HF}])$.
        * `3`: Free, i.e. $\mathrm{pH} = -\log_{10} [\mathrm{H}^+]$.
        * `4`: NBS, i.e. relative to [NBS/NIST](https://www.nist.gov/history/nist-100-foundations-progress/nbs-nist) reference standards.

    * `opt_k_carbonic`: which set of equilibrium constant parameterisations to use to model **carbonic acid dissociation:**
        * `1`: [RRV93](../refs/#r) (0 < *T* < 45 °C, 5 < *S* < 45, Total scale, artificial seawater).
        * `2`: [GP89](../refs/#g) (−1 < *T* < 40 °C, 10 < *S* < 50, Seawater scale, artificial seawater).
        * `3`: [H73a](../refs/#h) and [H73b](../refs/#h) refit by [DM87](../refs/#d) (2 < *T* < 35 °C, 20 < *S* < 40, Seawater scale, artificial seawater).
        * `4`: [MCHP73](../refs/#m) refit by [DM87](../refs/#d) (2 < *T* < 35 °C, 20 < *S* < 40, Seawater scale, real seawater).
        * `5`: [H73a](../refs/#h), [H73b](../refs/#h) and [MCHP73](../refs/#m) refit by [DM87](../refs/#d) (2 < *T* < 35 °C, 20 < *S* < 40, Seawater scale, artificial seawater).
        * `6`: [MCHP73](../refs/#m) aka "GEOSECS" (2 < *T* < 35 °C, 19 < *S* < 43, NBS scale, real seawater).
        * `7`: [MCHP73](../refs/#m) without certain species aka "Peng" (2 < *T* < 35 °C, 19 < *S* < 43, NBS scale, real seawater).
        * `8`: [M79](../refs/#m) (0 < *T* < 50 °C, *S* = 0, freshwater only).
        * `9`: [CW98](../refs/#c) (2 < *T* < 30 °C, 0 < *S* < 40, NBS scale, real estuarine seawater).
        * `10`: [LDK00](../refs/#l) (2 < *T* < 35 °C, 19 < *S* < 43, Total scale, real seawater) **(default)**.
        * `11`: [MM02](../refs/#m) (0 < *T* < 45 °C, 5 < *S* < 42, Seawater scale, real seawater).
        * `12`: [MPL02](../refs/#m) (−1.6 < *T* < 35 °C, 34 < *S* < 37, Seawater scale, field measurements).
        * `13`: [MGH06](../refs/#m) (0 < *T* < 50 °C, 1 < *S* < 50, Seawater scale, real seawater).
        * `14`: [M10](../refs/#m) (0 < *T* < 50 °C, 1 < *S* < 50, Seawater scale, real seawater).
        * `15`: [WMW14](../refs/#w) (0 < *T* < 45 °C, 0 < *S* < 45, Seawater scale, real seawater).
        * `16`: [SLH20](../refs/#s)  (−1.67 < *T* < 31.80 °C, 30.73 < *S* < 37.57, Total scale, field measurements).
        * `17`: [SB21](../refs/#s) (15 < *T* < 35 °C, 19.6 < *S* < 41, Total scale, real seawater).
        * `18`: [PLR18](../refs/#p) (–6 < *T* < 25 °C, 33 < *S* < 100, Total scale, real seawater).

    The brackets above show the valid temperature (*T*) and salinity (*S*) ranges, original pH scale, and type of material measured to derive each set of constants.

    * `opt_k_bisulfate`: which equilibrium constant parameterisation to use to model **bisulfate ion dissociation**:

        * `1`: [D90a](../refs/#d) **(default)**.
        * `2`: [KRCB77](../refs/#k).
        * `3`: [WM13](../refs/#w)/[WMW14](../refs/#w).

    * `opt_total_borate`: which **boron:salinity** relationship to use to estimate total borate (ignored if the `total_borate` argument is provided):

        * `1`: [U74](../refs/#u) **(default)**.
        * `2`: [LKB10](../refs/#l).
        * `3`: [KSK18](../refs/#k).

    * `opt_k_fluoride`: which equilibrium constant parameterisation to use for **hydrogen fluoride dissociation:**
        * `1`: [DR79](../refs/#d) **(default)**.
        * `2`: [PF87](../refs/#p).

    * `opt_buffers_mode`: how to calculate the various **buffer factors** (or not).
        * `1`: using automatic differentiation, which accounts for the effects of all equilibrating solutes **(default)**.
        * `2`: using explicit equations reported in the literature, which only account for carbonate, borate and water alkalinity.
        * `0`: not at all.

    For `opt_buffers_mode`, `1` is the recommended and most accurate calculation, and it is a little faster to compute than `2`.  If `0` is selected, then the corresponding outputs have the value `np.nan`.

    * `opt_gas_constant`: what value to use for the **gas constant** (*R*):
        * `1`: DOEv2 (consistent with other CO2SYS software before July 2020).
        * `2`: DOEv3.
        * `3`: [2018 CODATA](https://physics.nist.gov/cgi-bin/cuu/Value?r) **(default)**.

    * `opt_pressured_kCO2`: whether to correct the CO<sub>2</sub> solubility constant for hydrostatic pressure following [W74](../refs/#w) (see discussion in section 2.1.3 of [OE15](../refs/#o)):
        * `0`: do not apply the correction (**default** and the only option before v1.8.2).
        * `1`: apply the correction.

    * `opt_adjust_temperature`: how to convert <i>p</i>CO<sub>2</sub>, <i>ƒ</i>CO<sub>2</sub>, [CO<sub>2</sub>(aq)] and/or <i>x</i>CO<sub>2</sub> to a different temperature, when only one parameter is provided.
        * `1`: using the parameterised <i>υ<sub>h</sub></i> equation of [H24](../refs/#h) (**default**). 
        * `2`: using the constant <i>υ<sub>h</sub></i> fitted to the [TOG93](../refs/#t) dataset by [H24](../refs/#h).
        * `3`: using the constant theoretical <i>υ<sub>x</sub></i> of [H24](../refs/#h).
        * `4`: following the [H24](../refs/#h) approach but using a user-provided $b_h$ value (given with the kwarg `bh_upsilon`).
        * `5`: using the linear fit of [TOG93](../refs/#t).
        * `6`: using the quadratic fit of [TOG93](../refs/#t) (default before v1.8.3).

    * `opt_which_fCO2_insitu`: whether the input (`1`, **default**) or output (`2`) condition <i>p</i>CO<sub>2</sub>, <i>ƒ</i>CO<sub>2</sub>, [CO<sub>2</sub>(aq)] and/or <i>x</i>CO<sub>2</sub> values are at in situ conditions, for determining <i>b<sub>h</sub></i> with the parameterisation of [H24](../refs/#h).  Only applies when `opt_adjust_temperature` is `1`.

    #### Override equilibrium constants

    All the equilibrium constants needed by PyCO2SYS are estimated internally from temperature, salinity and pressure, and returned in the results.  However, you can also directly provide your own values for any of these constants instead.

    To do this, the arguments have the same keywords as the corresponding [results dict keys](#equilibrium-constants).  For example, to provide your own water dissociation constant value at input conditions of $10^{-14}$, use `k_water=1e-14`.

    If non-zero using `total_alpha` and/or `total_beta`, you should also supply the corresponding stoichiometric dissociation constant values as `k_alpha`/`k_alpha_out` and/or `k_beta`/`k_beta_out`.  If not provided, these default to p*K* = 7.

## Results

The results of `pyco2.sys` calculations are stored in a [dict](https://docs.python.org/3/tutorial/datastructures.html#dictionaries) of [NumPy arrays](https://docs.scipy.org/doc/numpy/reference/generated/numpy.array.html).  The keys to the dict are listed in the section below.

Scalar arguments, and results that depend only on scalar arguments, will be returned as scalars in the dict.  Array-like arguments, and results that depend on them, will all be broadcast to the same consistent shape.

The keys ending with `_out` are only available if at least one of the `temperature_out` or `pressure_out` arguments was provided.

!!! outputs "`pyco2.sys` results dict"

    #### Dissolved inorganic carbon

    * `"dic"`: **dissolved inorganic carbon** in μmol·kg<sup>−1</sup>.
    * `"carbonate"`/`"carbonate_out"`: **carbonate ion** at input/output conditions in μmol·kg<sup>−1</sup>.
    * `"bicarbonate"`/`"bicarbonate_out"`: **bicarbonate ion** at input/output conditions in μmol·kg<sup>−1</sup>.
    * `"aqueous_CO2"`/`"aqueous_CO2_out"`: **aqueous CO<sub>2</sub>** at input/output conditions in μmol·kg<sup>−1</sup>.
    * `"pCO2"`/`"pCO2_out"`: **seawater partial pressure of CO<sub>2</sub>** at input/output conditions in μatm.
    * `"fCO2"`/`"fCO2_out"`: **seawater fugacity of CO<sub>2</sub>** at input/output conditions in μatm.
    * `"xCO2"`/`"xCO2_out"`: **seawater dry mole fraction of CO<sub>2</sub>** at input/output conditions in ppm.
    * `"fugacity_factor"`/`"fugacity_factor_out"`: **fugacity factor** at input/output conditions for converting between CO<sub>2</sub> partial pressure and fugacity.
    * `"vp_factor"`/`"vp_factor_out"`: **vapour pressure factor** at input/output conditions for converting between <i>x</i>CO<sub>2</sub> and <i>p</i>CO<sub>2</sub>.

    #### Alkalinity and its components

    * `"alkalinity"`: **total alkalinity** in μmol·kg<sup>−1</sup>.
    * `"alkalinity_borate"`/`"alkalinity_borate_out"`: **borate alkalinity** at input/output conditions in μmol·kg<sup>−1</sup>.
    * `"alkalinity_phosphate"`/`"alkalinity_phosphate_out"`: **phosphate alkalinity** at input/output conditions in μmol·kg<sup>−1</sup>.
    * `"alkalinity_silicate"`/`"alkalinity_silicate_out"`: **silicate alkalinity** at input/output conditions in μmol·kg<sup>−1</sup>.
    * `"alkalinity_ammonia"`/`"alkalinity_ammonia_out"`: **ammonia alkalinity** at input/output conditions in μmol·kg<sup>−1</sup>.
    * `"alkalinity_sulfide"`/`"alkalinity_sulfide_out"`: **hydrogen sulfide alkalinity** at input/output conditions in μmol·kg<sup>−1</sup>.
    * `"alkalinity_alpha"`/`"alkalinity_alpha_out"`: **Hα alkalinity** at input/output conditions in μmol·kg<sup>−1</sup>.
    * `"alkalinity_beta"`/`"alkalinity_beta_out"`: **Hβ alkalinity** at input/output conditions in μmol·kg<sup>−1</sup>.
    * `"peng_correction"`: the **"Peng correction"** for alkalinity (applies only for `opt_k_carbonic = 7`) in μmol·kg<sup>−1</sup>.

    #### pH and water

    * `"pH"`/`"pH_out"`: **pH** at input/output conditions on the scale specified by input `opt_pH_scale`.
    * `"pH_total"`/`"pH_total_out"`: **pH** at input/output conditions on the **Total scale**.
    * `"pH_sws"`/`"pH_sws_out"`: **pH** at input/output conditions on the **Seawater scale**.
    * `"pH_free"`/`"pH_free_out"`: **pH** at input/output conditions on the **Free scale**.
    * `"pH_nbs"`/`"pH_nbs_out"`: **pH** at input/output conditions on the **NBS scale**.
    * `"hydrogen_free"`/`"hydrogen_free_out"`: **"free" proton** at input/output conditions in μmol·kg<sup>−1</sup>.
    * `"hydroxide"`/`"hydroxide_out"`: **hydroxide ion** at input/output conditions in μmol·kg<sup>−1</sup>.
    * `"fH"`/`"fH_out"`: **activity coefficient of H<sup>+</sup>** at input/output conditions for pH-scale conversions to and from the NBS scale.

    #### Carbonate mineral saturation

    * `"saturation_calcite"`/`"saturation_calcite_out"`: **saturation state of calcite** at input/output conditions.
    * `"saturation_aragonite"`/`"saturation_aragonite_out"`: **saturation state of aragonite** at input/output conditions.

    #### Buffer factors

    Whether these are evaluated using automatic differentiation, with explicit equations, or not at all is controlled by the input `opt_buffers_mode`.

    * `"revelle_factor"`/`"revelle_factor_out"`: **Revelle factor** at input/output conditions[^2].
    * `"psi"`/`"psi_out"`: *ψ* of [FCG94](../refs/#f) at input/output conditions.
    * `"gamma_dic"`/`"gamma_dic_out"`: **buffer factor *γ*<sub>DIC</sub>** of [ESM10](../refs/#e) at input/output conditions[^3].
    * `"beta_dic"`/`"beta_dic_out"`: **buffer factor *β*<sub>DIC</sub>** of [ESM10](../refs/#e) at input/output conditions.
    * `"omega_dic"`/`"omega_dic_out"`: **buffer factor *ω*<sub>DIC</sub>** of [ESM10](../refs/#e) at input/output conditions.
    * `"gamma_alk"`/`"gamma_alk_out"`: **buffer factor *γ*<sub>TA</sub>** of [ESM10](../refs/#e) at input/output conditions.
    * `"beta_alk"`/`"beta_alk_out"`: **buffer factor *β*<sub>TA</sub>** of [ESM10](../refs/#e) at input/output conditions.
    * `"omega_alk"`/`"omega_alk_out"`: **buffer factor *ω*<sub>TA</sub>** of [ESM10](../refs/#e) at input/output conditions.
    * `"isocapnic_quotient"`/`"isocapnic_quotient_out"`: **isocapnic quotient** of [HDW18](../refs/#h) at input/output conditions.
    * `"isocapnic_quotient_approx"`/`"isocapnic_quotient_approx_out"`: **isocapnic quotient approximation** of [HDW18](../refs/#h) at input/output conditions.
    * `"dlnpCO2_dT"`/`"dlnpCO2_dT_out"`: **temperature derivative of ln(<i>ƒ</i>CO<sub>2</sub>)** at input/output conditions (see [TOG93](../refs/#t)).
    * `"dlnpCO2_dT"`/`"dlnpCO2_dT_out"`: **temperature derivative of ln(<i>p</i>CO<sub>2</sub>)** at input/output conditions (see [TOG93](../refs/#t)).

    #### Biological properties

    Seawater properties related to the marine carbonate system that have a primarily biological application.

    * `"substrate_inhibitor_ratio"`/`"substrate_inhibitor_ratio_out"`: **substrate:inhibitor ratio** of [B15](../refs/#b) at input/output conditions in mol(HCO<sub>3</sub><sup>−</sup>)·μmol(H<sup>+</sup>)<sup>−1</sup>.

    #### Chemical speciation

    Molality of each individual chemical species involved in pH equilibria.

    * `"HCO3"`/`"HCO3_out"`: **bicarbonate** $[\text{HCO}_3^-]$ at input/output conditions in μmol·kg<sup>−1</sup>.
    * `"CO3"`/`"CO3_out"`: **carbonate** $[\text{CO}_3^{2-}]$ at input/output conditions in μmol·kg<sup>−1</sup>.
    * `"CO2"`/`"CO2_out"`: **aqueous carbon dioxide** $[\text{CO}_2(\text{aq})]$ at input/output conditions in μmol·kg<sup>−1</sup>.
    * `"BOH4"`/`"BOH4_out"`: **tetrahydroxyborate** $[\text{B(OH)}_4^-]$ at input/output conditions in μmol·kg<sup>−1</sup>.
    * `"BOH3"`/`"BOH3_out"`: **boric acid** $[\text{B(OH)}_3]$ at input/output conditions in μmol·kg<sup>−1</sup>.
    * `"OH"`/`"OH_out"`: **hydroxide** $[\text{OH}^-]$ at input/output conditions in μmol·kg<sup>−1</sup>.
    * `"Hfree"`/`"Hfree_out"`: **"free" protons** $[\text{H}^+]$ at input/output conditions in μmol·kg<sup>−1</sup>.
    * `"H3PO4"`/`"H3PO4_out"`: **phosphoric acid** $[\text{H}_3\text{PO}_4]$ at input/output conditions in μmol·kg<sup>−1</sup>.
    * `"H2PO4"`/`"H2PO4_out"`: **dihydrogen phosphate** $[\text{H}_2\text{PO}_4^-]$ at input/output conditions in μmol·kg<sup>−1</sup>.
    * `"HPO4"`/`"HPO4_out"`: **monohydrogen phosphate** $[\text{HPO}_4^{2-}]$ at input/output conditions in μmol·kg<sup>−1</sup>.
    * `"PO4"`/`"PO4_out"`: **phosphate** $[\text{PO}_4^{3-}]$ at input/output conditions in μmol·kg<sup>−1</sup>.
    * `"H4SiO4"`/`"H4SiO4_out"`: **orthosilicic acid** $[\text{Si(OH)}_4]$ at input/output conditions in μmol·kg<sup>−1</sup>.
    * `"H3SiO4"`/`"H3SiO4_out"`: **trihydrogen orthosilicate** $[\text{SiO(OH)}_3^-]$ at input/output conditions in μmol·kg<sup>−1</sup>.
    * `"NH3"`/`"NH3_out"`: **ammonia** $[\text{NH}_3]$ at input/output conditions in μmol·kg<sup>−1</sup>.
    * `"NH4"`/`"NH4_out"`: **ammonium** $[\text{NH}_4^+]$ at input/output conditions in μmol·kg<sup>−1</sup>.
    * `"HS"`/`"HS_out"`: **bisulfide** $[\text{HS}^-]$ at input/output conditions in μmol·kg<sup>−1</sup>.
    * `"H2S"`/`"H2S_out"`: **hydrogen sulfide** $[\text{H}_2\text{S}]$ at input/output conditions in μmol·kg<sup>−1</sup>.
    * `"HSO4"`/`"HSO4_out"`: **bisulfate** $[\text{HSO}_4^-]$ at input/output conditions in μmol·kg<sup>−1</sup>.
    * `"SO4"`/`"SO4_out"`: **sulfate** $[\text{SO}_4^{2-}]$ at input/output conditions in μmol·kg<sup>−1</sup>.
    * `"HF"`/`"HF_out"`: **hydrofluoric acid** $[\text{HF}]$ at input/output conditions in μmol·kg<sup>−1</sup>.
    * `"F"`/`"F_out"`: **fluoride** $[\text{F}^-]$ at input/output conditions in μmol·kg<sup>−1</sup>.

    #### Totals estimated from salinity

    * `"total_borate"`: **total borate** in μmol·kg<sup>−1</sup>.
    * `"total_fluoride"`: **total fluoride** μmol·kg<sup>−1</sup>.
    * `"total_sulfate"`: **total sulfate** in μmol·kg<sup>−1</sup>.
    * `"total_calcium"`: **total calcium** in μmol·kg<sup>−1</sup>.

    #### Equilibrium constants

    All equilibrium constants are returned on the pH scale of input `pHSCALEIN` except for `"KFinput"`/`"KFoutput"` and `"KSO4input"`/`"KSO4output"`, which are always on the Free scale.

    * `"k_CO2"`/`"k_CO2_out"`: **Henry's constant for CO<sub>2</sub>** at input/output conditions.
    * `"k_carbonic_1"`/`"k_carbonic_1_out"`: **first carbonic acid** dissociation constant at input/output conditions.
    * `"k_carbonic_2"`/`"k_carbonic_2_out"`: **second carbonic acid** dissociation constant at input/output conditions.
    * `"k_water"`/`"k_water_out"`: **water** dissociation constant at input/output conditions.
    * `"k_borate"`/`"k_borate_out"`: **boric acid** dissociation constant at input/output conditions.
    * `"k_fluoride"`/`"k_fluoride_out"`: **hydrogen fluoride** dissociation constant at input/output conditions.
    * `"k_bisulfate"`/`"k_bisulfate_out"`: **bisulfate** dissociation constant at input/output conditions.
    * `"k_phosphoric_1"`/`"k_phosphoric_1_out"`: **first phosphoric acid** dissociation constant at input/output conditions.
    * `"k_phosphoric_2"`/`"k_phosphoric_2_out"`: **second phosphoric acid** dissociation constant at input/output conditions.
    * `"k_phosphoric_3"`/`"k_phosphoric_3_out"`: **third phosphoric acid** dissociation constant at input/output conditions.
    * `"k_silicate"`/`"k_silicate_out"`: **silicic acid** dissociation constant at input/output conditions.
    * `"k_ammonia"`/`"k_ammonia_out"`: **ammonia** equilibrium constant at input/output conditions.
    * `"k_sulfide"`/`"k_sulfide_out"`: **hydrogen sulfide** equilibrium constant at input/output conditions.
    * `"k_alpha"`/`"k_alpha_out"`: **HA** equilibrium constant at input/output conditions.
    * `"k_beta"`/`"k_beta_out"`: **HB** equilibrium constant at input/output conditions.

    The ideal gas constant used in the calculations is also returned.  Note the unusual unit:

    * `"gas_constant"`: **ideal gas constant** in ml·bar<sup>−1</sup>·mol<sup>−1</sup>·K<sup>−1</sup>.

    #### Function arguments

    All the function arguments not already mentioned here are also returned as results with the same keys.

[^1]: See [ZW01](../refs/#z) for definitions of the different pH scales.

[^2]: In `opt_buffers_mode=2`, the Revelle factor is calculated using a simple finite difference scheme, just like the MATLAB version of CO2SYS.

[^3]: Equations for the buffer factors of [ESM10](../refs/#e) in `opt_buffers_mode=2` have all been corrected for typos following [RAH18](../refs/#r) and [OEDG18](../refs/#o).
