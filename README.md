# excavator_simulator_mujoco
This repository implements a simulator of a hydraulic excavator including soil digging, transportation, and dumping.
The simulator is built on the top of a customized version of [MuJoCo][MuJoCo], available [here][Mujoco2].
The only modification of this customized version is a better visualization of floating soil.

The soil dynamics is modeled using [soil_dyanmics_cpp][soil simulator] and a custom soil plugin serves as an interface with [MuJoCo][MuJoCo].
A description of the soil plugin as well as an usage example is provided [here][soil README].
An excavator model is also included that is thoroughly described in the corresponding [README](model/excavator/README.md).

A customized executable to launch [MuJoCo][MuJoCo] is also provided, as the default executable does not include the update of `HField`.

# Documentation / 文档

For detailed technical documentation, please refer to:

**English / Chinese (中英文双语)**:
- **[Soil Plugin Development Guide](plugin/soil/DEVELOPMENT.md)** - Comprehensive guide on soil plugin architecture, implementation, and development
  - 土壤插件开发指南 - 关于土壤插件架构、实现和开发的综合指南
- **[MuJoCo Development Guide](MUJOCO_DEVELOPMENT.md)** - Details about MuJoCo customization, plugin integration, and advanced topics
  - MuJoCo 开发指南 - 关于 MuJoCo 定制、插件集成和高级主题的详细信息
- **[Excavator Model README](model/excavator/README.md)** - Detailed description of the excavator model geometry and kinematics
  - 挖掘机模型说明 - 挖掘机模型几何和运动学的详细描述
- **[Soil Plugin Usage](plugin/soil/README.md)** - Quick reference for using the soil plugin
  - 土壤插件使用 - 使用土壤插件的快速参考

# Installation
A bash script is provided to make the installation of the simulator easier.
To install the simulator, simply execute the following command
```
bash <path_to_repo>/build.bash
```

This script would:
- (if necessary) clone the [soil_dyanmics_cpp][soil simulator] repository and move it to the appropriate folder.
- Build the simulator with CMake.
- Copy the custom plugin library to the appropriate folder.
- Copy the custom executable to the appropriate folder.

# Running the simulator
To run the simulator, simply execute the following command
```
.<path_to_repo>/build/bin/excavator_simulator <path_to_repo>/model/excavator/excavator.xml
```

A window will open with the hydraulic excavator.
It is suggested to toggle on the "Wireframe" view in the "Rendering" section in order to better visualize the soil.
The hydraulic excavator can be actuated using the four sliders in the "Control" section.

To change the initial pose of the excavator or the simulation properties (grid and soil), it is recommended to use the script `model_generation.py` in the `model/excavator/script/` folder.
This script can generate an excavator model following properties given in an input dictionary.
For more details, the user may refer to the [model README](model/excavator/README.md) and the docstrings inside the script.
Note that the script relies on the `Jinja2` templating engine that can be installed using the following command
```
pip install Jinja2.
```

# To-do list
There are several important features that are yet to be implemented.
These include, in order of priority:

- Improve the actuation. The excavator should not move when a zero velocity is requested.
- Add reaction force from the soil.

[MuJoCo]: https://mujoco.org/
[MuJoCo2]: https://github.com/KennyVilella/mujoco
[soil simulator]: https://github.com/KennyVilella/soil_dynamics_cpp
[soil README]: plugin/soil/README.md
