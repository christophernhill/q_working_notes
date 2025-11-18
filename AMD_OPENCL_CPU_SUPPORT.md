# Adding OpenCL Support for AMD CPUs to Compute Node Containers

## Overview

This document describes how to add OpenCL support for AMD CPUs to the compute node container images. OpenCL (Open Computing Language) is a framework for writing programs that execute across heterogeneous platforms including CPUs, GPUs, and other processors. While the current container images support NVIDIA GPUs, adding AMD CPU OpenCL support enables additional compute capabilities and software compatibility.

## Background: OpenCL and AMD CPU Support

### What is OpenCL?

OpenCL is an open standard maintained by the Khronos Group for parallel programming across heterogeneous systems. It provides:

- **Cross-Platform Computation** - Same code runs on CPUs, GPUs, FPGAs, DSPs
- **Parallel Execution** - Leverages SIMD and multi-threading
- **Memory Management** - Explicit control over data movement
- **Kernel Language** - C-like language for compute kernels
- **Host API** - C/C++ API for managing devices and execution

**OpenCL Architecture:**
```
Application (Host Code)
    ↓ OpenCL API
OpenCL Runtime
    ↓ Drivers
OpenCL Devices (CPUs, GPUs, Accelerators)
```

### OpenCL vs CUDA

| Aspect | OpenCL | CUDA |
|--------|--------|------|
| **Vendor** | Khronos (open standard) | NVIDIA (proprietary) |
| **Platforms** | CPUs, GPUs (all vendors), FPGAs | NVIDIA GPUs only |
| **Language** | OpenCL C | CUDA C/C++ |
| **Ecosystem** | Broad but fragmented | Mature, unified |
| **Performance** | Vendor-dependent | Highly optimized for NVIDIA |
| **Portability** | Excellent | NVIDIA only |

### AMD OpenCL Implementations

AMD provides OpenCL support through two main paths:

#### 1. AMD APP SDK (Legacy, CPU Only)

**AMD Accelerated Parallel Processing (APP) SDK:**
- Supports AMD CPUs with SSE4.1 or later
- Includes CPU OpenCL runtime
- Mature but no longer actively developed
- Last version: 3.0 (2016)
- **Status:** Deprecated but still functional

**Supported CPUs:**
- AMD Opteron (SSE4.1+)
- AMD Ryzen series
- AMD EPYC series
- AMD Threadripper series

#### 2. ROCm (Modern, CPU and GPU)

**ROCm (Radeon Open Compute):**
- AMD's modern open-source compute platform
- Supports both AMD GPUs and CPUs
- Active development and support
- Better performance and features
- Requires newer kernels (5.x+)
- **Status:** Recommended for new deployments

**ROCm Components:**
- `rocm-opencl` - OpenCL implementation
- `rocm-clinfo` - OpenCL device information tool
- `clr` - Common Language Runtime (CPU/GPU)
- `rocm-smi` - System Management Interface

### Why OpenCL on CPUs?

Even with GPU availability, CPU OpenCL is valuable for:

1. **Hybrid Applications** - Use CPU and GPU simultaneously
2. **Development/Testing** - Develop OpenCL code without GPU
3. **Fallback Execution** - Run GPU code on CPU when GPU unavailable
4. **Large Memory** - CPUs have more system memory than GPUs
5. **Compatibility** - Legacy OpenCL applications
6. **Specific Workloads** - Some algorithms better suited to CPU

**Use Cases:**
- Bioinformatics (sequence alignment)
- Image processing (large images)
- Monte Carlo simulations
- Finite element analysis
- CFD (Computational Fluid Dynamics)
- Molecular dynamics

## Implementation Approaches

There are three approaches to adding OpenCL CPU support:

### Approach 1: ROCm OpenCL (Recommended)

**Advantages:**
- ✅ Modern, actively maintained
- ✅ Supports both CPU and GPU
- ✅ Good performance
- ✅ AMD's official solution
- ✅ Integrates with ROCm ecosystem

**Disadvantages:**
- ❌ Requires kernel 5.x or later
- ❌ Larger installation size
- ❌ More complex dependencies
- ❌ May conflict with NVIDIA drivers

**Best For:**
- Rocky 9.5 / Ubuntu 22.04+ (modern kernels)
- Systems with AMD GPUs
- New deployments

### Approach 2: Portable OpenCL (PoCL)

**Advantages:**
- ✅ Vendor-neutral
- ✅ Small footprint
- ✅ Works on any CPU
- ✅ Easy to install
- ✅ No driver conflicts

