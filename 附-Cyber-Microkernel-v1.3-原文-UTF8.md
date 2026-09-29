Cyber-Microkernel Runtime Architecture v1.3
可一劳永逸的跨平台桌面微内核架构设计
（含豆包两轮评审 + 落地评审三次修正 + 工程可观测性模块 + 分期落地裁剪）


============================================================
零、核心设计哲学
============================================================
主链只跑纯数学，外挂负责感知物理世界。
拒绝在核心逻辑里写死任何平台判断，不在运行期进行任何硬件型号的推测。
通过“物理探测 → 能力抽象 → 策略合并 → 工厂回退”的四段式启动模型，
让核心逻辑在任何硬件上都能自动匹配最优渲染路径，并在环境巨变时实现无重启热切换。

Probe 给事实。
Capability 给抽象。
Config 给策略。
Factory 选链并回退。
Runtime 只认接口，不认硬件，不判平台。
接口层禁止暴露 Qt 类型。
所有模块通过 ILogger 输出日志。
热路径最小化分配、零类型判断。


============================================================
一、全局分层架构树
============================================================

+----------------------------------------------------------+
| L0  Core（纯逻辑层，零平台感知）                          |
|   ├─ 状态机 / 动画                                       |
|   ├─ 固定时间步 + 插值                                   |
|   ├─ 场景图 / 显示列表                                   |
|   └─ 脏矩形 / RenderList                                 |
+----------------------------------------------------------+
                          ↓ RenderList / DirtyRegion

+----------------------------------------------------------+
| L1  Bootstrap（启动期，两段式探测）                       |
|   ├─ PreProbe（无 Qt 依赖）                              |
|   │    ├─ EnvironmentProbe                              |
|   │    ├─ CpuProbe                                      |
|   │    ├─ GpuProbe                                      |
|   │    └─ ClockProbe                                    |
|   │                                                      |
|   ├─ [QApplication 初始化]                               |
|   │                                                      |
|   ├─ PostProbe（Qt 初始化后）                            |
|   │    └─ DisplayProbe                                  |
|   │                                                      |
|   ├─ CapabilityResolver  → Capabilities                 |
|   │    ├─ 含沙箱权限裁剪分支                             |
|   │    └─ 含能力推导与降级策略                           |
|   ├─ RuntimeConfigResolver → RuntimeConfig              |
|   └─ BackendFactory      → BackendBundle                |
+----------------------------------------------------------+
                          ↓ BackendBundle（按回退链组装）

+----------------------------------------------------------+
| L2  Runtime（运行期，热路径）                             |
|   ├─ IRenderer          → 绘制与呈现                     |
|   ├─ IPlatformBackend   → 窗口/事件/vsync                |
|   ├─ FrameScheduler     → 帧同步调度                     |
|   ├─ InputRegion        → 命中与穿透                     |
|   └─ ScreenDpi          → 多屏与缩放                     |
+----------------------------------------------------------+
                          ↑ 事件驱动

+----------------------------------------------------------+
| L3  RuntimeCapabilityChanged（热切换层）                  |
|   ├─ 防抖（按 event_type 配置，见第四节）                  |
|   ├─ 增量重配 (Incremental Rebuild)                       |
|   └─ 必要时整链重建 (Full Rebuild)                        |
+----------------------------------------------------------+

+----------------------------------------------------------+
| L4  Observability（可观测性层，全模块共用）               |
|   ├─ ILogger          → 统一日志抽象                     |
|   ├─ Metrics          → 性能埋点采集（输出通道见 6.4）    |
|   └─ HealthCheck      → 后端低频心跳探测                  |
+----------------------------------------------------------+


============================================================
二、启动期数据结构契约（Bootstrap 层）
============================================================

2.1 ProbeResult（纯事实，不做决策）

