# 在AutoDL实例中配置 PyTorch 2.11.0 + CUDA 12.8 安装问题总结

1. 从阿里云下载 CUDA 12.8 的 PyTorch wheel
curl -4 -L --retry 10 --retry-delay 2 -o 'torch-2.11.0+cu128-cp312-cp312-manylinux_2_28_x86_64.whl' 'https://mirrors.aliyun.com/pytorch-wheels/cu128/torch-2.11.0%2Bcu128-cp312-cp312-manylinux_2_28_x86_64.whl'
2. 安装到当前 .venv
python -m pip install './torch-2.11.0+cu128-cp312-cp312-manylinux_2_28_x86_64.whl'

## 1. 环境

目标环境：

* Ubuntu
* Python 3.12
* NVIDIA GeForce RTX 4090 D
* PyTorch 2.11.0
* CUDA 12.8

最终验证结果：

```text
PyTorch: 2.11.0+cu128
CUDA runtime: 12.8
CUDA available: True
GPU: NVIDIA GeForce RTX 4090 D
Capability: (8, 9)
GPU count: 1
```

说明 PyTorch + CUDA + GPU 已经正常工作。

---

## 2. 问题一：清华源找不到 `torch==2.11.0+cu128`

最开始尝试：

```bash
pip download torch==2.11.0+cu128 \
    --index-url https://pypi.tuna.tsinghua.edu.cn/simple
```

结果：

```text
ERROR: Could not find a version that satisfies the requirement torch==2.11.0+cu128
```

### 原因

清华 PyPI 镜像中虽然存在：

```text
torch 2.11.0
```

但没有同步：

```text
torch 2.11.0+cu128
```

也就是说：

```text
PyPI 镜像
└── torch 2.11.0          ✓
    torch 2.11.0+cu128    ✗
```

所以不能直接通过清华 PyPI 安装 CUDA 12.8 专用 wheel。

---

## 3. 问题二：PyTorch 官方源下载 `torch` 主 wheel 极慢

使用官方 CUDA 12.8 源：

```bash
pip install torch torchvision torchaudio \
    --index-url https://download.pytorch.org/whl/cu128
```

发现：

```text
torch-2.11.0+cu128
820.3 MB
```

下载速度只有：

```text
38.9 kB/s
```

但与此同时：

```text
nvidia-cudnn
16.1 MB/s

nvidia-cublas
16.3 MB/s

nvidia-nccl
16.6 MB/s
```

因此可以判断不是本机整体网络带宽不足，而是 `torch` 主 wheel 的下载链路存在问题。

---

## 4. 问题三：curl/wget/aria2 也出现持续降速

测试 `wget`：

```bash
wget -O /dev/null \
  --server-response \
  'https://download.pytorch.org/whl/cu128/torch-2.11.0%2Bcu128-cp312-cp312-manylinux_2_28_x86_64.whl'
```

开始能达到：

```text
1.56 MB/s
```

测试 `curl`：

```bash
curl -4 -L \
  -o /dev/null \
  -w '\n速度: %{speed_download} bytes/s\n' \
  'https://download.pytorch.org/whl/cu128/torch-2.11.0%2Bcu128-cp312-cp312-manylinux_2_28_x86_64.whl'
```

也出现过：

```text
2.82 MB/s
```

但完整下载过程中速度逐渐下降。

后来使用 aria2：

```bash
aria2c \
  -x 16 \
  -s 16 \
  -k 10M \
  --file-allocation=none \
  --continue=true \
  -o 'torch-2.11.0+cu128-cp312-cp312-manylinux_2_28_x86_64.whl' \
  'https://download.pytorch.org/whl/cu128/torch-2.11.0%2Bcu128-cp312-cp312-manylinux_2_28_x86_64.whl'
```

开始：

```text
DL: 6.5MiB/s
```

但随后逐渐变成：

```text
435KiB/s
276KiB/s
143KiB/s
32KiB/s
2.8KiB/s
0B
```

### 结论

这不是简单的 pip 配置问题。

从 HTTP 响应可以看到：

```text
Server: AmazonS3
X-Cache: Hit from cloudfront
X-Amz-Cf-Pop: NRT57-P10
```

