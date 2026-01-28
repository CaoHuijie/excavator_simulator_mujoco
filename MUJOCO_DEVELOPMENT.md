# MuJoCo 开发文档 (MuJoCo Development Documentation)

本文档详细介绍了 excavator_simulator_mujoco 项目中 MuJoCo 物理引擎的定制、集成和开发过程。

This document provides detailed information about the customization, integration, and development of the MuJoCo physics engine in the excavator_simulator_mujoco project.

## 目录 (Table of Contents)

1. [概述 (Overview)](#概述-overview)
2. [MuJoCo 定制版本 (Custom MuJoCo Version)](#mujoco-定制版本-custom-mujoco-version)
3. [自定义可执行文件 (Custom Executable)](#自定义可执行文件-custom-executable)
4. [插件系统集成 (Plugin System Integration)](#插件系统集成-plugin-system-integration)
5. [HField 可视化更新 (HField Visualization Update)](#hfield-可视化更新-hfield-visualization-update)
6. [开发指南 (Development Guide)](#开发指南-development-guide)
7. [高级主题 (Advanced Topics)](#高级主题-advanced-topics)

---

## 概述 (Overview)

### 为什么需要定制 MuJoCo？(Why Customize MuJoCo?)

excavator_simulator_mujoco 项目使用了一个定制版本的 MuJoCo，主要是为了：

1. **动态 HField 更新**: 标准 MuJoCo 可执行文件不支持在运行时更新 HField（高度场）
2. **改进土壤可视化**: 提供更好的浮动土壤可视化效果
3. **插件状态管理**: 实现插件与渲染循环之间的状态同步
4. **自定义物理循环**: 集成土壤动力学与 MuJoCo 的物理步进

### 定制内容摘要 (Customization Summary)

| 组件 | 修改内容 | 目的 |
|------|---------|------|
| `main.cc` | 添加 HField 更新逻辑 | 支持动态地形 |
| 可执行文件 | 自定义物理循环 | 集成土壤插件 |
| 插件系统 | 状态管理机制 | 控制可视化更新 |
| MuJoCo 库 | 改进浮动渲染 | 更好的视觉效果 |

---

## MuJoCo 定制版本 (Custom MuJoCo Version)

### 版本信息 (Version Information)

- **仓库**: https://github.com/KennyVilella/mujoco
- **分支**: `main`
- **基于**: MuJoCo 官方版本
- **主要改进**: 改善浮动土壤的可视化

### 核心修改 (Core Modifications)

#### 1. 浮动土壤可视化改进

定制的 MuJoCo 版本改进了浮动几何体（如铲斗中的土壤）的渲染方式。这包括：

```
改进的浮动体渲染:
- 更好的深度排序
- 改进的透明度处理
- 优化的 HField 网格渲染
- 减少视觉伪影
```

#### 2. HField 动态更新支持

标准 MuJoCo 将 HField 视为静态几何体。定制版本允许：

```cpp
// 运行时修改 HField 数据
m->hfield_data[index] = new_height_value;

// 通知渲染器需要更新
sim.UpdateHField(hfield_id);
```

### 构建定制 MuJoCo (Building Custom MuJoCo)

定制的 MuJoCo 版本通过 CMake 的 `FindOrFetch` 机制自动集成：

```cmake
findorfetch(
  USE_SYSTEM_PACKAGE OFF
  PACKAGE_NAME mujoco
  LIBRARY_NAME mujoco
  GIT_REPO https://github.com/KennyVilella/mujoco.git
  GIT_TAG main
  TARGETS mujoco::mujoco mujoco::platform_ui_adapter
  EXCLUDE_FROM_ALL
)
```

这会自动：
1. 检查系统是否已有 MuJoCo
2. 如果没有，克隆定制版本
3. 构建必要的库
4. 链接到项目

---

## 自定义可执行文件 (Custom Executable)

### 架构概览 (Architecture Overview)

```
┌───────────────────────────────────────────────────┐
│         Excavator Simulator Executable            │
├───────────────────────────────────────────────────┤
│                                                    │
│  ┌─────────────────────────────────────────────┐ │
│  │         Main Thread (Rendering)              │ │
│  │  • OpenGL 渲染循环                           │ │
│  │  • UI 事件处理                               │ │
│  │  • 场景更新                                  │ │
│  │  • HField 可视化刷新                         │ │
│  └─────────────────────────────────────────────┘ │
│                                                    │
│  ┌─────────────────────────────────────────────┐ │
│  │        Physics Thread (Simulation)           │ │
│  │  • 物理步进 (mj_step)                        │ │
│  │  • 插件计算                                  │ │
│  │  • 碰撞检测                                  │ │
│  │  • 状态管理                                  │ │
│  └─────────────────────────────────────────────┘ │
│                                                    │
│  ┌─────────────────────────────────────────────┐ │
│  │      Plugin System (Dynamic Loading)         │ │
│  │  • 插件扫描和加载                            │ │
│  │  • 插件注册                                  │ │
│  │  • 插件生命周期管理                          │ │
│  └─────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────┘
```

### 主要组件 (Main Components)

#### 1. main.cc 文件结构

**文件**: `excavator_simulator/main.cc`

```cpp
// 主要函数和功能

// 1. 插件处理 (Plugin Handling)
std::string getExecutableDir()           // 获取可执行文件目录
void scanPluginLibraries()               // 扫描和加载插件

// 2. 模型加载 (Model Loading)
mjModel* LoadModel(const char* file, mj::Simulate& sim)

// 3. 物理循环 (Physics Loop)
void PhysicsLoop(mj::Simulate& sim)     // 主物理仿真循环

// 4. 物理线程 (Physics Thread)
void PhysicsThread(mj::Simulate* sim, const char* filename)

// 5. 主函数 (Main Function)
int main(int argc, const char** argv)
```

#### 2. 插件自动加载机制

```cpp
void scanPluginLibraries() {
    // 1. 检查内置插件
    int nplugin = mjp_pluginCount();
    if (nplugin) {
        std::printf("Built-in plugins:\n");
        for (int i = 0; i < nplugin; ++i) {
            std::printf("    %s\n", mjp_getPluginAtSlot(i)->name);
        }
    }

    // 2. 从 mujoco_plugin 目录加载插件
    const std::string plugin_dir = getExecutableDir() + sep + MUJOCO_PLUGIN_DIR;
    mj_loadAllPluginLibraries(plugin_dir.c_str(), 
        +[](const char* filename, int first, int count) {
            std::printf("Plugins registered by library '%s':\n", filename);
            for (int i = first; i < first + count; ++i) {
                std::printf("    %s\n", mjp_getPluginAtSlot(i)->name);
            }
        });
}
```

**插件目录结构**:
```
build/bin/
├── excavator_simulator          # 可执行文件
└── mujoco_plugin/               # 插件目录
    └── libsoil.so               # 土壤插件库
```

#### 3. 物理循环实现

```cpp
void PhysicsLoop(mj::Simulate& sim) {
    // 声明土壤插件相关变量
    int terrain_id;
    int bucket_soil_1_id;
    int bucket_soil_2_id;
    int soil_id;
    bool soil_plugin;

    while (!sim.exitrequest.load()) {
        // 处理模型加载请求...
        
        // 检查土壤插件是否可用
        terrain_id = mj_name2id(m, mjOBJ_HFIELD, "terrain");
        bucket_soil_1_id = mj_name2id(m, mjOBJ_HFIELD, "bucket soil 1");
        bucket_soil_2_id = mj_name2id(m, mjOBJ_HFIELD, "bucket soil 2");
        soil_id = mj_name2id(m, mjOBJ_PLUGIN, "terrain");
        
        if ((terrain_id != -1) && (soil_id != -1)) {
            soil_plugin = true;
            // 验证插件状态大小
            int spec = mjSTATE_PLUGIN;
            int size = mj_stateSize(m, spec);
            if (size != 1) {
                mju_warning("Too many plugin state, disabling visual update");
                soil_plugin = false;
            }
        } else {
            soil_plugin = false;
        }

        // 物理步进
        {
            const std::unique_lock<std::recursive_mutex> lock(sim.mtx);
            if (m && sim.run) {
                // 执行物理步进
                mj_step(m, d);
            } else if (m) {
                // 暂停时仅更新前向运动学
                mj_forward(m, d);
            }
        }

        // 检查是否需要更新 HField 可视化
        if (soil_plugin) {
            int spec = mjSTATE_PLUGIN;
            int size = mj_stateSize(m, spec);
            std::vector<mjtNum> soil_state(size);
            mj_getState(m, d, soil_state.data(), spec);

            // 如果插件请求更新
            if (soil_state[0] == 1.0) {
                sim.UpdateHField(terrain_id);
                sim.UpdateHField(bucket_soil_1_id);
                sim.UpdateHField(bucket_soil_2_id);
            }
        }
    }
}
```

### 关键技术特性 (Key Technical Features)

#### 1. 线程安全

物理循环和渲染循环在不同的线程中运行，使用互斥锁保护共享数据：

```cpp
// 物理线程中锁定数据访问
{
    const std::unique_lock<std::recursive_mutex> lock(sim.mtx);
    if (m) {
        mj_step(m, d);  // 受保护的物理步进
    }
}  // 自动释放锁
```

#### 2. 时间同步

实现 CPU 时间与仿真时间的同步：

```cpp
// CPU-仿真同步点
std::chrono::time_point<mj::Simulate::Clock> syncCPU;
mjtNum syncSim = 0;

// 计算经过的时间
const auto elapsedCPU = startCPU - syncCPU;
double elapsedSim = d->time - syncSim;

// 检测不对齐
bool misaligned = mju_abs(Seconds(elapsedCPU).count()/slowdown - elapsedSim) 
                  > syncMisalign;

// 根据需要重新同步
if (misaligned) {
    syncCPU = startCPU;
    syncSim = d->time;
}
```

#### 3. 控制噪声注入

支持向控制输入注入 Ornstein-Uhlenbeck 噪声：

```cpp
if (sim.ctrl_noise_std) {
    // 转换率和比例为离散时间（Ornstein-Uhlenbeck）
    mjtNum rate = mju_exp(-m->opt.timestep / mju_max(sim.ctrl_noise_rate, mjMINVAL));
    mjtNum scale = sim.ctrl_noise_std * mju_sqrt(1-rate*rate);

    for (int i = 0; i < m->nu; i++) {
        // 更新噪声
        ctrlnoise[i] = rate * ctrlnoise[i] + scale * mju_standardNormal(nullptr);
        // 应用噪声
        d->ctrl[i] = ctrlnoise[i];
    }
}
```

---

## 插件系统集成 (Plugin System Integration)

### MuJoCo 插件架构

```
┌─────────────────────────────────────────────────────┐
│              MuJoCo Plugin System                    │
├─────────────────────────────────────────────────────┤
│                                                       │
│  ┌────────────────────────────────────────────────┐ │
│  │        Plugin Registration (mjpPlugin)         │ │
│  │  • name: "mujoco.soil"                         │ │
│  │  • capabilityflags: mjPLUGIN_PASSIVE           │ │
│  │  • nattribute: 5 (参数数量)                   │ │
│  │  • nstate: 1 (状态数量)                        │ │
│  └────────────────────────────────────────────────┘ │
│                                                       │
│  ┌────────────────────────────────────────────────┐ │
│  │           Plugin Callbacks                     │ │
│  │  • init()      - 初始化插件实例               │ │
│  │  • destroy()   - 清理插件实例                 │ │
│  │  • compute()   - 每步计算                     │ │
│  │  • reset()     - 重置插件（可选）             │ │
│  └────────────────────────────────────────────────┘ │
│                                                       │
│  ┌────────────────────────────────────────────────┐ │
│  │         Plugin Instance Data                   │ │
│  │  • d->plugin_data[instance]                    │ │
│  │    存储插件实例指针                            │ │
│  │  • d->plugin_state[offset]                     │ │
│  │    存储插件状态数据                            │ │
│  └────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

### 插件生命周期详解

#### 1. 注册阶段

```cpp
void Soil::RegisterPlugin() {
    mjpPlugin plugin;
    mjp_defaultPlugin(&plugin);

    // 设置插件属性
    plugin.name = "mujoco.soil";
    plugin.capabilityflags |= mjPLUGIN_PASSIVE;  // 被动插件（不施加力）

    // 定义参数
    const char* attributes[] = {
        "cell_size_z", "repose_angle", "max_iterations", 
        "cell_buffer", "amp_noise"
    };
    plugin.nattribute = sizeof(attributes) / sizeof(attributes[0]);
    plugin.attributes = attributes;

    // 定义状态大小
    plugin.nstate = +[](const mjModel* m, int instance) { return 1; };

    // 设置回调函数
    plugin.init = /* ... */;
    plugin.destroy = /* ... */;
    plugin.compute = /* ... */;

    // 注册到 MuJoCo
    mjp_registerPlugin(&plugin);
}
```

#### 2. 初始化阶段

```cpp
plugin.init = +[](const mjModel* m, mjData* d, int instance) {
    // 创建插件实例
    auto* Soil = Soil::Create(m, d, instance);
    if (!Soil) {
        return -1;  // 初始化失败
    }
    
    // 将实例指针存储在 MuJoCo 数据中
    d->plugin_data[instance] = reinterpret_cast<uintptr_t>(Soil);
    return 0;  // 成功
};
```

#### 3. 计算阶段

```cpp
plugin.compute = +[](const mjModel* m, mjData* d, int instance, int capability_bit) {
    // 检索插件实例
    auto* Soil = reinterpret_cast<class Soil*>(d->plugin_data[instance]);
    
    // 执行插件计算
    Soil->Compute(m, d, instance);
};
```

这个回调在每个物理步骤中被调用，由 `mj_step()` 或 `mj_forward()` 触发。

#### 4. 销毁阶段

```cpp
plugin.destroy = +[](mjData* d, int instance) {
    // 删除插件实例
    delete reinterpret_cast<Soil*>(d->plugin_data[instance]);
    d->plugin_data[instance] = 0;
};
```

### XML 配置

在 MuJoCo XML 模型中配置插件：

```xml
<mujoco>
  <extension>
    <plugin plugin="mujoco.soil">
      <instance name="terrain">
        <config key="cell_size_z" value="0.01"/>
        <config key="repose_angle" value="0.85"/>
        <config key="max_iterations" value="3"/>
        <config key="cell_buffer" value="4"/>
        <config key="amp_noise" value="50.0"/>
      </instance>
    </plugin>
  </extension>
  
  <!-- 其他模型定义 -->
</mujoco>
```

---

## HField 可视化更新 (HField Visualization Update)

### HField 系统概述

MuJoCo 的 HField (Height Field) 是一种高效的地形表示方法：

```
HField 数据结构:
┌──────────────────────────────────┐
│  nrow × ncol 高度值网格           │
│  每个值范围: [-1, 1]              │
│  实际高度 = value × size[2]       │
└──────────────────────────────────┘

属性:
• m->hfield_nrow[id]    : 行数
• m->hfield_ncol[id]    : 列数
• m->hfield_size[4*id]  : X 方向大小
• m->hfield_size[4*id+1]: Y 方向大小
• m->hfield_size[4*id+2]: Z 方向大小（高度范围）
• m->hfield_data[offset]: 高度数据数组
```

### 更新机制实现

#### 1. 插件状态标志

使用 MuJoCo 的插件状态系统来标记是否需要更新：

```cpp
// 在土壤插件中（soil.cc）
if (soil_update) {
    // 设置标志以允许可视化更新
    int spec = mjSTATE_PLUGIN;
    int size = mj_stateSize(m, spec);
    std::vector<mjtNum> soil_state(size);
    soil_state[0] = 1.0;  // 1.0 表示需要更新
    mj_setState(m, d, soil_state.data(), spec);
    
    // 更新 HField 数据...
}
```

#### 2. 主程序中检查更新

```cpp
// 在主程序中（main.cc）
if (soil_plugin) {
    // 读取插件状态
    int spec = mjSTATE_PLUGIN;
    int size = mj_stateSize(m, spec);
    std::vector<mjtNum> soil_state(size);
    mj_getState(m, d, soil_state.data(), spec);

    // 如果插件请求更新
    if (soil_state[0] == 1.0) {
        // 更新所有相关的 HField
        sim.UpdateHField(terrain_id);
        sim.UpdateHField(bucket_soil_1_id);
        sim.UpdateHField(bucket_soil_2_id);
    }
}
```

#### 3. 更新频率控制

为了性能，HField 只在必要时更新：

```cpp
// 土壤模拟器只在铲斗移动超过一个单元格大小时返回 true
bool soil_update = sim.Step(sim_out, pos, ori, grid, bucket, sim_param, 1e-5);
```

### 性能优化策略

```
更新策略层次结构:
┌─────────────────────────────────────┐
│ 1. 铲斗位置检查                      │
│    └─> 移动 < cell_size? 跳过       │
└─────────────────────────────────────┘
           ↓
┌─────────────────────────────────────┐
│ 2. 土壤动力学计算                    │
│    └─> 仅在活动区域计算             │
└─────────────────────────────────────┘
           ↓
┌─────────────────────────────────────┐
│ 3. HField 数据更新                   │
│    └─> 更新所有受影响的单元格        │
└─────────────────────────────────────┘
           ↓
┌─────────────────────────────────────┐
│ 4. 可视化刷新                        │
│    └─> 渲染线程更新 OpenGL 缓冲区    │
└─────────────────────────────────────┘
```

### 数据一致性保证

```cpp
// 使用互斥锁确保线程安全
{
    const std::unique_lock<std::recursive_mutex> lock(sim.mtx);
    
    // 1. 物理步进（更新土壤）
    mj_step(m, d);
    
    // 2. 这会调用插件的 compute() 回调
    //    插件更新 m->hfield_data[]
    
}  // 释放锁

// 3. 在物理循环外部检查更新标志
if (soil_state[0] == 1.0) {
    // 4. 通知渲染系统
    sim.UpdateHField(terrain_id);
}
```

---

## 开发指南 (Development Guide)

### 构建系统配置

#### CMakeLists.txt 结构

```cmake
# 主 CMakeLists.txt
cmake_minimum_required(VERSION 3.16)
project(excavator_simulator_mujoco VERSION 0.0.1)

# 查找或获取 MuJoCo
findorfetch(
  USE_SYSTEM_PACKAGE OFF
  PACKAGE_NAME mujoco
  LIBRARY_NAME mujoco
  GIT_REPO https://github.com/KennyVilella/mujoco.git
  GIT_TAG main
  TARGETS mujoco::mujoco mujoco::platform_ui_adapter
  EXCLUDE_FROM_ALL
)

# 添加子目录
add_subdirectory(plugin/soil)          # 构建土壤插件
add_subdirectory(excavator_simulator)  # 构建自定义可执行文件
```

#### excavator_simulator/CMakeLists.txt

```cmake
# 创建可执行文件
add_executable(excavator_simulator main.cc)

# 链接 MuJoCo 库
target_link_libraries(excavator_simulator
  PRIVATE
    mujoco::mujoco
    mujoco::platform_ui_adapter
    Threads::Threads
)

# 设置 C++ 标准
set_property(TARGET excavator_simulator PROPERTY CXX_STANDARD 17)
```

### 开发工作流程

#### 1. 修改可执行文件

```bash
# 1. 编辑 main.cc
vim excavator_simulator/main.cc

# 2. 重新构建
cmake --build build --target excavator_simulator

# 3. 复制到 bin 目录
cp build/excavator_simulator/excavator_simulator build/bin/

# 4. 测试
./build/bin/excavator_simulator model/excavator/excavator.xml
```

#### 2. 调试插件加载

```cpp
// 在 scanPluginLibraries() 中添加调试输出
void scanPluginLibraries() {
    std::printf("Executable directory: %s\n", getExecutableDir().c_str());
    std::printf("Plugin directory: %s\n", plugin_dir.c_str());
    
    int nplugin = mjp_pluginCount();
    std::printf("Total plugins loaded: %d\n", nplugin);
    
    // ... 其余代码
}
```

#### 3. 性能分析

```cpp
// 添加计时代码
#include <chrono>

void PhysicsLoop(mj::Simulate& sim) {
    auto start = std::chrono::high_resolution_clock::now();
    
    // 物理步进
    mj_step(m, d);
    
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
    
    std::printf("Physics step time: %ld µs\n", duration.count());
}
```

### 常见问题排查

#### 问题 1: 插件未加载

**症状**: 启动时看不到 "Plugins registered by library" 消息

**解决方案**:
```bash
# 检查插件目录
ls -la build/bin/mujoco_plugin/

# 应该看到 libsoil.so

# 检查动态链接依赖
ldd build/bin/mujoco_plugin/libsoil.so

# 确保所有依赖都能找到
```

#### 问题 2: HField 不更新

**症状**: 地形看起来是静态的，即使铲斗在移动

**解决方案**:
```cpp
// 添加调试输出
if (soil_plugin) {
    std::vector<mjtNum> soil_state(1);
    mj_getState(m, d, soil_state.data(), mjSTATE_PLUGIN);
    std::printf("Soil state: %f\n", soil_state[0]);
    
    if (soil_state[0] == 1.0) {
        std::printf("Updating HFields...\n");
        sim.UpdateHField(terrain_id);
        // ...
    }
}
```

#### 问题 3: 仿真运行缓慢

**可能原因和解决方案**:

1. **网格分辨率太高**
   ```xml
   <!-- 减小网格大小或增大单元尺寸 -->
   <config key="cell_size_z" value="0.02"/>  <!-- 从 0.01 增加到 0.02 -->
   ```

2. **迭代次数太多**
   ```xml
   <config key="max_iterations" value="2"/>  <!-- 从 3 减少到 2 -->
   ```

3. **更新太频繁**
   ```cpp
   // 增加更新阈值
   bool soil_update = sim.Step(sim_out, pos, ori, grid, bucket, sim_param, 2e-5);
   // 从 1e-5 增加到 2e-5
   ```

---

## 高级主题 (Advanced Topics)

### 多线程架构深入分析

#### 线程间通信

```
渲染线程 (Main Thread)           物理线程 (Physics Thread)
     │                                    │
     │   sim.exitrequest.load()          │
     │◄──────────────────────────────────│  设置退出标志
     │                                    │
     │   sim.droploadrequest.load()      │
     │◄──────────────────────────────────│  请求加载模型
     │                                    │
     │         sim.mtx (互斥锁)           │
     ├────────────────────────────────────┤  保护共享数据
     │                                    │
     │   sim.UpdateHField(id)             │
     │────────────────────────────────────┤  更新可视化
```

#### 原子操作的使用

```cpp
// 使用原子变量进行线程间通信
std::atomic<bool> exitrequest;
std::atomic<bool> droploadrequest;
std::atomic<int> uiloadrequest;

// 在物理线程中检查
while (!sim.exitrequest.load()) {
    // ...
}

// 在渲染线程中设置
if (exit_requested) {
    sim.exitrequest.store(true);
}
```

### 自定义物理步进器

可以修改物理循环以实现自定义行为：

```cpp
void PhysicsLoop(mj::Simulate& sim) {
    while (!sim.exitrequest.load()) {
        if (m && sim.run) {
            // === 自定义物理步进 ===
            
            // 1. 预处理
            PreProcessPhysics(m, d);
            
            // 2. MuJoCo 物理步进
            mj_step(m, d);
            
            // 3. 后处理（例如，自定义力计算）
            PostProcessPhysics(m, d);
            
            // 4. 检查约束违反
            CheckConstraints(m, d);
        }
    }
}

void PostProcessPhysics(mjModel* m, mjData* d) {
    // 示例：添加自定义外力
    for (int i = 0; i < m->nbody; i++) {
        // 计算自定义力
        mjtNum custom_force[3] = {0, 0, -9.81 * m->body_mass[i]};
        
        // 应用到身体
        mj_applyFT(m, d, custom_force, nullptr, 
                   d->xpos + 3*i, i, d->qfrc_applied);
    }
}
```

### 扩展插件系统

#### 添加新的插件类型

```cpp
// 1. 创建新插件类
class CustomPlugin {
  public:
    static CustomPlugin* Create(const mjModel* m, mjData* d, int instance);
    void Compute(const mjModel* m, mjData* d, int instance);
    static void RegisterPlugin();
};

// 2. 注册新插件
void CustomPlugin::RegisterPlugin() {
    mjpPlugin plugin;
    mjp_defaultPlugin(&plugin);
    
    plugin.name = "mujoco.custom";
    plugin.capabilityflags |= mjPLUGIN_ACTUATOR;  // 主动插件（施加力）
    
    // ... 设置回调
    
    mjp_registerPlugin(&plugin);
}

// 3. 在 main.cc 中加载
extern "C" void RegisterCustomPlugin();

int main(int argc, const char** argv) {
    // ...
    RegisterCustomPlugin();  // 注册自定义插件
    scanPluginLibraries();   // 扫描外部插件
    // ...
}
```

### 可视化增强

#### 自定义渲染

```cpp
// 在渲染循环中添加自定义绘制
void RenderCustomElements(mjvScene* scn, const mjModel* m, const mjData* d) {
    // 示例：绘制铲斗轨迹
    int bucket_id = mj_name2id(m, mjOBJ_BODY, "bucket");
    if (bucket_id != -1) {
        // 获取铲斗位置
        mjtNum* pos = d->xpos + 3*bucket_id;
        
        // 添加可视化几何体（如线条、点等）
        // ... 使用 mjv_* 函数
    }
}
```

### 性能监控和分析

```cpp
struct PerformanceMetrics {
    double physics_time;
    double plugin_time;
    double rendering_time;
    int hfield_updates;
};

class PerformanceMonitor {
  public:
    void StartPhysics() { physics_start = Clock::now(); }
    void EndPhysics() { 
        auto duration = Clock::now() - physics_start;
        metrics.physics_time = std::chrono::duration<double>(duration).count();
    }
    
    void PrintStats() {
        std::printf("Physics: %.3f ms, Plugin: %.3f ms, Rendering: %.3f ms\n",
                   metrics.physics_time * 1000,
                   metrics.plugin_time * 1000,
                   metrics.rendering_time * 1000);
    }
    
  private:
    using Clock = std::chrono::high_resolution_clock;
    Clock::time_point physics_start;
    PerformanceMetrics metrics;
};
```

---

## 最佳实践 (Best Practices)

### 1. 错误处理

```cpp
// 使用 MuJoCo 的错误处理机制
if (!model_loaded) {
    mju_error("Failed to load model: %s", filename);
    return nullptr;
}

// 对于警告
if (suspicious_condition) {
    mju_warning("Suspicious condition detected, continuing...");
}
```

### 2. 内存管理

```cpp
// 使用智能指针管理 MuJoCo 对象
struct ModelDeleter {
    void operator()(mjModel* m) const { mj_deleteModel(m); }
};
struct DataDeleter {
    void operator()(mjData* d) const { mj_deleteData(d); }
};

using ModelPtr = std::unique_ptr<mjModel, ModelDeleter>;
using DataPtr = std::unique_ptr<mjData, DataDeleter>;

// 使用
ModelPtr m(mj_loadXML(filename, nullptr, error, error_size));
if (m) {
    DataPtr d(mj_makeData(m.get()));
    // ... 使用 m 和 d
}  // 自动清理
```

### 3. 配置管理

```cpp
// 使用配置文件而不是硬编码
struct SimulatorConfig {
    std::string model_path;
    bool enable_wireframe;
    double physics_timestep;
    int max_iterations;
};

SimulatorConfig LoadConfig(const std::string& config_file) {
    // 从 JSON/YAML/INI 文件加载配置
    // ...
}
```

---

## 参考资料 (References)

### 官方文档

1. [MuJoCo 官方文档](https://mujoco.readthedocs.io/)
2. [MuJoCo 论坛](https://github.com/google-deepmind/mujoco/discussions)
3. [MuJoCo Python 绑定](https://mujoco.readthedocs.io/en/stable/python.html)

### 相关项目

1. [定制 MuJoCo 仓库](https://github.com/KennyVilella/mujoco)
2. [土壤动力学模拟器](https://github.com/KennyVilella/soil_dynamics_cpp)
3. [MuJoCo MPC](https://github.com/google-deepmind/mujoco_mpc)

### 学习资源

1. [MuJoCo 教程](https://mujoco.readthedocs.io/en/stable/tutorials.html)
2. [OpenGL 渲染](https://learnopengl.com/)
3. [C++ 并发编程](https://en.cppreference.com/w/cpp/thread)

---

## 贡献和支持 (Contributing and Support)

### 报告问题

如果您遇到问题，请：

1. 检查现有的 GitHub Issues
2. 提供完整的错误消息和堆栈跟踪
3. 包含您的系统信息（OS、编译器版本等）
4. 提供最小可复现示例

### 贡献代码

1. Fork 仓库
2. 创建功能分支
3. 遵循现有代码风格
4. 添加测试（如果适用）
5. 更新文档
6. 提交 Pull Request

---

**作者**: Vilella Kenny  
**维护者**: excavator_simulator_mujoco 项目团队  
**许可证**: 参见仓库 LICENSE 文件  
**最后更新**: 2026-01
