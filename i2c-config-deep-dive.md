# Hubris I2C Configuration Deep Dive

## Overview

Hubris uses a sophisticated build-time code generation system for I2C configuration. This document provides a comprehensive guide to understanding how I2C devices, controllers, and sensors are configured and how the build system generates runtime code.

## Architecture Overview

The I2C configuration system has three main components:

1. **Configuration Definition** - TOML-based device/controller definitions in `app.toml`
2. **Build-Time Code Generation** - `build/i2c/src/lib.rs` processes TOML and generates Rust code
3. **Runtime API** - `drv/i2c-api/src/lib.rs` provides the client interface for I2C operations

```
app.toml (I2C config) 
    ↓ 
build/i2c (code generator)
    ↓
i2c_config.rs (generated code)
    ↓
Runtime I2C operations
```

## Configuration Structure

### Controllers
Controllers represent the hardware I2C peripherals on the microcontroller:

```toml
[[i2c.controllers]]
controller = 1  # I2C1 peripheral
target = false  # Initiator mode (master)

[i2c.controllers.ports.B]  # GPIO Port B
scl = { pin = 8 }          # PB8 for SCL
sda = { pin = 9 }          # PB9 for SDA  
af = 4                     # Alternate function 4
```

Key fields:
- `controller`: Hardware peripheral number (0-7)
- `target`: `false` for initiator/master, `true` for target/slave
- `ports`: GPIO port configurations for different pin arrangements
- `scl`/`sda`: GPIO pins for clock and data lines
- `af`: Alternate function mapping for GPIO pins

### Devices
Devices represent individual I2C components on the bus:

```toml
[[i2c.devices]]
device = "ltc4282"
description = "Hot swap controller"
controller = 1
port = "B"
address = 0x44
mux = 1         # Optional: multiplexer ID
segment = 2     # Optional: mux segment
refdes = "U15"  # Reference designator

[i2c.devices.power]
rails = ["12V", "3V3"]
pmbus = true

[i2c.devices.sensors]
temperature = 1
current = 2
voltage = 2
```

Key fields:
- `device`: Device driver name (must match `drv-i2c-devices`)
- `controller`/`port`: Physical bus location
- `address`: 7-bit I2C device address
- `mux`/`segment`: Multiplexer hierarchy (optional)
- `power`: Power rail information for PMBus devices
- `sensors`: Sensor counts by type

### Multiplexers
Muxes allow multiple devices to share the same I2C address on different segments:

```toml
[[i2c.controllers.ports.B.muxes]]
driver = "pca9548"
address = 0x70
nreset = { port = "C", pin = 5 }  # Reset GPIO
```

## Build-Time Code Generation

### Code Generator (`build/i2c/src/lib.rs`)

The `ConfigGenerator` struct processes TOML configuration and generates different types of code based on the `Disposition`:

#### Disposition Types

1. **`Initiator`** - I2C master mode
   - Generates controller configurations
   - Creates pin mappings
   - Sets up mux configurations

2. **`Target`** - I2C slave mode  
   - Single controller configuration
   - Pin setup for target mode

3. **`Devices`** - Device factory functions
   - Creates device lookup tables
   - Generates factory functions by device type, name, and refdes

4. **`Sensors`** - Sensor mappings
   - Generates sensor ID constants
   - Creates sensor struct definitions

5. **`Validation`** - Device validation
   - Creates validation functions for each device

### Generated Code Structure

The build process creates `i2c_config.rs` with the following modules:

```rust
pub(crate) mod i2c_config {
    // Constants
    pub const NCONTROLLERS: usize = 2;
    pub const NMUXEDBUSES: usize = 1;
    
    // Controller configurations
    pub fn controllers() -> [I2cController<'static>; NCONTROLLERS] { ... }
    
    // Pin configurations  
    pub fn pins() -> [I2cPins; 4] { ... }
    
    // Mux configurations
    pub fn muxes() -> [I2cMux<'static>; 2] { ... }
    
    // Device factories
    pub mod devices { ... }
    
    // Sensor mappings
    pub mod sensors { ... }
    
    // Power rail mappings
    pub mod pmbus { ... }
    
    // Validation functions
    pub mod validation { ... }
}
```

## Device Factory Generation

### Factory Function Types

The code generator creates multiple types of factory functions:

