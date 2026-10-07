# reactingNanoParcelFoam

`reactingNanoParcelFoam` is a custom OpenFOAM solver derived from `reactingParcelFoam` for compressible reacting flow with a reacting multiphase Lagrangian cloud, surface-film coupling, and nano-scale droplet models. It is intended for cases where droplet diameter, droplet density, phase change, electric forces, and Brownian-scale transport effects matter more than in the stock reacting parcel solver.

This solver has been checked only with OpenFOAM-v2406 through OpenFOAM-v2506.

## Features

- Retains the main `reactingParcelFoam` carrier-gas workflow: compressible PIMPLE flow, species, energy, combustion, radiation, dynamic mesh support, surface film, and reacting multiphase parcels.
- Adds custom Lagrangian libraries and submodels including `CoulombForce`, `BrownianMotionForce`, `LiquidEvapFuchsKnudsen`, `massRosinRammler`, `PatchCollisionDensity`, and related parcel infrastructure.
- Adds droplet mass/volume/density handling for nano-scale evaporation or condensation, including controls such as `volumeUpdateMethod`, `constantVolume`, `minParcelMass`, and optional `lockParcelDiameter`.

## Why This Solver Exists

The stock `reactingParcelFoam` can become unstable for nano-order liquid droplet diameter change because its parcel-density update is not suitable for that scale. In practical nano-droplet calculations this can lead to divergence as the droplet mass and diameter evolve.

This solver carries custom parcel mass, volume, and density update logic so nano-scale droplet shrinkage or growth can be represented more robustly. This point is not obvious from looking only at the top-level solver file; the important changes are in the bundled Lagrangian parcel templates and phase-change submodels.

## Compilation

Load an OpenFOAM-v2406, v2412, or v2506 environment first:

```bash
source /path/to/OpenFOAM-v2506/etc/bashrc
```

Build everything from this directory:

```bash
./Allwmake
```

The build script compiles the bundled custom Lagrangian libraries first, then the solver executable:

```text
$FOAM_USER_APPBIN/reactingNanoParcelFoam
```

To clean generated build files:

```bash
./Allwclean
```

If your checkout loses executable bits, run:

```bash
chmod +x Allwmake Allwclean
```

## Required Case Setup

Start from a valid `reactingParcelFoam` case and replace the application in `system/controlDict`:

```text
application     reactingNanoParcelFoam;
```

The case must provide the normal compressible reacting-flow fields and dictionaries required by `reactingParcelFoam`, including thermophysical properties, chemistry/combustion settings as needed, turbulence settings, `fvSchemes`, `fvSolution`, and the `reactingCloud1Properties` dictionary.

For parcel calculations, configure:

```text
constant/reactingCloud1Properties
```

or the corresponding cloud dictionary used by your case.

Typical custom model entries include:

```text
subModels
{
    particleForces
    {
        sphereDrag;
        gravity;       // If you use only submicron particles, you do not need this
        BrownianMotion
        {
            turbulence false;
            lambda     68e-9;
        }
    }

    heatTransferModel  RanzMarshall; // fluid - particle heat transfer
    // RanzMarshall -> q=hA(T∞​−Tp​)
    RanzMarshallCoeffs
    {
        BirdCorrection  off;
    }

    compositionModel singleMixtureFraction; // Defines how particle composition is treated
    // singleMixtureFraction: uses a single mixture fraction to track composition
    
    singleMixtureFractionCoeffs
    {
        phases
        (
            gas
            {
            }
            liquid
            {
                H2O  1;
            }
            solid
            {
                NaCl 1;
            }
        );
    
        YGasTot0        0;
        YLiquidTot0     1;
        YSolidTot0      0;
    }

    phaseChangeModel  liquidEvapFuchsKnudsen;
    liquidEvapFuchsKnudsenCoeffs
    {
        gamma               6.8e-8;     // Mean gas free path
        alpham              1;          // The mass thermal accomodation
        solution            (H2O NaCl); // Solution (liquid solid)

        activityCoefficient Hoff;
        ic                  1.85;
        enthalpyTransfer    enthalpyDifference;
    }
}
```

## Usage

Run the solver in serial:

```bash
reactingNanoParcelFoam
```

For parallel cases:

```bash
decomposePar
mpirun -np <N> reactingNanoParcelFoam -parallel
reconstructPar
```

The solver supports split carrier-equation controls under `PIMPLE` in `system/fvSolution`:

```text
PIMPLE
{
    solveFlow      true;
    solveSpecies   true;
    solveEnergy    true;
}
```

Use these switches to freeze selected carrier equations while still advancing parcel, species, or thermal behavior as required by your study.

## Important Model Controls

For nano-droplet calculations, review the parcel `constantProperties` section carefully:

```text
constantProperties
{
    rho0            1000;             // water
    T0              298.15;
    Cp0             4187;
    minParcelMass   5.23599e-25;      // 1 nm water droplets 
    lockParcelDiameterSet   false;    // default "false"
    lockParcelDiameter      800e-9;   // unit is (m). vaild only when lockParcelDiameterSet is "true"

    volumeUpdateMethod  updateRhoAndVol;
}
```

Common choices:

- `volumeUpdateMethod constRho`: update droplet diameter at constant density.
- `volumeUpdateMethod constVol`: update density at constant volume.
- `volumeUpdateMethod updateRhoAndVol`: update both density and volume from component mass changes.
- `lockParcelDiameterSet true` with `lockParcelDiameter <value>`: stop diameter shrinkage below a selected cutoff and update density instead.

Use values consistent with your liquid mixture, time step, and minimum physically meaningful parcel mass.

## Notes

- The bundled Lagrangian source directories are part of the solver and must be compiled with the solver.
- Generated `lnInclude` directories and old object files are intentionally not stored in the repository; `wmake` recreates them.
- In this solver, configure `LiquidEvapFuchsKnudsen` with two `solution` entries: the volatile liquid species first and the non-volatile/solid component second.

