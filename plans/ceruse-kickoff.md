# Ceruse 引擎 · 启动方案（Kickoff Plan）

> 状态：草案 v0.2 · 目标：可直接照做的执行文档
> 上游项目：`ink3d_engine`（本文档记录从它继承什么、丢弃什么）
> 适用：新开仓库，从零实现

---

## 0. 项目命名

**项目名：Ceruse（`ceruse`，缩写 `cer`）**

### 0.1 词源与含义

*ceruse* /sɪˈruːs/ ≈「suh-ROOS」，源自拉丁语 *cerussa*、法语 *céruse*：
**铅白**（碱性碳酸铅，2PbCO₃·Pb(OH)₂）——文艺复兴到 19 世纪油画里**最重要、最亮的白色颜料**。
画家先用它（或 gesso 白底）"打底"，再一层层叠色；维米尔、伦勃朗的高光都是它。

### 0.2 在本项目里的三层隐喻

1. **打底（grounding）**：引擎的第一职责是"把画布准备好"——HDR 画布 + depth + RenderGraph
   这套基础设施。三种图元（mesh / splat / neural field）都只是往上叠的色层。
2. **最亮的白**：白 = 光。传统 PBR 的间接光、3DGS 的 view-dependent 高光、神经场的 radiance，
   归根结底都在解同一个问题：**这里有多白**。
3. **血统延续**：从 `ink3d` 的"墨"到 Ceruse 的"白颜料"，与 §2 的技术继承关系对称。

### 0.3 命名可用性（实测）

| 检查项 | 结果 |
|---|---|
| GitHub 仓库名含 `ceruse` | 5 个，全部无关（个人账号 `CandeCeruse`、若干杂项仓库） |
| GitHub 搜 `ceruse renderer` | **0 结果** |
| PyPI 包 `ceruse` | 未占用（404） |
| 已知图形/引擎项目同名 | 未发现 |

### 0.4 已知小风险（可接受，备查）

1. **拼读**：英语母语者可能读成 /səˈruːz/ 或 /ˈsiːruːs/。→ README 首行直接标音标，并在
   首次出现处括号注明「铅白 pigment」。
2. **语义**：铅白有毒，且伊丽莎白一世以它化妆导致慢性铅中毒。作为引擎名无碍，
   但宣传语里别玩"无毒/健康/皮肤"相关的梗。
3. **法语区搜索**：*céruse* 在法语里就是"铅白"，检索时会掺入颜料词条。
4. **拼写形近**：`ceruse` / `censure` / `cerise`（樱桃）/ `céruse`，口头沟通要说清。

### 0.5 备选（如需换名，已评估过）

| 备选 | 词源 / 本义 | 意象与取舍 |
|---|---|---|
| **Tessera** | 拉丁语，马赛克小块 | 语义最贴「图元 = 小片」；因撞 Tessera Technologies（图像传感封装）而退为备选 |
| **Loam** | 沃土（沙+黏土+腐殖质） | 极短、零占用，「多种材质混合成一体」；工程友好度最高 |
| **Ochre** | 赭石，最古老的颜料 | 延续 ink3d 颜料血统，好读好拼 |
| **Murmuration** | 椋鸟群飞 | 对 3DGS 语义最精确（大量个体→整体形态），但偏长 |
| **Lattice** | 晶格、栅格 | 概括 M5 的 hash-grid；撞 Lattice Semiconductor |
| **Nacre** | 珍珠母 | 薄膜干涉 → 呼应 view-dependent 色彩 |
| **Gyre** | 旋涡、环流 | 读音不直观（/ˈdʒaɪər/），代码里看不出含义 |
| **Inkwell** | 墨水瓶 | 延续 ink3d；撞 Rust 的 Inkwell（LLVM 封装） |
| **Vellum** | 犊皮纸 | 薄片合成；撞同领域软件 vellum.ai |

**命名约定（写进 CLAUDE.md / CONTRIBUTING.md）**

```
仓库          ceruse
命名空间      ceruse                 （不用 cer_ 前缀污染命名空间）
CMake target  cer_core / cer_rhi / cer_render / cer_app
宏            CER_*
环境变量      CER_*
可执行文件    ceruse
目录          src/...（平面）或 src/ceruse/...（二选一，锁定不要混用）
资产后缀      .cer（场景/资产包）、.csplat（splat 云）、.cweights（神经权重）
着色器后缀    .slang
文档 ADR      docs/adr/NNNN-*.md
```

