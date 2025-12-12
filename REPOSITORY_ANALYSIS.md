# BeaVR Repository - Complete Functionality Analysis

## Executive Summary

**BeaVR** (Bimanual, multi-Embodiment, Accessible VR Teleoperation for Robots) is an open-source, end-to-end teleoperation and robotics learning pipeline designed to make advanced robot control accessible and affordable. The system enables VR-based teleoperation of robots for data collection, policy training, and deployment using commodity hardware.

---

## 1. Project Overview

### 1.1 Core Purpose

BeaVR serves three primary functions:
1. **Real-time VR Teleoperation** - Control robots intuitively using VR headsets (Meta Quest 3, etc.)
2. **Demonstration Collection** - Record high-quality robot demonstrations for machine learning
3. **Policy Training & Deployment** - Train and evaluate imitation learning policies using collected data

### 1.2 Key Features

- **VR Teleoperation Out-of-the-Box**: Stream low-latency control and visual feedback through Meta Quest 3
- **Multi-Embodiment Support**: Ships with drivers for RX-1 humanoid, xArm robotic arms, and LeapHand dexterous hands
- **Simulation Parity**: Mirror real-world sessions in MuJoCo or Isaac Gym for sim-to-real transfer
- **Dexterous Demonstration Collection**: Capture single-hand, bi-manual, or whole-body demonstrations
- **Budget-Friendly**: Works on commodity PCs with accessible robotics hardware
- **LeRobot Integration**: Standardized data format compatible with the LeRobot ecosystem

### 1.3 Technology Stack

- **Language**: Python 3.10+
- **Core Framework**: PyTorch for deep learning
- **Messaging**: ZeroMQ for real-time inter-process communication
- **Visualization**: Rerun SDK for real-time 3D visualization
- **Data Format**: HuggingFace Datasets with LeRobot schema
- **Simulation**: PyBullet, MuJoCo support
- **Build System**: UV/Poetry package management

---

## 2. Repository Structure

```
beavr-bot/
├── src/beavr/                    # Main Python package
│   ├── teleop/                   # Teleoperation system
│   │   ├── components/           # Core teleoperation components
│   │   │   ├── detector/         # VR hand tracking detectors
│   │   │   ├── operator/         # Motion retargeting operators
│   │   │   ├── interface/        # Robot hardware interfaces
│   │   │   └── visualizer/       # Real-time visualization
│   │   ├── configs/              # Configuration system
│   │   │   ├── constants/        # Network ports, addresses
│   │   │   └── robots/           # Robot-specific configs
│   │   ├── common/               # Shared utilities
│   │   └── main.py               # Teleoperation entry point
│   ├── lerobot/                  # LeRobot integration
│   │   ├── common/               # Core LeRobot components
│   │   │   ├── datasets/         # Dataset handling
│   │   │   ├── policies/         # Learning policies
│   │   │   ├── robot_devices/    # Device drivers
│   │   │   └── envs/             # Environment wrappers
│   │   └── configs/              # Policy configurations
│   ├── scripts/                  # Utility scripts
│   │   └── control_robot.py      # Main robot control script
│   └── schemas/                  # Data schemas
├── configs/                       # YAML configurations
│   ├── environment/              # Dev/prod environment settings
│   └── robots/                   # Robot configuration files
├── assets/                        # Robot assets
│   ├── urdf/                     # URDF robot models
│   └── asset_templates/          # Asset templates
├── docs/                          # Documentation
│   ├── teleop/                   # Teleoperation docs
│   └── lerobot/                  # LeRobot integration docs
├── tests/                         # Test suite
├── docker/                        # Docker configurations
├── teleop.py                      # Convenience wrapper
└── pyproject.toml                # Project metadata and dependencies
```

---

## 3. Major Subsystems

### 3.1 Teleoperation System (`src/beavr/teleop/`)

The teleoperation system is the core of BeaVR, enabling real-time VR control of robots.

#### 3.1.1 Architecture Overview

The teleoperation system follows a **component-based architecture** with decoupled processes communicating via ZeroMQ:

```
VR Headset → Detector → Transform → Operator → Robot Interface → Physical Robot
              ↓           ↓           ↓            ↓
           Raw Data   Stabilized   Commands    Hardware API
                      Hand Frame
```