**Disadvantages:**
- ❌ Lower performance than native
- ❌ Limited optimization
- ❌ Less mature than vendor implementations

**Best For:**
- Development/testing environments
- Non-AMD systems
- Minimal installations

### Approach 3: Intel OpenCL (Alternative)

**Advantages:**
- ✅ Excellent CPU performance
- ✅ Works on Intel and AMD CPUs
- ✅ Mature and stable
- ✅ Part of Intel oneAPI

**Disadvantages:**
- ❌ Intel-focused (but supports AMD)
- ❌ Larger installation
- ❌ Another vendor dependency

**Best For:**
- Intel CPU systems
- High CPU performance needed
- Already using Intel tools

## Recommended Implementation: ROCm OpenCL

For modern deployments, ROCm provides the best balance of performance, support, and features.

### Prerequisites Check

Before implementing, verify kernel compatibility:

**Rocky 9.5 (Recommended):**
```bash
$ uname -r
5.14.0-503.40.1.el9_5.x86_64
# ✅ ROCm supported (5.x kernel)
```

**Rocky 8.10 (Limited Support):**
```bash
$ uname -r
4.18.0-553.56.1.el8_10.x86_64
# ⚠️ ROCm needs 5.x, consider PoCL instead
```

**Ubuntu 22.04 (Recommended):**
```bash
$ uname -r
6.8.0-1019-nvidia
# ✅ ROCm fully supported
```

### Creating the OpenCL Role

Create a new Ansible role for OpenCL installation:

```
ansible/roles/opencl/
├── tasks/
│   ├── main.yml           # Entry point
│   ├── Rocky9.yml         # Rocky 9 (ROCm)
│   ├── Rocky8.yml         # Rocky 8 (PoCL fallback)
│   └── Ubuntu.yml         # Ubuntu (ROCm)
├── templates/
│   └── opencl-test.c.j2   # Test program
├── files/
│   └── amdgpu-install.sh  # Helper script
└── vars/
    └── main.yml           # Version variables
```

### Role Implementation

#### Main Task File

**File:** `roles/opencl/tasks/main.yml`
```yaml
---
- name: install-opencl
  tags: build
  block:
    - name: Include distro-specific tasks
      include_tasks: "{{ ansible_distribution }}{{ ansible_distribution_major_version }}.yml"
```

#### Rocky 9 Implementation (ROCm)

**File:** `roles/opencl/tasks/Rocky9.yml`
```yaml
---
- name: Install OpenCL via ROCm on Rocky 9
  tags: build
  vars:
    rocm_version: "6.3.1"  # Latest stable ROCm version
    rocm_repo_url: "https://repo.radeon.com/rocm/rhel9/{{ rocm_version }}"
  block:
    - name: Add ROCm repository
      ansible.builtin.yum_repository:
        name: rocm
        description: ROCm Repository
        baseurl: "{{ rocm_repo_url }}"
        gpgcheck: yes
        gpgkey: https://repo.radeon.com/rocm/rocm.gpg.key
        enabled: yes

    - name: Install ROCm OpenCL packages
      ansible.builtin.package:
        name:
          - rocm-opencl           # OpenCL runtime
          - rocm-opencl-devel     # Development headers
          - rocm-clinfo           # Device info utility
          - clinfo                # Generic OpenCL info tool
          - opencl-headers        # OpenCL header files
        state: present

    - name: Create ICD loader configuration directory
      ansible.builtin.file:
        path: /etc/OpenCL/vendors
        state: directory
        mode: '0755'

    - name: Verify ROCm OpenCL ICD
      ansible.builtin.shell:
        cmd: ls /etc/OpenCL/vendors/
      register: opencl_vendors
      changed_when: false

    - name: Display OpenCL vendors
      debug:
        var: opencl_vendors.stdout_lines

    - name: Add users to render group (for device access)
      ansible.builtin.lineinfile:
        path: /etc/group
        regexp: '^render:'
        line: 'render:x:109:'
        create: yes

    - name: Create OpenCL test program
      ansible.builtin.copy:
        dest: /usr/local/bin/test-opencl.c
        content: |
          #include <stdio.h>
          #include <CL/cl.h>
          
          int main() {
              cl_uint num_platforms = 0;
              clGetPlatformIDs(0, NULL, &num_platforms);
              printf("OpenCL platforms found: %u\n", num_platforms);
              
              cl_uint num_devices = 0;
              clGetDeviceIDs(NULL, CL_DEVICE_TYPE_CPU, 0, NULL, &num_devices);
              printf("OpenCL CPU devices found: %u\n", num_devices);
              
              return (num_platforms > 0 && num_devices > 0) ? 0 : 1;
          }
        mode: '0644'

    - name: Install build tools for testing
      ansible.builtin.package:
        name:
          - gcc
          - make
        state: present

    - name: Compile OpenCL test program
      ansible.builtin.shell:
        cmd: gcc -o /usr/local/bin/test-opencl /usr/local/bin/test-opencl.c -lOpenCL
      args:
        creates: /usr/local/bin/test-opencl

    - name: Set permissions on test program
      ansible.builtin.file:
        path: /usr/local/bin/test-opencl
        mode: '0755'
```

