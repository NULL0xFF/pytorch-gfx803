# ROCm 6.1.5 + PyTorch 2.4.1 on RX 580 4G

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
  |Disk|40 GiB|/|
  |Disk|32 GiB|/opt|
  |Disk|40 GiB|/opt/rocm-6.1.5|
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

#### ROCm 6.1.5

```shell
# Setup AMD repository
wget https://repo.radeon.com/rocm/rocm.gpg.key -O - | gpg --dearmor | sudo tee /etc/apt/keyrings/rocm.gpg > /dev/null
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/rocm.gpg] https://repo.radeon.com/rocm/apt/6.1.5 jammy main" | sudo tee --append /etc/apt/sources.list.d/rocm.list
echo -e 'Package: *\nPin: release o=repo.radeon.com\nPin-Priority: 600' | sudo tee /etc/apt/preferences.d/rocm-pin-600
sudo apt update

# Install ROCm 6.1.5
sudo apt install rocm rocm-developer-tools rocm-ml-sdk rocm-ml-libraries rocm-hip-sdk rocm-hip-libraries
sudo sed -i 's|^PATH="|PATH="/opt/rocm/bin:|' /etc/environment
echo "ROCM_PATH=/opt/rocm" | sudo tee -a /etc/environment

# Logout to apply environment
exit
```

#### rocBLAS 4.0.0

```shell
# Re-enable conda environment
conda activate rocm-build

# Download rocBLAS
sudo mkdir /opt/rocBLAS
sudo chown $LOGNAME: /opt/rocBLAS
git clone --recursive https://github.com/ROCm/rocBLAS.git -b rocm-6.1.5 /opt/rocBLAS
cd /opt/rocBLAS

# Install required packages
pip install "cmake<4.0" joblib pyyaml

# Build rocBLAS
time ./install.sh -a "gfx803"

# Copy compiled library
sudo rsync -vrh /opt/rocBLAS/build/release/rocblas-install/lib/rocblas/library/ /opt/rocm/lib/rocblas/library/
sudo cp /opt/rocBLAS/build/release/rocblas-install/lib/librocblas.so.4.1 /opt/rocm/lib/librocblas.so.4.1.60105
```

#### rocSOLVER 3.25.0

```shell
# Download rocBLAS
sudo mkdir /opt/rocSOLVER
sudo chown $LOGNAME: /opt/rocSOLVER
git clone --recursive https://github.com/ROCm/rocSOLVER.git -b rocm-6.1.5 /opt/rocSOLVER
cd /opt/rocSOLVER

# Build rocBLAS
time ./install.sh -a "gfx803"

# Copy compiled library
sudo cp /opt/rocSOLVER/build/release/rocsolver-install/lib/librocsolver.so.0.1 /opt/rocm/lib/librocsolver.so.0.1.60105

# Reboot
sudo systemctl reboot
```

```shell
# Verify if ROCm is installed correctly
rocminfo
```

### Install PyTorch

#### PyTorch 2.4.1

```shell
# Re-enable conda environment
conda activate rocm-build

# Download PyTorch
sudo mkdir /opt/pytorch
sudo chown $LOGNAME: /opt/pytorch
git clone --recursive https://github.com/pytorch/pytorch.git -b v2.4.1 /opt/pytorch
cd /opt/pytorch

# Install required packages
pip install mkl-static mkl-include ninja -r requirements.txt

# Build PyTorch
export PYTORCH_ROCM_ARCH=gfx803
export PYTORCH_BUILD_VERSION=2.4.1 PYTORCH_BUILD_NUMBER=1
python tools/amd_build/build_amd.py
time python setup.py bdist_wheel

# Install PyTorch
pip install /opt/pytorch/dist/torch-2.4.1-cp312-cp312-linux_x86_64.whl
```

#### TorchVision 0.19.1

```shell
# Download TorchVision
sudo mkdir /opt/pytorch_vision
sudo chown $LOGNAME: /opt/pytorch_vision
git clone --recursive https://github.com/pytorch/vision.git -b v0.19.1 /opt/pytorch_vision
cd /opt/pytorch_vision

# Build TorchVision
export BUILD_VERSION=0.19.1
time python setup.py bdist_wheel

# Install TorchVision
pip install /opt/pytorch_vision/dist/torchvision-0.19.1-cp312-cp312-linux_x86_64.whl
```

