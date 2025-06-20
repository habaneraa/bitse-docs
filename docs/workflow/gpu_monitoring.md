# 应对 GPU 服务器的过热问题

## 本文背景

使用老旧的 GPU 机器，散热能力下降，影响性能甚至出现过热现象。现象包括：

1. 通过 `nvidia-smi` 命令查看 GPU 温度，发现空载时温度显著高于环境温度（例如 40 度）
2. 运行比较计算密集的任务时，由于散热较差，GPU 温度压不到合理区间（例如超过 85 度）
3. GPU 过热后会触发硬件保护强行切换电源，进程退出且不能再使用 GPU

过热后可能会出现如下报错：

```text
Unable to determine the device handle for 0000:02:00.0: Unknown Error
```

这里的序号是总线标识符，和 `nvidia-smi` 命令中查到的 GPU 是对应的，从这里可以得知是哪张 GPU 过热掉电。

## 解决方案

临时解决方式：过热掉电后，等温度降下来后重启机器即可恢复。

妥协方案：调整 GPU 电源模式。

首先，在 root 权限下执行：

```bash
nvidia-smi -pm 1
```

这将开启 GPU 的 "persistence mode"。然后执行：

```bash
nvidia-smi -pl 100
```

这将限制 GPU 的电源功率，从功耗上减少散热，以达到控制温度的目的。这里的 100 代表目标功率，可以随意调整，功率有特定范围，例如 RTX 3090 的功率范围是 100W 到 350W。如果输入了不合法的功率值，会有命令行提示。

此外，如果多卡机器中不同 GPU 的散热条件不一样，可以通过如下命令单独修改一张卡的功率上限。这在特定卡散热不佳且仍然希望使用多卡训练时很有用。

```bash
nvidia-smi -pl 100 -i 0
```

最后，一劳永逸解决方式：花点 w 买新机器。

## GPU 状态及进程监控

众所周知，`nvidia-smi` 用于查看 GPU 运行信息，但是它并不美观，而且不能动态更新内容，如果想持续监测 GPU 状态，需要反复执行该命令。因此，我简单列举两个小工具，可以在一定程度上替代 `nvidia-smi` 。他们均基于 Python，因此推荐使用 pipx 安装。

```bash
apt install pipx
pipx ensurepath
pipx install gpustat
pipx install nvitop
```

[gpustat](https://github.com/wookayin/gpustat) 是一款轻量命令行工具，定位是 `nvidia-smi` 的替代品。在命令行直接运行 `gpustat` 即可输入简洁版的 GPU 状态信息，也可以使用 `gpustat --watch` 来自动监测。

[nvitop](https://nvitop.readthedocs.io/en/latest/) 是另一款命令行工具，提供类似于 htop 的终端 UI，功能比较强大。同样地，在命令行可使用 `nvitop` 启动它的 UI。它很适合在一段时间范围内监测 GPU 运行状态。

## 其他技巧

如果希望多卡进程只使用一张或几张卡，可以通过设置环境变量来让进程只使用你指定的 GPU。

```bash
export CUDA_VISIBLE_DEVICES=0,1
```

这里的显卡号就是 `nvidia-smi` 命令最左侧显示的 GPU ID，可用逗号分隔多个。