@dataclass(frozen=True)
class ProbeResult:
    # 环境
    os_family: str              # linux / windows / darwin
    session_type: str           # x11 / wayland / win32 / quartz
    is_remote: bool             # SSH / 远程桌面
    is_sandbox: bool            # Flatpak / Snap / Sandboxie
    sandbox_type: str           # "flatpak" / "snap" / "sandboxie" / "none"
    sandbox_capabilities: frozenset[str]
                                # 沙箱允许的能力集，如 {"vulkan", "portal.display"}

    # CPU
    cpu_cores: int
    cpu_clock_hz: int           # 粗测主频
    clock_resolution_ns: int    # 系统最小调度粒度

    # GPU
    gpu_present: bool
    gpu_vendor: str             # nvidia / amd / intel / apple / none
    gpu_driver: str
    gpu_api_support: list[str]  # ["vulkan", "d3d11", "opengl", "metal"]

    # 显示（PostProbe 阶段填入）
    monitor_count: int
    primary_dpi_scale: float
    primary_refresh_hz: float

    # 时钟源
    steady_clock: bool
    high_res_clock: bool
    platform_timer: str         # "x11_present" / "dwm_flush" / "cvdisplaylink"


2.2 ProbeDiagnostics（诊断数据，独立于事实）

@dataclass(frozen=True)
class ProbeDiagnostics:
    total_duration_ms: float
    probe_timeout: bool
    sub_probe_status: dict[str, str]   # {"GpuProbe": "ok", "DisplayProbe": "timeout"}


说明：
* ProbeResult 只承载物理事实，纯输入。
* ProbeDiagnostics 承载耗时、超时、子探针状态，供日志与 Metrics 使用。
* 二者分离，CapabilityResolver 的输入更干净。


2.3 Capabilities（能力抽象，不涉及平台名称）

@dataclass(frozen=True)
class Capabilities:
    display_server_resolved: str    # "x11" / "wayland" / "dcomp" / "layered" / "quartz"
    gpu_candidate: bool
    gpu_api_hint: str               # "vulkan" / "d3d11" / "metal" / "software"
    partial_update_hint: bool
    region_input_hint: bool
    vsync_source_hint: str          # "x11_present" / "dwm_flush" / "cvdisplaylink"
    thread_budget: int
    refresh_hint: float
    dpi_hint: float


沙箱裁剪分支（修正版）：
* 若 ProbeResult.sandbox_type != "none"：
    - 优先尝试 GPU（Flatpak/Snap 可通过 portal 拿到 Vulkan/D3D12/GL）
    - 但要准备接受 gpu_driver_incompatible 或 permission_denied
    - 回退链中始终保留 raster 兜底
* 不要把沙箱直接等同于“禁用 GPU”。
* 裁剪规则以 CapabilityResolver 内的策略表实现。


2.4 RuntimeConfig（策略合并，引入用户偏好）

@dataclass(frozen=True)
class RuntimeConfig:
    renderer_preference: str       # "auto" / "gpu" / "cpu"
    frame_limit_policy: str        # "vsync" / "30fps" / "60fps" / "unlimited"
    oversample_scale: float        # 1.0 / 1.25 / 2.0
    worker_count: int              # 并行 Worker 数
    timer_policy: str              # "high_res" / "standard"
    input_region_policy: str       # "native" / "window_level" / "software"
    fallback_chain: list[str]      # 显式回退顺序


配置优先级（从高到低）：
  CLI 参数 > 环境变量 > 用户配置文件 > 硬件探测推导 > 内核默认值
  冲突时把最终决策写入日志。


2.5 BackendFactory（按回退链组装 + 错误分类）

class BackendError(Exception):
    error_code: str          # "gpu_driver_incompatible" / "x11_display_unreachable" ...
    recoverable: bool
    context: dict


