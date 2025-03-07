# stack-gen

### What is stack-gen
`stack-gen` is a simple python tool that takes a series of templates for a stackinator `UENV` recipe and generates a "final" recipe rfom them based on options passed on the command line.

### Why might you need stack-gen
In some cases you may have a `UENV` recipe that has slight differences depending on which platform you are using, a simple example is OpenGL on gh200 machines (eg `daint`, GPU available), and on multi-core machine (`eiger`, no GPUs). On these two machines, the bulk of a recipe for a project will be the same, but you will want different libraries for rendering/display.
`stack-gen` allows you to create a single recipe with some choices that are switched between based on variables such as platform architecture, compiler, cluster name, mpi vendor, etc.

### Simple example
When building paraview on a GPU enabled cluster, we want to use `EGL` for graphics, but on a multi-core cluster, we want to use `osmesa`, your recipe for both machines might be the same apart from some entries such as ...

<table>
<tr>
<th>gh200</th>
<th>multicore</th>
</tr>
<tr>
<td>

```yaml
  variants:
  - build_type=Release
  - +mpi
  - +cxx
  - +python
  - +cuda
  - cuda_arch=90
  - ~x11
  - ^[virtuals=gl] egl
```
</td>
<td>

```yaml
  variants:
  - build_type=Release
  - +mpi
  - +cxx
  - +python
  - ~cuda

  - ~x11
  - ^[virtuals=gl] osmesa ^mesa +osmesa
```
</td>
</tr>
</table>

At some point, you wish to make a change to part of the basic recipe but you have to go to several files (one for each cluster and/or build type) and keep them synchronized so that wheen you update libraries/dependencies/options on one machine, you also update them on another.
`stack-gen` allows you to write a yaml template using the following syntax, where (in this trivial example) we provide 3 options for different machine architectures. Note how the `cuda_arch` variable can be changed between `gh200` and `turing` and omitted completely for `zen2`
```yaml
  variants:
  - build_type=Release
  - +mpi
  - +cxx
  - +python

  - arch=gh200:
   - +cuda
   - cuda_arch=90
   - ~x11
   - ^[virtuals=gl] egl

 - arch=zen2:
   - ~cuda
   - ~x11
   - ^[virtuals=gl] osmesa ^mesa +osmesa

 - arch=turing:
   - +cuda
   - cuda_arch=89
   - ~x11
   - ^[virtuals=gl] egl
```
The aim is to make project maintenance more convenient, by reducing duplication of material in different files and instead putting common options in a single place.
When running `stack-gen` with the `arch` variable set to one of the above settings, only the entries applicable for that `arch` will be included and the other parts dropped.

Another example is as follows - here we might want to build the same `UENV` with either GCC, or LLVM compilers but would normally need to duplicate the entire recipe with a few lines changed. `stack-gen` simplifies this - especially when the number of options increases.
```yaml
  compiler:
    compiler=gcc:
      - toolchain: gcc
        spec: gcc@12
    compiler=llvm:
      - toolchain: llvm
        spec: nvhpc
```
Drop through or default choices are supported, as in the following example which has nested options, so that we can first select things based on a choice of `MPI` (either mpich, or openmpi), but then make sub choices based on which cluster we are targeting.
Here we specify customizations for the clusters `eiger` and `oryx`, but provide a default `mpich` choice for all other clusters.
```yaml
  mpi:
    mpi=mpich:
      cluster=eiger:
        spec: cray-mpich@8.1.30
        depends: [libfabric@main]
      cluster=oryx:
        spec: mpich
        gpu: cuda
      cluster=else:
        spec: cray-mpich@8.1.30
        gpu: cuda
        depends: [libfabric@main]
    mpi=ompi:
      spec: openmpi@git.mpi-continue-5.0.6=main
      xspec: +continuations +internal-pmix fabrics=cma,ofi,xpmem schedulers=slurm +cray-xpmem
      gpu: null
      depends: ["libfabric@main"]
```
!!! warning
    "ELSE:"
    The category `xxx=else` must be the final entry in a list of options

### Invocation
By default, the `arch` variable is generated from the cluster name, so specifying `daint` automaticallly produces `gh200` and `eiger` produces `zen2`
```
python /path/to/stack-gen.py -m mpich -c gcc -C daint -r paraview -t ./templates -o ./5.13.2
python /path/to/stack-gen.py -m mpich -c gcc -C eiger -r paraview -t ./templates -o ./5.13.2
python /path/to/stack-gen.py -m ompi  -c gcc  -C eiger -r my-dev -t ./templates -o ./
python /path/to/stack-gen.py -m ompi  -c llvm -C daint -r my-dev -t ./templates -o ./
```
### Complete example
!!! 'note'
    Environments.yaml