---

## 1. 范围与平台矩阵

**只做 PC 桌面端（含 macOS）。不做移动、不做浏览器、不做主机。**

| 平台 | 图形后端 | 状态 | 备注 |
|---|---|---|---|
| Windows 10/11 | **D3D12** | 一等 | 主开发平台，调试工具最全 |
| Windows 10/11 | **Vulkan 1.3** | 一等 | 用同一份代码验证跨后端正确性 |
| Linux (x86_64) | **Vulkan 1.3** | 二等 | 可延后到 M3 之后，但架构上不要写死 Win32 |
| macOS 13+ (arm64) | **Metal 3** | 一等 | Apple Silicon 优先；Intel Mac 不承诺 |

明确不做：

- 移动端（Android/iOS）、WebGPU/浏览器、D3D11、OpenGL、MoltenVK（macOS 直接走原生 Metal）
- 光栅化以外的老式管线（Geometry/Tessellation Shader 不规划）
- 主机平台

### 1.1 "PC 含 macOS"的架构含义（重要）

加上 macOS 不是"多一个后端"这么简单，有三条硬约束必须从第一天写进抽象层：

1. **着色器目标从 2 个变成 3 个**：Slang → HLSL（D3D12）/ SPIR-V（Vulkan）/ **MSL（Metal）**。
   `slangc` 的 Metal 输出路径要尽早验证（见 Spike-1）。
2. **TBDR（Apple Silicon 是 Tile-Based Deferred Renderer）**：RenderGraph 必须携带
   **attachment 的 load/store action**（`Clear/Load/DontCare` + `Store/DontCare`），
   而不是只携带资源状态。TBDR 上 on-chip tile memory 的读写行为决定性能量级，
   "用不用 `DontCare`" 在 Apple GPU 上是 2~3 倍带宽差异，在 D3D12/Vulkan 上几乎无感。
   → **抽象层取两者之交集偏 TBDR**：RenderGraph 的 pass 描述里 `loadOp/storeOp` 是一等公民。
3. **"间接计数"在 Metal 上缺失**：Vulkan 有 `vkCmdDrawIndexedIndirectCount`、
   D3D12 有 `ExecuteIndirect` 的 count 版本，Metal 的间接绘制**没有 count buffer 变体**。
   → RHI 抽象里 `multiDrawIndirect(countBuffer)` 必须提供"降级实现"：
   固定 max draw 数 + 无效实例退化为退化三角形，或走 `MTLIndirectCommandBuffer`。
   这条直接影响 3DGS 的"GPU 剔除后决定画多少个 splat-batch"。

---

## 2. 从 ink3d_engine 继承什么

### 2.1 直接搬（低耦合基础设施）

| 路径（ink3d_engine） | 用途 |
|---|---|
| `engine/src/core/log` | spdlog 封装 |
| `engine/src/core/math` | glm 别名、常用工具 |
| `engine/src/core/macro`, `core/std` | 宏、std 扩展 |
| `engine/src/platform` | 窗口/文件/计时（需改为 SDL3 或保留） |
| `engine/3rdparty/{entt,glm,spdlog-1.15.1,stb,tinygltf,tinyobjloader,tracy}` | 依赖全部可用 |

### 2.2 只搬"思路"，实现重写

| 路径 | 处理 |
|---|---|
| `engine/src/function/render/rhi/rhi_struct.ixx` | 枚举/描述符定义可参考；**语义不可复用**（WebGPU 的 bind group 数量、隐式 barrier、usage flag 会污染 slang-rhi 的显式状态模型） |
| `engine/src/function/render/passes/render_pass.ixx` | 接口形态参考，实现重写 |
| `engine/src/function/render/pipelines/render_pipeline.ixx` | 同上 |
| `engine/src/function/scene/systems` | ECS → 渲染的抽取思路可参考，但要求更严格（见 §4） |

### 2.3 不要搬

- `engine/src/function/render/rhi/webgpu/*`（`rhi_webgpu.cpp`、`webgpu-utils.*`、`webgpu_convert.ixx`、`webgpu_struct.ixx`）
- `engine/3rdparty/webgpu/*`（Dawn/wgpu-native 集成）
- `examples/framework/imgui_backends/webgpu.*`
- 任何 `Build/`、`build_old/`（dawn 相关 CMake 依赖，历史包袱）

