# Parallel Construction of Independent Spanning Trees in Bubble-Sort Networks (C++ / OpenMP / MPI)

This project builds the **n − 1 Independent Spanning Trees (ISTs)** of the bubble-sort network *Bₙ* using a non-recursive algorithm from a research paper, in which each vertex's parent can be computed independently. It has three implementations (serial, OpenMP, and hybrid MPI + OpenMP) and compares their runtimes. It was a group project, and the repository includes a written report and slides.

## Background

- In the bubble-sort network *Bₙ*, every vertex is a permutation of {1, …, n}, which gives *n!* vertices. Two vertices are connected when they differ by a swap of adjacent positions.
- Independent spanning trees that share a root give disjoint paths from the root to every vertex. This is useful for fault-tolerant routing and secure (split-message) transmission.
- The paper's algorithm finds the parent of a vertex *v* in tree *Tₜ* in constant time, using the last symbols of *v*, its inverse permutation and its rightmost misplaced symbol. This makes the algorithm O(n·n!) overall and lets every vertex be processed independently.

## What it does

- **Generates all n! permutations** (the code uses `n = 9`) and, for every non-root vertex and every *t* = 1 … n−1, computes its parent with `Parent1()`, `FindPosition()`, `Swap()` and `r()`. Each tree is stored as a child → parent map.
- **Serial version** (`serial.cpp`): a single-threaded baseline.
- **OpenMP version** (`Ists_openmp.cpp`): splits the loop over permutations with `#pragma omp parallel for` (36 threads). It caches each permutation's inverse and its `r(v)` value, and protects inserts into each tree with a per-tree `std::mutex`.
- **Hybrid MPI + OpenMP version** (`MPI+OPENMP.cpp`):
  - Splits the *n!* permutation indices into contiguous blocks, one per rank. Each rank rebuilds its permutations from their indices with **Lehmer-code** decoding (`index_to_permutation`).
  - Uses OpenMP threads inside each rank.
  - Serializes each local tree into flat integer arrays, gathers them on rank 0 with `MPI_Gather` / `MPI_Gatherv`, and times the run with `MPI_Wtime`.
- Includes helpers for printing the trees and doing a level-order (BFS) traversal. These calls are commented out in `main` by default.

## Results (from `report.project.docx`)

The report gives these runtimes for n = 9:

| Version | Execution time | Speedup vs serial |
|---|---|---|
| Serial | 22.8484 s | 1× |
| OpenMP | 5.1145 s | 4.47× |
| MPI + OpenMP | 21.456 s | 1.07× |

The report also includes gprof profiling output and a chart of execution time against speedup.

> **Note on METIS:** The report and slides describe using METIS to partition *Bₙ* across MPI processes. The MPI code in this repository does not call METIS. It divides the work into contiguous blocks of permutation indices.

## Tech stack

- **C++17**: `std::map`, `std::unordered_map`, structured bindings, `<chrono>`
- **OpenMP**: `parallel for` and `std::mutex` for thread-safe tree updates
- **MPI**: `MPI_Gather`, `MPI_Gatherv` and `MPI_Wtime`. According to the report, it was run on a multi-node cluster over SSH with a hostfile
- **gprof** for profiling (from the report)

## Project structure

```
serial.cpp           # Sequential IST construction (baseline)
Ists_openmp.cpp      # OpenMP-parallel IST construction
MPI+OPENMP.cpp       # Hybrid MPI + OpenMP version with Lehmer-code indexing
report.project.docx  # Project report (problem, design, profiling, results)
slides.pptx          # Presentation slides
```

## Build & run

The OpenMP and MPI commands match the ones shown in the report. The serial command is an example inferred from the code, which needs C++17.

```bash
# Serial (example)
g++ -std=c++17 -O2 serial.cpp -o ist_serial
./ist_serial

# OpenMP
g++ -fopenmp -std=c++17 Ists_openmp.cpp -o ist_parallel
./ist_parallel

# Hybrid MPI + OpenMP
mpic++ -fopenmp -std=c++17 "MPI+OPENMP.cpp" -o ist_mpi
mpirun -np 4 --hostfile hosts ./ist_mpi
```

To change the network size, edit `n` in `main()` in each source file. The default is 9.

---

**Author:** Raja Saif Ali — [GitHub](https://github.com/raja-saif) · [LinkedIn](https://www.linkedin.com/in/raja-saif-01702329a/)
