### 1. 需求演变矩阵 (Requirement Evolution Matrix)

| 阶段 | 用户输入 / 故障现象 | AI 误区 / 缺失环节 | 核心根源识别 | 最终验证的解决方案 |
| --- | --- | --- | --- | --- |
| **1. 初始回退**<br>*(Initial Fallback)* | ONNX Runtime 因缺少 `libcublasLt.so.12` 降级回退至 `CPUExecutionProvider`。 | 误以为是通用的环境配置问题，建议了大而全的 Conda/pip 安装命令。 | WSL 终端在运行测试脚本时，即使底层缺少实际的 `.so` 二进制文件，仍会从 wheel 包的元数据中读取并列出可用 Provider。 | 定位当前激活的虚拟环境中编译库文件的精确物理路径。 |
| **2. 连续依赖缺失**<br>*(Sequential Gap)* | 解决 CUDA 12 缺失问题后，系统立刻抛出下游组件缺少 `libcufft.so.11` 的错误。 | 误以为运行时环境只需要统一的单一 CUDA 大版本即可正常驱动。 | 存在混合版本的依赖冲突。应用层的 Provider 封装需要旧版的 API 签名（`.so.11`），而底层环境则提供了新版组件。 | 针对目标应用的执行约束，精确追踪其所需的特定依赖版本。 |
| **3. 运行时版本覆盖**<br>*(Shadowed Version)* | 用户发现环境中实际存在的是 `nvidia/cu13/lib/libcufft.so.12`，而非所需的 11 版本。 | 误以为标准的 pip 安装会自动处理好对旧版本的向后兼容布局。 | 隐式安装的较新依赖版本（CUDA 13 wheel 组件）发生覆盖，破坏了应用程序原本的运行时边界。 | 显式降低并锁死 Python wheel 包版本，使其与 ONNX Runtime 的编译需求完全对齐。 |
| **4. 最后的链接器失败**<br>*(Final Linker Fail)* | 尽管配置了路径，执行时依然报 `libcudadart.so.12` 缺失错误。 | 误以为 NVIDIA 的 PIP 包会将所有共享对象文件（`.so`）放置在单一的统一文件夹下。 | 特定的运行时组件将编译好的二进制文件隔离在了更深的离散子目录中（例如 `nvidia/cuda_runtime/lib/`）。 | 使用严格的冒号分隔符（`:`）进行多路径绝对路径配置，将深层子目录直接暴露给动态链接器。 |

---

### 2. 核心技术标准 (The Core Technical Standard)

为了在 Windows Subsystem for Linux (WSL) 环境中实现 ONNX Runtime 硬件加速的确定性执行，运行时配置必须强制执行以下架构标准：

> **系统强制性规则：**
> 1. **动态链接器显式可见性：** 切勿依赖虚拟环境内部的自动路径寻址。运行时环境必须将每个包含已编译二进制文件（`.so` 文件）的独立子包目录显式映射到 `LD_LIBRARY_PATH`。
> 2. **WSL 互操作性驱动映射：** 为了将计算指令从 Linux 子系统直接路由到宿主机 Windows 的 GPU 内核，必须将虚拟驱动桥接边界（`/usr/lib/wsl/lib`）包含在动态链接器的搜索路径中。
> 3. **Provider 级别隔离：** 执行命令中必须显式限制目标 Provider，防止因加载未配置的推理引擎（如 TensorRT）而引发乱序初始化开销与报错。

#### 确定性环境技术规范

* **活动运行时子系统：** WSL2 (Ubuntu 后端)
* **应用框架：** `deface` (基于 ONNX Runtime 引擎)
* **目标核心计算栈：** 兼容 CUDA 12.x / cuDNN 9.x 的运行层
* **链接器搜索变量配置：**
  ```bash
  export LD_LIBRARY_PATH="$LD_LIBRARY_PATH"
  export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:$HOME/miniconda3/lib/python3.13/site-packages/nvidia/cublas/lib"
  export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:$HOME/miniconda3/lib/python3.13/site-packages/nvidia/cudnn/lib"
  export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:$HOME/miniconda3/lib/python3.13/site-packages/nvidia/cufft/lib"
  export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:$HOME/miniconda3/lib/python3.13/site-packages/nvidia/cuda_runtime/lib"
  export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:/usr/lib/wsl/lib"
  ```

---

### 3. 操作手册 (Procedural Runbook)

请严格按照以下步骤顺序来恢复终端任务、对齐依赖布局并初始化 GPU 执行 Provider：

#### 步骤 1：恢复终端前台挂起作业

如果文本编辑器或进程之前被 `Ctrl + Z` 等信号意外挂起，请恢复或清理该作业边界：

* 将挂起的进程调回前台执行：
  ```bash
  fg
  ```

* 或者，若该进程已无响应，通过其作业标识符安全将其强制杀死：
  ```bash
  kill -9 %1
  ```

#### 步骤 2：清理冲突框架并对齐运行时版本

卸载由于自动升级引入的、可能覆盖所需低版本库的全新大版本（如 CUDA 13）计算 wheel 包，然后进行精准的版本锁定部署：

```bash
# 彻底清除不匹配或不稳定的上游冲突组件
pip uninstall -y \
  onnxruntime \
  onnxruntime-gpu \
  nvidia-cublas-cu11 \
  nvidia-cublas-cu12 \
  nvidia-cufft-cu11 \
  nvidia-cufft-cu12 \
  nvidia-cufft-cu13

# 安装针对 CUDA 12 编译的定制化 ONNX Runtime GPU 软件包
pip install onnxruntime-gpu \
  --extra-index-url https://aiinfra.pkgs.visualstudio.com/PublicPackages/_packaging/onnxruntime-cuda-12/pypi/simple/

# 补齐精准对应的 CUDA 12 核心运行时依赖组件
pip install \
  nvidia-cuda-runtime-cu12 \
  nvidia-cublas-cu12 \
  nvidia-cufft-cu12 \
  nvidia-cudnn-cu12
```

#### 步骤 3：映射系统路径与子系统桥接边界

将具体的内部包文件夹直接暴露给动态链接器。使用系统绝对路径下的二进制工具（如 `/usr/bin/nano`）以避开虚拟环境引起的版本信息警告：

1. 打开用户环境变量配置文件：
   ```bash
   /usr/bin/nano ~/.bashrc
   ```

2. 在文件**最底部**一行，追加写入以下完整的、用冒号分隔的路径映射布局：
   ```bash
   export LD_LIBRARY_PATH="$LD_LIBRARY_PATH"
   export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:$HOME/miniconda3/lib/python3.13/site-packages/nvidia/cublas/lib"
   export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:$HOME/miniconda3/lib/python3.13/site-packages/nvidia/cudnn/lib"
   export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:$HOME/miniconda3/lib/python3.13/site-packages/nvidia/cufft/lib"
   export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:$HOME/miniconda3/lib/python3.13/site-packages/nvidia/cuda_runtime/lib"
   export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:/usr/lib/wsl/lib"
   ```

3. 保存并关闭文件（`Ctrl+O` -> 回车 -> `Ctrl+X`），随后刷新当前终端使配置立即生效：
   ```bash
   source ~/.bashrc
   ```

#### 步骤 4：使用显式硬件指令执行任务

调用应用进程，并在命令行中传入严格的执行提供者（Execution Provider）参数，强制跳过标准的自动寻找例程，直达 CUDA 核心：

```bash
deface "./path/to/target_video.mp4" \
  --backend onnxrt \
  --execution-provider CUDAExecutionProvider
```