**规则：新仓库的 `src/render/` 目录下不允许出现任何 WebGPU / wgpu 词汇。**

---

## 3. 技术选型锁定表（第 1 天定稿，写入 README）

| 项 | 决策 | 理由 |
|---|---|---|
| 语言 | C++20，模块（`.ixx`） | 沿用现有习惯；MSVC/Clang 均支持 |
| 构建 | CMake ≥ 3.24 + `FetchContent` / vcpkg | — |
| 图形抽象 | **slang-rhi（锁定 commit/tag，禁止跟 `main`）** | 显式状态、贴近原生、天然吃 Slang 反射 |
| 上游版本 | `<在这里填写锁定的 commit sha>` | API 仍在演进，跟 main 会持续破坏 |
| ECS | entt（确认用稳定版，记录版本号） | 已知 |
| 窗口/输入 | SDL3 | Metal view（`SDL_Metal_CreateView`）、hidpi、跨平台一次性解决 |
| UI | Dear ImGui + 自写 slang-rhi backend + imnodes | 神经渲染调参面板需要 imnodes |
| 数学 | glm | 已知 |
| 图像/模型导入 | tinygltf（glTF）、stb_image（LDR）、自写 KTX2/Basis 读取（BC7/ASTC/BasisU 离线烘焙） | PC only，走 BC 系列 |
| 3DGS 资产 | `.ply`（原始）→ 自研紧凑格式（见 §8 M4） | 加载速度是刚需 |
| 着色器语言 | Slang（100%） | slang-rhi 反射红利 |
| 着色器编译 | 运行期 `slang::Session` 编译 + 热重载；发布版可预编译缓存 | 迭代速度优先 |
| 分析工具 | Tracy（CPU）+ GPU timestamp query（三后端都要实现） | 3DGS/神经渲染是性能项目 |
| 排序（3DGS） | 先 CPU 排序验证正确 → GPU 键值排序 / tile binning | 见 M4 |
| 测试 | Catch2 或 doctest；headless 截图回归 | 三后端一致性靠它 |

---

## 4. 仓库骨架

```
ceruse/
├─ CMakeLists.txt
├─ CMakePresets.json
├─ README.md
├─ CONTRIBUTING.md            # 含"render 层不得 include entt"等硬规则
├─ docs/
│   ├─ adr/
│   │   ├─ 0001-slang-rhi.md
│   │   ├─ 0002-no-webgpu.md
│   │   ├─ 0003-render-graph-shape.md
│   │   └─ 0004-ecs-render-boundary.md
│   ├─ upstream/slang-rhi.h    # 上游头文件快照存档
│   └─ platform-matrix.md
├─ 3rdparty/                   # entt glm spdlog stb tinygltf tracy imgui sdl3
├─ cmake/                      # toolchain、warnings、tracy、platform
├─ shaders/
│   ├─ common/                 # types.slang, binding.slang, sh.slang, noise.slang
│   ├─ mesh/                   # forward.slang, shadow.slang, pbr.slang
│   ├─ gs3d/                   # project.slang, sort.slang, render.slang
│   ├─ neural/                 # hashgrid.slang, mlp.slang, volume.slang
│   └─ post/                   # tonemap.slang, bloom.slang, taa.slang
├─ src/
│   ├─ core/                   # log math macro memory jobs(线程池) hash
│   ├─ platform/               # window(sdl3) file timer
│   ├─ ecs/                    # world.h components/ systems/ schedule
│   ├─ asset/                  # mesh texture material splat neural 导入 + 缓存
│   ├─ rhi/
│   │   ├─ device.h/.cpp       # 唯一接触 slang-rhi 的地方
│   │   ├─ state_tracker.h/.cpp
│   │   ├─ render_graph.h/.cpp
│   │   ├─ transient_alloc.h   # 每帧上传堆 / staging ring
│   │   └─ slang/              # session, module, reflection→binding layout, cache
│   ├─ render/
│   │   ├─ extract/            # ECS → RenderFrame
│   │   ├─ materials/
│   │   ├─ passes/             # shadow forward|gbuffer lighting splat neural post
│   │   └─ techniques/         # 组合 pass 的"技术"，见 §7
│   └─ app/                    # 帧循环 swapchain imgui profiler 命令行
├─ tests/
└─ tools/
    ├─ splat_convert/          # .ply → .tsplat 打包
    └─ weights_export/         # PyTorch 权重 → 二进制 + manifest（Python）
```

