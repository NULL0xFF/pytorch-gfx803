# ROCm 5.7.1 + PyTorch 2.3.1 on RX 580 4G

This walkthrough provides step-by-step instructions for setting up the environment to install ROCm, build custom rocBLAS, compile PyTorch with related libraries, and test using a UI tool.

This project is heavily inspired by the following links. For a different perspective on running PyTorch with ROCm on GCN hardware, refer to:

- [nikos230/Run-Pytorch-with-AMD-Radeon-GPU](https://github.com/nikos230/Run-Pytorch-with-AMD-Radeon-GPU)
- [robertrosenbusch/gfx803_rocm](https://github.com/robertrosenbusch/gfx803_rocm)

## Setup

### Environment

#### Hardware Configuration

- Host

  |Category|Setting|
  |-|-|
  |CPU|AMD EPYC 7551P|
  |RAM|192 GiB|
  |OS|Proxmox VE 8.3.5|

- Container

  |Category|Setting|Info|
  |-|-|-|
  |CPU|64 Cores||
  |RAM|64 GiB||
  |Disk|12 GiB|/|
  |Disk|48 GiB|/opt|
  |Device|/dev/dri/card1|gid=44 (video)|
  |Device|/dev/dri/renderD128|gid=108 (render)|
  |Device|/dev/kfd|gid=108 (render)|
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

#### Miniconda 3

```shell
# Download and Install Miniconda 3
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -P /tmp/
sudo bash /tmp/Miniconda3-latest-Linux-x86_64.sh -b -p /opt/conda
sudo sed -i 's|^PATH="|PATH="/opt/conda/bin:|' /etc/environment

# Logout to apply environment
exit

# Logged in
conda init
source ~/.bashrc
conda create -n rocm-build -y python=3.12
conda activate rocm-build
```

#### Install Prerequisites

```shell
sudo apt install -y build-essential ccache git libjpeg-dev libjpeg-turbo8-dev libpng-dev libmsgpack-dev libssl-dev
```

### Install ROCm

#### ROCm 5.7.1

```shell
# AMDGPU Install Script
wget https://repo.radeon.com/amdgpu-install/5.7.1/ubuntu/jammy/amdgpu-install_5.7.50701-1_all.deb -P /tmp/
sudo apt install -y /tmp/amdgpu-install_5.7.50701-1_all.deb
sudo apt update

# Install ROCm 5.7.1
sudo amdgpu-install -y --usecase=rocm,rocmdev,hiplibsdk,mlsdk --no-dkms
sudo sed -i 's|^PATH="|PATH="/opt/rocm/bin:|' /etc/environment
echo "ROCM_PATH=/opt/rocm" | sudo tee -a /etc/environment

# Logout to apply environment
exit
```

#### rocBLAS 3.1.0

```shell
# Re-enable conda environment
conda activate rocm-build

# Download rocBLAS
sudo mkdir /opt/rocBLAS
sudo chown $LOGNAME: /opt/rocBLAS
git clone --recursive https://github.com/ROCm/rocBLAS.git -b rocm-5.7.1 /opt/rocBLAS
cd /opt/rocBLAS

# Install required packages
pip install "cmake>=3.18.0,<3.19" joblib pyyaml

# Build rocBLAS
time ./install.sh -a "gfx803"

# Copy compiled library
sudo rsync -vrh /opt/rocBLAS/build/release/rocblas-install/lib/rocblas/library/ /opt/rocm/lib/rocblas/library/
sudo cp /opt/rocBLAS/build/release/rocblas-install/lib/librocblas.so.3.1.0 /opt/rocm/lib/librocblas.so.3.1.0.50701

# Reboot
sudo systemctl reboot
```

```shell-session
real    7m21.633s
user    68m6.908s
sys     4m39.870s
```

```shell
# Verify if ROCm is installed correctly
rocminfo
clinfo
```

### Install PyTorch

#### PyTorch 2.3.1

```shell
# Re-enable conda environment
conda activate rocm-build

# Download PyTorch
sudo mkdir /opt/pytorch
sudo chown $LOGNAME: /opt/pytorch
git clone --recursive https://github.com/pytorch/pytorch.git -b v2.3.1 /opt/pytorch
cd /opt/pytorch

# Install required packages
pip install mkl mkl-include "numpy<2.0" ninja -r requirements.txt

# Build PyTorch
export PYTORCH_ROCM_ARCH=gfx803
export PYTORCH_BUILD_VERSION=2.3.1 PYTORCH_BUILD_NUMBER=1
python tools/amd_build/build_amd.py
time python setup.py bdist_wheel

# Install PyTorch
pip install /opt/pytorch/dist/torch-2.3.1-cp312-cp312-linux_x86_64.whl
```

```shell-session
real    57m20.601s
user    1255m48.793s
sys     73m34.033s
```

#### TorchVision 0.18.1

```shell
# Download TorchVision
sudo mkdir /opt/pytorch_vision
sudo chown $LOGNAME: /opt/pytorch_vision
git clone --recursive https://github.com/pytorch/vision.git -b v0.18.1 /opt/pytorch_vision
cd /opt/pytorch_vision

# Build TorchVision
export BUILD_VERSION=0.18.1
time python setup.py bdist_wheel

# Install TorchVision
pip install /opt/pytorch_vision/dist/torchvision-0.18.1-cp312-cp312-linux_x86_64.whl
```

```shell-session
real    0m45.532s
user    8m18.431s
sys     0m48.950s
```

#### TorchAudio 2.3.1

```shell
# Download TorchAudio
sudo mkdir /opt/pytorch_audio
sudo chown $LOGNAME: /opt/pytorch_audio
git clone --recursive https://github.com/pytorch/audio.git -b v2.3.1 /opt/pytorch_audio
cd /opt/pytorch_audio

# Build TorchAudio
export BUILD_VERSION=2.3.1
time python setup.py bdist_wheel

# Install TorchAudio
pip install /opt/pytorch_audio/dist/torchaudio-2.3.1-cp312-cp312-linux_x86_64.whl
```

```shell-session
real    1m11.653s
user    27m53.991s
sys     3m49.060s
```

#### Backup

```shell
# Backup
conda list --export > ~/package_list.txt
```

## Test

### ComfyUI

#### Setup and Launch

```shell
# Create and activate conda environment
conda create -n comfyui -y python=3.12
conda activate comfyui

# Download ComfyUI
sudo mkdir /opt/comfyui
sudo chown $LOGNAME: /opt/comfyui
git clone https://github.com/comfyanonymous/ComfyUI.git /opt/comfyui
cd /opt/comfyui

# Install PyTorch and requirements
mkdir .pre-built
cp /opt/pytorch*/dist/torch*.whl .pre-built/
pip install -r requirements.txt .pre-built/*.whl "numpy<2.0"

# Run ComfyUI
python main.py --listen --lowvram
```

#### Text to Image

> You can drag-and-drop images into ComfyUI to load workflow.

Test 1 (v1-5-pruned-emaonly-fp16, Batch size: 1)

![Image #1](./images/ComfyUI_2025-03-26_145336_00001_.png)

Output

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
loaded completely 2542.42451171875 1639.406135559082 True
100%|██████████| 20/20 [00:20<00:00,  1.01s/it]
Requested to load AutoencoderKL
loaded completely 1183.9994140625001 319.11416244506836 True
Prompt executed in 39.48 seconds
```

<details>

<summary>Test 2 (NoobAI-XL, Batch size: 4)</summary>

|||
|-|-|
|Batch #1|Batch #2|
|![Image #2](./images/ComfyUI_2025-03-26_142303_00001_.png)|![Image #3](./images/ComfyUI_2025-03-26_142303_00002_.png)|
|Batch #3|Batch #4|
|![Image #4](./images/ComfyUI_2025-03-26_142303_00003_.png)|![Image #5](./images/ComfyUI_2025-03-26_142303_00004_.png)|

Output

```shell-session
got prompt
Requested to load SDXL
got prompt
got prompt
got prompt
loaded partially 1511.9998046875 1511.998908996582 0
100%|██████████| 28/28 [03:40<00:00,  7.89s/it]
Requested to load AutoencoderKL
0 models unloaded.
loaded completely 1512.0 319.11416244506836 True
Prompt executed in 224.83 seconds
Requested to load SDXL
loaded partially 1511.9998046875 1511.998908996582 0
100%|██████████| 28/28 [03:40<00:00,  7.87s/it]
Requested to load AutoencoderKL
0 models unloaded.
loaded completely 1512.0 319.11416244506836 True
Prompt executed in 224.02 seconds
Requested to load SDXL
loaded partially 1511.9998046875 1511.998908996582 0
100%|██████████| 28/28 [03:40<00:00,  7.87s/it]
Requested to load AutoencoderKL
0 models unloaded.
loaded completely 1512.0 319.11416244506836 True
Prompt executed in 224.02 seconds
Requested to load SDXL
loaded partially 1511.9998046875 1511.998908996582 0
100%|██████████| 28/28 [03:40<00:00,  7.87s/it]
Requested to load AutoencoderKL
0 models unloaded.
loaded completely 1512.0 319.11416244506836 True
Prompt executed in 223.87 seconds
```

</details>