def build_backend_bundle(probe, caps, cfg) -> BackendBundle:
    for backend_name in cfg.fallback_chain:
        try:
            platform = create_platform_backend(backend_name, probe, caps)
            renderer = create_renderer(cfg.renderer_preference, caps)
            scheduler = create_scheduler(caps.vsync_source_hint, cfg.timer_policy)
            bundle = BackendBundle(
                platform=platform,
                renderer=renderer,
                scheduler=scheduler,
                input_region=create_input_region(platform, caps),
                screen_dpi=create_screen_dpi(platform, probe),
                capabilities=caps,
            )
            post_probe(bundle)
            return bundle
        except BackendError as e:
            cleanup(backend_name)
            # 按 error_code 分类跳过，不互相牵连：
            #   gpu_*       → 跳过所有 GPU 后端
            #   display_*   → 跳过当前显示服务后端
            #   permission_* → 跳过同类权限受限后端
            if not e.recoverable:
                skip_related_backends(cfg.fallback_chain, e.error_code)
            continue
    raise RuntimeError("所有后端都创建失败")


回退链测试策略：
* 每个后端支持 FORCE_FAIL=1 环境变量。
* CI 通过 FORCE_FAIL 注入失败，验证回退链完整性。
* 这是架构级要求，不是测试细节。


============================================================
三、运行期主循环、线程模型与关闭时序
============================================================

3.1 主循环结构

VsyncSource（来自 IPlatformBackend）
    ↓
FrameScheduler
    ├─ 固定时间步累加
    ├─ Core::tick(fixed_dt)      ← 逻辑不在渲染线程跑
    ├─ 插值生成 RenderState
    └─ Renderer::render(RenderList)
    ↓
IRenderer::render → IPlatformBackend::present


热路径三大纪律：
1. 最小化分配：预分配缓冲区、复用对象、避免每帧创建 list/dict/临时字符串。
   （Python 里绝对零分配不现实；重点是避免无意义的每帧新对象。）
2. 零类型判断：每帧调用必须是直接函数引用（self._draw = backend.draw），
   绝不出现 isinstance 或 hasattr。
3. 逻辑与渲染分离：Core::tick 跑在 Worker 线程或主事件循环，
   Renderer::render 跑在渲染线程。两者通过 RenderList 通信。


3.2 线程安全规范

RenderList 通信协议（三选一）：
  ① 双缓冲 + 原子指针交换（SPSC，单生产者单消费者）—— 推荐
  ② 环形缓冲区 + 无锁队列
  ③ 锁 + 拷贝（简单但慢，仅低配回退路径）

后端资源所有权约束：
  · 交换链 / 句柄 / GPU 资源：仅渲染线程可持有
  · 窗口句柄：仅主线程可持有
  · 跨线程传递：只传 ID / 句柄副本，不传对象引用

热切换销毁时序：
  ① 暂停渲染线程
  ② 等待所有 Worker 任务 drain
  ③ 旧 Bundle 的 release() 被调用
  ④ 新 Bundle 注入
  ⑤ 恢复渲染线程


3.3 线程模型

主线程（平台事件循环）：窗口/表面生命周期、Vsync 分发、线程同步
渲染线程（可选）：QRhi 资源创建/销毁、绘制命令录制、present
Worker 线程池：场景图构建、布局/动画批处理、图像解码/缓存
FrameScheduler：只发“该渲染了”，不跑核心逻辑


3.4 优雅关闭（Shutdown 时序）

退出时严格反序执行：
  ① 停止 FrameScheduler（不再触发新帧）
  ② 等待当前 render 完成
  ③ 停止 Worker 线程池并 drain
  ④ IRenderer::release()
  ⑤ IPlatformBackend::release()
  ⑥ 关闭 ILogger / Metrics 输出

没有这一步，退出时大概率丢帧、崩溃或句柄泄漏。


============================================================
四、运行时热切换（RuntimeCapabilityChanged）
============================================================

触发条件（事件驱动，绝不轮询）：
* 多屏 DPI 变化
* 窗口跨屏
* GPU 设备丢失（TDR）
* Wayland compositor 能力变化
* 显示服务重启
* 用户修改配置


处理流程：

RuntimeCapabilityChanged
    ↓ 防抖（按 event_type 配置）
ReProbe（仅探测变化的部分）
    ↓
CapabilityResolver::update
    ↓
