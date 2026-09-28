# Lesieur Cristal – CPLEX Project

Two-echelon capacitated facility location MILP for Lesieur Cristal's
Moroccan distribution network. Solved with IBM ILOG CPLEX.

## Project structure

```
cplex_project/
├── opl/                         # IBM ILOG CPLEX Optimization Studio project
│   ├── LesieurCristal.mod       # OPL model (sets, parameters, variables, objective, constraints)
│   ├── LesieurCristal.ops       # CPLEX solver settings
│   ├── LesieurCristal.project   # OPL project descriptor (run configurations)
│   ├── baseline.dat             # Most-likely demand × Most-likely cost
│   ├── best_best.dat            # Best demand × Best cost scenario
│   └── worst_worst.dat          # Worst demand × Worst cost scenario
│
├── python/                      # Python docplex implementation
│   └── solve_cplex.py           # Solves all 9 scenario combinations
│
└── results/                     # Pre-computed results
    ├── console_output.txt       # What the script printed on the last run
    ├── cplex_solve.log          # Full CPLEX branch-and-bound log for all 9 runs
    ├── scenario_results.csv     # Tidy CSV of all 9 scenarios
    └── scenario_results.json    # Full JSON with flow detail per scenario
```

## How to run

### Option A — IBM ILOG CPLEX Optimization Studio (GUI)

1. Open CPLEX Optimization Studio.
2. File → Import → Existing OPL Project → select the `opl/` folder.
3. The project will appear in the OPL Projects view.
4. Right-click on `LesieurCristal.project` → "Run As" → choose a run
   configuration (`baseline`, `best_best`, or `worst_worst`).
5. Results print to the console and the solution browser opens with the
   optimal `y[]` and `x[][][]` values.

### Option B — Command line with `oplrun`

If you have CPLEX installed and `oplrun` is on your PATH:

```bash
cd opl
oplrun -p . baseline                  # solve baseline
oplrun -p . best_best                 # solve best/best
oplrun -p . worst_worst               # solve worst/worst
```

Or directly without the project descriptor:

```bash
oplrun LesieurCristal.mod baseline.dat
```

### Option C — Python (docplex)

This is the fastest path if you just want results:

```bash
cd python
pip install docplex cplex
python solve_cplex.py
```

This solves all 9 scenarios in under a second total and writes:
- `results/scenario_results.csv`  – summary of every scenario
- `results/scenario_results.json` – full solution detail
- `results/cplex_solve.log`       – CPLEX solver log

## Results (pre-computed)

All 9 scenarios solve to provable optimality (MIP gap = 0).
Baseline (Most-likely × Most-likely) objective = **141,001.90 kMAD/year**.

| Demand | Cost  | DCs opened           | Total (kMAD) | vs Base |
| ------ | ----- | -------------------- | ------------ | ------- |
| Best   | Best  | W1, W2, W3, W4       |   146,995.88 |   +4.3% |
| Best   | Most  | W1, W2, W3, W4       |   159,061.14 |  +12.8% |
| Best   | Worst | W1, W2, W3, W4       |   195,528.28 |  +38.7% |
| Most   | Best  | W1, W2, W3, W4       |   130,349.83 |   −7.6% |
| Most   | Most  | W1, W2, W3, W4       |   141,001.90 |  base   |
| Most   | Worst | W1, W2, W3, W4       |   173,111.95 |  +22.8% |
| Worst  | Best  | W1, W3, W4           |   109,094.80 |  −22.6% |
| Worst  | Most  | W1, W3, W4           |   118,038.41 |  −16.3% |
| Worst  | Worst | W1, W3, W4           |   145,050.70 |   +2.9% |

Where:
- **W1** = Tit Mellil (Casablanca)
- **W2** = Kenitra (Atlantic Free Zone)
- **W3** = Marrakech
- **W4** = Meknès
- **W5** = Tanger Med (never opened in any scenario)
- **W6** = Agadir (never opened in any scenario)

## Model details (in brief)

**Sets:** 2 supply nodes × 6 candidate DCs × 9 demand zones

**Decision variables:** 6 binary (DC opening) + 108 continuous (flows) = 114 total

**Constraints:** 17 functional constraints
- C1: 9 demand-satisfaction equalities
- C2: 2 supply-capacity inequalities
- C3: 6 DC-capacity (linked-to-opening) inequalities

**Objective:** minimise total annual logistics cost (kMAD), composed of
fixed DC opening costs + transport (supply → DC) + handling at DCs +
transport (DC → demand).

See `opl/LesieurCristal.mod` for the full MILP formulation.

## Cross-checks

The same model has been solved with three different engines and they
agree on every scenario:

| Engine                    | Algorithm                              | Baseline objective |
| ------------------------- | -------------------------------------- | ------------------ |
| IBM CPLEX 22.1            | Dynamic search (Simplex + B&B + cuts)  | 141,001.90 kMAD    |
| COIN-OR CBC (PuLP)        | Simplex + Branch-and-Bound             | 141,001.90 kMAD    |
| Excel Solver Simplex LP   | Simplex + Branch-and-Bound             | 141,001.90 kMAD    |

This triangulates the result and confirms the formulation is correct.
