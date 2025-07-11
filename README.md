# GitHub Action 自动内核驱动编译工具

这个 GitHub Actions 工作流可以帮助你在线编译 Android 内核驱动程序，无需本地编译环境。特别适合解决无法访问 Google 源码仓库、本地编译环境配置复杂等问题，能在 1 小时内完成完整编译过程。

## 主要功能

✅ 自动从 Google 官方源码仓库拉取指定版本的内核源码  
✅ 集成驱动代码到内核构建系统  
✅ 自动修改构建配置适配驱动模块  
✅ 智能选择构建方式（基于内核版本）  
✅ 一键下载编译结果（驱动模块 + boot.img）

## 使用指南

### 1. 准备工作
1. 在仓库中创建 `code` 目录，放入你的驱动源码：
   - 所有 `.c` 和 `.h` 源文件
   - `Makefile` 文件
2. 确定你的设备信息：
   - 内核大版本（设置 > 关于手机）
   - 设备架构（arm64/x86 等）
   - 驱动模块名称（Makefile 中定义的 .ko 文件名）

### 2. 配置工作流
修改 `.github/workflows/main.yml` 文件中的以下关键参数：

```yaml
name: Android14-6.1  # 修改为你的内核版本（格式：Android<大版本>-<内核版本>）

# ... 其他部分保持不变 ...

- name: Add module to GKI modules list
  run: |
    cd android-kernel
    awk -i inplace '
      # ... 脚本内容 ...
      print "    \"drivers/kerneldriver/rwProcMem_module.ko\","  # 修改为你的驱动模块名
      # ... 脚本内容 ...
    ' common/modules.bzl

- name: Build kernel module
  run: |
    cd android-kernel
    # 智能选择构建方式
    if [[ "${{ github.event.inputs.kernel_name }}" =~ Android(9|10|11) ]]; then
      echo "使用旧版构建系统 (Android 9-11)"
      build/build.sh
    else
      echo "使用新版 Bazel 构建系统 (Android 12+)"
      tools/bazel run //common:kernel_aarch64_dist  # 修改架构如 //common:kernel_x86_64_dist
    fi

- name: Upload kerneldriver.ko
  uses: actions/upload-artifact@v4.6.2
  with:
    name: kerneldriver
    path: android-kernel/out/kernel_aarch64  # 修改为对应架构路径
```

### 3. 需要修改的关键参数

| 参数位置 | 说明 | 示例值 |
|----------|------|--------|
| `name` | 内核版本 | `Android12-5.15` |
| 驱动模块名 | 你的驱动文件名 | `my_driver.ko` |
| 构建目标 | 设备架构 | `//common:kernel_x86_64_dist` |
| 输出路径 | 编译结果路径 | `android-kernel/out/kernel_x86_64` |

### 4. 启动编译
1. 提交代码到 GitHub 仓库
2. 访问仓库的 Actions 标签页
3. 选择 "Build Kernel Driver" 工作流
4. 点击 "Run workflow" 按钮
5. 等待约 45-60 分钟完成编译

### 5. 获取结果
编译完成后：
1. 在 Actions 页面找到成功的构建
2. 下载 "kerneldriver" 压缩包
3. 解压获取：
   - 编译好的驱动模块 (.ko)
   - 完整的内核镜像 (boot.img)
   - 构建日志

## 技术细节

### 智能构建系统选择
```bash
# 根据内核版本自动选择构建系统
if [[ "${{ github.event.inputs.kernel_name }}" =~ Android(9|10|11) ]]; then
  build/build.sh  # Android 9-11 使用传统构建系统
else
  tools/bazel run //common:kernel_aarch64_dist  # Android 12+ 使用 Bazel
fi
```

### 自动集成驱动到内核
```bash
# 将驱动代码复制到内核源码树
cp -r ../kerneldriver common/drivers

# 添加到内核构建系统
echo "obj-y += kerneldriver/" >> common/drivers/Makefile

# 注册为 GKI 模块
awk -i inplace '
  # 在模块列表的适当位置插入新驱动
  # 保持字母顺序避免编译错误
' common/modules.bzl
```

### 编译环境配置
```bash
# 解决常见编译错误
find . -type f -name "Makefile*" -exec sed -i \
  's/-Wframe-larger-than=[0-9]*/-Wframe-larger-than=4096/g' {} +

# 安装必备工具链
sudo apt-get install -y build-essential flex bison \
  libssl-dev libelf-dev bc
```

## 常见问题解决

**Q: 编译失败提示 "Module not found in GKI list"**  
A: 确保 `Add module to GKI modules list` 步骤中的驱动文件名与 Makefile 中的完全一致

**Q: 下载的 boot.img 刷入后无法启动**  
A: 检查设备架构配置是否正确，特别是：
- `tools/bazel run //common:kernel_xxx_dist` 中的架构
- 上传路径 `android-kernel/out/kernel_xxx`

**Q: 编译过程卡在 repo sync**  
A: GitHub 服务器访问 Google 源码库不稳定时可能发生，可尝试：
1. 重新运行工作流
2. 使用镜像源（需修改 repo init 的 URL）

**Q: 如何支持旧版 Android (9-11)?**  
A: 工作流已自动处理：
- Android 9-11：使用传统 build.sh
- Android 12+：使用 Bazel 构建系统

## 贡献与改进

欢迎提交 Issue 或 PR 改进此项目：
- 添加更多设备架构支持
- 优化构建速度
- 增加镜像源选择
- 改进错误处理机制

## 许可证

本项目采用 [MIT 许可证](LICENSE)，可自由用于个人和商业项目。
