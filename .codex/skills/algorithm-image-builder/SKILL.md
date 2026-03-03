---
name: algorithm-image-builder
description: 为 Python 或类似算法程序构建、整理和运行 Docker 算法镜像的通用技能。用于需要从零编写或优化 Dockerfile、梳理算法依赖、设计挂载运行方式、补齐 GitHub Actions 构建验证、生成可下载镜像包，或需要给出标准化构建步骤、运行步骤和注意事项时。
---

# 算法镜像构建

## 目标

- 为算法项目生成可复用、可验证、可发布的 Docker 镜像方案。
- 优先输出稳定的构建步骤、运行步骤和通用注意事项，而不是只给临时命令。
- 在不影响用户现有运行习惯的前提下，尽量兼容已有 `docker run` 命令。

## 通用构建步骤

1. 先确认入口脚本。
   常见入口是 `main.py`、`run.py`、`xxx_main.py`，也可能是用户给出的实际运行命令中的脚本路径。
2. 收集真实依赖。
   用 `rg "^(import|from) "` 扫描入口脚本及其直接引用模块，区分标准库、第三方包、项目内模块。
3. 区分三类依赖。
   第三方 Python 包：如 `numpy`、`pandas`、`netCDF4`。
   系统运行库：如 `libgfortran5`、`libeccodes0`。
   构建工具：如 `build-essential`、`gfortran`，仅放在 builder 阶段。
4. 使用多阶段构建。
   builder 阶段安装编译依赖并构建虚拟环境。
   runtime 阶段只复制虚拟环境和必要兼容文件，避免把编译工具带入最终镜像。
5. 固定脆弱依赖版本。
   对 `wrf-python`、`pygrib`、`h5py`、`scipy` 这类原生依赖，优先固定版本，并尽量选择有现成 wheel 的版本。
6. 优先构建纯运行时镜像。
   若用户说明源码会运行时挂载，则不要把业务源码、输入数据、临时输出打进镜像。
7. 兼容历史运行习惯。
   如果用户长期使用固定路径（例如 `/opt/python-3.10.13/bin/python` 或 `/mnt/...`），优先通过兼容包装脚本、软链或目录预创建保持命令形态不变。
8. 构建后做最小验证。
   至少执行一次 `docker build`，并在容器内做依赖导入验证。

## 推荐 Dockerfile 结构

1. 基础镜像。
   优先使用 `python:3.10-slim`、`python:3.11-slim` 或用户明确要求的基础镜像。
2. builder 阶段。
   安装 `build-essential`、`gfortran` 等构建工具。
   创建 `/opt/venv`。
   用固定版本的 `pip`、`setuptools`、`wheel` 安装依赖。
3. runtime 阶段。
   只安装运行时系统库。
   复制 `/opt/venv`。
   创建用户实际会使用的目录。
   如果要兼容旧命令，补充兼容路径，例如 `/opt/python-3.10.13/bin/python`。
4. 默认行为。
   如果用户习惯显式写出 `python 脚本 配置`，不要强制使用 `ENTRYPOINT` 接管。
   如果用户希望容器开箱即跑，可再单独加入口脚本。

## 通用运行步骤

1. 先构建镜像。

```bash
docker build -t your-image:tag .
```

2. 按用户真实目录挂载源码或工作目录。

```bash
docker run --rm \
  -v "$(pwd):/mnt/data2/DPS/WorkDir/EXE/" \
  your-image:tag \
  /opt/python-3.10.13/bin/python \
  /mnt/data2/DPS/WorkDir/EXE/YourProject/main.py \
  /mnt/data2/DPS/WorkDir/EXE/YourProject/config.json
```

3. 如果算法还依赖额外模型目录、输入数据目录或结果目录，再追加挂载。

```bash
docker run --rm \
  -v "$(pwd):/mnt/data2/DPS/WorkDir/EXE/" \
  -v /real/input:/data/input \
  -v /real/model:/data/model \
  your-image:tag \
  /opt/python-3.10.13/bin/python \
  /mnt/data2/DPS/WorkDir/EXE/YourProject/main.py \
  /mnt/data2/DPS/WorkDir/EXE/YourProject/config.json
```

4. 如果用户已有固定命令，优先保持命令结构不变，只替换镜像名。

## 验证步骤

1. 镜像构建验证。

```bash
docker build -t your-image:test .
```

2. Python 路径兼容验证。

```bash
docker run --rm your-image:test /opt/python-3.10.13/bin/python -c 'import sys; print(sys.executable)'
```

3. 依赖导入验证。

```bash
docker run --rm your-image:test /opt/python-3.10.13/bin/python -c 'import numpy, pandas, scipy'
```

4. 按真实运行命令做最小实测。
   不要求一次跑完整业务数据，但要至少推进到“真正读取配置或打开输入文件”这一步。

## 通用注意事项

- 先区分“镜像问题”和“数据问题”。
  `ModuleNotFoundError` 多半是镜像依赖缺失。
  `FileNotFoundError`、`PermissionError` 往往是挂载路径、配置路径或宿主机权限问题。
- 先区分“第三方包缺失”和“项目内模块缺失”。
  `import data_reader_sst` 这类报错，常常不是 `pip install` 能解决，而是代码文件根本没挂进容器。
- 不要只看 Dockerfile 静态内容。
  尽量实际运行一次容器，用真实命令验证。
- 对原生依赖优先选择 wheel。
  在 `arm64` 下，`h5py`、`pygrib`、`pyamg` 的源码编译成本高，也更容易失败。
- 避免无意义地改变用户习惯。
  用户已经有稳定命令时，优先兼容原命令，而不是强行改成新的入口方式。
- 谨慎处理默认配置路径。
  如果项目配置文件里仍然写着旧路径（例如 `/workspace`），即使镜像没问题，运行也会失败。
- 对 macOS 挂载路径做额外说明。
  Docker Desktop 只能挂载真实存在且已共享的宿主机路径，占位符路径不能直接运行。
- 发布镜像包时，默认使用 `docker save` 加 `.tar.gz`。
  除非用户明确要求，否则不要默认推送 Docker Hub。

## GitHub Actions 通用规则

- 仅做构建验证时，优先用普通 `docker build`。
- 多架构验证时，优先使用原生 Runner 分别构建，而不是默认依赖 `buildx`。
- 需要给用户下载镜像时，使用 `docker save` 导出 tar，再压缩为 `.tar.gz` 上传到 artifact 或 Release。
- 在工作流里只验证当前 Dockerfile，不混入与当前任务无关的发布逻辑。
## GitHub Actions 模板

- 仅做构建验证时，优先复用 `assets/github-actions-verify-native.yml`。
  这个模板会使用原生 Runner 分别验证 `amd64` 和 `arm64` 构建。
- 需要构建并发布可下载镜像包时，优先复用 `assets/github-actions-release-tarballs.yml`。
  这个模板会构建镜像、导出 `tar`、压缩为 `.tar.gz`，并在打标签时上传到 GitHub Release。
- 使用模板时，先替换其中的占位值。
  至少需要替换镜像名、触发分支、需要监听的路径，以及是否保留 `arm64` Runner。

## 输出要求

- 给出可直接执行的构建命令。
- 给出可直接执行的运行命令。
- 明确指出哪些路径是占位符，哪些路径必须替换成真实目录。
- 如果测试失败，明确说明失败是依赖缺失、路径不匹配、配置不匹配，还是权限问题。