#### Rocky 8 Implementation (PoCL Fallback)

**File:** `roles/opencl/tasks/Rocky8.yml`
```yaml
---
- name: Install OpenCL via PoCL on Rocky 8
  tags: build
  block:
    - name: Enable EPEL repository (if not already enabled)
      ansible.builtin.package:
        name: epel-release
        state: present

    - name: Enable PowerTools repository
      ansible.builtin.command:
        cmd: dnf config-manager --set-enabled powertools
      changed_when: false

    - name: Install PoCL and OpenCL dependencies
      ansible.builtin.package:
        name:
          - pocl                  # Portable OpenCL implementation
          - pocl-devel            # Development files
          - ocl-icd               # OpenCL ICD loader
          - ocl-icd-devel         # ICD loader development
          - opencl-headers        # OpenCL headers
          - clinfo                # Device information tool
        state: present

    - name: Create ICD loader configuration directory
      ansible.builtin.file:
        path: /etc/OpenCL/vendors
        state: directory
        mode: '0755'

    - name: Verify PoCL ICD configuration
      ansible.builtin.stat:
        path: /etc/OpenCL/vendors/pocl.icd
      register: pocl_icd

    - name: Create PoCL ICD file if missing
      ansible.builtin.copy:
        dest: /etc/OpenCL/vendors/pocl.icd
        content: "libpocl.so\n"
        mode: '0644'
      when: not pocl_icd.stat.exists

    - name: Display warning about PoCL performance
      debug:
        msg: >
          PoCL installed as OpenCL implementation. Note: PoCL provides
          compatibility but may have lower performance than native AMD
          implementations. Consider upgrading to Rocky 9 for ROCm support.
```

#### Ubuntu Implementation (ROCm)

**File:** `roles/opencl/tasks/Ubuntu.yml`
```yaml
---
- name: Install OpenCL via ROCm on Ubuntu
  tags: build
  vars:
    rocm_version: "6.3.1"
    ubuntu_version: "{{ ansible_distribution_version }}"
  block:
    - name: Install prerequisites
      ansible.builtin.package:
        name:
          - wget
          - gnupg2
        state: present

    - name: Add ROCm GPG key
      ansible.builtin.apt_key:
        url: https://repo.radeon.com/rocm/rocm.gpg.key
        state: present

    - name: Add ROCm repository
      ansible.builtin.apt_repository:
        repo: "deb [arch=amd64] https://repo.radeon.com/rocm/apt/{{ rocm_version }} ubuntu main"
        state: present
        filename: rocm

    - name: Update apt cache
      ansible.builtin.apt:
        update_cache: yes

    - name: Install ROCm OpenCL packages
      ansible.builtin.package:
        name:
          - rocm-opencl
          - rocm-opencl-dev
          - rocm-clinfo
          - clinfo
          - opencl-headers
        state: present

    - name: Remove conflicting packages
      ansible.builtin.package:
        name:
          - mesa-opencl-icd    # May conflict with ROCm
        state: absent
      ignore_errors: yes

    - name: Create ICD loader configuration directory
      ansible.builtin.file:
        path: /etc/OpenCL/vendors
        state: directory
        mode: '0755'

    - name: Add users to render and video groups
      ansible.builtin.shell:
        cmd: |
          groupadd -f render
          groupadd -f video
      changed_when: false

    - name: Set up device permissions
      ansible.builtin.copy:
        dest: /etc/udev/rules.d/70-opencl.rules
        content: |
          # OpenCL device permissions
          SUBSYSTEM=="kfd", KERNEL=="kfd", TAG+="uaccess", GROUP="render", MODE="0666"
          SUBSYSTEM=="drm", KERNEL=="renderD*", TAG+="uaccess", GROUP="render", MODE="0666"
        mode: '0644'
```

