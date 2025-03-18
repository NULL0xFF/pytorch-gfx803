# Run ROCm 5.4.1 on RX 580 4G

## Setup

### Environment

#### Hardware Configuration

- Host

  |Category|Info|
  |-|-|
  |CPU|AMD EPYC 7551P 32C64T|
  |RAM|192 GiB|
  |OS|Proxmox VE 8.4.3|

- Container

  |Category|Value|Info|
  |-|-|-|
  |CPU|64 Cores||
  |RAM|64 GiB|+ Swap 8 GiB|
  |Disk|16 GiB|/|
  |Disk|32 GiB|/opt|
  |OS|Ubuntu 22.04|LXC|

#### Global Variables

```shell
# Add GFX803 related variables
echo "ROC_ENABLE_PRE_VEGA=1" | sudo tee -a /etc/environment
echo "HSA_OVERRIDE_GFX_VERSION=8.0.3" | sudo tee -a /etc/environment

# Update and Reboot
sudo apt update
sudo apt upgrade -y
sudo systemctl reboot
```

#### Install Prerequisites

```shell
sudo apt install -y build-essential ccache cmake curl git libmsgpack-dev libssl-dev ninja-build python3-venv python3-pip tmux
```

#### Miniconda 3

```shell
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -P /tmp/
bash /tmp/Miniconda3-latest-Linux-x86_64.sh
source ~/.bashrc
conda create -n rocm-5.4.1 python=3.12
conda activate rocm-5.4.1
```

### Install ROCm

#### ROCm 5.4.1

```shell
# AMDGPU Install Script
wget https://repo.radeon.com/amdgpu-install/5.4.1/ubuntu/jammy/amdgpu-install_5.4.50401-1_all.deb -P /tmp/
sudo apt install -y /tmp/amdgpu-install_5.4.50401-1_all.deb
sudo apt update

# Install ROCm 5.4.1
sudo amdgpu-install -y --usecase=rocm,rocmdev,hiplibsdk,mlsdk --no-dkms
```

#### rocBLAS 2.46.0

```shell
# Download rocBLAS
sudo mkdir /opt/rocBLAS
sudo chown $LOGNAME: /opt/rocBLAS
git clone https://github.com/ROCm/rocBLAS.git -b rocm-5.4.1 /opt/rocBLAS
cd /opt/rocBLAS

# Build rocBLAS
time ./install.sh --merge-architectures

# If you are planning to use only gfx803
# specify architecture for reducing build time
time ./install.sh -a "gfx803"

# Copy compiled library
sudo rsync -vrh /opt/rocBLAS/build/release/rocblas-install/lib/rocblas/library/ /opt/rocm/lib/rocblas/library/
sudo cp /opt/rocBLAS/build/release/rocblas-install/lib/librocblas.so.0.1 /opt/rocm/lib/librocblas.so.0.1.50401

# Reboot
sudo systemctl reboot
```

```shell-session
real    63m47.511s
user    772m40.120s
sys     79m46.432s
```

```shell
# Verify if ROCm is installed correctly
rocminfo
clinfo
```

<details>

<summary>Example Output</summary>

