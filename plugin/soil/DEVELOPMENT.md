# 土壤插件开发文档 (Soil Plugin Development Documentation)

本文档详细介绍了 excavator_simulator_mujoco 项目中土壤插件的开发过程、架构设计和技术实现。

This document provides detailed information about the development process, architecture design, and technical implementation of the soil plugin in the excavator_simulator_mujoco project.

## 目录 (Table of Contents)

1. [概述 (Overview)](#概述-overview)
2. [架构设计 (Architecture Design)](#架构设计-architecture-design)
3. [核心组件 (Core Components)](#核心组件-core-components)
4. [实现细节 (Implementation Details)](#实现细节-implementation-details)
5. [开发指南 (Development Guide)](#开发指南-development-guide)
6. [技术要点 (Technical Highlights)](#技术要点-technical-highlights)

---

## 概述 (Overview)

### 项目背景 (Project Background)

土壤插件是 excavator_simulator_mujoco 项目的核心组件之一，它提供了真实的土壤动力学模拟，使挖掘机能够与可变形的土壤进行交互。该插件基于 [soil_dynamics_cpp](https://github.com/KennyVilella/soil_dynamics_cpp) 土壤模拟器，并作为与 MuJoCo 物理引擎的接口。

The soil plugin is one of the core components of the excavator_simulator_mujoco project. It provides realistic soil dynamics simulation, enabling the excavator to interact with deformable soil. The plugin is based on the [soil_dynamics_cpp](https://github.com/KennyVilella/soil_dynamics_cpp) soil simulator and serves as an interface with the MuJoCo physics engine.

### 主要功能 (Main Features)

1. **动态土壤模拟** - 实时计算土壤的挖掘、运输和堆放
2. **HField 集成** - 与 MuJoCo 的 HField（高度场）系统无缝集成
3. **参数化配置** - 支持自定义土壤属性和网格参数
4. **性能优化** - 仅在必要时更新可视化，提高仿真效率

---

## 架构设计 (Architecture Design)

### 系统架构图 (System Architecture)

```
┌─────────────────────────────────────────────────────────┐
│                   MuJoCo Physics Engine                  │
│  ┌────────────────────────────────────────────────────┐ │
│  │           Custom Excavator Simulator               │ │
│  │              (excavator_simulator)                 │ │
│  └───────────────────┬────────────────────────────────┘ │
│                      │                                   │
│                      │ Plugin Interface                  │
│                      ▼                                   │
│  ┌────────────────────────────────────────────────────┐ │
│  │              Soil Plugin (mujoco.soil)             │ │
│  │  ┌──────────────────────────────────────────────┐ │ │
│  │  │  • Parameter Validation                      │ │ │
│  │  │  • HField Management (terrain, bucket_soil)  │ │ │
│  │  │  • Bucket Position/Orientation Tracking      │ │ │
│  │  │  • Visual Update Control                     │ │ │
│  │  └──────────────────────────────────────────────┘ │ │
│  └───────────────────┬────────────────────────────────┘ │
└────────────────────────┼────────────────────────────────┘
                         │ C++ API
                         ▼
┌─────────────────────────────────────────────────────────┐
│           Soil Simulator (soil_dynamics_cpp)             │
│  ┌────────────────────────────────────────────────────┐ │
│  │  • Grid-based Terrain Representation               │ │
│  │  • Soil Relaxation Algorithm                       │ │
│  │  • Bucket-Soil Interaction                         │ │
│  │  • Repose Angle Physics                            │ │
│  └────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

### 插件生命周期 (Plugin Lifecycle)

1. **注册阶段 (Registration)**: `Soil::RegisterPlugin()` 向 MuJoCo 注册插件
2. **初始化阶段 (Initialization)**: `Soil::Create()` 创建插件实例
3. **运行阶段 (Computation)**: `Soil::Compute()` 在每个仿真步骤中更新土壤状态
4. **销毁阶段 (Destruction)**: 自动清理资源

---

## 核心组件 (Core Components)

### 1. 插件主类 (Main Plugin Class)

**文件**: `soil.h`, `soil.cc`

```cpp
class Soil {
  public:
    static Soil* Create(const mjModel* m, mjData* d, int instance);
    void Compute(const mjModel* m, mjData* d, int instance);
    static void RegisterPlugin();
    
  private:
    soil_simulator::SoilDynamics sim;      // 土壤动力学核心
    soil_simulator::Grid grid;              // 网格配置
    soil_simulator::Bucket *bucket;         // 挖掘机铲斗
    soil_simulator::SimParam sim_param;     // 仿真参数
    soil_simulator::SimOut *sim_out;        // 输出数据
};
```

**主要职责**:
- 管理土壤仿真器的生命周期
- 维护 HField 的 ID 引用（terrain, bucket_soil_1, bucket_soil_2）
- 协调铲斗位置与土壤交互
- 控制可视化更新时机

### 2. 插件注册 (Plugin Registration)

**文件**: `register.cc`

这个简单但关键的文件使用 MuJoCo 的插件库初始化宏来注册土壤插件：

```cpp
namespace mujoco::plugin::soil {
    mjPLUGIN_LIB_INIT { Soil::RegisterPlugin(); }
}
```

### 3. 参数配置 (Parameter Configuration)

插件支持以下可配置参数：

| 参数名称 | 类型 | 默认值 | 描述 |
|---------|------|--------|------|
| `cell_size_z` | float | 0.01 | 网格单元的高度（米） |
| `repose_angle` | float | 0.85 | 土壤的休止角（弧度） |
| `max_iterations` | int | 3 | 土壤松弛的最大迭代次数 |
| `cell_buffer` | int | 4 | 活动区域周围的缓冲单元数 |
| `amp_noise` | float | 50.0 | 地形初始化的噪声幅度 |

---

## 实现细节 (Implementation Details)

### HField 管理策略 (HField Management Strategy)

该插件使用三个 HField 来表示不同的土壤层：

1. **terrain**: 主要地形表面
2. **bucket soil 1**: 铲斗中的第一层土壤
3. **bucket soil 2**: 铲斗中的第二层土壤

#### HField 数据布局

```
MuJoCo HField 数据结构:
m->hfield_data[0 ... n_terrain-1]                    : terrain
m->hfield_data[n_terrain ... 2*n_terrain-1]          : bucket soil 1
m->hfield_data[2*n_terrain ... 3*n_terrain-1]        : bucket soil 2
```

### 更新机制 (Update Mechanism)

#### 1. 插件状态管理

插件使用 MuJoCo 的 `mjSTATE_PLUGIN` 状态来控制 HField 的可视化更新：

```cpp
// 设置状态以允许可视化更新
std::vector<mjtNum> soil_state(1);
soil_state[0] = 1.0;  // 1.0 = 需要更新, -1.0 = 不需要更新
mj_setState(m, d, soil_state.data(), mjSTATE_PLUGIN);
```

#### 2. 选择性更新策略

```cpp
bool soil_update = sim.Step(sim_out, pos, ori, grid, bucket, sim_param, 1e-5);

if (soil_update) {
    // 仅当铲斗移动超过一个单元格大小时才更新 HField
    // 这大大提高了性能
}
```

### 铲斗坐标转换 (Bucket Coordinate Transformation)

MuJoCo 和土壤模拟器使用不同的坐标约定，需要进行转换：

```cpp
// MuJoCo -> Soil Simulator 坐标转换
std::vector<float> ori = {
    static_cast<float>(d->xquat[4*bucket_id]),      // w (不变)
    static_cast<float>(-d->xquat[4*bucket_id+1]),   // x (取反)
    static_cast<float>(-d->xquat[4*bucket_id+2]),   // y (取反)
    static_cast<float>(-d->xquat[4*bucket_id+3])    // z (取反)
};
```

### 土壤可视化技巧 (Soil Visualization Trick)

对于铲斗土壤的 HField，插件使用了一个巧妙的技巧：

```cpp
if ((sim_out->body_soil_[0][ii][jj] != 0.0) || 
    (sim_out->body_soil_[1][ii][jj] != 0.0)) {
    // 有土壤 - 正常更新高度
    m->hfield_data[n_hfield_terrain + new_index] = 
        sim_out->body_soil_[1][ii][jj] / m->hfield_size[2];
} else {
    // 无土壤 - 设置为 mjMAXVAL 使其不可见
    m->hfield_data[n_hfield_terrain + new_index] = mjMAXVAL;
}
```

这种方法通过将空单元设置为极大值来有效地"隐藏"它们。

---

## 开发指南 (Development Guide)

### 环境设置 (Environment Setup)

#### 依赖项 (Dependencies)

1. **MuJoCo** (自定义版本): https://github.com/KennyVilella/mujoco
2. **soil_dynamics_cpp** (v1.1.0): https://github.com/KennyVilella/soil_dynamics_cpp
3. **CMake** >= 3.16
4. **C++ 编译器**: clang++ 或 g++ (支持 C++17)

#### 构建步骤 (Build Steps)

```bash
# 1. 克隆仓库
git clone https://github.com/CaoHuijie/excavator_simulator_mujoco
cd excavator_simulator_mujoco

# 2. 运行构建脚本（自动处理所有依赖）
bash build.bash

# 3. 插件库将被构建到: build/plugin/soil/libsoil.so
# 4. 并自动复制到: build/bin/mujoco_plugin/libsoil.so
```

### 修改土壤插件 (Modifying the Soil Plugin)

#### 添加新参数

1. **在 `soil.cc` 中添加参数验证**:

```cpp
if (!CheckAttr("new_parameter", m, instance))
    mju_error("Soil plugin: Invalid ``new_parameter`` specification");
```

2. **在构造函数中读取参数**:

```cpp
mjtNum new_parameter = strtod(
    mj_getPluginConfig(m, instance, "new_parameter"), nullptr);
```

3. **在 `RegisterPlugin()` 中注册参数**:

```cpp
const char* attributes[] = {
    "cell_size_z", "repose_angle", "max_iterations", 
    "cell_buffer", "amp_noise", "new_parameter"  // 添加新参数
};
```

4. **在 XML 模型中使用**:

```xml
<config key="new_parameter" value="10.0"/>
```

#### 调试技巧 (Debugging Tips)

1. **启用详细日志**:
```cpp
std::cout << "Debug: bucket position = " << pos[0] << ", " 
          << pos[1] << ", " << pos[2] << std::endl;
```

2. **验证 HField 更新**:
```cpp
if (soil_update) {
    std::cout << "HField updated at time: " << d->time << std::endl;
}
```

3. **检查插件状态**:
```cpp
std::vector<mjtNum> state(1);
mj_getState(m, d, state.data(), mjSTATE_PLUGIN);
std::cout << "Plugin state: " << state[0] << std::endl;
```

### 性能优化建议 (Performance Optimization Tips)

1. **减小网格分辨率**: 增大 `cell_size_xy` 和 `cell_size_z`
2. **减少迭代次数**: 降低 `max_iterations`（但可能影响物理精度）
3. **优化缓冲区**: 减小 `cell_buffer`（在不影响边界效果的前提下）
4. **选择性更新**: 仅在铲斗移动显著时更新 HField（已实现）

---

## 技术要点 (Technical Highlights)

### 1. 插件回调机制 (Plugin Callback Mechanism)

MuJoCo 的插件系统使用 Lambda 函数作为回调：

```cpp
plugin.init = +[](const mjModel* m, mjData* d, int instance) {
    auto* Soil = Soil::Create(m, d, instance);
    if (!Soil) return -1;
    d->plugin_data[instance] = reinterpret_cast<uintptr_t>(Soil);
    return 0;
};

plugin.compute = +[](const mjModel* m, mjData* d, int instance, int capability_bit) {
    auto* Soil = reinterpret_cast<class Soil*>(d->plugin_data[instance]);
    Soil->Compute(m, d, instance);
};
```

### 2. 闭环运动链约束 (Closed-Loop Kinematic Constraints)

插件需要正确处理挖掘机的闭环运动链（液压缸创建的闭环）。这通过以下方式实现：

- 跟踪多个铰链连接（D-E', E'-E, C-E）
- 确保液压缸的约束得到满足
- 正确传播位置和方向信息

### 3. 内存管理策略 (Memory Management Strategy)

```cpp
// 使用指针以避免大对象复制
soil_simulator::Bucket *bucket;
soil_simulator::SimOut *sim_out;

// 在构造函数中分配
bucket = new soil_simulator::Bucket(...);
sim_out = new soil_simulator::SimOut(grid);

// 使用默认析构函数自动清理（C++11 移动语义）
~Soil() = default;
```

### 4. 网格索引计算 (Grid Index Calculation)

HField 使用行优先顺序存储：

```cpp
// 对于位置 (ii, jj) 的单元格
int index = m->hfield_nrow[terrain_id] * jj + ii;

// 访问不同 HField 层
int terrain_index = index;
int bucket_soil_1_index = n_hfield_terrain + index;
int bucket_soil_2_index = 2 * n_hfield_terrain + index;
```

### 5. 土壤物理模型 (Soil Physics Model)

土壤模拟器实现了基于休止角的土壤松弛算法：

```
休止角 (Repose Angle): 土壤自然堆积时的最大斜角
典型值: 0.85 弧度 ≈ 48.7°

算法流程:
1. 检测斜度超过休止角的土壤单元
2. 计算需要移动的土壤体积
3. 将土壤重新分配到相邻单元
4. 迭代直到达到平衡状态或最大迭代次数
```

---

## 当前限制和未来改进 (Current Limitations and Future Improvements)

### 当前限制 (Current Limitations)

1. **硬编码的铲斗尺寸**: 铲斗的维度和形状是硬编码的
   ```cpp
   std::vector<float> b_pos_init = {0.7, 0.0, -0.5};
   std::vector<float> t_pos_init = {-0.14, 0.0, -0.97};
   float bucket_width = 0.68;
   ```

2. **硬编码的对象名称**: 要求特定的 HField 和 body 名称
   - 地形必须命名为 "terrain"
   - 铲斗必须命名为 "bucket"
   - 铲斗土壤必须命名为 "bucket soil 1" 和 "bucket soil 2"

3. **缺少土壤反作用力**: 土壤不会对挖掘机施加反作用力（在待办事项列表中）

### 未来改进计划 (Future Improvement Plans)

1. **参数化铲斗**: 从 XML 配置中读取铲斗尺寸
2. **灵活的命名**: 支持自定义 HField 和 body 名称
3. **力反馈**: 实现土壤对铲斗的反作用力
4. **多铲斗支持**: 支持多个铲斗或工具
5. **改进的可视化**: 更好的土壤颗粒渲染
6. **性能分析工具**: 内置的性能监控和分析

---

## 参考资料 (References)

1. [soil_dynamics_cpp 土壤模拟器](https://github.com/KennyVilella/soil_dynamics_cpp)
2. [MuJoCo 物理引擎](https://mujoco.org/)
3. [MuJoCo 插件文档](https://mujoco.readthedocs.io/en/stable/programming/plugin.html)
4. [自定义 MuJoCo 版本](https://github.com/KennyVilella/mujoco)

---

## 贡献指南 (Contributing Guidelines)

如果您想为土壤插件做出贡献，请：

1. Fork 本仓库
2. 创建您的功能分支 (`git checkout -b feature/AmazingFeature`)
3. 提交您的更改 (`git commit -m 'Add some AmazingFeature'`)
4. 推送到分支 (`git push origin feature/AmazingFeature`)
5. 开启一个 Pull Request

请确保您的代码：
- 遵循现有的代码风格
- 包含适当的注释（英文或中文）
- 通过所有测试
- 更新相关文档

---

## 许可证 (License)

Copyright, 2023, Vilella Kenny.

本项目遵循仓库根目录中指定的许可证。

---

**作者**: Vilella Kenny  
**维护者**: excavator_simulator_mujoco 项目团队  
**最后更新**: 2026-01