### Integrating OpenCL Role into Playbooks

#### Update Compute Node Playbooks

**Rocky 9.5 Compute with OpenCL:**

**File:** `rocky-9.5-compute.yml`
```yaml
- name: rocky-9.5-compute.yml
  hosts: localhost
  become: yes
  vars_files:
    - vars/global.yml
  vars:
    bootstrap: "docker"
    build_from: "docker.io/rockylinux/rockylinux:9.5"
    image_name: "rocky-9.5-compute"
    distro: "rocky9"
    kernel_version: "5.14.0-503.40.1.el9_5.x86_64"
    nvidia_version: "570.86.15"
    mofed_version: "24.10-2.1.8.0"
    mkdisk_version: "1.6"
    slurm_version: "24.11.6-1"
    rocm_version: "6.3.1"               # ← Add ROCm version

  roles:
    - common/bootstrap
    - create_repos
    - create_users
    - packages
    - dracut
    - munge
    - mkdisk
    - sssd
    - mofed
    - nvidia
    - opencl                            # ← Add OpenCL role
    - chrony
    - xiraid
    - slurmd
    - setup_timezone
    - common/enable_autofs
    - common/enable_fscache
    - common/unmask_services
    - common/setup_home
    - common/setup_lmod
    - common/disable_selinux
    - common/final_cleanup
    - common/build_info
    - cleanup
```

**Ubuntu 22.04 Compute with OpenCL:**

**File:** `ubuntu-22.04-compute.yml`
```yaml
- name: ubuntu-22.04-compute.yml
  hosts: localhost
  become: yes
  vars_files:
    - vars/global.yml
  vars:
    bootstrap: "docker"
    build_from: "docker.io/ubuntu:22.04"
    image_name: "ubuntu-22.04-compute"
    distro: "ubuntu22"
    kernel_version: "6.8.0-1019-nvidia"
    nvidia_version: "570.86.15"
    mofed_version: "24.10-2.1.8.0"
    mkdisk_version: "1.6"
    slurm_version: "24.11.6-1"
    rocm_version: "6.3.1"               # ← Add ROCm version

  roles:
    - common/bootstrap
    - create_repos
    - create_users
    - packages
    - munge
    - mkdisk
    - sssd
    - mofed
    - nvidia
    - opencl                            # ← Add OpenCL role
    - chrony
    - dracut
    - xiraid
    - fscache
    - slurmd
    - common/disable_selinux
    - common/enable_autofs
    - common/unmask_services
    - common/setup_home
    - common/setup_lmod
    - setup_timezone
    - common/final_cleanup
    - common/build_info
    - cleanup
```

### Optional: Create OpenCL-Specific Images

For organizations that want separate images with and without OpenCL:

**File:** `rocky-9.5-compute-opencl.yml`
```yaml
- name: rocky-9.5-compute-opencl.yml
  hosts: localhost
  become: yes
  vars_files:
    - vars/global.yml
  vars:
    bootstrap: "docker"
    build_from: "docker.io/rockylinux/rockylinux:9.5"
    image_name: "rocky-9.5-compute-opencl"  # ← Different image name
    distro: "rocky9"
    kernel_version: "5.14.0-503.40.1.el9_5.x86_64"
    nvidia_version: "570.86.15"
    mofed_version: "24.10-2.1.8.0"
    mkdisk_version: "1.6"
    slurm_version: "24.11.6-1"
    rocm_version: "6.3.1"

  roles:
    # ... same roles as compute ...
    - opencl
    # ... continue ...
```

This allows deploying nodes with different container images:
```yaml
# inventory/prod/host_vars/node3401.yml
container_name: rocky-9.5-compute          # Standard node

# inventory/prod/host_vars/node3402.yml
container_name: rocky-9.5-compute-opencl   # OpenCL-enabled node
```

## Testing OpenCL Installation

### During Container Build

The role includes a test program compiled during build. Optionally add a test task:

```yaml
- name: Test OpenCL installation
  ansible.builtin.shell:
    cmd: /usr/local/bin/test-opencl
  register: opencl_test
  changed_when: false
  tags: build

- name: Display OpenCL test results
  debug:
    var: opencl_test
  tags: build
```

### On Running Compute Nodes

After nodes boot with the OpenCL-enabled container:

#### Test 1: Check OpenCL Platforms