#### 3.1.2 Key Components

**A. Detectors (`components/detector/`)**
- **Purpose**: Capture raw hand tracking data from VR devices
- **Implementation**: 
  - `OculusVRHandDetector`: Single hand tracking
  - `BimanualOculusVRHandDetector`: Both hands simultaneously
- **Output**: Raw keypoint data published to ZMQ port 8000 (default)
- **Data Format**: XYZ positions for hand landmarks + button states

**B. Transform Components**
- **Purpose**: Convert raw VR keypoints into stable 6DoF hand frames
- **Process**: 
  - Filters noisy tracking data
  - Computes stable hand pose (position + orientation)
  - Applies coordinate frame transformations
- **Output**: Stabilized hand frame to ZMQ port (transforms)

**C. Operators (`components/operator/`)**
- **Purpose**: Map VR hand motion to robot commands (retargeting)
- **Key Classes**:
  - `XArmOperator`: Base class for arm control
  - `XArm7RightOperator` / `XArm7LeftOperator`: Side-specific operators
  - `LeapHandOperator`: Dexterous hand control
- **Process**:
  1. Subscribe to transformed hand frames
  2. Apply calibrated transformations (H_R_V: Robot→VR, H_T_V: Hand→VR)
  3. Compute relative motion since reset
  4. Map to robot workspace
  5. Apply optional filtering (complementary filter)
  6. Publish end-effector commands
- **Configuration**: Resolution modes (high/low), pause/continue states

**D. Robot Interfaces (`components/interface/`)**
- **Purpose**: Abstract hardware-specific APIs into unified interface
- **Implementations**:
  - `XArm7Robot`: xArm manipulator control via XArm SDK
  - `LeapHandRobot`: Leap Hand control (simulation + real)
  - `RX1RightRobot`: RX-1 humanoid arm interface
- **Functions**:
  - Receive commands from operators
  - Execute motion primitives
  - Publish robot state for recording
  - Handle homing, reset, and pause commands

**E. Visualizers (`components/visualizer/`)**
- **Purpose**: Real-time visualization of robot and VR states
- **Features**:
  - 2D plotting of joint trajectories
  - 3D visualization of robot pose
  - Hand tracking overlay
- **Backend**: Matplotlib-based plotters

#### 3.1.3 Networking Layer

**ZeroMQ Communication**:
- **Pattern**: Publish-Subscribe (PUB-SUB)
- **Location**: `src/beavr/teleop/common/utils/network.py`
- **Key Classes**:
  - `BasePublisher`: Generic ZMQ publisher
  - `BaseSubscriber`: Threaded subscriber with message processing
  - `ZMQPublisherManager`: Centralized publisher management
  - `HandshakeCoordinator`: ACK-based coordination for critical operations
  
**Default Port Mapping**:
```
Port 8000: Keypoint stream (raw VR data)
Port 8001: Control commands (to robots)
Port 8002: Robot state publishing
Port 8003+: Side-specific ports for bimanual setups
```

**Message Format**:
```python
[topic: bytes, payload: pickle.dumps(data)]
```

#### 3.1.4 Configuration System

**Structure**:
- **Dataclass-based**: Uses Python dataclasses for type safety
- **CLI Integration**: Draccus library for automatic CLI flag generation
- **YAML Support**: Load configurations from YAML files
- **Precedence**: CLI flags > YAML overrides > defaults

**Key Configuration Classes** (`configs/constants/models.py`):
- `TeleopConfig`: Master configuration
- `NetworkConfig`: IP addresses and network settings
- `PortsConfig`: Port numbers for all ZMQ channels
- `RobotConfig`: Robot-specific parameters
- `CameraConfig`: Camera settings for visual feedback

**Example Usage**:
```bash
# Single robot, right side
python teleop.py --robot_name=xarm7 --laterality=right

# Bimanual setup
python teleop.py --robot_name=leap,xarm7 --laterality=bimanual

# Custom network
python teleop.py --robot_name=leap --teleop.network.host_address=192.168.1.50
```

#### 3.1.5 Operational Controls

**Reset**: 
- Captures new baseline for hand and robot pose
- Subsequent motion is relative to this baseline
- Triggered by VR button or keyboard