```shell-session
ubuntu@rocm:~$ rocminfo
ROCk module is loaded
=====================    
HSA System Attributes    
=====================    
Runtime Version:         1.1
System Timestamp Freq.:  1000.000000MHz
Sig. Max Wait Duration:  18446744073709551615 (0xFFFFFFFFFFFFFFFF) (timestamp count)
Machine Model:           LARGE                              
System Endianness:       LITTLE                             

==========               
HSA Agents               
==========               
*******                  
Agent 1                  
*******                  
  Name:                    AMD EPYC 7551P 32-Core Processor   
  Uuid:                    CPU-XX                             
  Marketing Name:          AMD EPYC 7551P 32-Core Processor   
  Vendor Name:             CPU                                
  Feature:                 None specified                     
  Profile:                 FULL_PROFILE                       
  Float Round Mode:        NEAR                               
  Max Queue Number:        0(0x0)                             
  Queue Min Size:          0(0x0)                             
  Queue Max Size:          0(0x0)                             
  Queue Type:              MULTI                              
  Node:                    0                                  
  Device Type:             CPU                                
  Cache Info:              
    L1:                      32768(0x8000) KB                   
  Chip ID:                 0(0x0)                             
  ASIC Revision:           0(0x0)                             
  Cacheline Size:          64(0x40)                           
  Max Clock Freq. (MHz):   2000                               
  BDFID:                   0                                  
  Internal Node ID:        0                                  
  Compute Unit:            16                                 
  SIMDs per CU:            0                                  
  Shader Engines:          0                                  
  Shader Arrs. per Eng.:   0                                  
  WatchPts on Addr. Ranges:1                                  
  Features:                None
  Pool Info:               
    Pool 1                   
      Segment:                 GLOBAL; FLAGS: FINE GRAINED        
      Size:                    32845624(0x1f52f38) KB             
      Allocatable:             TRUE                               
      Alloc Granule:           4KB                                
      Alloc Alignment:         4KB                                
      Accessible by all:       TRUE                               
    Pool 2                   
      Segment:                 GLOBAL; FLAGS: KERNARG, FINE GRAINED
      Size:                    32845624(0x1f52f38) KB             
      Allocatable:             TRUE                               
      Alloc Granule:           4KB                                
      Alloc Alignment:         4KB                                
      Accessible by all:       TRUE                               
    Pool 3                   
      Segment:                 GLOBAL; FLAGS: COARSE GRAINED      
      Size:                    32845624(0x1f52f38) KB             
      Allocatable:             TRUE                               
      Alloc Granule:           4KB                                
      Alloc Alignment:         4KB                                
      Accessible by all:       TRUE                               
  ISA Info:                
*******                  
Agent 2                  
*******                  
  Name:                    AMD EPYC 7551P 32-Core Processor   
  Uuid:                    CPU-XX                             
  Marketing Name:          AMD EPYC 7551P 32-Core Processor   
  Vendor Name:             CPU                                
  Feature:                 None specified                     
  Profile:                 FULL_PROFILE                       
  Float Round Mode:        NEAR                               
  Max Queue Number:        0(0x0)                             
  Queue Min Size:          0(0x0)                             
  Queue Max Size:          0(0x0)                             
  Queue Type:              MULTI                              
  Node:                    1                                  
  Device Type:             CPU                                
  Cache Info:              
    L1:                      32768(0x8000) KB                   
  Chip ID:                 0(0x0)                             
  ASIC Revision:           0(0x0)                             
  Cacheline Size:          64(0x40)                           
  Max Clock Freq. (MHz):   2000                               
  BDFID:                   0                                  
  Internal Node ID:        1                                  
  Compute Unit:            16                                 
  SIMDs per CU:            0                                  
  Shader Engines:          0                                  
  Shader Arrs. per Eng.:   0                                  
  WatchPts on Addr. Ranges:1                                  
  Features:                None
  Pool Info:               
    Pool 1                   
      Segment:                 GLOBAL; FLAGS: FINE GRAINED        
      Size:                    66054060(0x3efe7ac) KB             
      Allocatable:             TRUE                               
      Alloc Granule:           4KB                                
      Alloc Alignment:         4KB                                
      Accessible by all:       TRUE                               
    Pool 2                   
      Segment:                 GLOBAL; FLAGS: KERNARG, FINE GRAINED
      Size:                    66054060(0x3efe7ac) KB             
      Allocatable:             TRUE                               
      Alloc Granule:           4KB                                
      Alloc Alignment:         4KB                                
      Accessible by all:       TRUE                               
    Pool 3                   
      Segment:                 GLOBAL; FLAGS: COARSE GRAINED      
      Size:                    66054060(0x3efe7ac) KB             
      Allocatable:             TRUE                               
      Alloc Granule:           4KB                                
      Alloc Alignment:         4KB                                
      Accessible by all:       TRUE                               
  ISA Info:                
*******                  
Agent 3                  
*******                  
  Name:                    AMD EPYC 7551P 32-Core Processor   
  Uuid:                    CPU-XX                             
  Marketing Name:          AMD EPYC 7551P 32-Core Processor   
  Vendor Name:             CPU                                
  Feature:                 None specified                     
  Profile:                 FULL_PROFILE                       
  Float Round Mode:        NEAR                               
  Max Queue Number:        0(0x0)                             
  Queue Min Size:          0(0x0)                             
  Queue Max Size:          0(0x0)                             
  Queue Type:              MULTI                              
  Node:                    2                                  
  Device Type:             CPU                                
  Cache Info:              
    L1:                      32768(0x8000) KB                   
  Chip ID:                 0(0x0)                             
  ASIC Revision:           0(0x0)                             
  Cacheline Size:          64(0x40)                           
  Max Clock Freq. (MHz):   2000                               
  BDFID:                   0                                  
  Internal Node ID:        2                                  
  Compute Unit:            16                                 
  SIMDs per CU:            0                                  
  Shader Engines:          0                                  
  Shader Arrs. per Eng.:   0                                  
  WatchPts on Addr. Ranges:1                                  
  Features:                None
  Pool Info:               
    Pool 1                   
      Segment:                 GLOBAL; FLAGS: FINE GRAINED        
      Size:                    33023932(0x1f7e7bc) KB             
      Allocatable:             TRUE                               
      Alloc Granule:           4KB                                
      Alloc Alignment:         4KB                                
      Accessible by all:       TRUE                               
    Pool 2                   
      Segment:                 GLOBAL; FLAGS: KERNARG, FINE GRAINED
      Size:                    33023932(0x1f7e7bc) KB             
      Allocatable:             TRUE                               
      Alloc Granule:           4KB                                
      Alloc Alignment:         4KB                                
      Accessible by all:       TRUE                               
    Pool 3                   
      Segment:                 GLOBAL; FLAGS: COARSE GRAINED      
      Size:                    33023932(0x1f7e7bc) KB             
      Allocatable:             TRUE                               
      Alloc Granule:           4KB                                
      Alloc Alignment:         4KB                                
      Accessible by all:       TRUE                               
  ISA Info:                
*******                  
Agent 4                  
*******                  
  Name:                    AMD EPYC 7551P 32-Core Processor   
  Uuid:                    CPU-XX                             
  Marketing Name:          AMD EPYC 7551P 32-Core Processor   
  Vendor Name:             CPU                                
  Feature:                 None specified                     
  Profile:                 FULL_PROFILE                       
  Float Round Mode:        NEAR                               
  Max Queue Number:        0(0x0)                             
  Queue Min Size:          0(0x0)                             
  Queue Max Size:          0(0x0)                             
  Queue Type:              MULTI                              
  Node:                    3                                  
  Device Type:             CPU                                
  Cache Info:              
    L1:                      32768(0x8000) KB                   
  Chip ID:                 0(0x0)                             
  ASIC Revision:           0(0x0)                             
  Cacheline Size:          64(0x40)                           
  Max Clock Freq. (MHz):   2000                               
  BDFID:                   0                                  
  Internal Node ID:        3                                  
  Compute Unit:            16                                 
  SIMDs per CU:            0                                  
  Shader Engines:          0                                  
  Shader Arrs. per Eng.:   0                                  
  WatchPts on Addr. Ranges:1                                  
  Features:                None
  Pool Info:               
    Pool 1                   
      Segment:                 GLOBAL; FLAGS: FINE GRAINED        
      Size:                    66008956(0x3ef377c) KB             
      Allocatable:             TRUE                               
      Alloc Granule:           4KB                                
      Alloc Alignment:         4KB                                
      Accessible by all:       TRUE                               
    Pool 2                   
      Segment:                 GLOBAL; FLAGS: KERNARG, FINE GRAINED
      Size:                    66008956(0x3ef377c) KB             
      Allocatable:             TRUE                               
      Alloc Granule:           4KB                                
      Alloc Alignment:         4KB                                
      Accessible by all:       TRUE                               
    Pool 3                   
      Segment:                 GLOBAL; FLAGS: COARSE GRAINED      
      Size:                    66008956(0x3ef377c) KB             
      Allocatable:             TRUE                               
      Alloc Granule:           4KB                                
      Alloc Alignment:         4KB                                
      Accessible by all:       TRUE                               
  ISA Info:                
*******                  
Agent 5                  
*******                  
  Name:                    gfx803                             
  Uuid:                    GPU-XX                             
  Marketing Name:          Radeon RX 580 Series               
  Vendor Name:             AMD                                
  Feature:                 KERNEL_DISPATCH                    
  Profile:                 BASE_PROFILE                       
  Float Round Mode:        NEAR                               
  Max Queue Number:        128(0x80)                          
  Queue Min Size:          64(0x40)                           
  Queue Max Size:          131072(0x20000)                    
  Queue Type:              MULTI                              
  Node:                    4                                  
  Device Type:             GPU                                
  Cache Info:              
    L1:                      16(0x10) KB                        
  Chip ID:                 26591(0x67df)                      
  ASIC Revision:           1(0x1)                             
  Cacheline Size:          64(0x40)                           
  Max Clock Freq. (MHz):   1340                               
  BDFID:                   8704                               
  Internal Node ID:        4                                  
  Compute Unit:            36                                 
  SIMDs per CU:            4                                  
  Shader Engines:          4                                  
  Shader Arrs. per Eng.:   1                                  
  WatchPts on Addr. Ranges:4                                  
  Features:                KERNEL_DISPATCH 
  Fast F16 Operation:      TRUE                               
  Wavefront Size:          64(0x40)                           
  Workgroup Max Size:      1024(0x400)                        
  Workgroup Max Size per Dimension:
    x                        1024(0x400)                        
    y                        1024(0x400)                        
    z                        1024(0x400)                        
  Max Waves Per CU:        40(0x28)                           
  Max Work-item Per CU:    2560(0xa00)                        
  Grid Max Size:           4294967295(0xffffffff)             
  Grid Max Size per Dimension:
    x                        4294967295(0xffffffff)             
    y                        4294967295(0xffffffff)             
    z                        4294967295(0xffffffff)             
  Max fbarriers/Workgrp:   32                                 
  Pool Info:               
    Pool 1                   
      Segment:                 GLOBAL; FLAGS: COARSE GRAINED      
      Size:                    4194304(0x400000) KB               
      Allocatable:             TRUE                               
      Alloc Granule:           4KB                                
      Alloc Alignment:         4KB                                
      Accessible by all:       FALSE                              
    Pool 2                   
      Segment:                 GROUP                              
      Size:                    64(0x40) KB                        
      Allocatable:             FALSE                              
      Alloc Granule:           0KB                                
      Alloc Alignment:         0KB                                
      Accessible by all:       FALSE                              
  ISA Info:                
    ISA 1                    
      Name:                    amdgcn-amd-amdhsa--gfx803          
      Machine Models:          HSA_MACHINE_MODEL_LARGE            
      Profiles:                HSA_PROFILE_BASE                   
      Default Rounding Mode:   NEAR                               
      Default Rounding Mode:   NEAR                               
      Fast f16:                TRUE                               
      Workgroup Max Size:      1024(0x400)                        
      Workgroup Max Size per Dimension:
        x                        1024(0x400)                        
        y                        1024(0x400)                        
        z                        1024(0x400)                        
      Grid Max Size:           4294967295(0xffffffff)             
      Grid Max Size per Dimension:
        x                        4294967295(0xffffffff)             
        y                        4294967295(0xffffffff)             
        z                        4294967295(0xffffffff)             
      FBarrier Max Size:       32                                 
*** Done ***

ubuntu@rocm:~$ clinfo
Number of platforms:     1
  Platform Profile:     FULL_PROFILE
  Platform Version:     OpenCL 2.1 AMD-APP (3513.0)
  Platform Name:     AMD Accelerated Parallel Processing
  Platform Vendor:     Advanced Micro Devices, Inc.
  Platform Extensions:     cl_khr_icd cl_amd_event_callback 


  Platform Name:     AMD Accelerated Parallel Processing
Number of devices:     1
  Device Type:      CL_DEVICE_TYPE_GPU
  Vendor ID:      1002h
  Board name:      Radeon RX 580 Series
  Device Topology:     PCI[ B#34, D#0, F#0 ]
  Max compute units:     36
  Max work items dimensions:    3
    Max work items[0]:     1024
    Max work items[1]:     1024
    Max work items[2]:     1024
  Max work group size:     256
  Preferred vector width char:    4
  Preferred vector width short:    2
  Preferred vector width int:    1
  Preferred vector width long:    1
  Preferred vector width float:    1
  Preferred vector width double:   1
  Native vector width char:    4
  Native vector width short:    2
  Native vector width int:    1
  Native vector width long:    1
  Native vector width float:    1
  Native vector width double:    1
  Max clock frequency:     1340Mhz
  Address bits:      64
  Max memory allocation:    3650722200
  Image support:     Yes
  Max number of images read arguments:   128
  Max number of images write arguments:   8
  Max image 2D width:     16384
  Max image 2D height:     16384
  Max image 3D width:     16384
  Max image 3D height:     16384
  Max image 3D depth:     8192
  Max samplers within kernel:    26591
  Max size of kernel argument:    1024
  Alignment (bits) of base address:   1024
  Minimum alignment (bytes) for any datatype:  128
  Single precision floating point capability
    Denorms:      No
    Quiet NaNs:      Yes
    Round to nearest even:    Yes
    Round to zero:     Yes
    Round to +ve and infinity:    Yes
    IEEE754-2008 fused multiply-add:   Yes
  Cache type:      Read/Write
  Cache line size:     64
  Cache size:      16384
  Global memory size:     4294967296
  Constant buffer size:     3650722200
  Max number of constant args:    8
  Local memory type:     Scratchpad
  Local memory size:     65536
  Max pipe arguments:     16
  Max pipe active reservations:    16
  Max pipe packet size:     3650722200
  Max global variable size:    3650722200
  Max global variable preferred total size:  4294967296
  Max read/write image args:    64
  Max on device events:     1024
  Queue on device max size:    8388608
  Max on device queues:     1
  Queue on device preferred size:   262144
  SVM capabilities:     
    Coarse grain buffer:    Yes
    Fine grain buffer:     Yes
    Fine grain system:     No
    Atomics:      No
  Preferred platform atomic alignment:   0
  Preferred global atomic alignment:   0
  Preferred local atomic alignment:   0
  Kernel Preferred work group size multiple:  64
  Error correction support:    0
  Unified memory for Host and Device:   0
  Profiling timer resolution:    1
  Device endianess:     Little
  Available:      Yes
  Compiler available:     Yes
  Execution capabilities:     
    Execute OpenCL kernels:    Yes
    Execute native function:    No
  Queue on Host properties:     
    Out-of-Order:     No
    Profiling :      Yes
  Queue on Device properties:     
    Out-of-Order:     Yes
    Profiling :      Yes
  Platform ID:      0x7f859dbf0eb0
  Name:       gfx803
  Vendor:      Advanced Micro Devices, Inc.
  Device OpenCL C version:    OpenCL C 2.0 
  Driver version:     3513.0 (HSA1.1,LC)
  Profile:      FULL_PROFILE
  Version:      OpenCL 1.2 
  Extensions:      cl_khr_fp64 cl_khr_global_int32_base_atomics cl_khr_global_int32_extended_atomics cl_khr_local_int32_base_atomics cl_khr_local_int32_extended_atomics cl_khr_int64_base_atomics cl_khr_int64_extended_atomics cl_khr_3d_image_writes cl_khr_byte_addressable_store cl_khr_fp16 cl_khr_gl_sharing cl_amd_device_attribute_query cl_amd_media_ops cl_amd_media_ops2 cl_khr_image2d_from_buffer cl_khr_subgroups cl_khr_depth_images cl_amd_copy_buffer_p2p cl_amd_assembly_program
```

