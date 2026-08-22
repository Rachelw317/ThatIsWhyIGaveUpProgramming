# 在AutoDL中配置 PyTorch 2.11.0 + CUDA 12.8

## 省流

1. 从阿里云下载 CUDA 12.8 的 PyTorch wheel

```bash
curl -4 -L --retry 10 --retry-delay 2 -o 'torch-2.11.0+cu128-cp312-cp312-manylinux_2_28_x86_64.whl' 'https://mirrors.aliyun.com/pytorch-wheels/cu128/torch-2.11.0%2Bcu128-cp312-cp312-manylinux_2_28_x86_64.whl'
```

2. 安装到当前 .venv

```bash
python -m pip install './torch-2.11.0+cu128-cp312-cp312-manylinux_2_28_x86_64.whl'
```

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

## 问题一：清华源找不到 `torch==2.11.0+cu128`

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

所以不能直接通过清华 PyPI 安装 CUDA 12.8 专用 wheel。

---

## 问题二：PyTorch 官方源下载 `torch` 主 wheel 极慢

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

## 问题三：curl/wget/aria2 也出现持续降速

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

从 HTTP 响应可以看到：

```text
Server: AmazonS3
X-Cache: Hit from cloudfront
X-Amz-Cf-Pop: NRT57-P10
```

说明请求经过了 CloudFront。

最终没有继续和 `download.pytorch.org` 的这条链路死磕，而是寻找国内 PyTorch wheel 镜像。

---

## 解决方案：使用阿里云 PyTorch wheel 镜像

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

同时直接使用已经下载好的 wheel：

```bash
python -m pip install './torch-2.11.0+cu128-cp312-cp312-manylinux_2_28_x86_64.whl'
```

这样：

* `torch` 主 wheel 不需要重新下载
* pip 从当前配置的阿里云 PyPI 源下载其他 Python/CUDA 依赖

这次下载速度正常。