---

## 5. 分层与硬边界

```
        ┌──────────────┐
        │     app      │  帧循环 / swapchain / imgui
        └──────┬───────┘
        ┌──────┴───────┐
        │ ECS (entt)   │  组件 + system 调度
        └──────┬───────┘
         Extract（每帧一次，单向数据流）
        ┌──────┴───────┐
        │ RenderFrame  │  纯 SoA 数据，不含 entt / 不含 RHI 句柄之外的东西
        └──────┬───────┘
        ┌──────┴───────┐
        │   render     │  materials / passes / techniques / RenderGraph
        └──────┬───────┘
        ┌──────┴───────┐
        │     rhi      │  slang-rhi 薄封装 + state tracker + RenderGraph 执行
        └──────┬───────┘
             slang-rhi
```

### 5.1 三条硬规则

1. **`src/render/**` 与 `src/ecs/**` 互不 include。** 二者只通过 `render/extract/render_frame.h`
   的数据结构通信。CI 加一条 grep 检查即可。
2. **只有 `src/rhi/` 可以 include `slang-rhi.h`。** 其余代码只见到 `ceruse::rhi::Device/Texture/Buffer/...`。
3. **资源状态转移只能由 `RenderGraph` 调 `state_tracker` 完成。** 业务代码禁止手写
   `setTextureState` / `globalBarrier`（Metal 上没有显式 barrier，规则一致才不会分叉）。

### 5.2 数据契约

```cpp
// src/render/extract/render_frame.h
namespace ceruse::render {

struct DrawItem {              // 一次实例化绘制的"桶"
  MeshHandle     mesh;
  MaterialHandle material;
  uint32_t       firstInstance;
  uint32_t       instanceCount;
};

struct SplatItem {
  SplatHandle cloud;
  uint32_t    transformIndex;
  float       lodBias;
  float       opacityScale;
};

struct NeuralFieldItem {
  NeuralFieldHandle field;
  AABB              bounds;
  float             densityScale;
};

struct RenderFrame {
  CameraData  camera;
  AABB        sceneBounds;
  std::pmr::vector<LightData>        lights;      // 已剔除、已排序
  std::pmr::vector<DrawItem>         draws;
  std::pmr::vector<InstanceData>     instances;   // 世界矩阵 + 材质参数
  std::pmr::vector<SplatItem>        splats;
  std::pmr::vector<NeuralFieldItem>  fields;
};

} // namespace ceruse::render
```

### 5.3 ECS 组件

```cpp
// src/ecs/components.h
namespace ceruse::ecs {

struct Transform        { Mat4 world; Entity parent; };
struct Camera           { float fovY, znear, zfar; uint32_t flags; };
struct MeshRenderer     { AssetHandle<Mesh> mesh; AssetHandle<Material> material; };
struct Light            { LightType type; Vec3 color; float intensity; /* ... */ };
struct GaussianSplatRenderer { AssetHandle<SplatCloud> cloud; float lodBias; };

// ⚠️ 一个 splat 一个 entity 是绝对禁止的：
//    100 万 splat 的场景 = 1 个 entity + 1 组 GPU buffer
struct NeuralFieldRenderer   { AssetHandle<NeuralField> field; };

} // namespace ceruse::ecs
```

---

## 6. RHI 层设计要点

### 6.1 封装的边界

`src/rhi/device.h` 暴露最小集合，**不暴露 slang-rhi 的类型**：

```cpp
namespace ceruse::rhi {

struct DeviceDesc { Backend backend; bool enableValidation; LogLevel log; };

class Device {                     // 单例式，进程内一个
 public:
  static std::unique_ptr<Device> create(const DeviceDesc&);
  CommandQueue& graphicsQueue();
  CommandQueue& computeQueue();
  CommandQueue& transferQueue();

  TextureHandle  createTexture(const TextureDesc&);
  BufferHandle   createBuffer(const BufferDesc&);
  SamplerHandle  createSampler(const SamplerDesc&);
  ShaderHandle   createShader(const ShaderDesc&);           // 走 slang 编译
  PipelineHandle createRenderPipeline(const RenderPipelineDesc&);
  PipelineHandle createComputePipeline(const ComputePipelineDesc&);
  ...
};

} // namespace ceruse::rhi
```