#### TorchAudio 2.4.1

```shell
# Download TorchAudio
sudo mkdir /opt/pytorch_audio
sudo chown $LOGNAME: /opt/pytorch_audio
git clone --recursive https://github.com/pytorch/audio.git -b v2.4.1 /opt/pytorch_audio
cd /opt/pytorch_audio

# Build TorchAudio
export BUILD_VERSION=2.4.1
time python setup.py bdist_wheel

# Install TorchAudio
pip install /opt/pytorch_audio/dist/torchaudio-2.4.1-cp312-cp312-linux_x86_64.whl
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
pip install -r requirements.txt /opt/pytorch*/dist/torch*.whl

# Run ComfyUI
python main.py --listen --lowvram
```

#### Text to Image

> You can drag-and-drop images into ComfyUI to load workflow.
> Each test ran after **Free model and node cache**

Test 1 (v1-5-pruned-emaonly-fp16, Batch size: 1)

![Image #1](./images/ComfyUI_2025-04-04_110532_00001_.png)

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
100%|██████████| 20/20 [00:19<00:00,  1.03it/s]
Requested to load AutoencoderKL
loaded completely 1183.9994140625001 319.11416244506836 True
Prompt executed in 40.78 seconds
```

<details>

<summary>Test 2 (NoobAI-XL, Batch size: 4)</summary>

|||
|-|-|
|Batch #1|Batch #2|
|![Image #2](./images/ComfyUI_2025-04-04_110712_00001_.png)|![Image #3](./images/ComfyUI_2025-04-04_112054_00001_.png)|
|Batch #3|Batch #4|
|![Image #4](./images/ComfyUI_2025-04-04_112450_00001_.png)|![Image #5](./images/ComfyUI_2025-04-04_113012_00001_.png)|

Output

```shell-session
got prompt
model weight dtype torch.float16, manual cast: None
model_type V_PREDICTION
Using split attention in VAE
Using split attention in VAE
VAE load device: cuda:0, offload device: cpu, dtype: torch.float32
Requested to load SDXLClipModel
loaded completely 9.5367431640625e+25 1560.802734375 True
CLIP/text encoder model load device: cpu, offload device: cpu, current: cpu, dtype: torch.float16
Requested to load SDXLClipModel
Token indices sequence length is longer than the specified maximum sequence length for this model (125 > 77). Running this sequence through the model will result in indexing errors
Token indices sequence length is longer than the specified maximum sequence length for this model (125 > 77). Running this sequence through the model will result in indexing errors
Requested to load SDXL
loaded partially 1504.6498046875001 1504.6496658325195 0
100%|██████████| 28/28 [03:43<00:00,  7.99s/it]
Requested to load AutoencoderKL
0 models unloaded.
loaded completely 1504.65 319.11416244506836 True
Prompt executed in 808.69 seconds
got prompt
Requested to load SDXL
loaded partially 1504.6498046875001 1504.6496658325195 0
100%|██████████| 28/28 [03:40<00:00,  7.86s/it]
Requested to load AutoencoderKL
0 models unloaded.
loaded completely 1504.65 319.11416244506836 True
Prompt executed in 224.96 seconds
got prompt
Requested to load SDXL
loaded partially 1504.7998046875 1504.799201965332 0
100%|██████████| 28/28 [03:39<00:00,  7.83s/it]
Requested to load AutoencoderKL
0 models unloaded.
loaded completely 1504.8000000000002 319.11416244506836 True
Prompt executed in 222.69 seconds
got prompt
Requested to load SDXL
loaded partially 1504.7998046875 1504.799201965332 0
100%|██████████| 28/28 [03:39<00:00,  7.83s/it]
Requested to load AutoencoderKL
0 models unloaded.
loaded completely 1504.8000000000002 319.11416244506836 True
Prompt executed in 222.69 seconds
```

</details>