RuntimeConfigResolver::update
    ↓
区分两类：
  ├─ 增量重配（仅 DPI / 尺寸变化）
  └─ 整链重建（显示服务切换 / GPU 丢失）
        BackendFactory::rebuildOrSwitch
        → 旧 BackendBundle 安全销毁
        → 新 BackendBundle 热注入


防抖策略（按 event_type 区分）：
* DPI / 跨屏：50–100ms 防抖
* GPU 丢失（TDR）：立即处理，不防抖
* 显示服务重启：事件本身已足够晚，无需防抖


热切换铁律：
* 防抖按事件类型配置，不搞一刀切。
* 增量优先：能局部重配的绝不整链重建。
* 重建时核心状态机与动画时间轴持续运行。


============================================================
五、资源生命周期统一接口（V0.3 引入）
============================================================

class ILifecycle(Protocol):
    def acquire(self) -> None: ...
    def release(self) -> None: ...
    def is_alive(self) -> bool: ...


规则：
* 所有 Backend / Renderer / 资源句柄实现 ILifecycle。
* 热切换时通过引用计数确保资源在所有引用者释放后才真正回收。
* 高频切换场景使用异步释放，避免主线程阻塞。
* 显式 release() 优先于 __del__。
* 资源管理器配套弱引用缓存容器（weakref.WeakValueDictionary），
  切断 Python 循环引用链，避免 GC 延迟释放显存。
* Qt 资源生命周期以 release() 为准，Python 弱引用仅管理包装对象。


============================================================
六、可观测性层
============================================================

6.1 ILogger（统一日志抽象）

class ILogger(Protocol):
    def debug(self, tag: str, msg: str, **ctx) -> None: ...
    def info(self, tag: str, msg: str, **ctx) -> None: ...
    def warn(self, tag: str, msg: str, **ctx) -> None: ...
    def error(self, tag: str, msg: str, **ctx) -> None: ...

规则：
* 所有模块禁止直接 import logging，必须通过 ILogger 抽象。
* 默认实现提供控制台 + 文件双通道。
* 日志级别可通过 CLI 动态调整。
* 为未来跨语言移植预留接口。


6.2 Metrics（性能埋点）

采集指标：
* 启动耗时（PreProbe / PostProbe 分项）
* 每帧耗时（帧起点 → present 完成）
* 后端重建次数与耗时
* 资源句柄数量（GPU / 窗口 / 输入区域）
* RenderList 队列深度
* GC 触发频率与暂停时长


6.3 HealthCheck（后端低频心跳）

性质说明：
* “绝不轮询”针对的是能力变化事件（DPI、跨屏、GPU 丢失）。
* HealthCheck 是低频心跳探测，性质不同，用于发现“事件丢失”的后端异常。

检查频率：每 10 秒一次。

检查项：
* Renderer 是否仍能成功绘制
* PlatformBackend 的 vsync 是否仍在触发
* GPU 句柄是否被系统回收

异常时触发 RuntimeCapabilityChanged → 自动热切换。


6.4 Metrics 输出通道（V0.4 前必须定）

候选方案：
  · 本地日志文件（V0.2 起步）
  · 共享内存（跨进程可观测）
  · Prometheus / OpenTelemetry（V0.4 正式接入）

V0.2 只需把基础指标写入日志，不引入外部依赖。


============================================================
七、模块责任边界表
============================================================

