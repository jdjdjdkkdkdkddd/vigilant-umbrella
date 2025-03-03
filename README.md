# **1. Zygote 启动前（Magisk 阶段）**
- **Zygisk 预加载**  
  Zygisk 是 Magisk 的一部分，会在系统启动早期（**init 进程阶段**）被加载。Zygisk 通过替换系统原生库（如 `libart.so` 或 `libmemtrack.so`）注入代码，但保留原库功能，避免直接修改系统分区。
- **LSPosed 核心库挂载**  
  LSPosed 作为 Magisk 模块，其核心文件（如 `lspd.dex`、配置文件）会被挂载到系统分区中，但未激活。

---

### **2. Zygote 进程启动阶段**
- **Zygisk 注入 Zygote**  
  Android 系统启动时，Zygote 进程由 `init` 进程 fork 生成。Zygisk 此时通过修改后的系统库，将自身代码注入 Zygote 进程的初始化流程。
- **加载 LSPosed 核心库**  
  Zygisk 在 Zygote 进程中加载 LSPosed 的核心组件（如 `liblspd.so`），触发 LSPosed 的初始化逻辑：
  1. **初始化环境**：创建 LSPosed 的独立 ClassLoader，与系统 ClassLoader 隔离。
  2. **加载配置**：读取 `/data/adb/lspd` 下的配置文件，确定已启用的模块和作用域。
  3. **建立通信通道**：启动 LSPosed 的守护进程（`daemon`），与后续的 LSPosed 管理器（Manager）通信。

---

### **3. Zygote 子进程孵化阶段**
- **劫持应用进程创建**  
  Zygote 孵化出的每个应用进程（包括系统服务如 `system_server`）都会继承 LSPosed 的注入代码。此时 LSPosed 根据配置判断是否需要对该进程生效：
  - **系统服务进程（如 system_server）**：若模块需要 Hook 系统服务（如修改权限管理），LSPosed 会在此进程加载模块代码。
  - **普通应用进程**：仅当应用在模块作用域内时，LSPosed 才会动态加载对应模块。

- **按需加载模块**  
  对于目标应用进程，LSPosed 通过动态加载模块的 APK 或 DEX 文件，调用其 `handleLoadPackage` 方法，执行模块预设的 Hook 逻辑。

---

### **4. 系统启动完成阶段**
- **LSPosed 管理器启动**  
  当用户解锁设备进入桌面后，系统启动 `com.android.systemui` 和用户应用。LSPosed 管理器（如 `LSPosed` App）作为普通应用启动，通过 Binder 与守护进程通信，同步模块配置和状态。
- **动态生效新配置**  
  用户修改模块作用域或启用新模块时，LSPosed 无需重启即可通过守护进程动态更新 Hook 规则。
