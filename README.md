# HEBI Robot Config
Documentation/examples for HEBI robot configuration files

## Overview

The HEBI robot.cfg file format is designed to store configuration information about a system in a human-readable, run-time modifiable, cross-API format.  By moving information from the code files to text files, the code should become shorter and more general and easier to port to different languages. 

## Format

The robot.cfg files are yaml files that contain the following information:
(1) Names + families of the actuators
(2) Path to an associated HRDF file
(3) Path to any associated gains files
(4) Parameters for any arm plugins that are to be used.
(5) Information about waypoints and paths
(6) Custom user attributes

### Names and Families (required)

These can be either given as a single string (applied to all modules in a group), or a list of strings.  For example:

```
names: ["J1_base", "J2_shoulder", "J3_elbow", "J4_wrist1", "J5_wrist2", "J6_wrist3"]
families: "Arm"
```

Note that yaml strings do not need to be enclosed in quotation marks.

### HRDF file (optional)

The HRDF file is parsed as relative to the current working directory of the executable.  Example:

```
hrdf: "hrdf/A-2085-06.hrdf"
```

### Gains files (optional)

The gains filenames are retrievable at run time through a loaded RobotConfig object, and any default gains (labeled with the 'default' key if multiple are provided) are automatically set by `Arm` code objects when initializing the `Arm` object.  Example:

```
gains:
  default: "gains/A-2085-06.xml"
  soft: "gains/A-2085-06-soft.xml"
```

### Plugins (optional)

The `plugins` list allows for storing `Arm` object plugin configurations.  Each listed plugin is parsed when a loaded `RobotConfig` object is used to create an `Arm` object, and as such only the parameter types will be validated when creating the `RobotConfig` object.  Example:

```
plugins:
  - name: 'gravityCompensation'
    type: GravityCompensation
  - name: 'dynamicCompensation'
    type: DynamicCompensation
  - name: 'impedanceController'
    type: ImpedanceController
    gainsInEndEffector: true
    # HOLD POSITION AND ROTATION - BUT ALLOW MOTION ALONG/AROUND Z-AXIS
    kp: [500, 500, 100, 0,  10, 0]  # (N/m) or (Nm/rad)
    kd: [ 10,  10,   1, 0, 0.1, 0]  # (N/(m/sec)) or (Nm/(rad/sec))
```

Note that comments can be added in the yaml file using the `#` character.

The `type` field for each plugin is used to determine which plugin to use.  If this plugin type is not supported by an API, there should be an error when creating the `Arm` object using the `RobotConfig` object.

The `name` field is used as a key for retrieving a specific plugin at run time.

The order of the plugins in the yaml file dictates how they are ordered when creating an `Arm` object.

The remaining items in each plugin depend on the plugin itself, and are evaluated when creating the plugin at the time the `Arm` object is created.  In general, plugins can have the following parameter types:
1. A float
2. A string
3. A bool (see the `gainsInEndEffector` items above)
4. A list of floats (see the `kp` and `kd` items above)
5. A list of strings
6. A list of bools

### Paths and waypoints:

TBD

### User data entries

TBD

## Full examples

In this repository, a full robot.cfg file example can be found.