#### 1. By Device Type
```rust
pub fn ltc4282(task: TaskId) -> [I2cDevice; 2] {
    [
        // Hot swap controller on I2C1:B
        I2cDevice::new(task, Controller::I2C1, PortIndex(0), None, 0x44),
        // Another instance...
        I2cDevice::new(task, Controller::I2C1, PortIndex(0), Some((Mux::M1, Segment::S2)), 0x44),
    ]
}
```

#### 2. By Device Name
```rust
pub fn ltc4282_hot_swap(task: TaskId) -> I2cDevice {
    I2cDevice::new(task, Controller::I2C1, PortIndex(0), None, 0x44)
}
```

#### 3. By Reference Designator
```rust
pub fn ltc4282_u15(task: TaskId) -> I2cDevice {
    I2cDevice::new(task, Controller::I2C1, PortIndex(0), None, 0x44)
}
```

#### 4. By Bus Name
```rust
pub fn ltc4282_main_bus(task: TaskId) -> [I2cDevice; 1] {
    [I2cDevice::new(task, Controller::I2C1, PortIndex(0), None, 0x44)]
}
```

### Factory Function Generation Logic

The code generator processes devices in this sequence:

1. **Parse Configuration** - Load and validate TOML
2. **Group Devices** - Organize by device type, name, refdes, and bus
3. **Generate Factories** - Create functions for each grouping
4. **Validation** - Ensure no duplicate names/refdes per device type

Example generation code:
```rust
for (device, devices) in all {
    write!(
        &mut self.output,
        r##"
        #[allow(dead_code)]
        pub fn {}(task: TaskId) -> [I2cDevice; {}] {{
            ["##,
        device,
        devices.len()
    )?;

    for d in devices {
        let out = self.generate_device(d, 16);
        write!(&mut self.output, "{out},")?;
    }

    writeln!(&mut self.output, "]\n        }}")?;
}
```

## Sensor Configuration

### Sensor Types
Hubris supports these sensor categories:
- `temperature` - Temperature sensors
- `power` - Power measurements  
- `current` - Current measurements
- `voltage` - Voltage measurements
- `input_current` - Input current measurements
- `input_voltage` - Input voltage measurements
- `speed` - Fan/motor speed sensors

### Sensor Struct Generation

For each device with sensors, the generator creates:

#### 1. Sensor Count Constants
```rust
pub const NUM_LTC4282_TEMPERATURE_SENSORS: usize = 1;
pub const NUM_LTC4282_CURRENT_SENSORS: usize = 2;
```

#### 2. Sensor ID Constants
```rust
pub const LTC4282_TEMPERATURE_SENSOR: SensorId = SensorId::new(0);
pub const LTC4282_CURRENT_SENSORS: [SensorId; 2] = [
    SensorId::new(1),
    SensorId::new(2),
];
```

#### 3. Sensor Structs
```rust
pub struct Sensors_ltc4282 {
    pub temperature: SensorId,
    pub current: [SensorId; 2],
    pub voltage: [SensorId; 2],
}

pub const LTC4282_HOT_SWAP_SENSORS: Sensors_ltc4282 = Sensors_ltc4282 {
    temperature: SensorId::new(0),
    current: [SensorId::new(1), SensorId::new(2)],
    voltage: [SensorId::new(3), SensorId::new(4)],
};
```

### Power Rail Integration

For PMBus devices, the generator links sensors to power rails:

```rust
// Power rail functions
pub fn v12_main(task: TaskId) -> (I2cDevice, u8) {
    (I2cDevice::new(task, Controller::I2C1, PortIndex(0), None, 0x44), 0)
}

// Phase information for multi-phase rails
pub const LTC4282_V12_MAIN_PHASES: Option<&'static [u8]> = Some(&[1, 2]);
```

## Multiplexer Support

### Mux Configuration
```rust
pub fn muxes() -> [I2cMux<'static>; 2] {
    [
        I2cMux {
            controller: Controller::I2C1,
            port: PortIndex(0),
            id: Mux::M1,
            driver: &drv_stm32xx_i2c::pca9548::Pca9548,
            nreset: Some(I2cGpio {
                gpio_pins: gpio_api::Port::C.pin(5),
            }),
            address: 0x70,
        },
        // Additional muxes...
    ]
}
```

### Mux State Management