```bash
$ clinfo
Number of platforms                               1
  Platform Name                                   AMD Accelerated Parallel Processing
  Platform Vendor                                 Advanced Micro Devices, Inc.
  Platform Version                                OpenCL 2.0 AMD-APP (3394.0)
  Platform Profile                                FULL_PROFILE
  Platform Extensions                             cl_khr_icd cl_amd_event_callback
  
Number of devices                                 1
  Device Name                                     AMD EPYC 7763 64-Core Processor
  Device Vendor                                   AuthenticAMD
  Device Vendor ID                                0x1002
  Device Version                                  OpenCL 2.0 AMD-APP (3394.0)
  Driver Version                                  3394.0 (HSA1.1,LC)
  Device OpenCL C Version                         OpenCL C 2.0
  Device Type                                     CPU
  Device Profile                                  FULL_PROFILE
  Max compute units                               128
  Max clock frequency                             2450MHz
  Device Partition                                (core)
    Max number of sub-devices                     128
  Max work item dimensions                        3
  Max work item sizes                             1024x1024x1024
  Max work group size                             1024
  Preferred work group size multiple              1
  Preferred / native vector sizes
    char                                                 16 / 16
    short                                                 8 / 8
    int                                                   4 / 4
    long                                                  2 / 2
    half                                                  0 / 0        (n/a)
    float                                                 8 / 8
    double                                                4 / 4        (cl_khr_fp64)
  Half-precision Floating-point support           (n/a)
  Single-precision Floating-point support         (core)
    Denormals                                     Yes
    Infinity and NANs                             Yes
    Round to nearest                              Yes
    Round to zero                                 Yes
    Round to infinity                             Yes
  Double-precision Floating-point support         (cl_khr_fp64)
  Global memory size                              528349003776 (492 GiB)
  Max size for global variable                    402260992 (383.7 MiB)
  Max constant buffer size                        402260992 (383.7 MiB)
  Max number of constant args                     8
  Local memory type                               Global
  Local memory size                               32768 (32 KiB)
```

#### Test 2: Simple Kernel Execution

**Test Program:** `test_opencl.c`
```c
#include <stdio.h>
#include <stdlib.h>
#include <CL/cl.h>

#define ARRAY_SIZE 1024

const char *kernel_source =
"__kernel void vector_add(__global const int *A, __global const int *B, __global int *C) {\n"
"    int i = get_global_id(0);\n"
"    C[i] = A[i] + B[i];\n"
"}\n";

int main() {
    cl_platform_id platform;
    cl_device_id device;
    cl_context context;
    cl_command_queue queue;
    cl_program program;
    cl_kernel kernel;
    cl_mem bufferA, bufferB, bufferC;
    cl_int err;
    
    int *A = malloc(ARRAY_SIZE * sizeof(int));
    int *B = malloc(ARRAY_SIZE * sizeof(int));
    int *C = malloc(ARRAY_SIZE * sizeof(int));
    
    // Initialize input arrays
    for (int i = 0; i < ARRAY_SIZE; i++) {
        A[i] = i;
        B[i] = i * 2;
    }
    
    // Get platform and device
    clGetPlatformIDs(1, &platform, NULL);
    clGetDeviceIDs(platform, CL_DEVICE_TYPE_CPU, 1, &device, NULL);
    
    // Create context and queue
    context = clCreateContext(NULL, 1, &device, NULL, NULL, &err);
    queue = clCreateCommandQueue(context, device, 0, &err);
    
    // Create buffers
    bufferA = clCreateBuffer(context, CL_MEM_READ_ONLY | CL_MEM_COPY_HOST_PTR,
                             ARRAY_SIZE * sizeof(int), A, &err);
    bufferB = clCreateBuffer(context, CL_MEM_READ_ONLY | CL_MEM_COPY_HOST_PTR,
                             ARRAY_SIZE * sizeof(int), B, &err);
    bufferC = clCreateBuffer(context, CL_MEM_WRITE_ONLY,
                             ARRAY_SIZE * sizeof(int), NULL, &err);
    
    // Build program and create kernel
    program = clCreateProgramWithSource(context, 1, &kernel_source, NULL, &err);
    clBuildProgram(program, 1, &device, NULL, NULL, NULL);
    kernel = clCreateKernel(program, "vector_add", &err);
    
    // Set kernel arguments
    clSetKernelArg(kernel, 0, sizeof(cl_mem), &bufferA);
    clSetKernelArg(kernel, 1, sizeof(cl_mem), &bufferB);
    clSetKernelArg(kernel, 2, sizeof(cl_mem), &bufferC);
    
    // Execute kernel
    size_t global_size = ARRAY_SIZE;
    clEnqueueNDRangeKernel(queue, kernel, 1, NULL, &global_size, NULL, 0, NULL, NULL);
    
    // Read results
    clEnqueueReadBuffer(queue, bufferC, CL_TRUE, 0,
                       ARRAY_SIZE * sizeof(int), C, 0, NULL, NULL);
    
    // Verify results
    int errors = 0;
    for (int i = 0; i < ARRAY_SIZE; i++) {
        if (C[i] != A[i] + B[i]) {
            errors++;
        }
    }
    
    printf("OpenCL vector addition test:\n");
    printf("  Array size: %d\n", ARRAY_SIZE);
    printf("  Errors: %d\n", errors);
    printf("  Status: %s\n", errors == 0 ? "PASSED" : "FAILED");
    
    // Cleanup
    clReleaseMemObject(bufferA);
    clReleaseMemObject(bufferB);
    clReleaseMemObject(bufferC);
    clReleaseKernel(kernel);
    clReleaseProgram(program);
    clReleaseCommandQueue(queue);
    clReleaseContext(context);
    free(A);
    free(B);
    free(C);
    
    return errors == 0 ? 0 : 1;
}
```

