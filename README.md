# GitHub Action automatic kernel driver compilation tool

This GitHub Actions workflow can help you compile Android kernel drivers online without the need for a local compilation environment. It is particularly suitable for solving problems such as inability to access Google source code repositories and complex local compilation environment configurations, and can complete the entire compilation process within 1 hour.

## Main functions

✅ Automatically pull the specified version of kernel source code from Google's official source code repository
✅ Integrate driver code into the kernel build system
✅ Automatically modify the build configuration to adapt the driver module
✅ Intelligently select the build method (based on the kernel version)
✅ One-click download of the compilation result (driver module + boot.img)

## Usage Guide

### 1. Preparation
1. Create a `code` directory in the repository and put your driver source code:
- All `.c` and `.h` source files
- `Makefile` file
2. Determine your device information:
- Kernel major version (Settings > About phone)
- Device architecture (arm64/x86, etc.)
- Driver module name (.ko file name defined in Makefile)

### 2. Configure workflow
Modify the following key parameters in the `.github/workflows/main.yml` file:

```yaml
name: Android14-6.1 # Change to your kernel version (format: Android<major version>-<kernel version>)

# ... Other parts remain unchanged ...

- name: Add module to GKI modules list
run: |
cd android-kernel
awk -i inplace '
# ... Script content ...
print " \"drivers/kerneldriver/rwProcMem_module.ko\"," # Change to your driver module name
# ... Script content ...
' common/modules.bzl

- name: Build kernel module
run: |
cd android-kernel
# Intelligent selection of build method
if [[ "${{ github.event.inputs.kernel_name }}" =~ Android(9|10|11) ]]; then
echo "Use the old version of the build system (Android 9-11)"
build/build.sh
else
echo "Use the new version of Bazel build system (Android 12+)"
tools/bazel run //common:kernel_aarch64_dist # Modify the architecture such as //common:kernel_x86_64_dist
fi

- name: Upload kerneldriver.ko
uses: actions/upload-artifact@v4.6.2
with:
name: kerneldriver
path: android-kernel/out/kernel_aarch64 # Modify to the corresponding architecture path
```

### 3. Key parameters that need to be modified

| Parameter location | Description | Example value |
|----------|----------|--------|
| `name` | Kernel version | `Android12-5.15` |
| Driver module name | Your driver file name | `my_driver.ko` |
| Build target | Device architecture | `//common:kernel_x86_64_dist` |
| Output path | Compilation result path | `android-kernel/out/kernel_x86_64` |

### 4. Start the build
1. Submit the code to the GitHub repository
2. Visit the Actions tab of the repository
3. Select the "Build Kernel Driver" workflow
4. Click the "Run workflow" button
5. Wait for about 45-60 minutes to complete the build

### 5. Get the results
After the build is complete:
1. Find the successful build on the Actions page
2. Download the "kerneldriver" compressed package
3. Unzip and get:
- Compiled driver module (.ko)
- Complete kernel image (boot.img)
- Build log

## Technical details

### Intelligent build system selection
```bash
# Automatically select the build system based on the kernel version
if [[ "${{ github.event.inputs.kernel_name }}" =~ Android(9|10|11) ]]; then
build/build.sh # Android 9-11 uses the traditional build system
else
tools/bazel run //common:kernel_aarch64_dist # Android 12+ uses Bazel
fi
```

### Automatically integrate drivers into the kernel
```bash
# Copy driver code to the kernel source tree
cp -r ../kerneldriver common/drivers

# Add to kernel build system
echo "obj-y += kerneldriver/" >> common/drivers/Makefile

# Register as GKI module
awk -i inplace '
# Insert new driver at the appropriate position in the module list
# Keep alphabetical order to avoid compilation errors
' common/modules.bzl
```

### Compilation environment configuration
```bash
# Solve common compilation errors
find . -type f -name "Makefile*" -exec sed -i \
's/-Wframe-larger-than=[0-9]*/-Wframe-larger-than=4096/g' {} +

# Install the required toolchain
sudo apt-get install -y build-essential flex bison \
libssl-dev libelf-dev bc
```

## Common Problem Solving

**Q: Compilation fails with "Module not found in GKI list"**
A: Make sure the driver file name in the `Add module to GKI modules list` step is exactly the same as that in the Makefile

**Q: Downloaded boot.img cannot be started after flashing**
A: Check whether the device architecture configuration is correct, especially:
- The architecture in `tools/bazel run //common:kernel_xxx_dist`
- The upload path `android-kernel/out/kernel_xxx`

**Q: The compilation process is stuck at repo sync**
A: This may happen when the GitHub server access to the Google source code library is unstable. You can try:
1. Rerun the workflow
2. Use a mirror source (need to modify the URL of repo init)

**Q: How to support older versions of Android (9-11)?**
A: The workflow has been automatically processed:
- Android 9-11: Use traditional build.sh
- Android 12+: Use Bazel build system

## Contributions and improvements

Welcome to submit issues or PRs to improve this project:
- Add more device architecture support
- Optimize build speed
- Increase mirror source selection
- Improve error handling mechanism

## License

This project adopts the [MIT License](LICENSE) and can be freely used for personal and commercial projects.
