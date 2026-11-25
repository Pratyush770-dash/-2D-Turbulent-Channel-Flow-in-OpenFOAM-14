# -2D-Turbulent-Channel-Flow-in-OpenFOAM-14
This is a steady, incompressible, turbulent flow through a straight 2D channel, solved in OpenFOAM 14 on Ubuntu with the standard k-epsilon model and wall functions. I built it while following the IIT Bombay Spoken Tutorial "Simulating Turbulent Flow in a 2D Channel", and then rebuilt the whole case from scratch for the newer OpenFOAM 14 setup, because a few things changed from the version the tutorial uses.

I did this mainly to get comfortable with the full turbulence workflow: picking a first cell height from a target y+, working out inlet k and epsilon, and setting up the matching wall-function boundary conditions.

## What the case does

Air-like fluid (nu = 1e-5 m²/s) enters a rectangular channel at a uniform 1 m/s. The top and bottom are walls, and the flow develops boundary layers along them as it moves downstream. I look at the velocity profile near the outlet to see the boundary layer shape.

| Item | Value |
|---|---|
| Channel length | 1 m |
| Channel height | 0.1 m |
| Thickness in z | 0.01 m (one cell, so it is effectively 2D) |
| Mesh | 100 x 10 x 1 cells, uniform |
| Inlet velocity | 1 m/s |
| Kinematic viscosity | 1e-5 m²/s |
| Turbulence model | k-epsilon (RAS) |
| Solver | `foamRun` with `solver incompressibleFluid` |
| Iterations | 2000 (steady state, written every 200) |

## How I picked the numbers

**First cell height (yp).** I wanted y+ around 30 so the wall functions are valid. Using a flat plate estimate:

- Re = U L / nu = 1e5
- Cf = 0.058 Re^-0.2 ≈ 0.0058
- wall shear stress = 0.5 Cf U² ≈ 0.0029
- friction velocity u_tau = sqrt(0.0029) ≈ 0.054 m/s
- yp = y+ nu / u_tau ≈ 0.0056 m

yp is the distance to the centre of the first cell, so the first cell is about twice that tall. That is why I used 10 cells across the 0.1 m height.

**Inlet turbulence.** I assumed 5% turbulence intensity and a length scale of 0.07 times the channel height.

- k = 1.5 (0.05 x 1)² = 0.00375 m²/s²
- epsilon = Cmu^0.75 k^1.5 / l ≈ 0.0054 m²/s³

## Boundary conditions

| Field | inlet | outlet | walls | frontAndBack |
|---|---|---|---|---|
| U | fixedValue (1 0 0) | zeroGradient | noSlip | empty |
| p | zeroGradient | fixedValue 0 | zeroGradient | empty |
| k | fixedValue 0.00375 | zeroGradient | kqRWallFunction | empty |
| epsilon | fixedValue 0.0054 | zeroGradient | epsilonWallFunction | empty |
| nut | calculated | calculated | nutkWallFunction | empty |

## Repository structure

```
openfoam_2D_turbulent_flow/
├── 0/                  initial and boundary conditions (U, p, k, epsilon, nut)
├── constant/           physicalProperties and momentumTransport
├── system/             blockMeshDict, controlDict, fvSchemes, fvSolution
├── case.foam           empty file so ParaView can open the case
├── log.run             solver output from the run
└── paraview.png        screenshot of the result from ParaView
```

## Running it

You need OpenFOAM 14 installed and sourced. From inside the case folder:

```bash
blockMesh
checkMesh
foamRun 2>&1 | tee log.run
```

The run stops on its own once the residuals drop below the limits in `fvSolution`, or at 2000 iterations, whichever comes first.

To check y+ on the walls:

```bash
foamPostProcess -func yPlus -latestTime
```

To look at the results:

```bash
touch case.foam
paraview case.foam
```

In ParaView I clicked Apply, jumped to the last time step, and coloured by U magnitude. For the outlet profile I used Plot Over Line from (0.995, 0, 0.005) to (0.995, 0.1, 0.005) and plotted U_x against Points_Y. I stayed slightly inside the outlet (x = 0.995) because the data is stored at cell centres.

## A few things that were different in OpenFOAM 14

- `simpleFoam` is not the main way to run this anymore. The steady incompressible solver is now `foamRun` with `solver incompressibleFluid`.
- The turbulence settings live in `constant/momentumTransport`, not `turbulenceProperties`.
- The viscosity lives in `constant/physicalProperties`, not `transportProperties`.
- I copied `fvSchemes`, `fvSolution` and `momentumTransport` from the `pitzDaily` tutorial in `incompressibleFluid` and then edited them, which was much less painful than writing the schemes by hand.

## Results

The flow develops boundary layers on both walls, and the velocity profile near the outlet is flat in the middle and drops steeply close to the walls, which is what a turbulent channel profile should look like. The ParaView screenshot is in `paraview.png`.

Solver converged after: [add iteration number from log.run]
Wall y+ (min / max / average): [add values from the yPlus output]

## Things I would improve

- Use a graded mesh toward the walls instead of a uniform one, so y+ is controlled better.
- Compare the outlet profile against the log law of the wall.
- Try k-omega SST and see how the profile and y+ requirements change.

## Author

**Pratyush Dash**
