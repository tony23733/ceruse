# Ceruse

**`ceruse`** /sɪˈruːs/ —— 源自拉丁语 *cerussa* / 法语 *céruse*：**铅白**（碱性碳酸铅，2PbCO₃·Pb(OH)₂），
文艺复兴到 19 世纪油画中最重要、最亮的白色颜料。

> 一个面向 PC 桌面的实时渲染引擎：三种图元（**mesh / 3DGS splat / neural field**）共用一套
> HDR 画布、depth 与 RenderGraph，可单独开关、可叠加。

**状态：M0 之前（项目启动阶段）** — 目标见 [里程碑](#里程碑)。

---

## ⛔ 硬规则（提 PR 前先看）

- ❌ `src/render/**` include `entt/*` 或 `ecs/*`
- ❌ `src/render/**`、`src/ecs/**` include `slang-rhi.h`
- ❌ 业务代码手写 `setTextureState` / `globalBarrier`
- ❌ 跟 slang-rhi `main` 分支（只跟锁定的 commit）
- ❌ 一个 splat 一个 entity
- ❌ 一个 shader 用 `#define` 打变体（用 `[SpecializationConstant]`）
- ❌ 未开校验层就调 pass

其余工程约定见 [CONTRIBUTING.md](CONTRIBUTING.md)，设计取舍见 [docs/adr/](docs/adr/)，
完整启动方案见 [plans/ceruse-kickoff.md](plans/ceruse-kickoff.md)。

---

## 平台矩阵

只做 **PC 桌面端（含 macOS）**。不做移动、浏览器、主机、D3D11/OpenGL。

| 平台 | 图形后端 | 状态 |
|---|---|---|
| Windows 10/11 | **D3D12** | 一等（主开发平台） |
| Windows 10/11 | **Vulkan 1.3** | 一等（跨后端正确性验证） |
| Linux x86_64 | **Vulkan 1.3** | 二等（M3 之后） |
| macOS 13+ (arm64) | **Metal 3** | 一等（Apple Silicon 优先） |

**"含 macOS"带来的三条硬约束**（从第一天就体现在抽象层里）：

1. **着色器目标 ×3**：Slang → HLSL（D3D12）/ SPIR-V（Vulkan）/ MSL（Metal）。
2. **TBDR**：Apple Silicon 是 Tile-Based Deferred Renderer，
   `loadOp / storeOp`（`Clear/Load/DontCare` + `Store/DontCare`）是 RenderGraph 的**一等公民**，
   不是附带优化项（Apple GPU 上 2~3 倍带宽差异）。
3. **能力交集**：Metal 没有 indirect count、没有原子 float 加法、不暴露 cooperative matrix。
   → RHI 只暴露三后端交集，降级路径写在 RHI 内部，**不污染业务代码**。

---

## 技术选型（已锁定）

| 项 | 决策 |
|---|---|
| 语言 | C++20（模块 `.ixx`） |
| 构建 | CMake ≥ 3.24 + `FetchContent` / vcpkg |
| 图形抽象 | **slang-rhi**（锁定 commit，禁止跟 `main`）— `<PINNED_SHA>` |
| ECS | entt（稳定版） |
| 窗口 / 输入 | SDL3（`SDL_Metal_CreateView`、hidpi） |
| UI | Dear ImGui（自写 slang-rhi backend）+ imnodes |
| 数学 | glm |
| 导入 | tinygltf / stb_image / 自写 KTX2 + Basis（离线 BC7/BC5/ASTC 烘焙） |
| 着色器 | **Slang 100%**，运行期编译 + 热重载 + 磁盘缓存 |
| 性能分析 | Tracy（CPU）+ GPU timestamp query（三后端都要有） |
| 3DGS 排序 | 先 CPU 排序验证正确 → GPU 键值排序 / tile binning |
| 测试 | Catch2 或 doctest；headless 截图回归（三后端矩阵） |

---

## 架构分层

```
app          帧循环 / swapchain / ImGui / 命令行
  │
ECS (entt)   组件 + system 调度
  │
  ├── Extract（每帧一次，单向数据流）
  │
RenderFrame  纯 SoA 数据，不含 entt
  │
render       materials / passes / techniques / RenderGraph
  │
rhi          slang-rhi 薄封装 + state tracker + RenderGraph 执行
  │
slang-rhi
```

三条硬边界的落地方式：

1. `src/render/**` 与 `src/ecs/**` 互不 include，只经 `render/extract/render_frame.h` 通信
   （CI 一条 grep 检查）。
2. 只有 `src/rhi/` 可见 `slang-rhi.h`，其余代码只用 `ceruse::rhi::Device/Texture/Buffer/...`。
3. 资源状态转移只能由 `RenderGraph` 调 `state_tracker` 完成。

所有 GPU 句柄使用 `Handle<T>{index, gen}`，而非裸指针（热重载 / 延迟释放 / 调试命名）。

---

## 仓库结构

```
ceruse/
├─ CMakeLists.txt / CMakePresets.json
├─ plans/            # 启动方案（ceruse-kickoff.md）
├─ docs/             # ADR、platform-matrix、上游头文件快照
├─ 3rdparty/         # entt glm spdlog stb tinygltf tracy imgui sdl3
├─ cmake/            # toolchain / warnings / tracy / platform
├─ shaders/          # common mesh gs3d neural post  *.slang
├─ src/              # core platform ecs asset rhi render app
├─ tests/
└─ tools/            # splat_convert / weights_export
```

---

## 命名约定

```
命名空间        ceruse                （不加 cer_ 前缀污染命名空间）
CMake target   cer_core / cer_rhi / cer_render / cer_app
宏 / 环境变量    CER_*
可执行文件      ceruse
资产后缀        .cer（场景/资产包） .csplat（splat 云） .cweights（神经权重）
着色器后缀      .slang
ADR            docs/adr/NNNN-*.md
```

---

## 里程碑

| # | 内容 | 估算 | 关键验收 |
|---|---|---|---|
| **M0** | 打通 slang-rhi（三后端 + 校验层 + GPU 计时） | 1~2 周 | 三后端 10 分钟无 validation error；`--backend=` 运行期切换 |
| **M1** | Slang 编译 + 反射绑定 + 热重载 | 1 周 | 同一 `.slang` 三后端视觉一致；热重载 < 300ms |
| **M2** | Mesh 最小闭环（glTF + Forward PBR + CSM） | 2~3 周 | Sponza 三后端一致；CPU 帧 < 3ms |
| **M3** | RenderGraph + 现代光照 + 后处理 | 3~4 周 | 1000 点光 + 100 阴影体；pass 重排只改图代码 |
| **M4** | 3DGS（M4a CPU 排序正确性 → M4b GPU 排序/LOD） | 3~5 周 | 1M splat @1080p ≥ 60fps |
| **M5** | 神经渲染（先 3DGS+MLP，再 hash-grid 体积） | 4~6 周 | 与 PyTorch 参考 PSNR 差 < 0.5dB |
| **M6** | 工程化（抓帧、热重载、序列化、截图回归） | 持续 | — |

局部目标不变，顺序不变：**M0 风险最高，必须最先做**。

---

## 早期 Spike（每个 ≤ 2 天，失败就改方案）

1. 同一份 `.slang` → HLSL / SPIR-V / MSL，三后端跑通
2. compute 生成 indirect args → `drawIndexedIndirect`（顺带暴露 Metal 无 indirect count）
3. 多 pass compute 链 + barrier，验证 `state_tracker` 设计
4. compute shader 里 fp16 GEMM（小矩阵）+ 计时
5. splat 与 mesh 的 depth 互相遮挡合成
6. Metal `LoadAction=DontCare` 的带宽/耗时差异（TBDR 验证）
7. Metal 无 float 原子的替代路径（tile 直方图）

---

## 构建（待 M0 落地后补全）

```bash
cmake --preset windows-d3d12     # 或 windows-vulkan / macos-metal
cmake --build --preset windows-d3d12
./build/ceruse --backend=d3d12 --scene <scene.cer>
```

---

## 参考

- slang-rhi：头文件即文档（`include/slang-rhi.h`）+ `examples/`
- Slang 反射：`slang::IModule::getReflection()` / `ProgramLayout` / `TypeLayout`
- 3DGS：Kerbl et al., *3D Gaussian Splatting for Real-Time Radiance Field Rendering*, SIGGRAPH 2023
- instant-ngp：Müller et al., *Instant Neural Graphics Primitives with a Multiresolution Hash Encoding*, 2022
- TBDR：Apple《Metal Best Practices》、WWDC "load/store actions for render passes"

---

*Ceruse 有毒。影射到引擎里，只剩一件事：光够不够白。*