句柄用 `Handle<T>`（`{uint32_t index, uint32_t gen}`）而不是裸指针：便于热重载、延迟释放、
序列化与调试命名。

### 6.2 `state_tracker`（必须，且要早写）

- 记录每个资源的当前状态（D3D12 `D3D12_RESOURCE_STATES` / Vulkan image layout+access / Metal 无）。
- 提供 `transition(cmd, resource, desired, syncScope)`，内部合并/裁剪，debug 下断言"读写冲突未加 barrier"。
- **Metal 后端在 tracker 内部是空实现**（无显式 barrier），但**接口保持一样**，这样 pass 代码三后端一致。
- Metal 的"依赖"靠 pass 边界隐式同步 + `memoryBarrier`（`MTLBarrierScope`），由 tracker 决定是否插入。

### 6.3 `transient_alloc`（每帧上传）

- 每个队列一个 ring buffer（默认 16MB/帧 × 3 帧在飞）。
- `upload(cmd, span) -> GPUPtr`：线性分配 + 自动 barrier。
- 所有 per-frame uniform / 实例数据 / indirect args 都走这里，避免每帧创建 buffer。

### 6.4 RenderGraph

**形态（保持小，不要抄通用 FrameGraph 全家桶）：**

```cpp
class RenderGraph {
 public:
  struct TextureUse {
    TextureHandle    texture;
    Access           access;      // Read / ReadWrite / Write
    LoadOp           loadOp;      // Clear / Load / DontCare      ← TBDR 必需
    StoreOp          storeOp;     // Store / DontCare             ← TBDR 必需
    Vec4             clearColor;
  };
  struct BufferUse { BufferHandle buffer; Access access; };

  using SetupFn   = std::function<void(PassBuilder&)>;
  using ExecuteFn = std::function<void(CommandContext&)>;

  void addPass(std::string_view name, QueueType queue,
               std::function<void(PassBuilder&)> setup, ExecuteFn exec);

  void compile();                 // 依赖排序、资源状态推导、别名
  void execute();                 // 按队列提交、跨队列 fence
};
```

必须支持的能力（按优先级）：

1. 读写依赖 → 拓扑排序（消费者在生产者之后）
2. 自动状态转移（交给 `state_tracker`）
3. 跨队列同步（graphics ↔ compute，Vulkan/Metal 都要；Metal 用 `MTLEvent`/`MTLSharedEvent`）
4. 别名（跨 pass 复用同一块内存，尤其是 G-buffer 和 splat 中间缓冲）
5. **in-place pass 检测**：写同一资源两次必须报错（这是显式 API 最常踩的坑）
6. pass 级 GPU 计时（每个 pass 一对 timestamp）

**不做**：异步计算重叠的自动调度（M6 之前）、自动并行化 pass（收益远小于复杂度）。

---

## 7. 三条渲染路径如何共存

### 7.1 统一约定

- 三条路径**共享** `HDRColor (RGBA16F)` + `Depth (D32F)` + 可选的 `Velocity`。
- **Mesh 先画**（写 depth），**splat/neural 后画**并按 depth 正确合成；
  或反过来按需用 depth prepass。这一条决定视觉正确性，不要中途改。
- 每条路径都是一个 **Technique**，可以单独开/关，也可叠加：

```cpp
// src/render/techniques/technique.h
struct TechniqueDesc { bool shadows, mesh, splats, neuralFields, post; };

class IRenderTechnique {
 public:
  virtual void build(RenderGraph&, const RenderFrame&, const TechniqueDesc&) = 0;
};
```

- **共享的 Slang 基础库** `shaders/common/`：类型定义、binding 约定、SH 求值、球谐/协方差、
  噪声。M4/M5 的数学（协方差投影、hash-grid 插值）都放这里，便于写 CPU 侧参考实现做单测。

### 7.2 各路径的 pass 形态

| 路径 | pass 序列 |
|---|---|
| Mesh 传统 | `shadow(cascade×N) → depthPrepass? → gbuffer` 或 `forward` → `lighting(clustered) → transparent → post` |
| 3DGS | `compute: 视锥剔除 → 2D 投影 & 协方差 → 深度键计算 → 排序(radix/tile binning) → 写 indirect args` → `raster: index draw + back-to-front blend` |
| 神经渲染 | `compute: 屏幕 tile/brick 生成 → hash-grid 特征插值 → MLP 推理 → (密度/颜色)` → `composite` |

---

## 8. 里程碑（含验收标准）