</details>

### Install PyTorch

#### PyTorch 2.1.1

```shell
# Download PyTorch
sudo mkdir /opt/pytorch
sudo chown $LOGNAME: /opt/pytorch
git clone --recursive https://github.com/pytorch/pytorch.git -b v2.1.1 /opt/pytorch
cd /opt/pytorch

# Install required packages
pip install -r requirements.txt "numpy<2.0"

# Build PyTorch
export PYTORCH_ROCM_ARCH=gfx803
export PYTORCH_BUILD_VERSION=2.1.1 PYTORCH_BUILD_NUMBER=1
python tools/amd_build/build_amd.py
time USE_ROCM=1 python setup.py bdist_wheel

# Install PyTorch
pip install /opt/pytorch/dist/torch-2.1.1-cp312-cp312-linux_x86_64.whl
```

```shell-session
real    22m26.395s
user    906m47.198s
sys     68m23.365s
```

#### TorchVision 0.16.1

```shell
# Download TorchVision
sudo mkdir /opt/pytorch_vision
sudo chown $LOGNAME: /opt/pytorch_vision
git clone --recursive https://github.com/pytorch/vision.git -b v0.16.1 /opt/pytorch_vision
cd /opt/pytorch_vision

# Install required packages
conda install libpng libjpeg-turbo -c pytorch
pip install flake8 typing mypy pytest pytest-mock scipy

# Build TorchVision
export BUILD_VERSION=0.16.1
time python setup.py bdist_wheel

# Install TorchVision
pip install /opt/pytorch_vision/dist/torchvision-0.16.1-cp312-cp312-linux_x86_64.whl
```

