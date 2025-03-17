# Run ROCm 5.4.1 on RX 580 4G with PyTorch 2.1.1

## Setup

### Environment

```shell
echo "ROC_ENABLE_PRE_VEGA=1" | sudo tee -a /etc/environment
echo "HSA_OVERRIDE_GFX_VERSION=8.0.3" | sudo tee -a /etc/environment
sudo systemctl reboot
```

### ROCm 5.4.1

#### Install AMD ROCm 5.4.1

```shell
wget https://repo.radeon.com/amdgpu-install/5.4.1/ubuntu/focal/amdgpu-install_5.4.50401-1_all.deb -P /tmp/
sudo apt install /tmp/amdgpu-install_5.4.50401-1_all.deb 
sudo apt update
sudo amdgpu install --usecase=rocm,hiplibsdk,mlsdk --no-dkms
```

#### Install [rocBLAS](https://github.com/xuhuisheng/rocm-gfx803/releases/tag/rocm541) compiled by [xuhuisheng](https://github.com/xuhuisheng/rocm-gfx803)

```shell
wget "https://github.com/xuhuisheng/rocm-gfx803/releases/download/rocm541/rocblas_2.46.0.50401-84.20.04_amd64.deb" -P /tmp/
sudo apt install /tmp/rocblas_2.46.0.50401-84.20.04_amd64.deb
sudo apt-mark hold rocblas
```

### Miniconda 3

```shell
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -P /tmp/
bash /tmp/Miniconda3-latest-Linux-x86_64.sh
source ~/.bashrc
```

### Build Environment

```shell
sudo apt install build-essential curl git libssl-dev
```

### CMake 3.18.6

#### Download CMake Source Code

```shell
sudo mkdir /opt/cmake
sudo chown $LOGNAME: /opt/cmake
git clone https://gitlab.kitware.com/cmake/cmake.git -b v3.18.6 /opt/cmake
```

#### Build and Install CMake

```shell
cd /opt/cmake
bash ./bootstrap
time make -j $(nproc)
sudo make install
cmake --version
```

#### Output

```shell-session
real    0m53.911s
user    23m49.459s
sys     2m50.737s

cmake version 3.18.6

CMake suite maintained and supported by Kitware (kitware.com/cmake).
```

### PyTorch 2.1.1

#### Download PyTorch

```shell
sudo mkdir /opt/pytorch
sudo chown $LOGNAME: /opt/pytorch
git clone --recursive https://github.com/pytorch/pytorch.git -b v2.1.1 /opt/pytorch
```

#### Setup Conda Environment

```shell
cd /opt/pytorch
conda create -n pytorch-build python=3.10
conda activate pytorch-build
pip install -r requirements.txt "numpy<2.0"
```

#### Build and Install PyTorch

```shell
export PATH=/opt/rocm/bin:$PATH ROCM_PATH=/opt/rocm HIP_PATH=/opt/rocm/hip
export PYTORCH_ROCM_ARCH=gfx803
export PYTORCH_BUILD_VERSION=2.1.1 PYTORCH_BUILD_NUMBER=1
python tools/amd_build/build_amd.py
time USE_ROCM=1 python setup.py bdist_wheel
pip install /opt/pytorch/dist/torch-2.1.1-cp310-cp310-linux_x86_64.whl
```

#### Output

```shell-session
real    30m29.837s
user    1005m39.444s
sys     72m58.383s
```

### TorchVision 0.16.1

#### Download TorchVision
```shell
sudo mkdir /opt/vision
sudo chown $LOGNAME: /opt/vision
git clone --recursive https://github.com/pytorch/vision.git -b v0.16.1 /opt/vision
```

#### Build and Install TorchVision

```shell
cd /opt/vision
time python setup.py bdist_wheel
pip install /opt/vision/dist/torchvision-0.16.1+fdea156-cp310-cp310-linux_x86_64.whl
```

#### Output

```shell-session
real    6m16.110s
user    5m49.001s
sys     0m35.238s
```

### TorchAudio

#### Download TorchAudio

```shell
sudo mkdir /opt/audio
sudo chown $LOGNAME: /opt/audio
git clone --recursive https://github.com/pytorch/audio.git -b v2.1.1 /opt/audio
```

#### Build and Install TorchAudio

```shell
cd /opt/audio
conda install ninja
time python setup.py bdist_wheel
pip install /opt/audio/dist/torchaudio-2.1.1+db62484-cp310-cp310-linux_x86_64.whl
```

#### Output

```shell-session
real    1m30.482s
user    27m33.686s
sys     3m14.386s
```

### Save Conda

```shell
conda list --export > ~/package_list.txt
```