> 时间按"一个人全职、熟悉图形"估算。可以并行推进的部分见 §8 备注。

### M0 · 打通 slang-rhi（1~2 周，**风险最高，必须最先做**）

任务：

1. CMake 集成 slang-rhi（锁 commit），先**只**编它自带的 `hello-triangle` 示例。
2. `Device` 封装：D3D12 / Vulkan / Metal 三后端可切换（命令行 `--backend=d3d12|vk|mtl`）。
3. swapchain + 帧循环 + 清屏 + Tracy + GPU timestamp。
4. **三种后端都开满校验层**：D3D12 debug layer、Vulkan validation + `VK_LAYER_KHRONOS_synchronization2` 检查、Metal validation + shader validation。

验收：

- [ ] 三后端在各自机器上跑 10 分钟不掉帧、无 validation error。
- [ ] `--backend` 切换不需要重编（运行期选择）。
- [ ] 帧计时显示 correct 的 GPU ms（三后端都有值，不为 0）。

### M1 · Slang 编译与热重载（1 周）

任务：

1. `src/rhi/slang/`：`Session` → `IModule` → entry point → 目标代码（HLSL/SPIR-V/MSL）。
2. 反射 → 生成 `BindingLayoutDesc` + `ShaderObject` 参数块，**不手写任何 binding 索引**。
3. 磁盘缓存（源码 hash → 序列化产物），热重载（文件监听 + 原子替换）。
4. 画：三角形 + 全屏 quad（验证 uniform/纹理/sampler 三类绑定）。

验收：

- [ ] 同一份 `forward.slang` 在 D3D12/Vulkan/Metal 上产生视觉一致的结果（截图 diff）。
- [ ] 改 shader 存盘到生效 < 300ms，不崩、不掉帧。
- [ ] 有编译错误时，错误信息含文件/行号/列号，且不破坏上一版程序。

### M2 · Mesh 最小闭环（2~3 周）

任务：

1. ECS + Transform 层级 + `Extract`（先单线程，接口预留多线程）。
2. glTF 导入（mesh/material/texture），纹理走离线 BC7/BC5 烘焙 + mip。
3. Forward + PBR（metallic-roughness）+ 方向光 + CSM 阴影。
4. 相机控制 + ImGui 面板（灯光/材质参数实时改）。

验收：

- [ ] 加载 Sponza，三后端视觉一致。
- [ ] CPU 帧 < 3ms（不含 GPU），GPU 各 pass 时间可见。
- [ ] `src/render/` 下 grep 不到 `entt`。

### M3 · RenderGraph + 现代光照（3~4 周）

任务：

1. RenderGraph 落地（§6.4 全部能力），M2 的所有 pass 迁入。
2. Clustered Forward（PC only 首选：透明友好、无需 G-buffer 带宽）或 Deferred + light culling。
3. 后处理：tonemap（AgX 或 ACES）、bloom、TAA（含 velocity）。
4. **TBDR 优化项**：attachment `DontCare` 应用、pass 合并、Metal 上验证带宽收益。

验收：

- [ ] 1000 点光源 + 100 阴影投射体稳定运行。
- [ ] 重排 pass 顺序只改图代码，不改任何 pass 实现。
- [ ] 三后端截图差异 < 阈值（写进 `tests/`）。

### M4 · 3DGS（3~5 周，分两小步）

**M4a · 正确性优先（CPU 排序）1~2 周**

1. `tools/splat_convert`：`.ply` → `.tsplat`（位置 fp32、SH 系数分档打包、opacity、
   scale、rot quat、AABB、预计算 `Σ` 与旋转），mmap 直读。
2. `compute: 视锥剔除 → 投影到 2D + 2×2 协方差 → 深度键`。
3. **CPU 端每帧排序**（`std::sort` on 64-bit key），上传索引。
4. `raster: 每 splat 一个 quad（4 顶点 + index buffer），按索引顺序 back-to-front blend`。
5. 与 mesh depth 合成（先 mesh 后 splat）。

验收：与参考 viewer 视觉对齐（PSNR/SSIM 记录基线），100k splat 可用。

**M4b · GPU 路径 2~3 周**

1. GPU 键值排序（radix sort，8-bit × 4 pass）或 **tile-based binning**（更省、更可扩展，推荐）。
   注意 Metal **没有原子 float 加法** → 直方图/累加用 `atomic_uint` 编码或 CAS 循环。
