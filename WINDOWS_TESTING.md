For your commit 953a0b9, I would validate as follows.

1. Run the portable-kernel and supporting C++ tests using MSVC.

   Use an x64 Visual Studio Developer PowerShell, so MSVC’s environment is initialized, then work from the repository root. You can use your existing venv directly without activating it:

   $py = (Resolve-Path .venv/Scripts/python.exe).Path

cmake -S . -B cmake-out-warning-tests -G Ninja `
  -DCMAKE_C_COMPILER=cl `
  -DCMAKE_CXX_COMPILER=cl `
  -DCMAKE_BUILD_TYPE=Release `
  "-DPYTHON_EXECUTABLE=$py" `
  -DEXECUTORCH_BUILD_TESTS=ON `
  -DEXECUTORCH_BUILD_EXTENSION_TENSOR=ON `
  -DEXECUTORCH_BUILD_EXTENSION_FLAT_TENSOR=ON `
  -DEXECUTORCH_BUILD_KERNELS_OPTIMIZED=OFF `
  -DEXECUTORCH_BUILD_KERNELS_QUANTIZED=OFF `
  -DEXECUTORCH_BUILD_XNNPACK=OFF

   Build the relevant test targets:

   cmake --build cmake-out-warning-tests --parallel 8 --target `
  portable_kernels_test `
  kernels_portable_cpu_util_test `
  extension_tensor_test `
  runtime_core_exec_aten_util_test `
  runtime_core_portable_type_test

   Run those executables through CTest:

   ctest --test-dir cmake-out-warning-tests `
  --output-on-failure --no-tests=error `
  -R '^(portable_kernels_test|kernels_portable_cpu_util_test|extension_tensor_test|runtime_core_exec_aten_util_test|runtime_core_portable_type_test)$'

   These commands are derived from this checkout’s CMake targets; I have not built or run them yet. They require the repository’s source dependencies/submodules to be populated.

   Building specific targets also avoids an existing Windows executor-test helper that unconditionally invokes Conda.
2. Run individual GoogleTest cases when investigating a failure.

   CTest sees portable_kernels_test as one test executable. GoogleTest provides finer filtering:

   .\cmake-out-warning-tests\kernels\test\portable_kernels_test.exe `
  --gtest_filter="OpGeluTest.*"

   To see the available individual cases:

   .\cmake-out-warning-tests\kernels\test\portable_kernels_test.exe `
  --gtest_list_tests

   For final validation, run the complete portable suite: your modified shared headers can affect operators beyond the files directly edited.
3. Treat the quantized C++ suite as a separate requirement.

   Its executable is quantized_kernels_test, enabled by:

   -DEXECUTORCH_BUILD_KERNELS_QUANTIZED=ON

   However, this checkout’s MSVC CI explicitly disables quantized kernels. Therefore I cannot claim this suite currently works with MSVC. Enabling it and building quantized_kernels_test is the direct way to determine the remaining compatibility problems.

   If it fails to compile, that is a validation infrastructure gap—not evidence that your changes broke kernel behavior. Under your constraints, resolving those MSVC build problems or providing target-side tests would be necessary to cover this part of the commit.
4. Rebuild and run the Cortex-M implementation tests on your existing FVP.

   The relevant tests include:

   & $py -m pytest -c backends/arm/test/pytest.ini `
  backends/cortex_m/test/ops/test_conv.py `
  backends/cortex_m/test/ops/test_linear.py `
  backends/cortex_m/test/ops/test_activation_quant.py `
  --cortex-m-target=cortex-m55 `
  -k implementation

   This assumes Python imports the checkout and the matching runner has been rebuilt.

   Use Cortex-M55 to exercise the MVE code you changed in op_quantize_per_tensor.cpp; an x64 host test cannot execute that code.

   The runner must contain your rebuilt C++ kernels. Its expected build directory is:

   arm_test/arm_semihosting_executor_runner_corstone-300_cortex-m55

   The existing shell build wrappers require a generated setup_path.sh. With your constraints, the native alternative is a direct CMake build from:

   examples/arm/executor_runner/standalone

   using your existing Arm cross-toolchain, SDK and local CMSIS-NN sources. This needs Windows path configuration; the repository does not provide a turnkey PowerShell equivalent of build_test_runner.sh.

   Also, this runner registers only a subset of portable operators. Passing its tests does not cover all the portable and quantized files in your commit.

Two details about your current environment matter: your venv imports ExecuTorch from site-packages, and pytest-timeout is absent. Before using pytest, ensure imports resolve to this checkout. If needed, install only the missing plugin with:

uv pip install --python $py pytest-timeout

For confidence in warning-related numerical changes, compare parent and patched commits using the same compiler, configuration and tests, then rebuild your embedded configuration with its original warning options. Passing host tests establishes useful functional coverage; the target build and FVP tests cover target-specific compilation and execution.

I added your Windows, MSVC, existing-toolchain, no-setup-script, and uv-only constraints to root AGENTS.md (/C:/Users/chrfav01/benchresults/executorch_matthias/AGENTS.md:3). No toolchains or Python packages were installed, and no validation suite has been run yet.