```yaml
${ENVNAME}:

  views:
    develop:
      exclude: []
      uenv:
        add_compilers: true
        prefix_paths:
          LD_LIBRARY_PATH: [lib, lib64]

    paraview:
      link: run
      exclude: []
      uenv:
        add_compilers: true
        prefix_paths:
          LD_LIBRARY_PATH: [lib, lib64]

  unify: true

  compiler:
    compiler=gcc:
      - toolchain: gcc
        spec: gcc@12
    compiler=llvm:
      - toolchain: llvm
        spec: nvhpc
    compiler=gcc+llvm:
      - toolchain: gcc
        spec: gcc@12
      - toolchain: llvm
        spec: nvhpc

  mpi:
    mpi=mpich:
      cluster=eiger:
        spec: cray-mpich@8.1.30
        depends: [libfabric@main]
      cluster=oryx:
        spec: laptop-mpich
        gpu: cuda
      cluster=else:
        spec: cray-mpich@8.1.30
        gpu: cuda
        depends: [libfabric@main]
    mpi=ompi:
      spec: openmpi@git.mpi-continue-5.0.6=main
      xspec: +continuations +internal-pmix fabrics=cma,ofi,xpmem schedulers=slurm +cray-xpmem
      gpu: null
      depends: ["libfabric@main"]

  packages:
  - git
  - perl

  specs:
  # Build + repo tools
  - git-lfs
  - cmake
  - direnv
  - ninja
  - libtree
  - llvm@18 ~gold ~cuda

  # sys tools
  - hwloc
  - numactl

  # profiling/testing/logging
  - gperftools
  - googletest
  - protobuf
  - spdlog

  # maths
  - blaspp
  - eigen
  - fftw
  - lapackpp
  - openblas
  - proj

  # c++ libraries
  - fmt@10
  - stdexec@main
  - boost +atomic +chrono +container +context +coroutine +date_time +filesystem +graph +json +mpi +multithreaded +program_options +regex +serialization +shared +system +test +thread

  # IO and parallelism
  - hdf5 +mpi +cxx +hl +threadsafe +shared ~java
  - netcdf-c +mpi
  - adios2 +python +hdf5 ~zfp
  - h5hut@master
  - lz4

  # allocators/memory management
  - jemalloc
  - mimalloc

  # profiling/testing
  - gperftools
  - googletest

  # multithreading
  - tbb

  # in-situ support
  - libcatalyst +mpi +python

  # vtk external deps
  - cgns@4.4.0
  - double-conversion@3.3.0
  - gl2ps
  - glew
  - jpeg
  - jsoncpp
  - libharu
  - libtiff
  - nlohmann-json
  - libtheora@master
  - pugixml
  - pegtl
  - protobuf@:3.21
  - seacas ~fortran ~applications ~legacy ~tests ~x11
  - utf8cpp

  # climate/weather
  - cdi

  # raytracing in VTK/ParaView
  - ospray@3.2 ~mpi +denoiser +volumes ~apps ~glm
  - ispc@1.24
  - openvkl
  - embree
  - rkcommon
  - openimagedenoise

  # python
  - python@3.11
  - py-numpy
  - py-pandas
  - py-matplotlib
  - py-mpi4py
  - py-cftime
  - py-h5py

  - arch=gh200:
    # CUDA/HIP support
    - cuda
    - whip@main

    # Maths
    - nvpl-blas      threads=none
    - nvpl-lapack    threads=none
    - openblas

  - arch=zen2:
    # offscreen rendering
    - mesa         +osmesa
    - osmesa
    - glew

    # Maths
    - intel-oneapi-mkl

  - arch=turing:
    # CUDA
    - cuda
    - whip@main

    # Maths
    - openblas
    - intel-oneapi-mkl

  variants:
  - build_type=Release
  - cxxstd=17
  - ~fortran
  - ~examples
  - ~tests
  - ~testing
  - +mpi
  - +cxx
  - +python

  - arch=gh200:
    - +cuda
    - cuda_arch=90
    - ~x11
    - ^[virtuals=gl] egl

  - arch=zen2:
    - ~cuda
    - ~x11
    - ^[virtuals=gl] osmesa ^mesa +osmesa

  - arch=turing:
    - +cuda
    - cuda_arch=89
    - ~x11
    - ^[virtuals=gl] egl
```

!!! 'note'
    Packages.yaml
```yaml
packages:
  arch=gh200:
    all:
      providers:
        gl: [egl]
    egl:
      externals:
      - spec: egl@1.0.0
        prefix: /usr
      buildable: false
    llvm:
      require: llvm ~gold ~cuda

  arch=turing:
    all:
      providers:
        gl: [egl]
    egl:
      externals:
      - spec: egl@1.0.0
        prefix: /usr
      buildable: false
    llvm:
      require: llvm ~gold ~cuda

  arch=zen2:
    all:
      providers:
        gl: [osmesa]
    llvm:
      require: llvm ~gold ~cuda
```
!!! 'note'
    Compilers.yaml
```yaml
compiler=gcc:
  bootstrap:
    spec: gcc@12
  gcc:
    specs:
    - gcc@12

compiler=llvm:
  bootstrap:
    spec: gcc@11
  gcc:
    specs:
    - gcc@11
  llvm:
    requires: gcc@11
    specs:
    - llvm@13
```