**Pause/Continue**:
- `ARM_TELEOP_STOP`: Halt motion while maintaining connection
- `ARM_TELEOP_CONT`: Resume operation
- Synchronized across operator and robot via handshake

**Resolution Modes**:
- **High Resolution**: Larger hand motion → smaller end-effector motion (precision)
- **Low Resolution**: Larger workspace coverage
- Toggle via VR button

#### 3.1.6 Data Recording

**Recorder Component**:
- Subscribes to robot state streams
- Synchronizes camera images with robot states
- Captures:
  - Joint positions and velocities
  - End-effector poses
  - Commanded actions
  - Camera frames
  - Timestamps
- **Output Format**: HuggingFace Dataset compatible with LeRobot

---

### 3.2 LeRobot Integration (`src/beavr/lerobot/`)

The LeRobot stack provides dataset management and policy learning capabilities.

#### 3.2.1 Architecture

```
BeaVR Teleop → Dataset Creation → Policy Training → Policy Evaluation → Deployment
                      ↓                   ↓               ↓
                 HF Dataset         PyTorch Model    Robot Control
```

#### 3.2.2 Key Components

**A. Robot Devices (`common/robot_devices/`)**
- **Purpose**: Unified robot control API for LeRobot
- **Components**:
  - Camera drivers (Intel RealSense, OpenCV)
  - Robot control wrappers
  - Mobile manipulator support
  - Utility functions for control

**B. Datasets (`common/datasets/`)**
- **Purpose**: Dataset creation and management
- **Format**: HuggingFace Datasets with LeRobot schema
- **Features**:
  - Episode-based storage
  - Video compression
  - Metadata tracking
  - Cloud storage via HuggingFace Hub
- **Schema**:
  ```python
  {
    "observation.state": joint_positions,
    "observation.images.{camera_name}": image_arrays,
    "action": commanded_actions,
    "episode_index": episode_id,
    "frame_index": frame_id,
    "timestamp": unix_timestamp,
  }
  ```

**C. Policies (`common/policies/`)**
- **Purpose**: Imitation learning policy implementations
- **Supported Policies**:
  - ACT (Action Chunking Transformer)
  - Diffusion Policy
  - TDMPC (Temporal Difference Model Predictive Control)
  - And more...
- **Training**: PyTorch-based with GPU acceleration
- **Evaluation**: Real robot or simulation

**D. Environments (`common/envs/`)**
- **Purpose**: Gym-compatible robot environments
- **Features**:
  - Real robot environments
  - Simulation wrappers
  - Standard Gym API
  - Episode management

#### 3.2.3 Control Script (`scripts/control_robot.py`)

The main entry point for LeRobot operations:

**Modes**:
1. **Teleoperate**: Live robot control for testing
2. **Record**: Collect demonstration episodes
3. **Replay**: Replay recorded episodes
4. **Remote Robot**: Run on edge devices

**Example Usage**:
```bash
# Record demonstrations
python beavr/scripts/control_robot.py \
  --robot.type=so100 \
  --control.type=record \
  --control.repo_id=$USER/my_dataset \
  --control.num_episodes=50 \
  --control.fps=30

# Train policy
python beavr/scripts/train.py \
  --dataset.repo_id=$USER/my_dataset \
  --policy.type=act \
  --output_dir=outputs/train/act_model

# Evaluate policy
python beavr/scripts/control_robot.py \
  --robot.type=so100 \
  --control.type=record \
  --control.policy.path=outputs/train/act_model/checkpoints/080000/pretrained_model
```

#### 3.2.4 Configuration System

**Location**: `lerobot/configs/`
- Policy hyperparameters
- Dataset configurations
- Training parameters
- Environment settings

---

### 3.3 Supported Robots

#### 3.3.1 xArm7 Manipulator
- **Type**: 7-DOF robotic arm
- **Control**: Cartesian end-effector commands
- **SDK**: xArm Python SDK
- **Features**: Left/right arms, bimanual support
- **Config**: `configs/robots/xarm7_config.py`

#### 3.3.2 Leap Hand
- **Type**: Dexterous robotic hand
- **Control**: Joint-level control (16 DOF)
- **Features**: Simulated and real hardware support
- **Integration**: Can be mounted on xArm for manipulation tasks
- **Config**: `configs/robots/leap_config.py`