2. 写 indirect args → `drawIndexedIndirect`；**Metal 无 indirect count** → 走降级路径（§1.1-3）。
3. LOD / 分块裁剪（大场景 5M+ splat 必需）。

验收：

- [ ] 1M splat @1080p ≥ 60fps（RTX 4070 级），排序耗时单独可见。
- [ ] 三后端一致；Metal 上不因缺失 indirect count 而崩溃或数量错。

### M5 · 神经渲染（4~6 周，**必须先选最窄的一条路**）

按顺序推进，任一步失败不影响前面的成果：

1. **3DGS + MLP 着色**（推荐第一步）：每 splat 颜色由小 MLP 从（特征 + 视角）预测，
   替换 SH。与 M4 复用，2 周内可见"神经化"效果。
2. **hash-grid + 小 MLP 体积渲染**（instant-ngp 风格）：ray marching + 三线性特征插值 + fp16 GEMM。
3. **性能**：cooperative matrix（D3D12/Vulkan 有，**Metal 无对应暴露** → Apple 上退回
   `simdgroup_matrix` 手写或 fp16 dot + threadgroup 分块）。

配套（这条路径 80% 的摩擦在这里）：

- `tools/weights_export`（Python）：PyTorch 训练（gsplat / tiny-cuda-nn）→ **contiguous
  fp16/fp32 二进制 + JSON manifest**（层形状、激活、量化方式、精度）。
- Slang 侧权重的内存布局固定为 `StructuredBuffer<float16_t4>` 之类的显式约定，
  写进 `shaders/neural/README`。
- 逐层单测：先把单层 MLP 的 GPU 结果与 PyTorch 对齐（误差 < 1e-3），再叠深。

验收：

- [ ] 与 PyTorch 参考渲染 PSNR 差 < 0.5dB。
- [ ] 权重换了不用重编 shader（运行时上传）。

### M6 · 工程化（持续）

- pass 级 GPU 计时面板、RenderDoc 一键抓帧（`--capture-frame=N`）
- asset 热重载、场景序列化
- headless 离屏渲染 + 截图回归（三后端矩阵）
- 发布构建：shader 预编译缓存、包体、`--scene`/`--benchmark` 命令行

---

## 9. 早期必做的 Spike（每个 ≤2 天，失败就改方案）

| # | Spike | 为什么必须先验证 |
|---|---|---|
| 1 | 同一份 `.slang` 编出 HLSL + SPIR-V + **MSL** 并在三后端跑通 | 整个项目的技术前提；MSL 输出路径最容易踩坑 |
| 2 | compute 生成 indirect args → `drawIndexedIndirect` | 3DGS 的命脉；同时暴露 Metal 无 indirect count 的问题 |
| 3 | 多 pass compute 链 + barrier（UAV↔SRV / image layout / Metal 无 barrier） | 验证 `state_tracker` 设计是否成立 |
| 4 | fp16 GEMM（小矩阵）在 compute shader 里跑通并计时 | 神经渲染的命脉；同时量出 coop matrix 收益 |
| 5 | splat 与 mesh 的 depth 正确合成（互相遮挡） | 决定 M4 是否需要 per-tile 深度缓冲 |
| 6 | **TBDR 验证**（Metal）：attachment `LoadAction=DontCare` 的带宽/耗时差异 | 决定 RenderGraph 的 `loadOp/storeOp` 是否值得做成核心概念（结论：值得） |
| 7 | Metal 上 float 原子缺失的替代路径（tile 直方图） | 决定 M4b 排序方案 |

这 7 个 spike 全过 → 项目没有"方向性"风险，剩下全是工程量。

---

## 10. 风险清单（按杀伤力排序）

1. **slang-rhi API 演进 + 文档极少。**
   对策：锁 commit、把上游 `slang-rhi.h` 快照进 `docs/upstream/`、封装隔离、升级走独立 PR 并跑截图回归。
2. **校验弱，错误静默。**
   对策：所有资源转移强制走 tracker；debug 包装里存名字 + 创建调用栈；三后端校验层默认开。
3. **显式状态写错（尤其 Vulkan layout）导致画面"偶尔不对"。**
   对策：RenderGraph + tracker + RenderDoc 抓帧；不做"看起来对"就提交。