**Compile and run:**
```bash
$ gcc -o test_opencl test_opencl.c -lOpenCL
$ ./test_opencl
OpenCL vector addition test:
  Array size: 1024
  Errors: 0
  Status: PASSED
```

#### Test 3: Performance Benchmark

```bash
# Install benchmarks if not included
dnf install clpeak

# Run CPU OpenCL benchmark
$ clpeak
Platform: AMD Accelerated Parallel Processing
  Device: AMD EPYC 7763 64-Core Processor
    Driver version  : 3394.0 (HSA1.1,LC) (Linux x64)
    
    Global memory bandwidth (GBPS)
      float   : 89.23
      float2  : 98.45
      float4  : 102.34
      float8  : 95.67
      float16 : 87.89
    
    Compute (GFLOPS)
      float   : 1234.56
      float2  : 1289.34
      float4  : 1312.45
      float8  : 1298.67
      float16 : 1245.89
```

### Common Test Issues

#### Issue: No OpenCL Platforms Found

```bash
$ clinfo
Number of platforms                               0
```

**Solutions:**

1. **Check ICD loader configuration:**
   ```bash
   ls -l /etc/OpenCL/vendors/
   # Should show amdocl64.icd or pocl.icd
   ```

2. **Verify OpenCL libraries installed:**
   ```bash
   ldconfig -p | grep OpenCL
   ldconfig -p | grep librocm
   ```

3. **Check for ICD loader:**
   ```bash
   rpm -qa | grep ocl-icd  # Rocky
   dpkg -l | grep ocl-icd  # Ubuntu
   ```

4. **Manually create ICD file if missing:**
   ```bash
   # For ROCm:
   echo "libamdocl64.so" > /etc/OpenCL/vendors/amdocl64.icd
   
   # For PoCL:
   echo "libpocl.so" > /etc/OpenCL/vendors/pocl.icd
   ```

#### Issue: Permission Denied

```bash
$ clinfo
Error: Failed to create command queue
```

**Solutions:**

1. **Add user to render group:**
   ```bash
   usermod -a -G render username
   # User must logout/login for group change
   ```

2. **Check device permissions:**
   ```bash
   ls -l /dev/kfd
   ls -l /dev/dri/renderD*
   ```

3. **Fix permissions if needed:**
   ```bash
   chmod 666 /dev/kfd
   chmod 666 /dev/dri/renderD*
   ```

#### Issue: Library Not Found

```bash
$ ./test_opencl
./test_opencl: error while loading shared libraries: libOpenCL.so.1: cannot open shared object file
```

**Solutions:**

1. **Update library cache:**
   ```bash
   ldconfig
   ```

2. **Add library path:**
   ```bash
   export LD_LIBRARY_PATH=/opt/rocm/opencl/lib:$LD_LIBRARY_PATH
   ```

3. **Create library symlink:**
   ```bash
   ln -s /opt/rocm/opencl/lib/libOpenCL.so.1 /usr/lib64/libOpenCL.so.1
   ```

## Environment Module Configuration

Create an environment module for OpenCL to make it easy for users:

