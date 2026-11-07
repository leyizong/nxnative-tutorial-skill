# 实战：把一款不在表里的游戏调到能玩 —— 《文明 7》全记录

> 2026-11-07 · 一天从"0 个着色器"到"回合 1 开局可玩"（Mac）
> 配套主教程：[README.zh-CN.md](README.zh-CN.md)。本文是"新游戏带起"路线的完整实例：每一步怎么查、每个坑怎么定位、每个修复为什么那样修。

[nxnative](https://github.com/seedds/nxnative) 游戏表外的游戏也能构建，但"可能开不了机"。本文记录《文明 7》（Switch 版，Title ID `0100C3601518C000`，v1.5.0）从 NSP 到 Mac 可玩的全过程——它踩到的每个坑都是新游戏带起的典型形态。

## 0. 基线：标准流程走到哪停了

```bash
python3 scripts/build_ipa_from_nsp.py /path/to/nsps -k prod.keys \
  --romfs separate -l zh-Hans -o Civ7.ipa
```

| 流水线步骤 | 结果 |
|---|---|
| 提取 / 勘测 / 模块改写 / 堆检查（3.01 GB 堆） | ✅ |
| Mac 宿主启动到 `nnMain` | ✅ |
| 地址区域检查 | ⚠️ 游戏开局预留 **8 GB** 别名空间，超 iPhone 5 GB 上限 → **Mac 优先**（Mac 的别名区即 Switch 的 64 GB） |
| 着色器拆分 | ❌ **78,523 个文件扫完 0 个着色器** |

## 1. 坑 A：全游戏找不到一个着色器

**排查链**（每步都排除一个假设）：

1. 媒体解码 NRO 的导入表 → 0 个 glslc/nvn 字符串（排除"编译器藏在解码器里"）；
2. Mac 实跑（`DM_NVN_SHDUMP` 抓运行时提交的着色器）→ 12 个着色器**逐字节命中** RomFS 根目录的 `ShaderAutoGen_Linux_NVN_FinalRelease.bs`。

**根因**：这个 11.4 MB 的 `CIVBSF` 文件把**预编译 Maxwell 二进制和 NV 汇编源码打包在一起**（801 个着色器：357 vs + 343 ps + 101 cs），游戏运行时直接提交二进制，**全程零编译**——所以二进制里没有任何 `glslcCompile`。

**修复**：给 `shader_split.py` 写一个 `civbsf_shaders` 容器读取器（仿 frostbite 模式）。格式全解在函数 docstring 里；验证标准是"运行时 dump 的键全部能从读取器输出复现"（12/12）。

> **方法论**：着色器找不到时，先 `DM_NVN_SHDUMP=<dir>` 抓游戏实际提交的字节，再拿这些字节去游戏文件里 `grep`——命中的容器就是真正出处，比猜测格式高效得多。

## 2. 坑 B：确认设置屏后必崩（SIGBUS）

**症状**：`CRASH sig=10 addr=0 pc=main+0x4e8a18`，确定性复现。

**排查**：64 MB 主栈照崩（排除栈溢出）→ 反汇编崩溃点：崩溃函数 `sub sp, sp, #0x98`（**奇数帧**，152=16×9+8）后第一条 `str x1,[sp]`。

**根因**：游戏脚本桥（4 份拷贝）用奇数立即数调栈。Horizon 不检查 sp 对齐；**Apple 内核对未对齐 sp 的访问直接 SIGBUS**。

**修复**：`patch_nso.py` 新增 `sp_imm` 改写站点——奇数 `sub/add sp,sp,#imm` 原地取整到 16（sub/add 成对同改，sp 进出函数不变）。库中已有同机制先例（Paper Mario 的寄存器形式 `sp_align`）。

> **方法论**：SIGBUS + addr=0 + sp 相对访问 → 先查 sp 对齐，再看栈大小。同家族问题在 `docs/reference/rewriter.md` 的 "Stack arguments reserved" 条目。

## 3. 坑 C：无头全绿，开窗就崩（宿主 bug）

**症状**：几十轮无头跑零崩溃；`DM_WINDOW=1` 后 33 秒 `sig=6` abort。

**根因**：开窗后 present 走 display-link 线程；游戏有一张**首次使用即 present** 的窗口纹理 → present 线程触发首次上传 → 宿主 `up_owner_check`（上传批次仅限 submit 线程）abort。无头模式 present 在合法线程做同样的事，所以从不触发。

**当前绕过**：`DM_PRESENT_LINK=0`。正修需宿主上传路径改动（待上游 PR）。

## 4. 坑 D：没有声音

Mac 上宿主**默认静音**（仅 iOS 默认出声）：`DM_AUDIO=1` 即可。

## 5. 坑 E/F：两类"画了但看不见"（着色器链接/混合）

| 症状 | 根因 | 修法 |
|---|---|---|
| 悬停地块无高亮 | 高亮用 NVN **双源混合**；profile 的 `dual_want.tsv` 为空 → 第二混合源按零（画了=隐形） | 日志 RED 给出着色器键 → 加进 `dual_want.tsv` → `dmsh_build.py <pkg> <cache> <key>` 单键重建。多个变体会陆续冒头，挂 `DM_NVN_DUAL_WANT=<file>` 自动收集 |
| 放置建筑无预览 | 运行时 VS/FS 配对不在任何配对表 → `vs_pad` 没补顶点输出 → Metal 建管线失败 | 两对补进 `pso_pairs.tsv` + 重建两个顶点着色器 |

> **方法论**：游戏逻辑正常但某视觉效果缺失 → 先看 `[RED]`。这个仓库的设计哲学是每类问题都有带键名的 RED 指路，顺着修就行。

## 6. 调试环境备忘（Mac 手动跑）

```bash
DM_ROMFS=<RomFS.img 文件本身！传目录会静默失效，游戏"秒退">
DM_MSL_DIR=<shader/cache>          # 宿主自己拼 b-<chain>
DM_NVN_SHMAP=<profile>/shader_map.tsv
DM_SAVE / DM_TEMP                  # 存档与临时目录
DM_KEYS='8:X,12:X'                 # 脚本化按键（无头过菜单）
DM_CTL=<普通文件>                  # 追加行：'A'/'Right' 按键、'shot <path>' 截图
DM_WINDOW=1 + DM_PRESENT_LINK=0 + DM_AUDIO=1   # 可玩三件套
# 键盘=手柄：J=A K=B U=X I=Y，WASD=左摇杆，方向键=十字键
```

## 7. 结果

- **Mac 可玩**：菜单 → 领袖/文明选择 → 开局（回合 1，公元前 4000 年，完整世界地图 + HUD + 中文 + 声音），60 fps 节奏，0 崩溃，内存 2.8 GB / GPU 1.9 GB 稳定
- CIVBIG/Oodle 资产流送（RomFS 的 86%）无需任何修改即可工作
- 待办：present 线程正修、MSAA 纹理上传、`.bs` 元数据配对表解析、iPhone 内存实测

## 8. 通用心法

1. **让日志说话**：每类问题都有 `[RED]`，且大多自带修法提示（连文件名都给出）；
2. **字节对质**：拿运行时抓到的字节去文件里搜，直接定位真正的容器；
3. **一次一个假设**：崩溃先排除（栈大小→执行权限→对齐），每步都要有证据；
4. **修在根上**：识别出"游戏行为合法、宿主语义缺失"时，修宿主/工具的通用路径，而不是给游戏打补丁。
