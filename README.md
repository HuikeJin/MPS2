# GraceQ/MPS2
A high-performance matrix product state(MPS) algorithms library based on GraceQ/tensor

_"Easy push your D to 10k"_


## Design Goals

- High-performance MPS algorithms implementation based on the power of [GraceQ/tensor](https://github.com/gracequantum/tensor).
- Perform MPS optimization on kinds of HPC hardware architectures which also bases on the GraceQ/tensor project.
- Flexible API design to cope with complex research tasks. We do not offer something like t-J model, but you can define it in 5 minutes.


## Installation

GraceQ/MPS2 library depends [GraceQ/tensor](https://github.com/gracequantum/tensor) as its basic tensor library and [nlohmann/json](https://github.com/nlohmann/json) to parser input JSON file. So you need to install these dependencies first or use `--recurse-submodules` option to clone them as submodules.

```
git clone --recurse-submodules https://github.com/gracequantum/mps2.git gqmps2
```

Then you can use the [CMake](https://cmake.org/) tool to build the library.

```
cd gqmps2
mkdir build && cd build
cmake ..
make
```

Finally, you can get the library file `libgqmps2.a` at `./lib/libmps2.a`.


## Minimal tutorial

### Using GraceQ/MPS2
It is easy to use GraceQ/MPS2.

```cpp
#include "gqmps2/gqmps2.h"
using namespace gqmps2;
```

Because this library highly depends on the GraceQ/tensor, you also want to include it.

```cpp
#include "gqten/gqten.h"
using namespace gqten;
```

Then we calculate the ground state of the one dimensional [Heisenberg model](https://en.wikipedia.org/wiki/Heisenberg_model_(quantum)) to how to use GraceQ/MPS2.

### Input JSON file and `CaseParams` parser
A typical input [JSON](https://en.wikipedia.org/wiki/JSON) file looks like

```json
{
  "CaseParams": {
    "N": 20,
    "Sweeps": 10,
    "Dmin": 32,
    "Dmax": 32,
    "CutOff": 1e-8,
    "LanczErr": 1e-9,
    "MaxLanczIter": 100,
    "TenTransNumThreads": 4
  }
}
```
`"CaseParams"` object is necessary for the parameter parser. `"N"` is the system size. `"Sweeps"` is the MPS update time. `"Dmin"` is the minimal kept bond dimensions. `"Dmax"` is the maximal kept bond dimensions. `"CutOff"` is the truncation error. `"LanczErr"` is the [Lanczos algorithm](https://en.wikipedia.org/wiki/Lanczos_algorithm) iteration error. `"MaxLanczIter"` is the maximal Lanczos iteration time. `"TenTransNumThreads"` tells the GraceQ/tensor library use how many threads to perform tensor transpose.

To parser this JSON file, you can define your own parameter parser class.
```cpp
struct CaseParams : public CaseParamsParserBasic {
  CaseParams(const char *pf) : CaseParamsParserBasic(pf) {
    N = ParseInt("N");
    Sweeps = ParseInt("Sweeps");
    Dmin = ParseInt("Dmin");
    Dmax = ParseInt("Dmax");
    CutOff = ParseDouble("CutOff");
    LanczErr = ParseDouble("LanczErr");
    MaxLanczIter = ParseInt("MaxLanczIter");
    TenTransNumThreads = ParseInt("TenTransNumThreads");
  }

  long N;
  long Sweeps;
  long Dmin;
  long Dmax;
  double CutOff;
  double LanczErr;
  long MaxLanczIter;
  int TenTransNumThreads;
};
```
In the `main` function, use the JSON file path to initialize the parser.
```cpp
int main(int argc, char *argv[]) {
  CaseParams params(argv[1]);
}
```
Then you can use these parameters in you program.

### Define operators
Quantum operator can be represented as two ranks tensor. About how to define tensor in GraceQ/tensor library, you can refer [GraceQ/tensor](https://github.com/gracequantum/tensor).
```cpp
auto pb_out = Index({
                   QNSector(QN({QNNameVal("Sz", 1)}), 1),
                   QNSector(QN({QNNameVal("Sz", -1)}), 1)}, OUT);
auto pb_in = InverseIndex(pb_out);

auto sz = GQTensor({pb_in, pb_out});
auto sp = GQTensor({pb_in, pb_out});
auto sm = GQTensor({pb_in, pb_out});
auto id = GQTensor({pb_in, pb_out});
sz({0, 0}) = 0.5;
sz({1, 1}) = -0.5;
sp({0, 1}) = 1;
sm({1, 0}) = 1;
```
Where the physical bond `pb_out` will also be used latter.

### Generate matrix product operator(MPO)
You can use `gqmps2::MPOGenerator` to generate MPO contains any one-site and two-site terms. The following example tells you how to generate the Hamiltonian of the Heisenberg model.

```cpp
auto zero_div = QN({QNNameVal("Sz", 0)});
auto mpo_gen = MPOGenerator(params.N, pb_out, zero_div);
for (long i = 0; i < params.N-1; ++i) {
  mpo_gen.AddTerm(1, {OpIdx(sz, i), OpIdx(sz, i+1)});
  mpo_gen.AddTerm(0.5, {OpIdx(sp, i), OpIdx(sm, i+1)});
  mpo_gen.AddTerm(0.5, {OpIdx(sm, i), OpIdx(sp, i+1)});
}
auto mpo = mpo_gen.Gen();
```
The `OpIdx` defines the operator and the site which lives on. The type of the result MPO `mpo` is `std::vector<GQTensor *>`.

### Define initial MPS
You can define a base direct product state as the initial MPS. Because the U1 symmetry is kept during the iteration process, The quantum number of this initial MPS also labels the sector you are working in the whole Hilbert space. Temperately, MPS is also defined as a `std::vector<GQTensor *>` now.

```cpp
std::vector<GQTensor *> mps(params.N);
std::vector<long> stat_labs;
for (int i = 0; i < params.N; ++i) { stat_labs.push_back(i % 2); }
DirectStateInitMps(mps, stat_labs, pb_out, zero_div);
```

### Define sweep parameters
It is straightforward to define the sweep parameter class.

```cpp
auto sweep_params = SweepParams(
                        params.Sweeps,
                        params.Dmin,
                        params.Dmax,
                        params.CutOff,
                        true,
                        kTwoSiteAlgoWorkflowInitial,
                        LanczosParams(params.LanczErr, params.MaxLanczIter));
```
Where `kTwoSiteAlgoWorkflowInitial` tells the GraceQ/MPS2 to use the "initial" workflow to run the calculation.

### Run the two-site MPS update algorithm
Set the number of threads which tensor transpose calculation will use and call the algorithm function.

```cpp
gqten::GQTenSetTensorTransposeNumThreads(params.TenTransNumThreads);
auto energy0 = TwoSiteAlgorithm(mps, mpo, sweep_params);
```

### The demo you can run
Copy, compile, and run your first GraceQ/MPS2 application now.

```cpp
#include "gqmps2/gqmps2.h"
#include "gqten/gqten.h"


using namespace gqmps2;
using namespace gqten;


struct CaseParams : public CaseParamsParserBasic {
  CaseParams(const char *pf) : CaseParamsParserBasic(pf) {
    N = ParseInt("N");
    Sweeps = ParseInt("Sweeps");
    Dmin = ParseInt("Dmin");
    Dmax = ParseInt("Dmax");
    CutOff = ParseDouble("CutOff");
    LanczErr = ParseDouble("LanczErr");
    MaxLanczIter = ParseInt("MaxLanczIter");
    TenTransNumThreads = ParseInt("TenTransNumThreads");
  }

  long N;
  long Sweeps;
  long Dmin;
  long Dmax;
  double CutOff;
  double LanczErr;
  long MaxLanczIter;
  int TenTransNumThreads;
};


int main(int argc, char *argv[]) {
  CaseParams params(argv[1]);

  auto pb_out = Index({
                     QNSector(QN({QNNameVal("Sz", 1)}), 1),
                     QNSector(QN({QNNameVal("Sz", -1)}), 1)}, OUT);
  auto pb_in = InverseIndex(pb_out);

  auto sz = GQTensor({pb_in, pb_out});
  auto sp = GQTensor({pb_in, pb_out});
  auto sm = GQTensor({pb_in, pb_out});
  auto id = GQTensor({pb_in, pb_out});
  sz({0, 0}) = 0.5;
  sz({1, 1}) = -0.5;
  sp({0, 1}) = 1;
  sm({1, 0}) = 1;

  auto zero_div = QN({QNNameVal("Sz", 0)});
  auto mpo_gen = MPOGenerator(params.N, pb_out, zero_div);
  for (long i = 0; i < params.N-1; ++i) {
    mpo_gen.AddTerm(1, {OpIdx(sz, i), OpIdx(sz, i+1)});
    mpo_gen.AddTerm(0.5, {OpIdx(sp, i), OpIdx(sm, i+1)});
    mpo_gen.AddTerm(0.5, {OpIdx(sm, i), OpIdx(sp, i+1)});
  }
  auto mpo = mpo_gen.Gen();

  std::vector<GQTensor *> mps(params.N);
  std::vector<long> stat_labs;
  for (int i = 0; i < params.N; ++i) { stat_labs.push_back(i % 2); }
  DirectStateInitMps(mps, stat_labs, pb_out, zero_div);

  auto sweep_params = SweepParams(
                          params.Sweeps,
                          params.Dmin,
                          params.Dmax,
                          params.CutOff,
                          true,
                          kTwoSiteAlgoWorkflowInitial,
                          LanczosParams(params.LanczErr, params.MaxLanczIter));

  gqten::GQTenSetTensorTransposeNumThreads(params.TenTransNumThreads);
  auto energy0 = TwoSiteAlgorithm(mps, mpo, sweep_params);

  std::cout << "E0/site: " << energy0 / params.N << std::endl;

  return 0;
}
```

Parameters JSON file.

```json
{
  "CaseParams": {
    "N": 20,
    "Sweeps": 10,
    "Dmin": 32,
    "Dmax": 32,
    "CutOff": 1e-8,
    "LanczErr": 1e-9,
    "MaxLanczIter": 100,
    "TenTransNumThreads": 4
  }
}
```

Finally do
```
./gqmps2_demo params.json
```

## TODO list

This TODO list is *not* sorted by expected completion order.

- Release N-body terms MPO generator.
- Finer workflow control for these MPS algorithms.
- Abstract MPS and MPO objects to specific classes.
- ...


## License

GraceQ/MPS2 is freely available under the [LGPL v3.0](https://www.gnu.org/licenses/lgpl-3.0.en.html) licence.


## How to cite

Cite GraceQ/MPS2 as
> "GraceQ/MPS2: A high-performance matrix product state(MPS) algorithms library based on GraceQ/tensor", https://github.com/gracequantum/mps2 .


## Acknowledgments

The author(s) highly acknowledge the following people, project(s) and organization(s) (sorted in alphabetical order):

ALPS project, Cheng Peng, Chunyu Sun, D. N. Sheng, Grace Song, Hong-Chen Jiang, itensor.org, Le Zhao, Shuo Yang, Thomas P. Devereaux, Wayne Zheng, Xiaoyu Dong, Yifan Jiang, Zheng-Yu Weng

You can not meet this library without anyone of them.