The I2C server tracks mux state per bus:
- **`Enabled(mux, segment)`** - Specific mux+segment active
- **`Unknown`** - State uncertain, requires reset
- **No entry** - No mux segments enabled

## Validation System

### Device Validation Generation

For each device, the generator creates validation code:

```rust
pub fn validate(task: TaskId, index: usize) -> Result<I2cValidation, ResponseCode> {
    match index {
        0 => {
            // Device with driver validation
            if drv_i2c_devices::ltc4282::Ltc4282::validate(&device)? {
                Ok(I2cValidation::Good)
            } else {
                Ok(I2cValidation::Bad)
            }
        }
        1 => {
            // Device without driver - raw read test
            device.read::<u8>()?;
            Ok(I2cValidation::RawReadOk)
        }
        _ => Err(ResponseCode::BadArg)
    }
}
```

### Validation Types
- **`Good`** - Device-specific validation passed
- **`Bad`** - Device-specific validation failed  
- **`RawReadOk`** - Basic I2C communication works

## Integration with Runtime API

### Build Integration

Tasks integrate generated configuration in their `build.rs`:

```rust
use build_i2c::{codegen, Disposition};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    codegen(Disposition::Devices)?;
    Ok(())
}
```

### Runtime Usage

Tasks use generated factories:

```rust
use drv_i2c_api::I2cDevice;

include!(concat!(env!("OUT_DIR"), "/i2c_config.rs"));

fn main() -> ! {
    let i2c_task = I2C.get_task_id();
    
    // Use generated factory
    let hot_swap = i2c_config::devices::ltc4282_hot_swap(i2c_task);
    
    // Perform I2C operations
    let temp: u16 = hot_swap.read_reg(0x8D)?;
}
```

## Configuration Best Practices

### 1. Device Organization
- Group related devices on the same controller/port
- Use descriptive names and reference designators
- Document device purposes in `description` field

### 2. Multiplexer Design
- Minimize mux depth (avoid mux-behind-mux)
- Use consistent segment numbering
- Provide reset GPIO for mux recovery

### 3. Sensor Configuration
- Match sensor counts to actual device capabilities
- Use meaningful rail names for power devices
- Provide sensor names for disambiguation

### 4. Address Management
- Avoid address conflicts on the same bus segment
- Reserve addresses for future expansion
- Document address allocation strategy

## Debugging Configuration Issues

### Common Build Errors

1. **Duplicate Names/Refdes**
   ```
   duplicate name hot_swap for device ltc4282
   ```
   Solution: Use unique names or add `flavor` field

2. **Invalid Bus References**
   ```
   device ltc4282 specifies unknown bus "main_bus"
   ```
   Solution: Ensure bus names match controller port names

3. **Mux/Segment Mismatches**
   ```
   device specifies a mux but no segment
   ```
   Solution: Always specify both mux and segment together

### Runtime Debugging

Use the validation system to test device connectivity:
```rust
let result = i2c_config::validation::validate(i2c_task, device_index)?;
match result {
    I2cValidation::Good => println!("Device validated successfully"),
    I2cValidation::Bad => println!("Device validation failed"),  
    I2cValidation::RawReadOk => println!("Basic I2C communication works"),
}
```

## Advanced Topics

### Custom Device Drivers

To add support for new devices:

1. **Add driver to `drv-i2c-devices`**
2. **Implement `Validate` trait**
3. **Update device configuration in `app.toml`**
4. **Rebuild to regenerate factories**

### Multi-Controller Systems

For systems with multiple I2C controllers:
- Configure each controller separately
- Use different dispositions for different roles
- Consider controller-specific features (speed, capabilities)

### Dynamic Device Discovery

While most configuration is static, you can combine generated factories with runtime discovery:

```rust
// Start with generated configuration
let mut devices = i2c_config::devices::temperature_sensor(i2c_task);

// Add dynamically discovered devices
for addr in discovered_addresses {
    let device = I2cDevice::new(i2c_task, controller, port, None, addr);
    devices.push(device);
}
```

## Conclusion

Hubris's I2C configuration system provides a powerful, type-safe way to manage complex I2C topologies. The build-time code generation ensures that device configurations are validated at compile time and provides ergonomic factory functions for runtime use. Understanding this system is crucial for effective I2C device management in Hubris applications.
