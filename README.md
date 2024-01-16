# HEBI Robot Config
Documentation/examples for HEBI robot configuration files

## Overview

The HEBI robot.cfg file format is designed to store configuration information about a system in a human-readable, run-time modifiable, cross-API format.  By moving information from the code files to text files, the code should become shorter and more general and easier to port to different languages. 

## Format

The robot.cfg files are yaml files that contain the following information:
1. Names + families of the actuators
2. Path to an associated HRDF file
3. Path to any associated gains files
4. Parameters for any arm plugins that are to be used.
5. Custom user attributes
6. Information about waypoints and paths (future work)

Comments can be added in the yaml file using the `#` character.

### Names and Families (required)

The `version` attribute refers to the version of the file. It commences with `1.0`. Both `names` and `families` must be present and must either be a singular string or a list of strings. If a single family is provided, it is assigned to all modules within its group. For example:

```
version: 1.0
families: "Arm"
names: ["J1_base", "J2_shoulder", "J3_elbow", "J4_wrist1", "J5_wrist2", "J6_wrist3"]
```

Note that yaml strings do not need to be enclosed in quotation marks.

### HRDF file (optional)

The HRDF file is parsed as needed, but is assumed relative to the directory of the configuration file.  Example:

```
hrdf: "hrdf/A-2085-06.hrdf"
```

### Gains files (optional)

The gains filenames are retrievable at run time through a loaded RobotConfig object. The APIs may choose to automatically load the default gains (defined with the `default` key if multiple are provided).  The gain files are parsed as needed, but are assumed relative to the configuration file. Example:

```
gains:
  default: "gains/A-2085-06.xml"
  soft: "gains/A-2085-06-soft.xml"
```

or

```
gains:
  hard: "gains/A-2085-06.xml"
  soft: "gains/A-2085-06-soft.xml"
```

or 

```
gains: "gains/A-2085-06.xml"
```

(Note -- the second example has no default gains, and the third example stores the `gains` file with the `default` key)

### Plugins (optional)

The `plugins` list allows for storing `Arm` plugin configurations.  Each listed plugin is parsed when a loaded `RobotConfig` object is used to create an `Arm` object, and as such only the parameter types will be validated when creating the `RobotConfig` object. Each plugin requires at least a `type` and a `name` parameter, and it is recommended to always set a `ramp_time`. They can be disabled by setting `disabled` to false, and may have an arbitrary number of additional parameters.

- The `type` field for each plugin is used to determine which plugin to use.  If this plugin type is not supported by an API, there should be an error when creating the `Arm` object using the `RobotConfig` object. 
- The `name` field can be used as a key for retrieving a specific plugin at run time.
- The `ramp_time` field defines the time in seconds that it should take to scale the effect. The default `0` signifies no scaling and may result in sudden accelerations and jerks.
- The `enabled` field can be used to start a plugin in a deactivated state. It is often simpler to disable a plugin instead of commenting it out.

The order of the plugins in the yaml file dictates how they are ordered when creating an `Arm` object.

The remaining items in each plugin depend on the plugin itself, and are evaluated when creating the plugin at the time the `Arm` object is created.  In general, plugins can have the following parameter types:
1. A float
2. A string
3. A bool (see the `gains_in_end_effector_frame` item below)
4. A list of floats (see the `kp` and `kd` items below)
5. A list of strings
6. A list of bools

Below is a list of the currently supported plugins. 

- Required parameters are marked in **`bold`**.
- The default value of optional parameters is in `(brackets)`. 

| Type                           | Parameters                                                                        | Description                                                                                                                                                     |
|--------------------------------|-----------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Common to all plugins**         | **`type`** <br> **`name`** <br> `ramp_time (0)` <br> `enabled (true)`     |      Parameters that are used across all the plugins below.  See definitions above and the examples below for more details.                                                                                                                                                           |
| `GravityCompensationEffort`  | `imu_feedback_index (0)` <br> `imu_frame_index (0)` <br> `imu_rotation_offset (identity)` | Adds efforts to compensate for gravity                                                                                                                          |
| `DynamicsCompensationEffort` |                                                                                   | Adds efforts to compensate for joint accelerations. The masses are determined from the robot model.                                                             |
| `EffortOffset`               | **`offset`**                                                                            | Adds efforts to compensate for static offsets due to hardware configurations such as a mechanical spring assist.                                                |
| `ImpedanceController`        | **`gains_in_end_effector_frame`** <br> **`kp`** <br> **`kd`** <br> `ki (zeros)` <br> `i_clamp (zeros)`   | Adds efforts to result in the desired end-effector impedances.                                                                                                  |
| `DoubledJoint`              | **`group_family`** <br> **`group_name`** <br> **`index`** <br> `mirror (true)`                        | Copies actuator commands to assist with a second actuator. This simplifies working with double shoulder configurations while treating an arm as a serial chain. |

Examples:

```
plugins:
  - type: GravityCompensationEffort
    name: gravComp
    imu_feedback_index: 0 # index of the device within a group. Defaults to zero
    imu_frame_index: 0 # frame index that should be transformed. Defaults to zero
    imu_rotation_offset: [1, 0, 0, 0, 1, 0, 0, 0, 1] # row major 3x3 rot matrix, eye 3 default
    enabled: true
    
  - type: DynamicsCompensationEffort
    name: dynamicsComp
    ramp_time: 1
    enabled: true
    
  - name: 'impedanceController'
    type: ImpedanceController
    gains_in_end_effector_frame: true
    # HOLD POSITION AND ROTATION - BUT ALLOW MOTION ALONG/AROUND Z-AXIS
    kp: [500, 500, 100, 0,  10, 0]  # (N/m) or (Nm/rad)
    kd: [ 10,  10,   1, 0, 0.1, 0]  # (N/(m/sec)) or (Nm/(rad/sec))
    
  # Kits with a gas spring need to add a shoulder compensation torque.
  # It should be around -7 Nm for most kits, but it may need to be tuned
  # for your specific setup.
  - type: EffortOffset
    name: gasSpringCompensation
    ramp_time: 0
    enabled: false
    offset: [0, -7, 0, 0, 0, 0]
```

### User data entries

The optional `user_data` field may contain key:value data that gets stored in a "user data" parameter map. The keys must be alphanumeric with optional underscores and do not start with a number. Depending on the API, the values may be exposed as strings or as dynamic types. Example:

```
user_data:
  robot_display_name: "Friendly Bot"
  max_power: "25.9"
  scale: 0.9
  enable_logging: true
```

### Paths and waypoints:

TBD

### Other entries

Any entry that is not `names`, `families`, `hrdf`, `gains`, `plugins`, or `user_data` results in an error.

## Full examples

In this repository, a full robot.cfg file example can be found.