| 模块 | 输入 | 输出 | 严禁做的事 |
|------|------|------|------------|
| PreProbe | 系统调用 | ProbeResult（部分） | 不做决策、不依赖 Qt |
| PostProbe | Qt 屏幕 API | ProbeResult（显示） | 不做决策 |
| Resolver | ProbeResult + 导入期常量 | Capabilities | 不读用户偏好、不选后端 |
| Config | Capabilities + 用户偏好 | RuntimeConfig | 不实例化后端 |
| Factory | RuntimeConfig | BackendBundle | 不在运行期被频繁调用 |
| IRenderer | RenderList | NativeSurface | 不判平台、不暴露 Qt 类型 |
| IPlatformBackend | 事件/vsync | NativeSurface | 不管渲染算法、不暴露 Qt 类型 |
| FrameScheduler | Vsync | tick 信号 | 不跑核心逻辑 |
| InputRegion | 鼠标事件 | 命中结果 | 不判平台 |
| ScreenDpi | 屏幕事件 | DPI/缩放 | 不重建后端（除非必要） |
| ILogger | 日志调用 | 输出流 | 不依赖 Qt、不依赖 Python logging |
| Metrics | 埋点事件 | 指标数据 | 不影响热路径性能 |


关于“编译期平台常量”：
* Python 没有编译期，改为“导入期常量 / 构建期平台标记”
  （sys.platform、打包时的 target）。


============================================================
八、落地任务清单（V0.2 / V0.3 / V0.4 分期）
============================================================

 #【V0.2 骨架期】
目标：最小 Cap可运行骨架 + 196 FPS 回归。

核心 4 文件：
  core/probe.py     # PreProbe + PostProbe 合并的同步版本
  core/resolver.py abilityResolver + 沙箱裁剪
  core/config.py    # RuntimeConfig + 优先级合并
  core/factory.py   # BackendFactory + 最简回退链

后端 2 文件：
  backends/linux_x11.py    # X11 原生穿透 + QPainter 光栅化
  backends/raster.py       # 纯 QPainter 光栅化，终极回退

必做：
  · 定义 ILogger / IRenderer / IPlatformBackend 接口
  · 接口层禁止暴露 Qt 类型
  · 沙箱裁剪基础版（放宽 GPU 假设）
  · 弱引用缓存基础版
  · 最基础帧耗时埋点（logging 输出）
  · 优雅关闭时序

不做：
  · ILifecycle（挪到 V0.3 与热切换一起引入）
  · 异步探测
  · Metrics 层
  · HealthCheck
  · RuntimeCapabilityChanged 热切换
  · QRhiRenderer / DCompBackend / WaylandBackend
  · 脱离 Qt 的独立底层


【V0.3 加固期】
目标：跨平台 + 热切换 + 可观测性初步。

范围：
  · ILifecycle 接口 + 引用计数
  · Win32LayeredBackend + MacBackend
  · RuntimeCapabilityChanged（事件驱动 + 按事件类型防抖）
  · 增量重配路径
  · BackendError 错误分类 + 智能跳过
  · 异步探测 + 超时保护
  · 完整帧耗时埋点
  · 弱引用缓存的引用计数扩展


【V0.4 成熟期】
目标：GPU 加速 + 完整可观测性 + 脱离 Qt 评估。

范围：
  · QRhiRenderer（GPU 加速）
  · Win32DCompBackend
  · WaylandBackend
  · Metrics 输出通道（Prometheus / OpenTelemetry）
  · HealthCheck 自动热切换触发
  · 评估脱离 Qt 依赖（SDL / 原生 Win32 / libinput）
  · 完成 ILogger 与 Metrics 的独立部署能力


============================================================
九、核心原则（写进代码最上方）
============================================================

Probe 给事实。
Capability 给抽象。
Config 给策略。
Factory 选链并回退。
Runtime 只认接口，不认硬件，不判平台。
接口层禁止暴露 Qt 类型。
所有模块通过 ILogger 输出日志。
热路径最小化分配、零类型判断。
V0.2 只跑最小骨架，不提前实现 V0.3+ 的模块。
退出时严格反序关闭，杜绝句柄泄漏。


============================================================
十、一句话结论
============================================================
v1.3 已经收敛到设计完整、落地清醒、措辞严谨的版本。
真正要动手的只有三件事：
1. 把“零分配”改成“最小化分配”；
2. 补 Shutdown 时序和回退链 FORCE_FAIL 测试策略；
3. V0.2 砍掉 ILifecycle，沙箱裁剪放宽 GPU 假设。
其余按分期落地即可。