#### 3.3.3 RX-1 Humanoid
- **Type**: Full-size humanoid robot
- **Control**: Arm and torso control
- **Features**: Whole-body teleoperation
- **Config**: Custom configuration for RX-1

#### 3.3.4 Extensibility
- **Template**: `components/operator/robots/template.py`
- **Process**: Implement robot-specific operator and interface classes
- **Registration**: Add to robot config registry

---

## 4. Data Flow Example

### Scenario: Right-handed xArm7 teleoperation

```
1. VR Headset (Meta Quest 3)
   ↓ [Hand tracking data via WiFi]
   
2. OculusVRHandDetector (Process 1)
   - Receives raw hand data on PULL socket
   - Parses XYZ keypoints
   - Publishes to ZMQ port 8000, topic "right"
   ↓
   
3. Transform Component (Process 2)
   - Subscribes to port 8000, topic "right"
   - Computes stable 6DoF hand frame
   - Publishes to transform port, topic "right"
   ↓
   
4. XArm7RightOperator (Process 3)
   - Subscribes to transformed hand frame
   - Applies calibration transforms (H_R_V, H_T_V)
   - Computes relative motion since reset
   - Maps to robot workspace
   - Applies complementary filter
   - Publishes end-effector command to port 8001
   ↓
   
5. XArm7Robot (Process 4)
   - Subscribes to port 8001 for commands
   - Validates command
   - Calls DexArmControl.move_arm_cartesian()
   - Executes motion on hardware
   - Publishes robot state to port 8002
   ↓
   
6. Recorder (Process 5) [Optional]
   - Subscribes to port 8002
   - Synchronizes with camera frames
   - Saves to HuggingFace Dataset
```

---

## 5. Development Workflow

### 5.1 Installation

```bash
# Install uv package manager
curl -LsSf https://astral.sh/uv/install.sh | sh

# Setup Python environment
uv python install 3.10.13
uv venv --python 3.10.13
source .venv/bin/activate

# Install dependencies
uv sync --extra dev

# Setup pre-commit hooks
pre-commit install
```

### 5.2 Running Teleoperation

```bash
# Single robot
python teleop.py --robot_name=leap --laterality=right

# Bimanual
python teleop.py --robot_name=leap,xarm7 --laterality=bimanual

# Simulation mode
python teleop.py --robot_name=xarm7 --teleop.flags.sim_env=true
```

### 5.3 Recording Data

```bash
python beavr/scripts/control_robot.py \
  --robot.type=so100 \
  --control.type=record \
  --control.repo_id=$USER/dataset \
  --control.num_episodes=50
```

### 5.4 Testing

```bash
# Run all tests
pytest tests/ -v

# Run specific test
pytest tests/components/test_operator.py -v

# With coverage
pytest tests/ -v --cov=beavr --cov-report=html
```

---

## 6. Configuration and Customization

### 6.1 Environment Configuration

**Development** (`configs/environment/dev.yaml`):
- Localhost networking
- Debug logging enabled
- Simulation mode available

**Production** (`configs/environment/prod.yaml`):
- Network addresses for distributed setup
- Optimized logging
- Hardware acceleration

### 6.2 Robot Configuration

Each robot has a dedicated config file defining:
- Network ports (per laterality)
- Operator parameters (transforms, filters)
- Robot interface settings (IP, modes)
- Recording configuration

### 6.3 Calibration

**Purpose**: Align VR coordinate frame with robot base frame

**Process**:
1. Move robot to known position
2. Place VR controller at robot end-effector
3. Record both poses
4. Compute transformation matrices (H_R_V, H_T_V)
5. Save to calibration file

**Location**: `calibration_files/`

---

## 7. Advanced Features

### 7.1 Simulation Support

**PyBullet Integration**:
- `components/operator/robots/xarm_pybullet_sim.py`
- Visual debugging
- Rapid prototyping

**MuJoCo/Isaac Gym**:
- Mentioned in docs but requires additional setup
- For large-scale policy training

### 7.2 Multi-Camera Recording

- Supports multiple camera streams
- Intel RealSense drivers included
- Synchronization with robot states
- Compression for efficient storage