说明请求经过了 CloudFront。

最终没有继续和 `download.pytorch.org` 的这条链路死磕，而是寻找国内 PyTorch wheel 镜像。

---

## 5. 解决方案：使用阿里云 PyTorch wheel 镜像

最终找到阿里云的 PyTorch CUDA 12.8 wheel 镜像。

目标 wheel：

```text
torch-2.11.0+cu128-cp312-cp312-manylinux_2_28_x86_64.whl
```

下载：

```bash
curl -4 -L --retry 10 --retry-delay 2 \
  -o 'torch-2.11.0+cu128-cp312-cp312-manylinux_2_28_x86_64.whl' \
  'https://mirrors.aliyun.com/pytorch-wheels/cu128/torch-2.11.0%2Bcu128-cp312-cp312-manylinux_2_28_x86_64.whl'
```

这样成功避开了 `download.pytorch.org` 的慢速链路。

---

## 6. 问题四：wheel 下载成功，但 `.venv` 里没有 torch

下载完成后第一次检查：

```bash
python -c "import torch; print(torch.__version__)"
```

结果：

```text
ModuleNotFoundError: No module named 'torch'
```

检查：

```bash
which python && which pip && python -m pip --version && python -m pip show torch
```

发现：

```text
/root/autodl-tmp/MyProjects/syntacticbootstrapping/.venv/bin/python
/root/autodl-tmp/MyProjects/syntacticbootstrapping/.venv/bin/pip
```

但：

```text
WARNING: Package(s) not found: torch
```

### 原因

之前只是**下载了 wheel**，没有把它安装到当前 `.venv`。

---

## 7. 从本地 wheel 安装 PyTorch

直接使用已经下载好的 wheel：

```bash
python -m pip install './torch-2.11.0+cu128-cp312-cp312-manylinux_2_28_x86_64.whl'
```

这样：

* `torch` 主 wheel 不需要重新下载
* pip 从当前配置的阿里云 PyPI 源下载其他 Python/CUDA 依赖

例如：

```text
nvidia-cudnn-cu12
nvidia-cublas-cu12
nvidia-nccl-cu12
nvidia-cusparse-cu12
nvidia-cusolver-cu12
...
```

这次下载速度正常。

---

## 9. 最终验证命令

验证当前 PyTorch/CUDA/GPU 环境，可以直接用这一条：

```bash
python -c "import torch; print('PyTorch:', torch.__version__); print('CUDA runtime:', torch.version.cuda); print('CUDA available:', torch.cuda.is_available()); print('GPU:', torch.cuda.get_device_name(0) if torch.cuda.is_available() else 'N/A'); print('Capability:', torch.cuda.get_device_capability(0) if torch.cuda.is_available() else 'N/A'); print('GPU count:', torch.cuda.device_count())"
```

最终得到：

```text
PyTorch: 2.11.0+cu128
CUDA runtime: 12.8
CUDA available: True
GPU: NVIDIA GeForce RTX 4090 D
Capability: (8, 9)
GPU count: 1
```


## 11. pip 缓存清理

查看 pip cache：

```bash
python -m pip cache dir
```

清空 pip cache：

```bash
python -m pip cache purge
```

这不会卸载已经安装的 PyTorch、CUDA 或其他 Python 包，只会删除 pip 的下载缓存。

---

## 12. 最终推荐的镜像策略

这次踩坑后，比较适合这个环境的策略是：

```text
普通 Python 包
        ↓
阿里云 / 清华 PyPI 镜像

PyTorch CUDA 专用 wheel
        ↓
国内 PyTorch wheel 镜像
        ↓
mirrors.aliyun.com/pytorch-wheels/cu128/

本地 wheel
        ↓
python -m pip install ./xxx.whl
```

而不是强制让：

```text
torch + cu128
        ↓
download.pytorch.org
```

因为这次实际证明，AutoDL 环境到 PyTorch 官方 CloudFront 的这条大文件下载链路非常不稳定。



**核心问题已经全部解决，现在 PyTorch + CUDA 12.8 + RTX 4090 D 已经正常工作。**
