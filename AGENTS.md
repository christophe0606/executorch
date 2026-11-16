CLAUDE.md

## Local development and validation constraints

- Work natively on Windows using PowerShell or Git Bash. Do not require or install Linux, WSL, virtual machines, or containers that require them.
- Use the existing MSVC toolchain for Windows host builds and tests. Do not install or require Clang, clang-cl, GCC, or MinGW for host validation.
- Embedded/FVP validation may use the already installed target toolchains and FVP tools. Reuse them; do not reinstall them or confuse target cross-compilers with host compiler requirements.
- Do not run `setup.sh`, toolchain bootstrap scripts, or other setup/install wrappers that reinstall existing tools. Use explicit build/test commands and configure paths to existing tools.
- Use the existing Python virtual environment created with `uv`. Use `uv` for any necessary Python dependency installation into that environment. Do not create a replacement environment, use Conda, or switch Python environment managers.
- Explain validation coverage and any Windows/MSVC limitations explicitly. Distinguish host C++ tests, Python/reference tests, and tests executing the changed C++ kernels on an FVP.