### 7.3 Real-Time Visualization

- **Rerun SDK**: 3D visualization of robot and VR
- **Matplotlib**: 2D plots of trajectories
- **Web Interface**: Flask-based monitoring (optional)

### 7.4 Remote Operation

- **Distributed Setup**: Separate machines for VR, compute, and robot
- **Network Configuration**: Customizable IP addresses and ports
- **Latency Optimization**: ZMQ with low latency settings

---

## 8. CI/CD and Quality Assurance

### 8.1 Continuous Integration

**GitHub Actions** (`.github/workflows/`):
- `fast_tests.yml`: Quick unit tests on PR
- `full_tests.yml`: Comprehensive tests on main branch
- Python 3.10 test environment
- Code coverage reporting

### 8.2 Code Quality

**Pre-commit Hooks** (`.pre-commit-config.yaml`):
- Ruff linting
- Black formatting
- Import sorting with isort
- Type checking (when enabled)

**Linting Configuration** (`pyproject.toml`):
```toml
[tool.ruff.lint]
select = ["E", "W", "F", "I", "B", "C4", "T20", "N"]
ignore = ["E501", "T201"]  # Line length, print statements
```

### 8.3 Testing Strategy

**Test Structure** (`tests/`):
```
tests/
├── components/           # Component-level tests
│   ├── test_detector.py
│   ├── test_operator.py
│   └── test_interface.py
├── conftest.py          # Pytest fixtures
└── __init__.py
```

**Test Categories**:
- Unit tests: Individual functions and classes
- Integration tests: Component interactions
- System tests: End-to-end workflows (manual)

---

## 9. Dependencies and Ecosystem

### 9.1 Core Dependencies

**Deep Learning**:
- PyTorch ≥2.2.1
- torchvision ≥0.21.0
- Transformers ≥4.50.3

**Robotics**:
- xArm Python SDK (xArm control)
- Dynamixel SDK (servo control)
- PyBullet ≥3.2.6 (simulation)
- pyrealsense2 (Intel cameras)

**Data and ML**:
- datasets ≥2.19.0 (HuggingFace)
- Wandb ≥0.16.3 (experiment tracking)
- imageio[ffmpeg] (video processing)

**Networking**:
- pyzmq ≥26.2.1 (messaging)
- Flask ≥3.0.3 (web interface)

**Utilities**:
- Hydra ≥1.3.2 (configuration)
- Draccus 0.10.0 (CLI generation)
- omegaconf ≥2.3.0 (config management)

### 9.2 Optional Dependencies

**Development** (`[dev]`):
- pytest, pytest-cov, pytest-timeout
- Black, isort, ruff
- pre-commit
- JupyterLab

**LeRobot Integration** (`[lerobot]`):
- lerobot ≥0.1.0

**Teleoperation** (`[teleop]`):
- pygame ≥2.5.1
- hidapi ≥0.14.0

---

## 10. Use Cases and Applications

### 10.1 Research Applications

1. **Imitation Learning Research**
   - Collect large-scale demonstration datasets
   - Train behavior cloning policies
   - Evaluate sim-to-real transfer

2. **Teleoperation Studies**
   - Human-robot interaction research
   - Ergonomics and usability studies
   - Latency and control fidelity experiments

3. **Multi-Modal Learning**
   - Vision-language-action models
   - Cross-embodiment policy transfer
   - Multi-task learning

### 10.2 Industry Applications

1. **Remote Operations**
   - Hazardous environment manipulation
   - Space/underwater robotics
   - Medical robotics

2. **Training Data Generation**
   - Bootstrap learning systems
   - Quality control demonstrations
   - Assembly task libraries

3. **Rapid Prototyping**
   - Test robot capabilities
   - Validate task feasibility
   - Human-in-the-loop development

### 10.3 Education

1. **Robotics Courses**
   - Hands-on teleoperation labs
   - ML for robotics projects
   - System integration exercises

2. **Research Training**
   - Dataset collection workflows
   - Policy training pipelines
   - Real-robot experimentation

---

## 11. Community and Contribution

### 11.1 Contributing

See `CONTRIBUTING.md` for:
- Code style guidelines
- PR process
- Adding new robots
- Extending policies
- Documentation standards

