Fork of https://github.com/alugowski/fast_matrix_market.git

For CombBLAS single MPI process matrix market I/O use. 


```bash
#jobname initenv

# on NERSC
# export SPKERNELS_DIR=/Users/hongy0a/Documents/GitHub
# export FMM_DIR=/Users/hongy0a/Documents/GitHub/fast_matrix_market
# export CMAKE_C_COMPILER=cc
# export CMAKE_CXX_COMPILER=CC
# export CMAKE_CUDA_COMPILER=${which nvcc}
export SPKERNELS_DIR=/Users/hongy0a/Documents/GitHub
export FMM_DIR=/Users/hongy0a/Documents/GitHub/fast_matrix_market
export CMAKE_C_COMPILER=/opt/homebrew/Cellar/llvm/17.0.6_1/bin/clang
export CMAKE_CXX_COMPILER=/opt/homebrew/Cellar/llvm/17.0.6_1/bin/clang++
```


```bash
#jobname config
rm -rf ${FMM_DIR}/build 
cmake \
-S${FMM_DIR} \
-B${FMM_DIR}/build \
-DCMAKE_INSTALL_PREFIX=${SPKERNELS_DIR}/install -DINSTALLPROJECT=ON 
```


```bash
#jobname build-install
#dep config
cmake --build ${FMM_DIR}/build -- -j 
cmake --install ${FMM_DIR}/build 
```