```shell-session
real    0m41.325s
user    7m21.693s
sys     0m48.703s
```

#### TorchAudio 2.1.1

```shell
# Download TorchAudio
sudo mkdir /opt/pytorch_audio
sudo chown $LOGNAME: /opt/pytorch_audio
git clone --recursive https://github.com/pytorch/audio.git -b v2.1.1 /opt/pytorch_audio
cd /opt/pytorch_audio

# Install required packages
conda install ninja

# Build TorchAudio
export BUILD_VERSION=2.1.1
time python setup.py bdist_wheel

# Install TorchAudio
pip install dist/torchaudio-2.1.1-cp312-cp312-linux_x86_64.whl
```

```shell-session
real    1m30.482s
user    27m33.686s
sys     3m14.386s
```

```shell
conda list --export > ~/package_list.txt
```

## Test

### ComfyUI

```shell
# Create and activate conda environment
conda create -n comfyui python=3.12
conda activate comfyui

# Download ComfyUI
sudo mkdir /opt/comfyui
sudo chown $LOGNAME: /opt/comfyui
git clone https://github.com/comfyanonymous/ComfyUI.git /opt/comfyui
cd /opt/comfyui

# Install PyTorch and requirements
pip install /opt/pytorch/dist/torch-2.1.1-cp312-cp312-linux_x86_64.whl
pip install /opt/pytorch_vision/dist/torchvision-0.16.1-cp312-cp312-linux_x86_64.whl
pip install /opt/pytorch_audio/dist/torchaudio-2.1.1-cp312-cp312-linux_x86_64.whl
pip install -r requirements.txt "numpy<2.0"

# Run ComfyUI
python main.py --listen --lowvram
```

```shell-session
got prompt
model weight dtype torch.float16, manual cast: None
model_type EPS
Using split attention in VAE
Using split attention in VAE
VAE load device: cuda:0, offload device: cpu, dtype: torch.float32
Requested to load SD1ClipModel
loaded completely 9.5367431640625e+25 235.84423828125 True
CLIP/text encoder model load device: cpu, offload device: cpu, current: cpu, dtype: torch.float16
Requested to load BaseModel
loaded completely 2876.79951171875 1639.406135559082 True
100%|███████████████████████████████████████████| 20/20 [00:29<00:00,  1.49s/it]
Requested to load AutoencoderKL
loaded completely 1175.8974609375 319.11416244506836 True
Prompt executed in 55.41 seconds
got prompt
loaded completely 2485.5258911132814 1639.406135559082 True
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 20/20 [00:20<00:00,  1.01s/it]
Prompt executed in 22.20 seconds
```

||ComfyUI_00001_.png|ComfyUI_00002_.png|
|-|-|-|
|Image|![First image of glass bottle generated by RX 580 using ComfyUI](./images/ComfyUI_00001_.png)|![Second image of glass bottle generated by RX 580 using ComfyUI](./images/ComfyUI_00002_.png)|
|Seed|156680208700286|1|