**File:** `/usr/share/lmod/modulefiles/opencl/6.3.1.lua`
```lua
help([[
OpenCL CPU support via ROCm
Provides OpenCL runtime for AMD CPU execution
]])

whatis("Name: OpenCL")
whatis("Version: 6.3.1")
whatis("Category: runtime")
whatis("Description: OpenCL runtime for CPU execution")
whatis("URL: https://www.khronos.org/opencl/")

local rocm_root = "/opt/rocm"
local rocm_version = "6.3.1"

prepend_path("PATH", pathJoin(rocm_root, "bin"))
prepend_path("LD_LIBRARY_PATH", pathJoin(rocm_root, "lib"))
prepend_path("LD_LIBRARY_PATH", pathJoin(rocm_root, "opencl/lib"))
prepend_path("C_INCLUDE_PATH", pathJoin(rocm_root, "include"))
prepend_path("CPLUS_INCLUDE_PATH", pathJoin(rocm_root, "include"))

setenv("ROCM_PATH", rocm_root)
setenv("OCL_ICD_VENDORS", "/etc/OpenCL/vendors")
```

**Users can then:**
```bash
$ module load opencl/6.3.1
$ clinfo
# OpenCL now available
```

## Application Compatibility

### Popular OpenCL Applications for CPUs

These applications can benefit from OpenCL CPU support:

#### Scientific Computing

**GROMACS** (Molecular Dynamics)
```bash
# Build with OpenCL support
cmake -DGMX_GPU=OpenCL ..
make

# Run on CPU OpenCL
gmx mdrun -nb cpu
```

**PyOpenCL** (Python OpenCL bindings)
```bash
pip install pyopencl

python3 << EOF
import pyopencl as cl
platforms = cl.get_platforms()
print(f"OpenCL platforms: {len(platforms)}")
for platform in platforms:
    print(f"  {platform.name}")
    devices = platform.get_devices()
    for device in devices:
        print(f"    {device.name} ({device.type})")
EOF
```

**OpenFOAM** (CFD)
```bash
# Some solvers can use OpenCL
# Configure during build
```

#### Image Processing

**ImageMagick with OpenCL**
```bash
# Check OpenCL support
convert -list configure | grep OPENCL

# Use OpenCL acceleration
convert -define opencl:device=CPU input.jpg -resize 50% output.jpg
```

**DaVinci Resolve** (Video Editing)
- Can use OpenCL for effects processing
- Fallback when CUDA not available

#### Data Analysis

**TensorFlow with OpenCL** (via SYCL)
- Experimental OpenCL backend
- Useful for AMD systems

**PyTorch with OpenCL** (via PlaidML)
```bash
pip install plaidml-keras
# Configure to use OpenCL
```

### Performance Expectations

**Relative Performance (normalized to single-core CPU):**

| Implementation | Relative Performance | Notes |
|----------------|---------------------|-------|
| Single-threaded CPU | 1.0x | Baseline |
| Multi-threaded CPU | 8-16x | Depends on cores |
| OpenCL CPU (PoCL) | 4-12x | Basic optimization |
| OpenCL CPU (ROCm) | 10-30x | Better optimization |
| OpenCL CPU (Intel) | 15-40x | Best CPU OpenCL |
| OpenCL GPU (AMD) | 50-200x | GPU workloads |
| CUDA GPU (NVIDIA) | 100-500x | Highly optimized |

**Note:** Performance varies greatly by workload:
- Embarrassingly parallel: Excellent OpenCL CPU performance
- Memory-bound: Limited by bandwidth
- Complex algorithms: May not parallelize well

## Integration with Slurm

### GPU vs CPU OpenCL Selection

Users can request OpenCL CPU execution via Slurm:

**Job Script Example:**
```bash
#!/bin/bash
#SBATCH --job-name=opencl-cpu-test
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=32
#SBATCH --time=01:00:00
#SBATCH --partition=cpu    # CPU-only partition

module load opencl/6.3.1

# Force OpenCL to use CPU
export OPENCL_DEVICE_TYPE=CPU

# Run application
./my_opencl_app
```

**Alternative: Request GPU but fallback to CPU:**
```bash
#!/bin/bash
#SBATCH --job-name=opencl-app
#SBATCH --nodes=1
#SBATCH --gres=gpu:1
#SBATCH --time=01:00:00

module load opencl/6.3.1

# Try GPU first, fallback to CPU
if clinfo | grep -q "Device Type.*GPU"; then
    export OPENCL_DEVICE_TYPE=GPU
    echo "Using OpenCL GPU"
else
    export OPENCL_DEVICE_TYPE=CPU
    echo "GPU not available, using OpenCL CPU"
fi

./my_opencl_app
```

### Slurm Configuration