### 11.2 Code of Conduct

See `CODE_OF_CONDUCT.md` for community standards

### 11.3 License

**MIT License** - Permissive open-source license
- Commercial use allowed
- Modification allowed
- Distribution allowed
- Private use allowed
- Liability and warranty disclaimers apply

**Note**: The LICENSE file and README both specify MIT License, though there is a mention of "BSD-3-licensed" in the README's "Why use BeaVR?" section which may be outdated or refer to specific components.

---

## 12. Future Directions

Based on `docs/TODO.md` and roadmap:

1. **Expanded Robot Support**
   - UR series arms
   - Quadrupeds
   - Tactile grippers

2. **Enhanced VR Support**
   - Apple Vision Pro native integration
   - Hand-held controller alternatives
   - Haptic feedback

3. **Advanced Learning**
   - Reinforcement learning integration
   - Online learning during teleoperation
   - Multi-modal policies (vision + language)

4. **Performance Optimization**
   - GPU-accelerated retargeting
   - Reduced latency networking
   - Real-time policy inference

5. **User Experience**
   - Web-based configuration UI
   - Improved visualization
   - Automated calibration

---

## 13. Troubleshooting and Common Issues

### 13.1 Installation Issues

**Problem**: CUDA version mismatch
**Solution**: Install PyTorch via conda with correct CUDA version

**Problem**: Port already in use
**Solution**: Kill zombie processes or change ports in config

### 13.2 Runtime Issues

**Problem**: No VR data flowing
**Solution**: 
- Check VR app is running
- Verify network connectivity
- Check firewall settings
- Verify port configuration

**Problem**: Robot not responding
**Solution**:
- Check robot power and connection
- Verify IP address in config
- Test robot independently
- Check safety limits

### 13.3 Performance Issues

**Problem**: High latency
**Solution**:
- Reduce VR polling frequency
- Use wired ethernet
- Disable unnecessary visualizers
- Check CPU usage

---

## 14. Technical Specifications

### 14.1 Performance Metrics

- **VR Tracking Rate**: Up to 200 Hz
- **Control Loop Frequency**: 60-200 Hz (configurable)
- **Recording Frequency**: 30-60 Hz (typical)
- **Network Latency**: <50ms on LAN
- **Supported Cameras**: 640x480 to 1920x1080 @ 30fps

### 14.2 System Requirements

**Minimum**:
- Ubuntu 20.04+ (Linux)
- Python 3.10
- 8GB RAM
- Quad-core CPU
- GPU: None (CPU only mode)

**Recommended**:
- Ubuntu 22.04
- Python 3.10.13
- 16GB+ RAM
- 8-core CPU
- NVIDIA GPU with CUDA 11.8+
- SSD storage

**VR Hardware**:
- Meta Quest 3 (primary)
- Meta Quest 2 (supported)
- Apple Vision Pro (via third-party app)

---

## 15. Conclusion

BeaVR is a comprehensive, production-ready system for VR-based robot teleoperation and learning. It combines:

1. **Real-time teleoperation** with low latency and intuitive VR control
2. **Multi-robot support** with extensible architecture
3. **Data collection** in standardized LeRobot format
4. **Policy training** with state-of-the-art imitation learning
5. **Deployment** back to real robots for evaluation

The modular design, comprehensive documentation, and active development make it suitable for both research and practical applications. The MIT license and budget-friendly approach lower barriers to entry for robotics research and development.

### Key Strengths

- ✅ Open-source and permissively licensed
- ✅ Well-documented with clear examples
- ✅ Modular and extensible architecture
- ✅ Active development and community
- ✅ Integration with popular ML frameworks
- ✅ Support for simulation and real hardware

### Getting Started

1. Read the main README.md
2. Follow installation instructions
3. Start with teleoperation examples
4. Progress to data collection
5. Train your first policy
6. Deploy and evaluate

For detailed information, refer to the documentation in the `docs/` directory.

---

**Repository**: https://github.com/ARCLab-MIT/beavr-bot  
**Paper**: https://arxiv.org/abs/2508.09606  
**Project Page**: https://arclab-mit.github.io/beavr-landing/  
**License**: MIT  
**Version**: 1.0.0