4. **Metal 与 D3D12/Vulkan 的能力差**（无 indirect count、无 float 原子、无 coop matrix、TBDR）。
   对策：抽象层从一开始就取"三后端交集"，降级路径写在 RHI 内部而不是业务代码里。
5. **3DGS 排序性能**（5M splat 全局排序必挂）。
   对策：CPU 先正确 → tile binning 优先于全局 radix → 必须有 LOD/裁剪。
6. **神经渲染的权重/精度对齐**。
   对策：导出工具 + 逐层单测 + 运行时上传（不编译进 shader）。
7. **ECS 与渲染耦合**。
   对策：`Extract` 硬隔断 + CI grep 规则 + Code Review 驳回。
8. **Slang 编译期慢 / 变体爆炸**。
   对策：磁盘缓存、`[SpecializationConstant]` 替代宏组合、并行编译（多线程 slangc session）。
9. **macOS 上 GPU 计时的实现差异**（`MTLCounterSampleBuffer`）。
   对策：M0 就实现，别留到性能优化阶段。

---

## 11. 第一天 Checklist（可直接照做）

1. `git init ceruse`；创建 `docs/adr/0001-slang-rhi.md`，记录三条取舍：为什么 slang-rhi、
   为什么不要 WebGPU、为什么包含 macOS/Metal。
2. 写 `CMakeLists.txt` + `CMakePresets.json`，`FetchContent` 拉 slang-rhi **并锁 commit**；
   目标：只编通上游 `hello-triangle`。
3. 把 `slang-rhi.h` 关键类型抄成一页笔记放 `docs/upstream/slang-rhi-notes.md`
   （`IDevice/ICommandQueue/ICommandBuffer/ITexture/IBuffer/IBindingLayout/IShaderObject/IRenderPipeline`）。
4. 建 `src/rhi/device.h/.cpp` 第一版封装（**不泄漏 slang-rhi 头文件**）。
5. 建 `src/rhi/state_tracker.h` 空壳 + 断言骨架。
6. 接 Tracy + GPU timestamp（三后端各一份实现，先只做 D3D12）。
7. `src/ecs/world.h` 包一层 entt（渲染侧永远拿不到 `registry`）。
8. 建 `shaders/common/types.slang`（`float3` 别名、上下文结构、binding 约定注释）。
9. 建 `docs/platform-matrix.md`，把 §1 与 §1.1 的三条硬约束写进去。
10. 跑 Spike-1（同一 `.slang` → HLSL/SPIR-V/MSL），当天出结论。

---

## 12. 参考与术语

- `slang-rhi` 上游：头文件即文档（`include/slang-rhi.h`）+ `examples/`。
- Slang 反射 API：`slang::IModule::getReflection()`、`ProgramLayout`、`TypeLayout`。
- 3DGS 原始实现：Kerbl et al., *3D Gaussian Splatting for Real-Time Radiance Field Rendering* (SIGGRAPH 2023)。
- instant-ngp：Müller et al., *Instant Neural Graphics Primitives with a Multiresolution Hash Encoding* (2022)。
- TBDR 优化：Apple《Metal Best Practices》与 WWDC "Metal 中渲染 pass 的 load/store action"。

---

## 附录 A · CMake 骨架（示意）

```cmake
cmake_minimum_required(VERSION 3.24)
project(ceruse LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

include(FetchContent)

FetchContent_Declare(slang_rhi
  GIT_REPOSITORY https://github.com/shader-slang/slang-rhi.git
  GIT_TAG        <PINNED_SHA>        # 禁止 main
)
FetchContent_MakeAvailable(slang_rhi)

add_library(cer_core   STATIC ...)
add_library(cer_rhi    STATIC ...)   # 唯一 link slang_rhi 的 target
add_library(cer_render STATIC ...)
target_link_libraries(cer_rhi PRIVATE slang-rhi::slang-rhi)
add_executable(ceruse ...)
target_link_libraries(ceruse PRIVATE cer_core cer_rhi cer_render)
```

## 附录 B · 禁止事项速查（贴在新仓库 README 顶部）

- ❌ `src/render/**` include `entt/*` 或 `ecs/*`
- ❌ `src/render/**`、`src/ecs/**` include `slang-rhi.h`
- ❌ 业务代码手写 `setTextureState` / `globalBarrier`
- ❌ 跟 slang-rhi `main` 分支
- ❌ 一个 splat 一个 entity
- ❌ 一个 shader 用 `#define` 打变体
- ❌ 未开校验层就调 pass