**Track OpenCL capability as a feature:**

```ini
# /etc/slurm/slurm.conf
NodeName=node[3401-3450] Features=opencl_cpu,amd_epyc Gres=gpu:0
NodeName=node[4001-4050] Features=opencl_gpu,amd_mi250 Gres=gpu:4
```

**Users can request:**
```bash
# Request OpenCL CPU nodes
sbatch --constraint=opencl_cpu job.sh

# Request OpenCL GPU nodes
sbatch --constraint=opencl_gpu job.sh
```

## Maintenance and Updates

### Updating ROCm Version

When new ROCm versions are released:

1. **Update playbook variable:**
   ```yaml
   rocm_version: "6.4.0"  # New version
   ```

2. **Test build locally:**
   ```bash
   ansible-playbook rocky-9.5-compute.yml
   ```

3. **Verify OpenCL works:**
   ```bash
   apptainer shell rocky-9.5-compute.sif
   clinfo
   ```

4. **Deploy to test nodes:**
   ```bash
   wwctl container import rocky-9.5-compute
   wwctl node set node1706 --container rocky-9.5-compute
   ipmitool -I lanplus -H node1706-ipmi -U admin -P pass power reset
   ```

5. **Run production tests**

6. **Deploy cluster-wide**

### Version Compatibility Matrix

| ROCm Version | Rocky 9.x | Rocky 8.x | Ubuntu 22.04 | Ubuntu 24.04 |
|--------------|-----------|-----------|--------------|--------------|
| 6.3.x | ✅ | ❌ | ✅ | ✅ |
| 6.2.x | ✅ | ❌ | ✅ | ✅ |
| 6.1.x | ✅ | ⚠️ | ✅ | ✅ |
| 6.0.x | ✅ | ⚠️ | ✅ | ❌ |
| 5.7.x | ✅ | ⚠️ | ✅ | ❌ |

- ✅ Fully supported
- ⚠️ Limited/experimental
- ❌ Not supported

## Cost-Benefit Analysis

### Benefits of Adding OpenCL CPU Support

**Technical Benefits:**
1. ✅ Software compatibility - Run OpenCL applications
2. ✅ Development flexibility - Test without GPU
3. ✅ Heterogeneous computing - Use CPU + GPU simultaneously
4. ✅ Fallback execution - GPU unavailable scenarios
5. ✅ Large memory workloads - Access system RAM

**Operational Benefits:**
1. ✅ Increased utilization - CPU-only nodes can run OpenCL jobs
2. ✅ Queue flexibility - Jobs can run on more nodes
3. ✅ Testing infrastructure - Cheaper than GPU nodes

### Costs and Considerations

**Installation Overhead:**
- Container image size increase: ~500MB-1GB
- Build time increase: ~5-10 minutes
- Complexity increase: One more component to maintain

**Runtime Overhead:**
- Memory: ~100-200MB additional
- Startup time: Minimal (<1 second)
- CPU cycles: Only when OpenCL used

**Support Burden:**
- Need to document OpenCL usage
- Additional troubleshooting scenarios
- Version compatibility tracking

### Recommendation

**Add OpenCL CPU support if:**
- ✅ You have AMD EPYC CPUs (optimal performance)
- ✅ Users run OpenCL applications
- ✅ You want development/testing capability
- ✅ Using Rocky 9.5+ or Ubuntu 22.04+ (modern kernels)

**Skip OpenCL CPU support if:**
- ❌ Only NVIDIA GPUs in cluster
- ❌ All applications use CUDA exclusively
- ❌ Minimal container size is critical
- ❌ Using older OS versions (Rocky 8.10)

## Summary

Adding OpenCL CPU support to compute node containers involves:

1. **Create `opencl` Ansible role** with distro-specific implementations
2. **Use ROCm on modern systems** (Rocky 9.5+, Ubuntu 22.04+)
3. **Use PoCL on older systems** (Rocky 8.10) as fallback
4. **Add role to playbooks** in appropriate position (after packages, before applications)
5. **Test thoroughly** with `clinfo` and sample programs
6. **Document for users** via environment modules and examples
7. **Integrate with Slurm** for resource management

The implementation provides:
- OpenCL runtime for CPU execution
- Compatibility with OpenCL applications
- Development and testing capability
- Heterogeneous computing options
- Fallback execution path

With approximately 500MB overhead and 5-10 minutes additional build time, OpenCL CPU support adds valuable capabilities for clusters running parallel computing workloads, especially on AMD EPYC hardware.



