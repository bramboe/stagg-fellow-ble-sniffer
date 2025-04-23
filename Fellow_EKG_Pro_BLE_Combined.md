# Fellow Stagg EKG Pro BLE Protocol Comprehensive Guide

## Protocol Architecture Overview

After extensive testing and reverse engineering, we've determined that the Fellow Stagg EKG Pro kettle uses a sophisticated dual control system architecture:

1. **Display/State System**:
   - Controls what's shown on the kettle's display
   - Managed through standard BLE characteristic read/write operations
   - Changes don't directly affect physical heating elements

2. **Hardware Control System**:
   - Controls the physical heating elements and standby state
   - Managed through direct text commands or scheduled events
   - Commands that control physical elements may cause BLE disconnection

This dual architecture explains why attempts to toggle standby mode by modifying state bytes alone are unsuccessful - the physical heating state isn't directly represented in the BLE state data.

## BLE Service Architecture

### Primary Service
- **Service UUID**: `7AEBF330-6CB1-46E4-B23B-7CC2262C605E`

### Key Characteristics

| UUID | Name | Function | Length Requirements |
|------|------|----------|---------------------|
| `2291C4B1-5D7F-4477-A88B-B266EDB97142` | STATUS_CHAR | Status notifications | Variable |
| `2291C4B2-5D7F-4477-A88B-B266EDB97142` | STATUS2_CHAR | Secondary status | Variable |
| `2291C4B3-5D7F-4477-A88B-B266EDB97142` | STATUS3_CHAR | Tertiary status | Variable |
| `2291C4B4-5D7F-4477-A88B-B266EDB97142` | CONTROL_CHAR | Control authorization | Exactly 8 bytes |
| `2291C4B5-5D7F-4477-A88B-B266EDB97142` | MULTI_CHAR | Multi-purpose state data | Exactly 17 bytes |
| `2291C4B6-5D7F-4477-A88B-B266EDB97142` | COMMAND_CHAR | Text commands | Variable |
| `2291C4B7-5D7F-4477-A88B-B266EDB97142` | EXTRA_CHAR | Additional control/info | 2-20 bytes |
| `2291C4B8-5D7F-4477-A88B-B266EDB97142` | READ_ONLY_CHAR1 | Device name | Read-only |
| `2291C4B9-5D7F-4477-A88B-B266EDB97142` | READ_ONLY_CHAR2 | MAC address | Read-only |

### Command Length Requirements
- **MULTI_CHAR**: Must be exactly 17 bytes
- **CONTROL_CHAR**: Must be exactly 8 bytes
- **EXTRA_CHAR**: Accepts commands of length 2-20 bytes
- **COMMAND_CHAR**: Variable length for text commands

Sending commands with incorrect lengths results in error: `Error Domain=CBATTErrorDomain Code=13 "The value's length is invalid."`

## State Data Format

The MULTI_CHAR uses a 17-byte message format with this structure:

```
f7 17 00 80 50 80 c0 80 00 00 28 00 01 00 00 00 00
```

| Byte Position | Function | Values |
|---------------|----------|--------|
| 0-1 | Header/Mode | `f7 17` (Celsius), `ff 17` (special commands/schedule) |
| 2 | Fixed | `00` (fixed value) |
| 3 | Schedule Flag | `00` (normal), `80` (schedule mode), `04` (alternative mode) |
| 4 | Temperature | Temperature × 2 (e.g., 40°C = `50` hex = 80 decimal ÷ 2) |
| 5 | Unknown | Usually `80` |
| 6 | Display State | `80` (standby display), `D0`/`C0` (heating display), `6F` (activation display) |
| 7 | Display State 2 | Usually `80` |
| 8-9 | Schedule Time | Hour (byte 9) and minute (byte 8) for scheduled activation |
| 10 | Sequence Number | Increments with each state change |
| 11 | Clock Format | `01` (24-hour format), `00` (12-hour format) |
| 12 | Execution Flag | `01` (standard update), `02` (execute change), `33` (found to affect time) |
| 13 | Schedule Mode | `00` (normal), `80` (schedule mode), `1E` (observed in some states) |
| 14-15 | Unknown | Usually `00 00` but observed to affect time display when modified |
| 16 | Checksum | Sum of bytes 0-15 modulo 256 |

## Mode Transition Mechanism

The kettle's mode transitions involve multiple byte changes simultaneously:

#### Initial State (Standby Display)
```
f7 17 00 80 63 80 c0 80 00 00 15 00 01 80 00 00 98
```

#### Transition State (Heating Display)
```
ff 17 00 04 63 80 6f 80 00 00 18 01 02 80 00 00 a8
```

Key byte changes during transition:
- Byte 0: `0xf7` → `0xff` (Header/Mode change)
- Byte 3: `0x80` → `0x04` (Mode flag change)
- Byte 6: `0xc0` → `0x6f` (Display state change)
- Byte 10: `0x15` → `0x18` (Sequence number increment)
- Byte 12: `0x01` → `0x02` (Execution flag change)
- Byte 16: `0x98` → `0xa8` (Checksum recalculation)

### Display Control vs. Actual Heating Control

A critical discovery is that changes to byte 6 in the MULTI_CHAR only affect what's shown on the kettle's display:
- Value `0x80`: Shows STANDBY on display
- Value `0xD0` or `0xC0`: Shows HEATING on display
- Value `0x6F`: Special activation display value used in schedule commands

These display changes do not activate or deactivate the actual heating elements.

### State Data Identical Between Modes

The kettle's BLE state data doesn't distinguish between actual standby and heating modes:
```
STANDBY: f7 17 00 00 50 80 c0 80 00 00 26 09 01 1e 00 00 09
HEATING: f7 17 00 00 50 80 c0 80 00 00 26 09 01 1e 00 00 09
```

This confirms that the physical heating state isn't represented in the BLE state data.

### Schedule Mode Pattern
Comparison between normal and schedule modes:
```
Normal mode:   f7 17 00 00 50 80 80 00 00 00 25 01 01 00 00 00 6b
Schedule mode: f7 17 00 80 6a 80 c0 80 00 00 28 01 01 00 00 00 73
```

Key differences:
- Byte 3: `00` → `80` (Schedule mode flag)
- Byte 4: `50` → `6a` (Temperature change from 40°C to 53°C)
- Byte 6: `80` → `c0` (Display state change)
- Byte 10: `25` → `28` (Sequence number increment)

### Factory Reset Command Pattern

During factory reset, we observed a special pattern with `cd` values in bytes 4 and 6:
```
f7 15 00 00 cd 00 cd 00 00 00 19 01 02 1e 00 00 34
```

This might represent a special control sequence used during reset/standby operations.

## Clock Format Control

Byte 11 controls the clock display format:
- Value `01`: 24-hour format (e.g., 13:38)
- Value `00`: 12-hour format (e.g., 1:38 AM)

Implementation:
```python
# To set 12-hour format:
state = await client.read_gatt_char(MULTI_CHAR)
modified_state = bytearray(state)
modified_state[11] = 0x00  # 12-hour format
modified_state[12] = 0x02  # Execution flag
modified_state[16] = sum(modified_state[0:16]) % 256  # Checksum

# Send authorization
await client.write_gatt_char(CONTROL_CHAR, bytes([0x02, 0, 0, 0, 0, 0, 0, 0]))

# Send modified state
await client.write_gatt_char(MULTI_CHAR, modified_state)
```

## Time Control System

The kettle implements a sophisticated time control system that is deeply integrated with its scheduling functionality:

### Dual Time State System
- The kettle appears to maintain multiple internal time references
- Under certain conditions, the display can alternate between different time states
- Time changes observed during testing showed clear patterns:
  1. **Normal Advancement**: Time incrementing minute by minute as expected
  2. **State Switching**: Abrupt changes to a completely different time
  3. **Fixed Reference**: Returning to a specific time (12:28) after certain operations

### Critical Time Control Bytes
Several bytes have significant effects on the kettle's time display:

| Byte Position | Normal Function | Effect on Time | Observed Behavior |
|---------------|-----------------|----------------|-------------------|
| 8-9 | Schedule Time | Minor | Store schedule time but don't directly control display time |
| **12** | Execute Flag | **Major** | Setting to 0x33 caused time to jump from 12:27 → 11:28 (-59 min) |
| 13 | Schedule Mode | Significant | Modified display time and appeared to change time mode |
| 14-15 | Unknown | Moderate | Continued to affect time in the altered state |

## Firmware Version Information

Data from the EXTRA_CHAR reveals firmware information:
```
31 2e 31 2e 37 35 53 53 50 20 43 00 f6 c2 00 40 ff ff ff ff f0 3e 08 40 23 0c 06 00 10 26 09 40
```

When decoded as ASCII for the first part, this contains:
- `1.1.75SSP C` - Firmware version
- Followed by binary data (possibly calibration or configuration data)

## Display Control Methods

To change what's shown on the kettle's display (without affecting actual heating), use this two-step process:

### 1. Authorization Step

```python
# Send Control authorization
await client.write_gatt_char(CONTROL_CHAR, bytes([0x02, 0, 0, 0, 0, 0, 0, 0]))
```

### 2. Command with Execution Flag

```python
# Read current state
state = await client.read_gatt_char(MULTI_CHAR)

# Create modified state
modified_state = bytearray(state)
modified_state[6] = 0xD0  # Set to heating display (0x80 for standby)
modified_state[12] = 0x02  # Set execution flag
modified_state[16] = sum(modified_state[0:16]) % 256  # Recalculate checksum

# Send command with execution flag
await client.write_gatt_char(MULTI_CHAR, modified_state)
```

This process successfully changes the kettle's display between heating and standby indicators without affecting the actual heating elements.

## Heating Control Methods

### Schedule-Based Activation (RECOMMENDED)

The schedule-based approach uses a special command format and appears to respect temperature limits:

```python
async def activate_via_schedule(client, temp_celsius):
    # Read current state
    current_state = await client.read_gatt_char(MULTI_CHAR)
    
    # Create schedule command
    schedule_cmd = bytearray(17)
    
    # Basic structure
    schedule_cmd[0] = 0xff  # Special schedule prefix
    schedule_cmd[1] = 0x17  # Mode (Celsius)
    schedule_cmd[2] = 0x00  # Fixed
    schedule_cmd[3] = 0x80  # Schedule flag
    schedule_cmd[4] = int(temp_celsius * 2)  # Temperature × 2
    schedule_cmd[5] = 0x80  # Standard value
    schedule_cmd[6] = 0x6f  # Special activation byte
    schedule_cmd[7] = 0x80  # Standard value
    
    # Time bytes - currently using zeros but this affects the internal clock
    schedule_cmd[8] = 0x00  # Minutes
    schedule_cmd[9] = 0x00  # Hours
    
    # Increment sequence number
    schedule_cmd[10] = (current_state[10] + 1) % 256
    
    # Additional bytes
    schedule_cmd[11] = 0x00
    schedule_cmd[12] = 0x02  # Execution flag
    schedule_cmd[13] = 0x80  # Schedule mode
    schedule_cmd[14] = 0x00
    schedule_cmd[15] = 0x00
    
    # Calculate checksum
    checksum = sum(schedule_cmd[0:16]) % 256
    schedule_cmd[16] = checksum
    
    # Send authorization
    await client.write_gatt_char(CONTROL_CHAR, bytes([0x02, 0, 0, 0, 0, 0, 0, 0]))
    
    # Send schedule command
    await client.write_gatt_char(MULTI_CHAR, schedule_cmd)
```

**Limitation**: This approach affects the kettle's internal clock time. The time display may change as a side effect of using this method.

### Turning OFF Heating

The following text commands can turn off heating (all cause BLE disconnection):

```python
# Option 1: Reset command
await client.write_gatt_char(COMMAND_CHAR, b"reset")

# Option 2: Reset variants
await client.write_gatt_char(COMMAND_CHAR, b"reset on")
# or
await client.write_gatt_char(COMMAND_CHAR, b"reset off")

# Option 3: Factory command
await client.write_gatt_char(COMMAND_CHAR, b"factory")
```

All these commands cause BLE disconnection, requiring reconnection afterward.

### Direct Text Commands (NOT RECOMMENDED)

⚠️ **SAFETY WARNING** ⚠️

The following direct text commands have significant safety concerns and are **NOT RECOMMENDED** for general use:

- `heaton`: Activates heating elements without temperature control
  - **UNSAFE**: Will heat indefinitely, ignoring temperature settings
  - No automatic shutoff based on temperature
  - Could lead to dangerous boil-dry conditions

- `heatoff`: Attempts to deactivate heating elements

If "heaton" is ever used for testing purposes, **ALWAYS** implement strict external timeouts and send "heatoff" afterward:

```python
# UNSAFE - Only for testing with extreme caution
await client.write_gatt_char(COMMAND_CHAR, b"heaton")

# ALWAYS use a strict timeout
await asyncio.sleep(30)  # Never exceed 30 seconds

# ALWAYS turn off heating afterward
await client.write_gatt_char(COMMAND_CHAR, b"heatoff")
```

## Temperature Control

Temperature can be set by modifying byte 4 in the MULTI_CHAR:

```python
# Read current state
state = await client.read_gatt_char(MULTI_CHAR)

# Modify temperature (byte 4 = temperature × 2)
new_state = bytearray(state)
new_state[4] = int(desired_temp * 2)  # Temperature in Celsius

# Write back modified state
await client.write_gatt_char(MULTI_CHAR, new_state)
```

Valid temperature range: 40°C to 100°C (or 104°F to 212°F).

Note: Setting temperature only affects the display and the kettle's internal target. The actual heating behavior depends on which heating control method is used.

## Text Commands Tested

### WiFi-Related Commands
- `wifion` - Enable WiFi
- `wifioff` - Disable WiFi
- `wifista` - Enable WiFi station mode
- `httpfw` - Firmware-related function

### Display Control Commands Tested (All Unsuccessful)
The following commands were tested but did not affect the display as intended:
- "displayon", "display:on", "display:heat", "display:heating"
- "showheating", "show:heating", "show:heat", "show:on"
- "screenon", "screen:on", "screen:heat", "screen:heating"
- "displayheating", "display_heating", "display_heat", "display_on"
- "knob:on", "knob:heat", "knob:heating", "knob:display"
- "toggle:display", "toggle:heat", "toggle:heating"
- "wakeon", "wake:on", "wake:heat", "wake:heating"

### Heating-Related Commands
- "heaton" - Activates heating (UNSAFE)
- "heatoff" - Attempts to stop heating
- "knob" - No effect observed
- "temperature:off" - No effect observed
- "off off" - No effect observed

### Standby Control Commands
- "reset", "reset on", "reset off" - Turn off heating (cause disconnection)
- "factory" - Factory reset (causes disconnection)
- Other tested but unsuccessful: "power", "power:on", "power:off", "standby", "standby:toggle", "heat:on", "heat:off", "toggle", "mode", "status"

## Display Control Testing Approaches

Several approaches were tested to control the kettle's display:

### 1. Two-Step Display Control
- Send authorization to CONTROL_CHAR (`0x02`)
- Set display byte (6) to different values: `0xD0`, `0xC0`
- Set execution flag (byte 12) to `0x02`

### 2. Schedule Display Pattern
- Try mimicking the schedule command format with special display values
- Test different schedule flag (byte 3) values: `0x00`, `0x80`
- Test different display state (byte 6) values: `0x80`, `0xC0`, `0xD0`, `0x6F`

### 3. Combined Approaches
- Authorization + schedule pattern
- Special byte combinations
- Command sequences

### 4. Direct Text Commands
- Testing various text commands as listed above

## Comparison to Previous Model

The EKG Pro uses a completely different protocol design compared to the older EKG+ model:

### Previous Model (EKG+)
- **Service UUID**: `00001820-0000-1000-8000-00805f9b34fb`
- **Command Structure**:
  ```
  0xEF, 0xDD,       # Magic header
  0x0A,             # Command flag
  sequence_number,  # Incremented with each command
  command_type,     # 0=power, 1=temperature
  value,            # For power: 1=on, 0=off
  checksum1,        # (sequence + value) & 0xFF
  checksum2         # command_type
  ```
- **Power Control**: Simple command with command_type=0, value=1 (on) or 0 (off)

### Current Model (EKG Pro)
- Uses a more complex dual control architecture
- Text commands for physical control
- State bytes for display control
- Two-step authorization + execution pathway
- Testing confirmed that the older model's commands have no effect on the EKG Pro.

## Complete Connection and Control Example

```python
import asyncio
from bleak import BleakClient, BleakScanner

# Characteristic UUIDs
STATUS_CHAR = "2291C4B1-5D7F-4477-A88B-B266EDB97142"
MULTI_CHAR = "2291C4B5-5D7F-4477-A88B-B266EDB97142"
CONTROL_CHAR = "2291C4B4-5D7F-4477-A88B-B266EDB97142"
COMMAND_CHAR = "2291C4B6-5D7F-4477-A88B-B266EDB97142"
EXTRA_CHAR = "2291C4B7-5D7F-4477-A88B-B266EDB97142"

def notification_handler(sender, data):
    """Handle incoming notifications from the kettle."""
    print(f"Notification: {data.hex(' ')}")
    
    # Parse the data if possible
    if len(data) >= 7:
        display_byte = data[6]
        if display_byte == 0xD0:
            print("DISPLAY STATE: HEATING (D0)")
        elif display_byte == 0xC0:
            print("DISPLAY STATE: HEATING ALTERNATIVE (C0)")
        elif display_byte == 0x80:
            print("DISPLAY STATE: STANDBY (80)")
        else:
            print(f"DISPLAY STATE: UNKNOWN (0x{display_byte:02x})")
        
        if len(data) >= 5:
            temp_raw = data[4]
            if data[0] == 0xf7:  # Celsius mode
                temp = temp_raw / 2
                print(f"Temperature setting: {temp}°C")
            elif data[0] == 0xf5:  # Fahrenheit mode
                print(f"Temperature setting: {temp_raw}°F")

async def connect_to_kettle():
    """Find and connect to the Fellow Stagg EKG Pro kettle."""
    devices = await BleakScanner.discover()
    kettle = next((d for d in devices if d.name and "EKG" in d.name), None)
    
    if not kettle:
        print("No kettle found!")
        return None
    
    print(f"Connecting to {kettle.name}...")
    client = BleakClient(kettle.address)
    await client.connect()
    print(f"Connected to {kettle.name}")
    
    # Enable notifications
    await client.start_notify(STATUS_CHAR, notification_handler)
    print("Notifications enabled")
    
    # Read firmware version
    try:
        fw_data = await client.read_gatt_char(EXTRA_CHAR)
        fw_text = fw_data.decode('ascii', errors='ignore').split('\0')[0]
        print(f"Firmware version: {fw_text}")
    except Exception as e:
        print(f"Could not read firmware: {e}")
    
    return client

async def set_display_heating(client, show_heating=True):
    """Change the kettle's display to show heating or standby."""
    # Send authorization
    await client.write_gatt_char(CONTROL_CHAR, bytes([0x02, 0, 0, 0, 0, 0, 0, 0]))
    
    # Get current state
    state = await client.read_gatt_char(MULTI_CHAR)
    
    # Modify display state
    modified_state = bytearray(state)
    modified_state[6] = 0xD0 if show_heating else 0x80  # 0xD0 = heating, 0x80 = standby
    modified_state[12] = 0x02  # Execution flag
    modified_state[16] = sum(modified_state[0:16]) % 256  # Checksum
    
    # Send command
    await client.write_gatt_char(MULTI_CHAR, modified_state)
    print(f"Display set to {'HEATING' if show_heating else 'STANDBY'}")

async def set_temperature(client, temp_celsius):
    """Set the kettle's temperature."""
    # Get current state
    state = await client.read_gatt_char(MULTI_CHAR)
    
    # Modify temperature
    modified_state = bytearray(state)
    modified_state[4] = int(temp_celsius * 2)  # Temperature × 2
    modified_state[16] = sum(modified_state[0:16]) % 256  # Checksum
    
    # Send command
    await client.write_gatt_char(MULTI_CHAR, modified_state)
    print(f"Temperature set to {temp_celsius}°C")

async def set_clock_format(client, use_24h=True):
    """Set the clock format (12h or 24h)."""
    # Send authorization
    await client.write_gatt_char(CONTROL_CHAR, bytes([0x02, 0, 0, 0, 0, 0, 0, 0]))
    
    # Get current state
    state = await client.read_gatt_char(MULTI_CHAR)
    
    # Modify clock format
    modified_state = bytearray(state)
    modified_state[11] = 0x01 if use_24h else 0x00  # 0x01 = 24h, 0x00 = 12h
    modified_state[12] = 0x02  # Execution flag
    modified_state[16] = sum(modified_state[0:16]) % 256  # Checksum
    
    # Send command
    await client.write_gatt_char(MULTI_CHAR, modified_state)
    print(f"Clock format set to {('24-hour' if use_24h else '12-hour')}")

async def activate_via_schedule(client, temp_celsius):
    """Activate heating using schedule command."""
    # Read current state
    current_state = await client.read_gatt_char(MULTI_CHAR)
    
    # Create schedule command
    schedule_cmd = bytearray(17)
    
    # Basic structure
    schedule_cmd[0] = 0xff  # Special schedule prefix
    schedule_cmd[1] = 0x17  # Mode (Celsius)
    schedule_cmd[2] = 0x00  # Fixed
    schedule_cmd[3] = 0x80  # Schedule flag
    schedule_cmd[4] = int(temp_celsius * 2)  # Temperature × 2
    schedule_cmd[5] = 0x80  # Standard value
    schedule_cmd[6] = 0x6f  # Special activation byte
    schedule_cmd[7] = 0x80  # Standard value
    schedule_cmd[8] = 0x00  # Minutes
    schedule_cmd[9] = 0x00  # Hours
    schedule_cmd[10] = (current_state[10] + 1) % 256  # Sequence number
    schedule_cmd[11] = 0x00
    schedule_cmd[12] = 0x02  # Execution flag
    schedule_cmd[13] = 0x80  # Schedule mode
    schedule_cmd[14] = 0x00
    schedule_cmd[15] = 0x00
    schedule_cmd[16] = sum(schedule_cmd[0:16]) % 256  # Checksum
    
    # Send authorization
    await client.write_gatt_char(CONTROL_CHAR, bytes([0x02, 0, 0, 0, 0, 0, 0, 0]))
    
    # Send schedule command
    await client.write_gatt_char(MULTI_CHAR, schedule_cmd)
    print(f"Heating activated via schedule at {temp_celsius}°C")
    print("Note: This may change the kettle's clock time")

async def turn_off_heating(client):
    """Turn off heating (causes disconnection)."""
    print("Turning off heating (this will cause disconnection)...")
    await client.write_gatt_char(COMMAND_CHAR, b"reset")

async def main():
    client = await connect_to_kettle()
    if not client:
        return
    
    try:
        # Set temperature to 85°C
        await set_temperature(client, 85)
        
        # Change clock format to 24h
        await set_clock_format(client, True)
        await asyncio.sleep(2)
        
        # Change display to show heating (doesn't activate actual heating)
        await set_display_heating(client, True)
        await asyncio.sleep(2)
        
        # Change display to show standby
        await set_display_heating(client, False)
        await asyncio.sleep(2)
        
        # Activate heating via schedule (this affects the clock time)
        await activate_via_schedule(client, 85)
        await asyncio.sleep(10)
        
        # Turn off heating (causes disconnection)
        await turn_off_heating(client)
    
    except Exception as e:
        print(f"Error: {e}")
    
    finally:
        # May not reach here due to disconnection
        if client.is_connected:
            await client.disconnect()

if __name__ == "__main__":
    asyncio.run(main())
```

## Unsuccessful Approaches

Many approaches were tested but did not successfully control the actual heating elements:

### 1. Direct Display State Changes
- Changing byte 6 from `0x80` to `0xD0` only changes the display
- No effect on actual heating elements

### 2. Direct Standby Toggle via State Bytes
- Attempts to toggle standby mode by modifying bytes in MULTI_CHAR
- No effect on actual heating function

### 3. Sequence Number Manipulation
- Incrementing byte 10 (sequence number) in various patterns
- No effect on standby function despite observable changes in states

### 4. Display Text Commands
- Numerous text commands as listed in "Display Control Commands Tested"
- None successfully toggled the display or heating function

### 5. EKG+ Style Commands
- Attempting to use the previous model's command structure
- Different error message ("The request is not supported") but no success

### 6. Bit Toggles in Control Characteristic
- Individual bit toggles in CONTROL_CHAR bytes
- Various patterns (all bits ON, half ON, etc.)
- No effect on heating function

## Safety Considerations

1. **Temperature Control Limitation**: The direct "heaton" command activates heating regardless of temperature settings and doesn't automatically stop.

2. **Temperature Measurement Limitation**: You cannot read the current water temperature through BLE.

3. **Safety Implications**: Using direct heating commands without safety measures could lead to overheating.

4. **Physical Priority**: The physical button always takes precedence over BLE commands, ensuring the user can manually override any remote control.

5. **Recommended Approach**: Use the schedule-based approach which respects temperature settings, though it affects the clock time.

## Future Research Directions

1. Discover a method to control heating that doesn't affect the clock time
2. Explore alternative text commands that might offer safer control
3. Investigate ways to read the current water temperature via BLE
4. Find a way to toggle heating without disconnection
5. Better understand the time encoding for schedules
6. Explore the relationship between the kettle's internal clock and command authorization

## Conclusion

The Fellow Stagg EKG Pro kettle uses a sophisticated dual control system with separate display and hardware control mechanisms. While the kettle's display can be easily controlled via BLE, actual heating control is more limited and relies on either scheduled events or direct text commands with safety concerns.

This protocol architecture appears to be designed intentionally for safety reasons, preventing accidental remote activation of the heating elements through simple BLE commands.

---

# Fellow Stagg EKG Pro BLE Protocol Documentation

## Protocol Architecture Overview

After extensive testing and reverse engineering, we've determined that the Fellow Stagg EKG Pro kettle uses a sophisticated dual control system architecture:

1. **Display/State System**:
   - Controls what's shown on the kettle's display
   - Managed through standard BLE characteristic read/write operations
   - Changes don't directly affect physical heating elements

2. **Hardware Control System**:
   - Controls the physical heating elements and standby state
   - Managed through direct text commands
   - Commands that control physical elements cause BLE disconnection

This dual architecture explains why earlier attempts to toggle standby mode by modifying state bytes were unsuccessful - the physical standby state isn't represented in the BLE state data.

## BLE Service Architecture

### Primary Service
- **Service UUID**: `7AEBF330-6CB1-46E4-B23B-7CC2262C605E`

### Key Characteristics
| UUID | Name | Function | Length Requirements |
|------|------|----------|---------------------|
| `2291C4B1-5D7F-4477-A88B-B266EDB97142` | STATUS_CHAR | Status notifications | Variable |
| `2291C4B5-5D7F-4477-A88B-B266EDB97142` | MULTI_CHAR | Multi-purpose state data | Exactly 17 bytes |
| `2291C4B4-5D7F-4477-A88B-B266EDB97142` | CONTROL_CHAR | Control authorization | Exactly 8 bytes |
| `2291C4B6-5D7F-4477-A88B-B266EDB97142` | COMMAND_CHAR | Text commands | Variable |

## State Data Format

The MULTI_CHAR uses a 17-byte message format with this structure:

```
f7 17 00 80 50 80 c0 80 00 00 28 00 01 00 00 00 00
```

| Byte Position | Function | Values |
|---------------|----------|--------|
| 0-1 | Header/Mode | `f7 17` (standard), `ff 17` (special commands) |
| 3 | Schedule Flag | `00` (normal), `80` (schedule mode) |
| 4 | Temperature | Temperature × 2 (e.g., 40°C = `50` hex = 80 decimal ÷ 2) |
| 5-7 | Display State | Various values control display appearance |
| 8-9 | Schedule Time | Encodes schedule time parameters |
| 10 | Sequence Number | Increments with each state change |
| 12 | Execution Flag | `01` (standard update), `02` (execute change) |
| 16 | Checksum/Control | Varies with packet contents |

## Standby Control

### Turning OFF (Heating → Standby)
Only text commands through COMMAND_CHAR can trigger the transition from heating to standby:

- `reset`
- `reset on`
- `reset off`
- `factory`

All these commands cause BLE disconnection as a side effect, requiring reconnection afterward.

### Turning ON (Standby → Heating)
There are only two ways to turn the kettle ON:

1. **Physical Button**: Press the physical knob button on the kettle
2. **Schedule Activation**: When a scheduled event triggers, the kettle can turn itself on

Notably, there is **no direct BLE command** to turn the kettle ON without using the physical button or a schedule.

## Temperature Control

Temperature can be set by modifying byte 4 in the MULTI_CHAR state data:

```python
# Read current state
state = await client.read_gatt_char(MULTI_CHAR)

# Modify temperature (byte 4 = temperature × 2)
new_state = bytearray(state)
new_state[4] = int(desired_temp * 2)  # Temperature in Celsius

# Write back modified state
await client.write_gatt_char(MULTI_CHAR, new_state)
```

Valid temperature range: 40°C to 100°C (or 104°F to 212°F).

## Schedule Control

Schedules can be set using a specific byte pattern:

```
ff 17 00 80 xx 80 6f 80 0b 00 yy 00 01 80 00 00 zz
```

Where:
- `xx` = Temperature × 2
- `yy` = Sequence number (incremented)
- `zz` = Checksum

Byte 3 = `80` indicates schedule mode, and bytes 8-10 appear to encode time information.

## Display Control Execution Pathway

To execute physical display changes (like toggling clock format), use:

1. **Authorization Step**:
   ```python
   # Send Control authorization
   await client.write_gatt_char(CONTROL_CHAR, bytes([0x02, 0, 0, 0, 0, 0, 0, 0]))
   ```

2. **Command with Execution Flag**:
   ```python
   # Read current state
   state = await client.read_gatt_char(MULTI_CHAR)
   
   # Set execution flag (byte 12 = 0x02)
   modified_state = bytearray(state)
   modified_state[12] = 0x02
   
   # Send command with execution flag
   await client.write_gatt_char(MULTI_CHAR, modified_state)
   ```

This two-step process is required for display changes to take effect.

## Implementation Recommendations

Based on our findings, here's the recommended approach for controlling the kettle:

### Turning OFF
```python
# Send reset command (will cause disconnection)
await client.write_gatt_char(COMMAND_CHAR, b"reset")

# Handle reconnection
# Wait for disconnection then reconnect
```

### Setting Temperature
```python
# Read current state
state = await client.read_gatt_char(MULTI_CHAR)

# Modify temperature (byte 4 = temp × 2)
new_state = bytearray(state)
new_state[4] = int(desired_temp * 2)

# Write modified state
await client.write_gatt_char(MULTI_CHAR, new_state)
```

### Setting Schedule
```python
# Read current state
state = await client.read_gatt_char(MULTI_CHAR)

# Create schedule command
schedule_cmd = bytearray(state)
schedule_cmd[0] = 0xff  # Special prefix
schedule_cmd[3] = 0x80  # Schedule mode
schedule_cmd[4] = int(temp * 2)  # Temperature
# Set time bytes (8-10)
# Increment sequence counter (byte 10)

# Send schedule command
await client.write_gatt_char(MULTI_CHAR, schedule_cmd)
```

## Key Protocol Discoveries

1. **State Data Identical Between Modes**:
   ```
   STANDBY: f7 17 00 00 50 80 c0 80 00 00 26 09 01 1e 00 00 09
   HEATING: f7 17 00 00 50 80 c0 80 00 00 26 09 01 1e 00 00 09
   ```
   The kettle's BLE state data doesn't distinguish between standby and heating modes.

2. **Schedule Mode Pattern**:
   ```
   Normal:   f7 17 00 00 50 80 80 00 00 00 25 01 01 00 00 00 6b
   Schedule: f7 17 00 80 6a 80 c0 80 00 00 28 01 01 00 00 00 73
   ```
   Key differences in bytes 3, 4, and 13 indicate schedule mode.

3. **Clock Format Control**:
   - Byte 11 = `01`: 24-hour format
   - Byte 11 = `00`: 12-hour format

4. **Firmware Version Information** (from EXTRA_CHAR):
   ```
   31 2e 31 2e 37 35 53 53 50 20 43
   ```
   Decodes to: `1.1.75SSP C` - Firmware version

## Comparison to Previous Model

The EKG Pro uses a completely different protocol design compared to the older model:

1. The older model used binary command patterns for on/off:
   - On: `efdd0a0000010100`
   - Off: `efdd0a0400000400`

2. The EKG Pro uses:
   - Text commands for physical control
   - State bytes for display control
   - A two-step authorization + execution pathway

Testing confirmed that the older model's commands have no effect on the EKG Pro.

## Safety Considerations

The kettle's protocol design prioritizes safety:
- Remote activation is intentionally limited
- Disconnection during standby toggle is likely a safety feature
- Only scheduled events can activate heating without physical interaction

This dual control architecture with restricted activation capabilities is common in appliances with heating elements to prevent accidents or unintended activation.

---

# Fellow Stagg EKG Pro BLE Protocol Documentation

## Protocol Architecture Overview

The Fellow Stagg EKG Pro kettle uses a sophisticated dual control system architecture specifically designed with safety in mind:

1. **Display/State System**:
   - Controls what's shown on the kettle's display
   - Managed through standard BLE characteristic read/write operations
   - Changes don't directly affect physical heating elements

2. **Hardware Control System**:
   - Controls the physical heating elements and standby state
   - Managed through either direct text commands or scheduled events
   - Commands that control physical elements may cause BLE disconnection

This dual architecture is a critical safety feature that prevents unintentional or unsafe remote activation of the heating elements.

## BLE Service Architecture

### Primary Service
- **Service UUID**: `7AEBF330-6CB1-46E4-B23B-7CC2262C605E`

### Key Characteristics
| UUID | Name | Function | Length Requirements |
|------|------|----------|---------------------|
| `2291C4B1-5D7F-4477-A88B-B266EDB97142` | STATUS_CHAR | Status notifications | Variable |
| `2291C4B5-5D7F-4477-A88B-B266EDB97142` | MULTI_CHAR | Multi-purpose state data | Exactly 17 bytes |
| `2291C4B4-5D7F-4477-A88B-B266EDB97142` | CONTROL_CHAR | Control authorization | Exactly 8 bytes |
| `2291C4B6-5D7F-4477-A88B-B266EDB97142` | COMMAND_CHAR | Text commands | Variable |
| `2291C4B7-5D7F-4477-A88B-B266EDB97142` | EXTRA_CHAR | Additional information | Variable |
| `2291C4B8-5D7F-4477-A88B-B266EDB97142` | READ_ONLY_CHAR1 | Device name | Read-only |
| `2291C4B9-5D7F-4477-A88B-B266EDB97142` | READ_ONLY_CHAR2 | MAC address | Read-only |

## State Data Format

The MULTI_CHAR uses a 17-byte message format with this structure:

```
f7 17 00 80 50 80 c0 80 00 00 28 00 01 00 00 00 00
```

| Byte Position | Function | Values |
|---------------|----------|--------|
| 0-1 | Header/Mode | `f7 17` (standard), `ff 17` (schedule mode) |
| 3 | Schedule Flag | `00` (normal), `80` (schedule mode) |
| 4 | Temperature | Temperature × 2 (e.g., 40°C = `50` hex = 80 decimal ÷ 2) |
| 5-7 | Display State | Various values control display appearance |
| 8-9 | Schedule Time | Hours (byte 8) and minutes (byte 9) for scheduled activation |
| 10 | Sequence Number | Increments with each state change |
| 12 | Execution Flag | `01` (standard update), `02` (execute change) |
| 16 | Checksum | Sum of bytes 0-15 modulo 256 |

## Control Methods

### Heating Control Commands

The following text commands control the kettle's heating functionality and display options:

| Command | Effect | Notes |
|---------|--------|-------|
| `heaton` | Activates heating elements | **UNSAFE**: Ignores temperature limits and will heat indefinitely without automatic shutoff |
| `heatoff` | Deactivates heating elements | May cause BLE disconnection |
| `reset` | Deactivates heating elements | Causes BLE disconnection |
| `reset on` | Deactivates heating elements | Causes BLE disconnection |
| `reset off` | Deactivates heating elements | Causes BLE disconnection |
| `factory` | Deactivates heating elements | Causes BLE disconnection, resets some settings |
| `setdigital` | Sets clock to digital display | Changes display format only |
| `setanalog` | Sets clock to analog display | Changes display format only |

### Important Safety Limitations

1. **Temperature Control Limitation**: The `heaton` command activates heating regardless of the target temperature set in the kettle. It will heat water indefinitely without automatic shutoff based on temperature.

2. **Temperature Measurement Limitation**: It is not possible to measure or read the current water temperature through BLE. The kettle does not expose this data via BLE.

3. **Safety Implications**: Using the `heaton` command without additional safety measures could lead to overheating and potential fire hazard. This makes it less ideal for unattended operation or integration with home automation systems where safety is a concern.

### Schedule-Based Activation (RECOMMENDED)

The schedule-based approach is the **RECOMMENDED** method for controlling heating as it respects temperature limits and automatically stops when temperature is reached.

Schedule commands use a special format with the MULTI_CHAR characteristic:
```
ff 17 00 80 xx 80 6f 80 MM HH yy 00 02 80 00 00 zz
```

Where:
- `xx` = Temperature × 2
- `MM` = Minute encoding
- `HH` = Hour encoding
- `yy` = Sequence number (incremented)
- `zz` = Checksum

Implementation pattern:
```python
# Create schedule command
schedule_cmd = bytearray(17)
schedule_cmd[0] = 0xff  # Special schedule prefix
schedule_cmd[1] = 0x17  # Mode (Celsius)
schedule_cmd[2] = 0x00  # Fixed
schedule_cmd[3] = 0x80  # Schedule flag
schedule_cmd[4] = int(temperature * 2)  # Temperature × 2
schedule_cmd[5] = 0x80  # Standard value
schedule_cmd[6] = 0xd0  # Power byte (heating)
schedule_cmd[7] = 0x80  # Standard value
schedule_cmd[8] = hours  # Target hour (0-23)
schedule_cmd[9] = minutes  # Target minutes (0-59)
schedule_cmd[10] = next_seq  # Sequence number (incremented)
schedule_cmd[11] = 0x00
schedule_cmd[12] = 0x02  # Execute flag
schedule_cmd[13] = 0x80  # Schedule mode
schedule_cmd[14] = 0x00
schedule_cmd[15] = 0x00
schedule_cmd[16] = checksum  # Sum of bytes 0-15 modulo 256

# Send authorization
await client.write_gatt_char(CONTROL_CHAR, bytes([0x02, 0, 0, 0, 0, 0, 0, 0]))

# Send schedule command
await client.write_gatt_char(MULTI_CHAR, schedule_cmd)
```

### Temperature Setting

Temperature can be set by modifying byte 4 in the MULTI_CHAR state data:

```python
# Read current state
state = await client.read_gatt_char(MULTI_CHAR)

# Modify temperature (byte 4 = temperature × 2)
new_state = bytearray(state)
new_state[4] = int(desired_temp * 2)  # Temperature in Celsius

# Write back modified state
await client.write_gatt_char(MULTI_CHAR, new_state)
```

Valid temperature range: 40°C to 100°C (or 104°F to 212°F).

Note: Setting temperature only affects the display and the kettle's internal target. The `heaton` command ignores these settings, but scheduled activations respect them.

### Display Control Execution Pathway

To execute physical display changes (like toggling clock format), use:

1. **Authorization Step**:
   ```python
   # Send Control authorization
   await client.write_gatt_char(CONTROL_CHAR, bytes([0x02, 0, 0, 0, 0, 0, 0, 0]))
   ```

2. **Command with Execution Flag**:
   ```python
   # Read current state
   state = await client.read_gatt_char(MULTI_CHAR)
   
   # Set execution flag (byte 12 = 0x02)
   modified_state = bytearray(state)
   modified_state[12] = 0x02
   
   # Send command with execution flag
   await client.write_gatt_char(MULTI_CHAR, modified_state)
   ```

This two-step process is required for display changes to take effect.

## Implementation Recommendations

Based on our findings, here's the recommended approach for controlling the kettle:

### Turning ON (with schedule - RECOMMENDED)
```python
# Set up a schedule for immediate activation
# This respects temperature settings and safety features
temp_celsius = 85
current_hour = 12  # Current hour or when you want activation to occur
current_minute = 30  # Current minute or when you want activation to occur

# Create schedule command (see schedule implementation pattern above)
# ...

# Send authorization and schedule command
await client.write_gatt_char(CONTROL_CHAR, bytes([0x02, 0, 0, 0, 0, 0, 0, 0]))
await client.write_gatt_char(MULTI_CHAR, schedule_cmd)
```

### Turning ON (direct method - USE WITH CAUTION)
```python
# Send heaton command (activates heating without temperature control)
await client.write_gatt_char(COMMAND_CHAR, b"heaton")

# IMPORTANT: Implement your own safety timeout
await asyncio.sleep(max_duration)  # e.g., 120 seconds

# Turn off after timeout
await client.write_gatt_char(COMMAND_CHAR, b"heatoff")
```

### Turning OFF
```python
# Send heatoff command
await client.write_gatt_char(COMMAND_CHAR, b"heatoff")

# Alternative: Send reset command (will cause disconnection)
# await client.write_gatt_char(COMMAND_CHAR, b"reset")
# Handle reconnection
```

### Setting Temperature
```python
# Read current state
state = await client.read_gatt_char(MULTI_CHAR)

# Modify temperature (byte 4 = temp × 2)
new_state = bytearray(state)
new_state[4] = int(desired_temp * 2)

# Write modified state
await client.write_gatt_char(MULTI_CHAR, new_state)
```

## Key Protocol Insights

1. **Dual Control Architecture**:
   - No BLE data is transmitted when the kettle executes a scheduled heating event
   - This confirms that heating activation happens via internal logic, not BLE commands
   - The schedule-based approach works with the kettle's internal safety systems

2. **Display vs. Hardware Control**:
   - Changing the power byte (position 6) from `80` to `D0` only affects the display
   - It shows "HEATING" on screen but doesn't activate the actual heating elements

3. **State Data Identical Between Modes**:
   ```
   STANDBY: f7 17 00 00 50 80 c0 80 00 00 26 09 01 1e 00 00 09
   HEATING: f7 17 00 00 50 80 c0 80 00 00 26 09 01 1e 00 00 09
   ```
   The kettle's BLE state data doesn't distinguish between standby and heating modes.

4. **Clock Settings and Time Effects**:
   - Byte 11 controls the clock display format:
     - Value `01`: 24-hour format (e.g., 13:38)
     - Value `00`: 12-hour format (e.g., 1:38 AM)
   - Some BLE commands unexpectedly affect the kettle's internal clock
   - Schedule commands particularly may cause time changes

## Safety Considerations

1. **Temperature Control Limitation**:
   - The `heaton` command activates heating regardless of the target temperature
   - It will heat water indefinitely without automatic shutoff based on temperature

2. **Temperature Measurement Limitation**:
   - It is not possible to measure or read the current water temperature through BLE
   - The kettle does not expose this data via BLE

3. **Safety Implications**:
   - Using the `heaton` command without additional safety measures could lead to overheating
   - Always implement strict timeouts when using direct heating commands
   - The schedule-based approach is the only recommended method for remote heating

4. **Physical Priority**:
   - The physical button always takes precedence over BLE commands
   - This ensures the user can always manually override any remote control

## Firmware Information

The EXTRA_CHAR provides firmware version information:
```
31 2e 31 2e 37 35 53 53 50 20 43 00
```

When decoded as ASCII, this reveals:
- `1.1.75SSP C` - Firmware version information

## Comparison to Previous Model

The EKG Pro uses a completely different protocol design compared to the older model:

1. The older model used binary command patterns for on/off:
   - On: `efdd0a0000010100`
   - Off: `efdd0a0400000400`

2. The EKG Pro uses:
   - Text commands for physical control
   - State bytes for display control
   - A two-step authorization + execution pathway

Testing confirmed that the older model's commands have no effect on the EKG Pro.

## Notes on Earlier Assumptions

Some earlier documentation contained different information that has been updated as the protocol was more thoroughly researched:

1. **Display vs. Heating Control**: Initially, it was thought that bytes 6-7 in the MULTI_CHAR controlled the actual heating elements. Later research showed these only control the display.

2. **Power Control**: Early attempts to toggle standby mode by modifying state bytes were unsuccessful because the physical standby state isn't represented in the BLE state data.

3. **Control Command Paths**: Earlier documentation didn't identify the text commands (`heaton`, `heatoff`, etc.) or their safety implications.

## Conclusion

The Fellow Stagg EKG Pro kettle uses a sophisticated and safety-oriented BLE protocol design. The dual control architecture with limited remote activation capabilities is an intentional safety feature. When implementing control solutions, always prioritize the schedule-based approach which works with the kettle's internal safety systems rather than trying to bypass them.

> **Note:** Be aware that some BLE commands, particularly scheduling commands, may inadvertently change the kettle's internal clock time. Consider this side effect when implementing automation solutions.

This documentation represents findings from extensive reverse engineering and testing. While the protocol details described here have been verified to work with the Fellow Stagg EKG Pro model, implementation details may vary across firmware versions.

---

# Fellow Stagg EKG Pro BLE Protocol Guide

This document provides a comprehensive guide to the reverse-engineered Bluetooth Low Energy (BLE) protocol for the Fellow Stagg EKG Pro kettle.

## Connection Establishment

### Service and Characteristic UUIDs

- **Primary Service UUID**: `7AEBF330-6CB1-46E4-B23B-7CC2262C605E`

- **Key Characteristics**:
  - Status Notifications: `2291C4B1-5D7F-4477-A88B-B266EDB97142` (Handle 0x0027)
  - Multi-purpose Control: `2291C4B5-5D7F-4477-A88B-B266EDB97142`
  - Command Channel: `2291C4B6-5D7F-4477-A88B-B266EDB97142`

- **Additional Characteristics**:
  - `2291C4B2-5D7F-4477-A88B-B266EDB97142` (Additional notifications)
  - `2291C4B3-5D7F-4477-A88B-B266EDB97142` (Additional notifications)
  - `2291C4B4-5D7F-4477-A88B-B266EDB97142` (Control characteristic)
  - `2291C4B7-5D7F-4477-A88B-B266EDB97142` (Additional control)
  - `2291C4B8-5D7F-4477-A88B-B266EDB97142` (Read-only)
  - `2291C4B9-5D7F-4477-A88B-B266EDB97142` (Read-only)

### Connection Steps

1. Scan for BLE devices with "EKG" in the name
2. Connect to the device
3. Enable notifications on the status characteristic
4. Send WiFi initialization commands (optional)
5. Begin sending control commands

### Example Connection Code (Python with Bleak)

```python
import asyncio
from bleak import BleakClient, BleakScanner

# Characteristic UUIDs
STATUS_CHAR = "2291C4B1-5D7F-4477-A88B-B266EDB97142"
MULTI_CHAR = "2291C4B5-5D7F-4477-A88B-B266EDB97142"

def notification_handler(sender, data):
    """Handle incoming notifications from the kettle."""
    print(f"Notification: {data.hex()}")

async def connect_to_kettle():
    # Scan for kettle
    devices = await BleakScanner.discover()
    kettle = next((d for d in devices if d.name and "EKG" in d.name), None)
    
    if not kettle:
        print("No kettle found!")
        return
    
    print(f"Connecting to {kettle.name} at {kettle.address}...")
    async with BleakClient(kettle.address) as client:
        print("Connected!")
        
        # Enable notifications
        await client.start_notify(STATUS_CHAR, notification_handler)
        
        # Now ready to send commands
        # ...
```

## Command Structure

### Command Format

Commands sent to the kettle must follow a specific format. All commands sent to the MULTI_CHAR must be exactly 17 bytes (34 hex characters) in length.

#### Temperature Setting Commands

Format: `f7 [mode] 00 00 [temp] 00 [power] 00 00 00 00 [control bytes] 00 00 [checksum]`

Where:
- `mode`: `17` for Celsius, `15` for Fahrenheit
- `temp`: Temperature value (2× Celsius, 1× Fahrenheit)
- `power`: `D0` for heating, `80` for standby
- `control bytes`: Varies based on command type
- `checksum`: Simple sum of all bytes modulo 256 (optional)

#### Observed Working Command Patterns

1. **Celsius Base Command**: `f717006400000000000000000000000000`
2. **Fahrenheit Base Command**: `f715006400000000000000000000000000`
3. **Celsius (from packet capture)**: `f7170000c00c800000001304011e00000b`
4. **Fahrenheit (from packet capture)**: `f7150000c00cd000000012040a1e0000`

### Text Commands

Text commands can be sent to the COMMAND_CHAR. Known commands include:

- `wifion` - Enable WiFi
- `wifista` - Enable WiFi station mode
- `httpfw` - Firmware-related function

## Temperature Control

### Setting Temperature

Temperature values are encoded differently between Celsius and Fahrenheit:

- **Celsius**: Temperature value is 2× the desired temperature
  - Example: 85°C is encoded as `0xAA` (170 decimal)
- **Fahrenheit**: Temperature value is the same as the desired temperature
  - Example: 185°F is encoded as `0xB9` (185 decimal)

### Temperature Mode Selection

- **Celsius Mode**: Use prefix `f7170000`
- **Fahrenheit Mode**: Use prefix `f7150000`

### Example Temperature Commands

1. **Set to 85°C**:
   ```python
   temp = 85
   temp_raw = int(temp * 2)  # 170
   cmd = bytearray.fromhex("f717006400000000000000000000000000")
   cmd[4] = temp_raw
   # Add checksum if needed
   checksum = sum(cmd) % 256
   cmd.append(checksum)
   await client.write_gatt_char(MULTI_CHAR, cmd)
   ```

2. **Set to 195°F**:
   ```python
   temp = 195
   cmd = bytearray.fromhex("f715006400000000000000000000000000")
   cmd[4] = temp
   # Add checksum if needed
   checksum = sum(cmd) % 256
   cmd.append(checksum)
   await client.write_gatt_char(MULTI_CHAR, cmd)
   ```

## Power/Standby Control

The kettle appears to have two different power states that can be controlled via BLE:

1. **Complete Power OFF**: Disconnects BLE connection
2. **Standby Mode**: Like pressing the physical knob, keeps BLE connected

### Standby Mode Control

To toggle between heating and standby (like pressing the physical knob), byte 7 (index 6) in the command appears to be the control byte:

- `0xD0`: Heating mode
- `0x80`: Standby mode

### Example Standby Toggle Command

```python
# Set to standby mode
cmd = bytearray.fromhex("f717006400000000000000000000000000")
cmd[6] = 0x80  # Standby byte
await client.write_gatt_char(MULTI_CHAR, cmd)

# Set to heating mode
cmd = bytearray.fromhex("f717006400000000000000000000000000")
cmd[6] = 0xD0  # Heating byte
await client.write_gatt_char(MULTI_CHAR, cmd)
```

## Status Notification Interpretation

When notifications are enabled, the kettle sends status updates that can be interpreted:

- First 4 bytes: Mode (`f7170000` = Celsius, `f7150000` = Fahrenheit)
- Byte 5 (index 4): Current temperature (2× for Celsius)
- Byte 7 (index 6): Possibly power state (`0xD0` = Heating, `0x80` = Standby)

Example notification parser:

```python
def parse_notification(data):
    if len(data) >= 4:
        mode_bytes = data[0:4]
        if mode_bytes == bytes.fromhex("F7150000"):
            mode = "Fahrenheit"
            if len(data) >= 5:
                temp = data[4]
                print(f"Temperature: {temp}°F")
        elif mode_bytes == bytes.fromhex("F7170000"):
            mode = "Celsius"
            if len(data) >= 5:
                temp = data[4] / 2
                print(f"Temperature: {temp}°C")
        
        if len(data) >= 7:
            if data[6] == 0xD0:
                state = "HEATING"
            elif data[6] == 0x80:
                state = "STANDBY"
            else:
                state = f"UNKNOWN (0x{data[6]:02X})"
            print(f"Kettle state: {state}")
```

## Putting It All Together

### Complete Kettle Control Example

```python
import asyncio
from bleak import BleakClient, BleakScanner

# Characteristic UUIDs
STATUS_CHAR = "2291C4B1-5D7F-4477-A88B-B266EDB97142"
MULTI_CHAR = "2291C4B5-5D7F-4477-A88B-B266EDB97142"
COMMAND_CHAR = "2291C4B6-5D7F-4477-A88B-B266EDB97142"

# Command templates
CELSIUS_TEMPLATE = "f717006400000000000000000000000000"
FAHRENHEIT_TEMPLATE = "f715006400000000000000000000000000"

def notification_handler(sender, data):
    """Handle incoming notifications from the kettle."""
    print(f"Notification: {data.hex()}")
    
    # Parse kettle state
    if len(data) >= 4:
        # Parse temperature mode
        if data[0:2] == bytes.fromhex("F717"):
            print("Mode: Celsius")
            if len(data) >= 5:
                print(f"Current temperature: {data[4]/2}°C")
        elif data[0:2] == bytes.fromhex("F715"):
            print("Mode: Fahrenheit")
            if len(data) >= 5:
                print(f"Current temperature: {data[4]}°F")
        
        # Parse power state (if present)
        if len(data) >= 7:
            if data[6] == 0xD0:
                print("Kettle is HEATING")
            elif data[6] == 0x80:
                print("Kettle is in STANDBY")

async def set_temperature(client, temp, use_fahrenheit=False):
    """Set kettle temperature."""
    if use_fahrenheit:
        template = FAHRENHEIT_TEMPLATE
        temp_raw = temp  # Direct value for Fahrenheit
    else:
        template = CELSIUS_TEMPLATE
        temp_raw = int(temp * 2)  # 2× for Celsius
    
    # Create command
    cmd = bytearray.fromhex(template)
    cmd[4] = temp_raw  # Set temperature byte
    cmd[6] = 0xD0      # Set to heating mode
    
    # Add checksum (optional)
    checksum = sum(cmd) % 256
    cmd.append(checksum)
    
    # Send command
    print(f"Setting temperature to {temp}{'°F' if use_fahrenheit else '°C'}")
    await client.write_gatt_char(MULTI_CHAR, cmd)
    print("Temperature set!")

async def toggle_standby(client, standby=True):
    """Toggle between heating and standby mode."""
    # Create command
    cmd = bytearray.fromhex(CELSIUS_TEMPLATE)
    cmd[6] = 0x80 if standby else 0xD0  # 0x80 for standby, 0xD0 for heating
    
    # Add checksum (optional)
    checksum = sum(cmd) % 256
    cmd.append(checksum)
    
    # Send command
    print(f"Setting kettle to {'STANDBY' if standby else 'HEATING'} mode")
    await client.write_gatt_char(MULTI_CHAR, cmd)
    print("Mode changed!")

async def control_kettle():
    """Connect to kettle and provide control interface."""
    # Scan for kettle
    devices = await BleakScanner.discover()
    kettle = next((d for d in devices if d.name and "EKG" in d.name), None)
    
    if not kettle:
        print("No kettle found!")
        return
    
    print(f"Connecting to {kettle.name} at {kettle.address}...")
    async with BleakClient(kettle.address) as client:
        print("Connected!")
        
        # Enable notifications
        await client.start_notify(STATUS_CHAR, notification_handler)
        print("Notifications enabled")
        
        # Main control loop
        while True:
            print("\n=== KETTLE CONTROL ===")
            print("1. Set temperature (Celsius)")
            print("2. Set temperature (Fahrenheit)")
            print("3. Enter heating mode")
            print("4. Enter standby mode")
            print("5. Exit")
            
            choice = input("Select option: ")
            
            if choice == "1":
                temp = float(input("Enter temperature (°C): "))
                await set_temperature(client, temp, False)
            elif choice == "2":
                temp = float(input("Enter temperature (°F): "))
                await set_temperature(client, temp, True)
            elif choice == "3":
                await toggle_standby(client, False)  # Heating mode
            elif choice == "4":
                await toggle_standby(client, True)   # Standby mode
            elif choice == "5":
                print("Exiting...")
                break
            else:
                print("Invalid option")
            
            # Wait for notifications
            await asyncio.sleep(1)

if __name__ == "__main__":
    asyncio.run(control_kettle())
```

## Notes and Limitations

1. **Command Length**: All commands to the MULTI_CHAR must be exactly 17 bytes.
2. **BLE Connection**: The kettle must be powered on (not completely off) for BLE to connect.
3. **Temperature Encoding**: Remember to multiply Celsius values by 2 when sending.
4. **Checksum**: Some commands may require a checksum byte (simple sum modulo 256).
5. **Timing**: Some commands may require delays between them to take effect.

## Further Exploration

Areas that may warrant further investigation:

1. The full meaning of bytes 10-14 in command packets
2. Additional text commands beyond the WiFi-related ones
3. Other control characteristics that might offer additional functionality
4. Hold temperature mode control
5. Firmware update mechanism

## References

This guide was compiled through reverse engineering and Wireshark packet capture analysis. Commands and structures were determined through systematic testing and observation of kettle behavior.

---

# Fellow Stagg EKG Pro BLE Protocol Findings

This document summarizes the findings from reverse engineering the Fellow Stagg EKG Pro kettle's Bluetooth Low Energy (BLE) protocol, with specific focus on standby mode control and command formats.

## BLE Architecture Overview

The Fellow Stagg EKG Pro kettle uses a Bluetooth Low Energy protocol with a custom service and several characteristics for different control functions.

### Service and Characteristic UUIDs

- **Primary Service UUID**: `7AEBF330-6CB1-46E4-B23B-7CC2262C605E`

- **Key Characteristics**:
  - Status Notifications: `2291C4B1-5D7F-4477-A88B-B266EDB97142` (Handle 0x0027)
  - Secondary Status: `2291C4B2-5D7F-4477-A88B-B266EDB97142` 
  - Tertiary Status: `2291C4B3-5D7F-4477-A88B-B266EDB97142`
  - Control Characteristic: `2291C4B4-5D7F-4477-A88B-B266EDB97142` 
  - Multi-purpose Control: `2291C4B5-5D7F-4477-A88B-B266EDB97142`
  - Command Channel: `2291C4B6-5D7F-4477-A88B-B266EDB97142`
  - Extra Control: `2291C4B7-5D7F-4477-A88B-B266EDB97142`
  - Read-only Info: `2291C4B8-5D7F-4477-A88B-B266EDB97142`
  - Read-only Info: `2291C4B9-5D7F-4477-A88B-B266EDB97142`

## Command Length Requirements

A critical discovery is that each characteristic has specific length requirements for commands:

- **MULTI_CHAR (2291C4B5)**: Requires exactly 17 bytes
- **CONTROL_CHAR (2291C4B4)**: Requires exactly 8 bytes
- **EXTRA_CONTROL (2291C4B7)**: Accepts commands of length 2-20 bytes
- **COMMAND_CHAR (2291C4B6)**: Used for text commands, variable length

Sending commands with incorrect lengths results in error: `Error Domain=CBATTErrorDomain Code=13 "The value's length is invalid."`

## Command Formats and Effects

### MULTI_CHAR Commands (17 bytes)

Format: `f7 [mode] 00 00 [temp] 00 [power] 00 00 00 00 [control bytes] 00 00 [checksum]`

- Used for temperature settings and primary kettle control
- **CRITICAL DISTINCTION**: Commands to this characteristic affect the clock display, NOT power/standby
- **WARNING**: The byte at index 6 (7th byte) with values 0xD0/0x80 is NOT a power/standby control as originally documented - it only affects the clock display
- **DO NOT USE** these commands for power/standby control as they only turn off the clock display, not the kettle
- Changing bytes 7-8 (index 6-7) with values like C0/E0 or C8/80 similarly only affects the clock display

This characteristic should not be used for standby control as it does not control power functions.

### CONTROL_CHAR Commands (8 bytes)

- Requires exactly 8 bytes
- Purpose of this characteristic is still being explored
- May be related to specialized control functions
- No specific command format has been confirmed yet

### EXTRA_CONTROL Commands (2-20 bytes)

- Accepts variable length commands (2-20 bytes)
- Highly flexible in terms of length
- Purpose and command structure still being investigated

### Text Commands via COMMAND_CHAR

Known text commands from documentation:
- `wifion` - Enable WiFi
- `wifioff` - Disable WiFi
- `wifista` - Enable WiFi station mode
- `httpfw` - Firmware-related function

Attempted text commands for standby control:
- `power:on` / `power:off`
- `standby` / `heat`
- `toggle`
- `start` / `stop`
- `boil` / `hold`

None of these text commands have been confirmed to control standby mode.

## Comparison with Older EKG+ Model

The older Fellow Stagg EKG+ model uses a completely different BLE protocol:

- **Service UUID**: `00001820-0000-1000-8000-00805f9b34fb`
- **Characteristic UUID**: `00002A80-0000-1000-8000-00805f9b34fb`
- **Command Structure**: 
  ```
  0xEF, 0xDD,       # Magic header
  0x0A,             # Command flag
  sequence_number,  # Incremented with each command
  command_type,     # 0=power, 1=temperature
  value,           # For power: 1=on, 0=off
  checksum1,        # (sequence + value) & 0xFF
  checksum2         # command_type
  ```
- **Power Control**: Simple command with command_type=0, value=1 (on) or 0 (off)

Attempts to use this command structure on the Pro model have not been successful.

## Known Issues and Limitations

1. **Clock Display vs. Power Control - Important Distinction**:
   - Commands to the MULTI_CHAR that were initially thought to control power (D0/80 values) actually only control the clock display
   - **DO NOT CONFUSE** clock display commands with power/standby commands - they are entirely different functions
   - The original protocol documentation incorrectly suggests that the D0/80 bytes control heating/standby, when they only affect the display
   - True standby control requires a different command and possibly a different characteristic entirely

2. **Standby Control Challenges**:
   - The exact command to toggle standby mode without affecting the display has not been identified
   - Multiple command structures and byte positions have been tested

3. **Command Length Validation**:
   - Strict length requirements for each characteristic
   - Commands must be padded to the exact required length

## Tested Commands and Results

Extensive testing with various command patterns has yielded the following results:

### CONTROL_CHAR (8-byte) Command Testing Results

The following command patterns were tested on the CONTROL_CHAR characteristic and did NOT affect standby state:

- Individual bit toggles in byte 0 (bits 0-7)
- Individual bit toggles in byte 1 (bits 0,7)
- Traditional power values in byte 0 (0xD0, 0x80)
- EKG+ style commands
- All bits ON pattern (0xFF in all bytes)
- First half ON pattern (0xFF in first 4 bytes)
- Second half ON pattern (0xFF in last 4 bytes)

Notably, one command pattern (EKG+ style on CONTROL_CHAR) resulted in a different error message: "The request is not supported," suggesting this command pattern may be recognized but rejected.

### EXTRA_CONTROL Command Testing Results

The following command patterns were tested on the EXTRA_CONTROL characteristic and did NOT affect standby state:

- Simple ON/OFF patterns (0x01/0x00 in first byte) with 2-byte length
- Simple ON/OFF patterns (0x01/0x00 in first byte) with 4-byte length
- Simple ON/OFF patterns (0x01/0x00 in first byte) with 8-byte length
- Traditional power values (0xD0, 0x80) with 8-byte length
- EKG+ style command with 8-byte length

## Recommendations for Further Exploration

Based on our extensive testing results, here are updated recommendations for further investigation:

1. **Try Other Characteristics**:
   - None of the tested patterns on CONTROL_CHAR or EXTRA_CONTROL affected standby
   - Consider exploring the COMMAND_CHAR with more complex text commands
   - Try investigating any undocumented characteristics or services

2. **Different Command Structure Approaches**:
   - The command structures we've tested (bit toggles, EKG+ style) don't control standby
   - Consider a completely different command structure or format
   - Try exploring command sequences (multiple commands in sequence) rather than single commands

3. **Deeper BLE Analysis**:
   - Use a dedicated BLE sniffer with the official app if possible
   - Focus on capturing the exact moment when standby is toggled
   - Look for commands or notifications that differ from what we've tested

4. **Hardware Analysis**:
   - Consider if there might be a hardware button that directly controls standby
   - Investigate if standby might be controlled by a different mechanism entirely

5. **Alternative Control Methods**:
   - Explore if there are other control methods beyond BLE (e.g., WiFi connectivity)
   - Investigate if power control might be part of a larger command sequence or state machine

## Next Steps

The most promising approach is to continue systematic testing of various command patterns on the CONTROL_CHAR (8-byte commands) and EXTRA_CONTROL characteristic. The script `kettle-standby-tester.py` provides a framework for this testing.

---

# Combined Fellow Stagg EKG Pro BLE Protocol Documentation

## Protocol Architecture Overview

After extensive testing and reverse engineering, we've determined that the Fellow Stagg EKG Pro kettle uses a sophisticated dual control system architecture:

1. **Display/State System**:
   - Controls what's shown on the kettle's display
   - Managed through standard BLE characteristic read/write operations
   - Changes don't directly affect physical heating elements

2. **Hardware Control System**:
   - Controls the physical heating elements and standby state
   - Managed through direct text commands
   - Commands that control physical elements cause BLE disconnection

This dual architecture explains why earlier attempts to toggle standby mode by modifying state bytes were unsuccessful - the physical standby state isn't represented in the BLE state data.

## BLE Service Architecture

### Primary Service
- **Service UUID**: `7AEBF330-6CB1-46E4-B23B-7CC2262C605E`

### Key Characteristics
| UUID | Name | Function | Length Requirements |
|------|------|----------|---------------------|
| `2291C4B1-5D7F-4477-A88B-B266EDB97142` | STATUS_CHAR | Status notifications | Variable |
| `2291C4B5-5D7F-4477-A88B-B266EDB97142` | MULTI_CHAR | Multi-purpose state data | Exactly 17 bytes |
| `2291C4B4-5D7F-4477-A88B-B266EDB97142` | CONTROL_CHAR | Control authorization | Exactly 8 bytes |
| `2291C4B6-5D7F-4477-A88B-B266EDB97142` | COMMAND_CHAR | Text commands | Variable |

## State Data Format

The MULTI_CHAR uses a 17-byte message format with this structure:

```
f7 17 00 80 50 80 c0 80 00 00 28 00 01 00 00 00 00
```

| Byte Position | Function | Values |
|---------------|----------|--------|
| 0-1 | Header/Mode | `f7 17` (standard), `ff 17` (special commands) |
| 3 | Schedule Flag | `00` (normal), `80` (schedule mode) |
| 4 | Temperature | Temperature × 2 (e.g., 40°C = `50` hex = 80 decimal ÷ 2) |
| 5-7 | Display State | Various values control display appearance |
| 8-9 | Schedule Time | Encodes schedule time parameters |
| 10 | Sequence Number | Increments with each state change |
| 12 | Execution Flag | `01` (standard update), `02` (execute change) |
| 16 | Checksum/Control | Varies with packet contents |

## Standby Control

### Turning OFF (Heating → Standby)
Only text commands through COMMAND_CHAR can trigger the transition from heating to standby:

- `reset`
- `reset on`
- `reset off`
- `factory`

All these commands cause BLE disconnection as a side effect, requiring reconnection afterward.

### Turning ON (Standby → Heating)
There are only two ways to turn the kettle ON:

1. **Physical Button**: Press the physical knob button on the kettle
2. **Schedule Activation**: When a scheduled event triggers, the kettle can turn itself on

Notably, there is **no direct BLE command** to turn the kettle ON without using the physical button or a schedule.

## Temperature Control

Temperature can be set by modifying byte 4 in the MULTI_CHAR state data:

```python
# Read current state
state = await client.read_gatt_char(MULTI_CHAR)

# Modify temperature (byte 4 = temperature × 2)
new_state = bytearray(state)
new_state[4] = int(desired_temp * 2)  # Temperature in Celsius

# Write back modified state
await client.write_gatt_char(MULTI_CHAR, new_state)
```

Valid temperature range: 40°C to 100°C (or 104°F to 212°F).

## Schedule Control

Schedules can be set using a specific byte pattern:

```
ff 17 00 80 xx 80 6f 80 0b 00 yy 00 01 80 00 00 zz
```

Where:
- `xx` = Temperature × 2
- `yy` = Sequence number (incremented)
- `zz` = Checksum

Byte 3 = `80` indicates schedule mode, and bytes 8-10 appear to encode time information.

## Display Control Execution Pathway

To execute physical display changes (like toggling clock format), use:

1. **Authorization Step**:
   ```python
   # Send Control authorization
   await client.write_gatt_char(CONTROL_CHAR, bytes([0x02, 0, 0, 0, 0, 0, 0, 0]))
   ```

2. **Command with Execution Flag**:
   ```python
   # Read current state
   state = await client.read_gatt_char(MULTI_CHAR)
   
   # Set execution flag (byte 12 = 0x02)
   modified_state = bytearray(state)
   modified_state[12] = 0x02
   
   # Send command with execution flag
   await client.write_gatt_char(MULTI_CHAR, modified_state)
   ```

This two-step process is required for display changes to take effect.

## Implementation Recommendations

Based on our findings, here's the recommended approach for controlling the kettle:

### Turning OFF
```python
# Send reset command (will cause disconnection)
await client.write_gatt_char(COMMAND_CHAR, b"reset")

# Handle reconnection
# Wait for disconnection then reconnect
```

### Setting Temperature
```python
# Read current state
state = await client.read_gatt_char(MULTI_CHAR)

# Modify temperature (byte 4 = temp × 2)
new_state = bytearray(state)
new_state[4] = int(desired_temp * 2)

# Write modified state
await client.write_gatt_char(MULTI_CHAR, new_state)
```

### Setting Schedule
```python
# Read current state
state = await client.read_gatt_char(MULTI_CHAR)

# Create schedule command
schedule_cmd = bytearray(state)
schedule_cmd[0] = 0xff  # Special prefix
schedule_cmd[3] = 0x80  # Schedule mode
schedule_cmd[4] = int(temp * 2)  # Temperature
# Set time bytes (8-10)
# Increment sequence counter (byte 10)

# Send schedule command
await client.write_gatt_char(MULTI_CHAR, schedule_cmd)
```

## Key Protocol Discoveries

1. **State Data Identical Between Modes**:
   ```
   STANDBY: f7 17 00 00 50 80 c0 80 00 00 26 09 01 1e 00 00 09
   HEATING: f7 17 00 00 50 80 c0 80 00 00 26 09 01 1e 00 00 09
   ```
   The kettle's BLE state data doesn't distinguish between standby and heating modes.

2. **Schedule Mode Pattern**:
   ```
   Normal:   f7 17 00 00 50 80 80 00 00 00 25 01 01 00 00 00 6b
   Schedule: f7 17 00 80 6a 80 c0 80 00 00 28 01 01 00 00 00 73
   ```
   Key differences in bytes 3, 4, and 13 indicate schedule mode.

3. **Clock Format Control**:
   - Byte 11 = `01`: 24-hour format
   - Byte 11 = `00`: 12-hour format

4. **Firmware Version Information** (from EXTRA_CHAR):
   ```
   31 2e 31 2e 37 35 53 53 50 20 43
   ```
   Decodes to: `1.1.75SSP C` - Firmware version

## Comparison to Previous Model

The EKG Pro uses a completely different protocol design compared to the older model:

1. The older model used binary command patterns for on/off:
   - On: `efdd0a0000010100`
   - Off: `efdd0a0400000400`

2. The EKG Pro uses:
   - Text commands for physical control
   - State bytes for display control
   - A two-step authorization + execution pathway

Testing confirmed that the older model's commands have no effect on the EKG Pro.

## Safety Considerations

The kettle's protocol design prioritizes safety:
- Remote activation is intentionally limited
- Disconnection during standby toggle is likely a safety feature
- Only scheduled events can activate heating without physical interaction

This dual control architecture with restricted activation capabilities is common in appliances with heating elements to prevent accidents or unintended activation.

### 10. Physical Button Behavior

The physical knob press that toggles standby mode:
- Does not consistently produce BLE notifications
- Is not directly observable through standard BLE monitoring
- May involve a different command pattern than the schedule activation### 9. Standby Control Discovery!

We've captured a critical notification when the kettle's schedule activated:
```
NOTIFICATION [MULTI_CHAR]: f7 17 00 64 6a 80 c0 80 00 00 28 01 01 00 00 00 73
```

Comparing to previous state:
```
Previous state: f7 17 00 64 50 80 80 00 00 00 25 01 01 00 00 00 6b
Schedule activation: f7 17 00 64 6a 80 c0 80 00 00 28 01 01 00 00 00 73
```

The key differences reveal:
1. **Byte 4**: `50` → `6a` (Temperature change from 40°C to 53°C)
2. **Byte 6**: `80` → `c0` (**This appears to be the heating control**)
3. **Byte 7**: `00` → `80` (Possibly part of the control sequence)
4. **Byte 10**: `25` → `28` (Sequence counter increment)

This is a breakthrough discovery! The standby control appears to be in byte 6, but with a different value than we expected. Rather than toggling between `80` and `D0`, it appears to toggle between `80` and `C0`.# Fellow Stagg EKG Pro BLE Protocol Investigation

## Investigation Overview

We've been working to reverse engineer the Bluetooth Low Energy (BLE) protocol of the Fellow Stagg EKG Pro kettle, with a specific focus on controlling the standby functionality. This document summarizes our findings, observations, and current understanding of the protocol.

## Kettle BLE Architecture

### Primary Service
- **Service UUID**: `7AEBF330-6CB1-46E4-B23B-7CC2262C605E`

### Key Characteristics
| UUID | Name (Assigned) | Function | Length Requirements |
|------|-----------------|----------|---------------------|
| `2291C4B1-5D7F-4477-A88B-B266EDB97142` | STATUS_CHAR | Status notifications | Variable |
| `2291C4B2-5D7F-4477-A88B-B266EDB97142` | STATUS2_CHAR | Secondary status | Variable |
| `2291C4B3-5D7F-4477-A88B-B266EDB97142` | STATUS3_CHAR | Tertiary status | Variable |
| `2291C4B4-5D7F-4477-A88B-B266EDB97142` | CONTROL_CHAR | Control characteristic | Exactly 8 bytes |
| `2291C4B5-5D7F-4477-A88B-B266EDB97142` | MULTI_CHAR | Multi-purpose | Exactly 17 bytes |
| `2291C4B6-5D7F-4477-A88B-B266EDB97142` | COMMAND_CHAR | Command channel | Variable (text) |
| `2291C4B7-5D7F-4477-A88B-B266EDB97142` | EXTRA_CHAR | Extra control | 2-20 bytes |
| `2291C4B8-5D7F-4477-A88B-B266EDB97142` | INFO1_CHAR | Read-only info | Variable |
| `2291C4B9-5D7F-4477-A88B-B266EDB97142` | INFO2_CHAR | Read-only info | Variable |

## Key Discoveries

### 1. MULTI_CHAR Message Format

We've identified that the MULTI_CHAR uses a 17-byte message format:

```
f7 17 00 64 50 80 80 00 00 00 1f 01 01 00 00 00 5f
```

Where:
- Bytes 0-1 (`f7 17`): Temperature mode indicator (Celsius)
- Byte 4 (`50`): Temperature setting (80 decimal = 40°C when divided by 2)
- Bytes 5-6 (`80 80`): Appear related to display state (standby/heating)
- Byte 10 (`1f`): Incrementing control byte that changes with each toggle
- Byte 11 (`01`): **Clock format control** (01 = 24-hour format, 00 = 12-hour format)
- Byte 12 (`01`): Additional control flag
- Byte 16 (`5f`): Likely a checksum

### 2. Display Control vs. Heating Control

A key finding is that we initially confused display indicators with actual heating control:
- Byte 6 value `D0` indicates HEATING on the display
- Byte 6 value `80` indicates STANDBY on the display
- However, changing this value only affects the display, not the actual heating function

### 3. Standby Toggle Pattern

When physically pressing the kettle's knob to toggle standby mode, we observed:
- Byte 10 increments with each toggle: `1a` → `1b` → `1c` → `1d` → etc.
- This byte appears to function as a counter or command sequence number
- The kettle accepts and stores our modified commands with incremented values
- However, incrementing this byte alone doesn't trigger the actual standby toggle

### 4. Clock Format Control

We discovered that byte 11 controls the clock display format:
- Value `01`: 24-hour format (e.g., 13:38)
- Value `00`: 12-hour format (e.g., 1:38 AM)
- Changing this value immediately affects the clock display
- The kettle remembers this setting when reading back the state

### 5. Text Commands

We tested various text commands on COMMAND_CHAR including:
- `power`, `power:on`, `power:off`
- `standby`, `standby:toggle`
- `heat:on`, `heat:off`
- `toggle`, `mode`, `status`

None of these text commands had any observable effect on the kettle's standby mode.

### 7. EXTRA_CHAR and Version Information

We've captured data from the EXTRA_CHAR which reveals:
```
31 2e 31 2e 37 35 53 53 50 20 43 00 f6 c2 00 40 ff ff ff ff f0 3e 08 40 23 0c 06 00 10 26 09 40
```

When decoded as ASCII for the first part, this contains:
- `1.1.75SSP C` - Likely firmware version information
- Followed by binary data - Possibly calibration or configuration data

This characteristic may provide important information about the kettle's firmware version and capabilities.

## Execution Pathway Discovery

We've made a critical breakthrough in understanding how to make the kettle execute physical changes:

```
The combination of:
1. CONTROL_CHAR with value 0x02 (first byte)
2. MULTI_CHAR with byte 12 set to 0x02 (execution flag)
```

This command pattern successfully changes the clock display from digital to analog. This is our first confirmed instance of being able to trigger a physical change in the kettle (beyond just temperature setting).

The execution pathway appears to involve:

1. **"Authorization" Step**: CONTROL_CHAR command with specific first byte values grants permission for certain operations:
   - `0x02` appears to authorize display-related changes

2. **"Execution Flag"**: Byte 12 in MULTI_CHAR seems to act as an execution trigger:
   - `0x01`: Standard state update (no physical effect)
   - `0x02`: Execute the requested change physically

3. **Command Sequence**: The correct sequence appears to be:
   - Send WiFi initialization command
   - Send CONTROL_CHAR command with the appropriate "authorization" value
   - Send MULTI_CHAR command with byte 12 set to the execution flag

This discovery provides a clear path to attempt control of other kettle functions, particularly heating control, using the same execution pathway.

## Further Testing Ideas

Based on our latest findings from the BLE packet capture, here are suggested approaches for achieving direct control of the kettle:

1. **Schedule Command Emulation**:
   - Try to emulate the schedule command format with byte 6 set to `8C` and byte 7 set to `08`
   - This might allow direct control by mimicking the schedule command pathway

2. **CONTROL_CHAR + Schedule Format**:
   - Combine CONTROL_CHAR commands with MULTI_CHAR commands using the schedule format
   - Use the pattern observed in bytes 10-12 (`01 01 01`) as seen in the captured packet

3. **Modified Schedule Testing**:
   - Set up a schedule on the kettle, capture the BLE traffic
   - Then try to modify just the temperature or time parameters while preserving the schedule format

4. **Schedule-like Command Construction**:
   - Create commands that use the schedule byte pattern (`8C` in byte 6, `08` in byte 7)
   - But change only the temperature and heating state, keeping other values intact

5. **Schedule Mode Activation**:
   - Explore if there's a command to activate schedule mode, which might enable a different command path
   - Look for commands that might toggle the kettle between direct and schedule control modes

6. **Direct Schedule Programming via BLE**:
   - Rather than trying to control heating directly, explore if schedules themselves can be programmed via BLE
   - This would provide indirect but effective control

7. **Time/Clock Correlation**:
   - Investigate if there's a relationship between the kettle's internal clock and command authorization
   - The schedule specifies a time (1:16 AM) - this might be relevant to authentication

These approaches focus on the newly discovered schedule command pathway, which seems to have the necessary permissions to control heating functions.

## Script Development Progress

We've developed several scripts to investigate the protocol:

1. **kettle-standby-tester.py**: Initial testing script for systematic exploration of command patterns.

2. **kettle-focused-tester.py**: Focused testing of display control and bit patterns.

3. **kettle-specialized-tester.py**: Testing of initialization sequences and protocol mechanisms.

4. **kettle-listener.py**: Passive monitoring script with user action logging.

5. **kettle-standby-toggler.py**: Testing the byte 10 increment theory.

6. **kettle-advanced-toggler.py**: Comprehensive testing of multiple toggle approaches.

## Progress and Successes

Our investigation has yielded significant discoveries about the Fellow Stagg EKG Pro kettle's BLE protocol:

1. **Successfully Implemented Functions**:
   - Temperature setting and reading (fully functional)
   - Clock display format toggling (digital/analog)
   - Reading the current state of all kettle parameters

2. **Key Protocol Insights**:
   - Identified command formats for different kettle states
   - Discovered the execution pathway that makes changes take effect
   - Mapped the relationship between display state and actual function

3. **Execution System Discovery**:
   - CONTROL_CHAR with value `0x02` authorizes display changes
   - Byte 12 with value `0x02` serves as an execution flag
   - This combination successfully triggered physical changes

## Conclusions and Implications

The Fellow Stagg EKG Pro kettle implements a sophisticated BLE protocol with a multi-step command execution system:

1. **Authorization Step**: 
   - CONTROL_CHAR commands with specific values grant permission for certain operations
   - Different values likely authorize different types of operations

2. **Command Step**:
   - MULTI_CHAR commands contain the actual state changes
   - Byte 12 serves as an execution flag that determines if changes are applied

3. **Safety Design**:
   - This two-step process prevents accidental activation of heating elements
   - It explains why our earlier commands were accepted but not executed

# Fellow Stagg EKG Pro BLE Protocol Investigation

## New Standby Control Discoveries (March 15, 2025)

### 11. Multiple Direct Standby Control Commands

We've discovered several text commands that can control the kettle's standby mode:

1. **"factory" command**: 
   - Causes the kettle to enter standby mode
   - Triggers a BLE disconnection as side effect
   - Appears to be a type of "soft reset" function

2. **"reset" command and variants**:
   - "reset", "reset on", "reset off" all cause disconnection
   - Similar behavior to the "factory" command
   - Likely toggles the kettle's standby state

These commands bypass the normal state data representation, acting directly on the kettle's hardware control systems rather than changing the state bytes.

### 12. Dual Control System Architecture

Our testing has revealed that the kettle operates with two separate control systems:

1. **Display/State System**:
   - Represented in the MULTI_CHAR state bytes
   - Changes to these bytes only affect what the display shows
   - State bytes are identical between standby and heating modes
   - Controlled through the standard BLE characteristic read/write flow

2. **Hardware Control System**:
   - Controlled by direct text commands ("factory", "reset") 
   - These commands don't change state bytes but affect physical hardware
   - Cause disconnection when executed
   - Not reflected in the BLE state data

This dual architecture explains why our earlier attempts to toggle standby mode by modifying state bytes were unsuccessful - the physical standby state isn't represented in the state data.

### 13. State Data Identical Between Modes

We confirmed that the kettle's state data is completely identical between standby and heating modes:

```
STANDBY: f7 17 00 00 50 80 c0 80 00 00 26 09 01 1e 00 00 09
HEATING: f7 17 00 00 50 80 c0 80 00 00 26 09 01 1e 00 00 09
```

This explains why our attempts to toggle standby by modifying bytes 6 and 7 (which we thought indicated heating state) were unsuccessful. The kettle doesn't represent its physical state in the BLE data.

### 14. Schedule Mode Byte Pattern

We identified a different byte pattern in schedule mode from the Wireshark capture:

```
Normal mode: f7 17 00 00 50 80 80 00 00 00 25 01 01 00 00 00 6b
Schedule mode: f7 17 00 80 6a 80 c0 80 00 00 28 01 01 00 00 00 73
```

Key differences:
- Byte 3: `00` → `80` (Schedule mode flag)
- Byte 4: `50` → `6a` (Temperature change from 40°C to 53°C)
- Byte 13: `00` → `1e` (Different in normal vs. schedule mode)

However, testing these byte combinations through the automated schedule byte tester revealed that:
- In standby mode: Only the time display changed
- In heating mode: No changes were observed

This confirms that these bytes don't directly control the standby function but might be related to schedule display or other secondary functions.

### 15. Command Channel Testing Findings

Systematic testing of text commands through the command channel revealed:

1. Several commands changed the sequence number but didn't affect standby state:
   - "knob"
   - "temperature:off"
   - "off off"

2. Commands that caused disconnection (similar to "factory"):
   - "reset"
   - "reset on"
   - "reset off"

The disconnection behavior appears to be strongly associated with commands that control the kettle's physical standby state.

## Previous Protocol Investigation Findings

### 1. MULTI_CHAR Message Format

We've identified that the MULTI_CHAR uses a 17-byte message format:

```
f7 17 00 64 50 80 80 00 00 00 1f 01 01 00 00 00 5f
```

Where:
- Bytes 0-1 (`f7 17`): Temperature mode indicator (Celsius)
- Byte 4 (`50`): Temperature setting (80 decimal = 40°C when divided by 2)
- Bytes 5-6 (`80 80`): Appear related to display state (standby/heating)
- Byte 10 (`1f`): Incrementing control byte that changes with each toggle
- Byte 11 (`01`): **Clock format control** (01 = 24-hour format, 00 = 12-hour format)
- Byte 12 (`01`): Additional control flag
- Byte 16 (`5f`): Likely a checksum

### 2. Display Control vs. Actual Control

A key finding is that the state bytes only control the kettle's display, not its actual function:
- Byte 6 value `D0` indicates HEATING on the display
- Byte 6 value `80` indicates STANDBY on the display
- However, changing this value only affects the display, not the actual heating function

### 3. Standby Toggle Pattern

When physically pressing the kettle's knob to toggle standby mode, we observed:
- Byte 10 increments with each toggle: `1a` → `1b` → `1c` → `1d` → etc.
- This byte appears to function as a counter or command sequence number
- The kettle accepts and stores our modified commands with incremented values
- However, incrementing this byte alone doesn't trigger the actual standby toggle

### 4. Clock Format Control

We discovered that byte 11 controls the clock display format:
- Value `01`: 24-hour format (e.g., 13:38)
- Value `00`: 12-hour format (e.g., 1:38 AM)
- Changing this value immediately affects the clock display
- The kettle remembers this setting when reading back the state

### 5. Text Commands

We tested various text commands on COMMAND_CHAR including:
- `power`, `power:on`, `power:off`
- `standby`, `standby:on`, `standby:off`
- `heat:on`, `heat:off`
- `toggle`, `mode`, `status`
- `factory` - discovered to put the kettle in standby mode
- `reset` - also discovered to possibly control standby mode

### 7. EXTRA_CHAR and Version Information

We've captured data from the EXTRA_CHAR which reveals:
```
31 2e 31 2e 37 35 53 53 50 20 43 00 f6 c2 00 40 ff ff ff ff f0 3e 08 40 23 0c 06 00 10 26 09 40
```

When decoded as ASCII for the first part, this contains:
- `1.1.75SSP C` - Likely firmware version information
- Followed by binary data - Possibly calibration or configuration data

This characteristic may provide important information about the kettle's firmware version and capabilities.

## Execution Pathway Discovery

We've made a critical breakthrough in understanding how to make the kettle execute physical changes:

```
The combination of:
1. CONTROL_CHAR with value 0x02 (first byte)
2. MULTI_CHAR with byte 12 set to 0x02 (execution flag)
```

This command pattern successfully changes the clock display from digital to analog. This is our first confirmed instance of being able to trigger a physical change in the kettle (beyond just temperature setting).

The execution pathway appears to involve:

1. **"Authorization" Step**: CONTROL_CHAR command with specific first byte values grants permission for certain operations:
   - `0x02` appears to authorize display-related changes

2. **"Execution Flag"**: Byte 12 in MULTI_CHAR seems to act as an execution trigger:
   - `0x01`: Standard state update (no physical effect)
   - `0x02`: Execute the requested change physically

3. **Command Sequence**: The correct sequence appears to be:
   - Send WiFi initialization command
   - Send CONTROL_CHAR command with the appropriate "authorization" value
   - Send MULTI_CHAR command with byte 12 set to the execution flag

This discovery provides a clear path to attempt control of other kettle functions, particularly heating control, using the same execution pathway.

## Factory Reset Command Pattern

During factory reset, we observed a special pattern with `cd` values in bytes 4 and 6:

```
f7 15 00 00 cd 00 cd 00 00 00 19 01 02 1e 00 00 34
```

This might represent a special control sequence used during reset/standby operations.

## Physical Button Behavior

The physical knob press that toggles standby mode:
- Does not consistently produce BLE notifications
- Is not directly observable through standard BLE monitoring
- Appears to use a different command pattern than display changes
- Does not disconnect the BLE connection, unlike the "factory" command
- Doesn't change any of the state bytes we can observe

## Standby Control Approaches

Our investigation has revealed multiple approaches to controlling standby:

1. **Display Control**: We can change what the kettle's display shows (using bytes 6-7) but this doesn't affect actual heating state.

2. **Direct Commands**: We can control standby using the "factory" and "reset" commands, but these cause a disconnect.

3. **Physical Button**: The cleanest toggle method, but we haven't discovered how to replicate this over BLE yet.

## Further Investigation Needed

Based on our latest findings, further investigation should focus on:

1. **Clean Toggle Command**: Discovering a text command that toggles standby without disconnecting, if one exists

2. **Traffic Capture**: Capturing the exact BLE traffic when the physical button is pressed to understand how it toggles standby without changing state bytes

3. **Reconnection Handling**: Creating a script that can automatically reconnect after using the "factory" or "reset" commands to provide a seamless standby control experience

# Fellow Stagg EKG Pro BLE Protocol Guide

This document provides a comprehensive guide to the reverse-engineered Bluetooth Low Energy (BLE) protocol for the Fellow Stagg EKG Pro kettle.

## Connection Establishment

### Service and Characteristic UUIDs

- **Primary Service UUID**: `7AEBF330-6CB1-46E4-B23B-7CC2262C605E`

- **Key Characteristics**:
  - Status Notifications: `2291C4B1-5D7F-4477-A88B-B266EDB97142` (Handle 0x0027)
  - Multi-purpose Control: `2291C4B5-5D7F-4477-A88B-B266EDB97142`
  - Command Channel: `2291C4B6-5D7F-4477-A88B-B266EDB97142`

- **Additional Characteristics**:
  - `2291C4B2-5D7F-4477-A88B-B266EDB97142` (Additional notifications)
  - `2291C4B3-5D7F-4477-A88B-B266EDB97142` (Additional notifications)
  - `2291C4B4-5D7F-4477-A88B-B266EDB97142` (Control characteristic)
  - `2291C4B7-5D7F-4477-A88B-B266EDB97142` (Additional control)
  - `2291C4B8-5D7F-4477-A88B-B266EDB97142` (Read-only)
  - `2291C4B9-5D7F-4477-A88B-B266EDB97142` (Read-only)

### Connection Steps

1. Scan for BLE devices with "EKG" in the name
2. Connect to the device
3. Enable notifications on the status characteristic
4. Send WiFi initialization commands (optional)
5. Begin sending control commands

### Example Connection Code (Python with Bleak)

```python
import asyncio
from bleak import BleakClient, BleakScanner

# Characteristic UUIDs
STATUS_CHAR = "2291C4B1-5D7F-4477-A88B-B266EDB97142"
MULTI_CHAR = "2291C4B5-5D7F-4477-A88B-B266EDB97142"

def notification_handler(sender, data):
    """Handle incoming notifications from the kettle."""
    print(f"Notification: {data.hex()}")

async def connect_to_kettle():
    # Scan for kettle
    devices = await BleakScanner.discover()
    kettle = next((d for d in devices if d.name and "EKG" in d.name), None)
    
    if not kettle:
        print("No kettle found!")
        return
    
    print(f"Connecting to {kettle.name} at {kettle.address}...")
    async with BleakClient(kettle.address) as client:
        print("Connected!")
        
        # Enable notifications
        await client.start_notify(STATUS_CHAR, notification_handler)
        
        # Now ready to send commands
        # ...
```

## Command Structure

### Command Format

Commands sent to the kettle must follow a specific format. All commands sent to the MULTI_CHAR must be exactly 17 bytes (34 hex characters) in length.

#### Temperature Setting Commands

Format: `f7 [mode] 00 00 [temp] 00 [power] 00 00 00 00 [control bytes] 00 00 [checksum]`

Where:
- `mode`: `17` for Celsius, `15` for Fahrenheit
- `temp`: Temperature value (2× Celsius, 1× Fahrenheit)
- `power`: `D0` for heating, `80` for standby
- `control bytes`: Varies based on command type
- `checksum`: Simple sum of all bytes modulo 256 (optional)

#### Observed Working Command Patterns

1. **Celsius Base Command**: `f717006400000000000000000000000000`
2. **Fahrenheit Base Command**: `f715006400000000000000000000000000`
3. **Celsius (from packet capture)**: `f7170000c00c800000001304011e00000b`
4. **Fahrenheit (from packet capture)**: `f7150000c00cd000000012040a1e0000`

### Text Commands

Text commands can be sent to the COMMAND_CHAR. Known commands include:

- `wifion` - Enable WiFi
- `wifista` - Enable WiFi station mode
- `httpfw` - Firmware-related function

## Temperature Control

### Setting Temperature

Temperature values are encoded differently between Celsius and Fahrenheit:

- **Celsius**: Temperature value is 2× the desired temperature
  - Example: 85°C is encoded as `0xAA` (170 decimal)
- **Fahrenheit**: Temperature value is the same as the desired temperature
  - Example: 185°F is encoded as `0xB9` (185 decimal)

### Temperature Mode Selection

- **Celsius Mode**: Use prefix `f7170000`
- **Fahrenheit Mode**: Use prefix `f7150000`

### Example Temperature Commands

1. **Set to 85°C**:
   ```python
   temp = 85
   temp_raw = int(temp * 2)  # 170
   cmd = bytearray.fromhex("f717006400000000000000000000000000")
   cmd[4] = temp_raw
   # Add checksum if needed
   checksum = sum(cmd) % 256
   cmd.append(checksum)
   await client.write_gatt_char(MULTI_CHAR, cmd)
   ```

2. **Set to 195°F**:
   ```python
   temp = 195
   cmd = bytearray.fromhex("f715006400000000000000000000000000")
   cmd[4] = temp
   # Add checksum if needed
   checksum = sum(cmd) % 256
   cmd.append(checksum)
   await client.write_gatt_char(MULTI_CHAR, cmd)
   ```

## Power/Standby Control

The kettle appears to have two different power states that can be controlled via BLE:

1. **Complete Power OFF**: Disconnects BLE connection
2. **Standby Mode**: Like pressing the physical knob, keeps BLE connected

### Standby Mode Control

To toggle between heating and standby (like pressing the physical knob), byte 7 (index 6) in the command appears to be the control byte:

- `0xD0`: Heating mode
- `0x80`: Standby mode

### Example Standby Toggle Command

```python
# Set to standby mode
cmd = bytearray.fromhex("f717006400000000000000000000000000")
cmd[6] = 0x80  # Standby byte
await client.write_gatt_char(MULTI_CHAR, cmd)

# Set to heating mode
cmd = bytearray.fromhex("f717006400000000000000000000000000")
cmd[6] = 0xD0  # Heating byte
await client.write_gatt_char(MULTI_CHAR, cmd)
```

## Status Notification Interpretation

When notifications are enabled, the kettle sends status updates that can be interpreted:

- First 4 bytes: Mode (`f7170000` = Celsius, `f7150000` = Fahrenheit)
- Byte 5 (index 4): Current temperature (2× for Celsius)
- Byte 7 (index 6): Possibly power state (`0xD0` = Heating, `0x80` = Standby)

Example notification parser:

```python
def parse_notification(data):
    if len(data) >= 4:
        mode_bytes = data[0:4]
        if mode_bytes == bytes.fromhex("F7150000"):
            mode = "Fahrenheit"
            if len(data) >= 5:
                temp = data[4]
                print(f"Temperature: {temp}°F")
        elif mode_bytes == bytes.fromhex("F7170000"):
            mode = "Celsius"
            if len(data) >= 5:
                temp = data[4] / 2
                print(f"Temperature: {temp}°C")
        
        if len(data) >= 7:
            if data[6] == 0xD0:
                state = "HEATING"
            elif data[6] == 0x80:
                state = "STANDBY"
            else:
                state = f"UNKNOWN (0x{data[6]:02X})"
            print(f"Kettle state: {state}")
```

## Putting It All Together

### Complete Kettle Control Example

```python
import asyncio
from bleak import BleakClient, BleakScanner

# Characteristic UUIDs
STATUS_CHAR = "2291C4B1-5D7F-4477-A88B-B266EDB97142"
MULTI_CHAR = "2291C4B5-5D7F-4477-A88B-B266EDB97142"
COMMAND_CHAR = "2291C4B6-5D7F-4477-A88B-B266EDB97142"

# Command templates
CELSIUS_TEMPLATE = "f717006400000000000000000000000000"
FAHRENHEIT_TEMPLATE = "f715006400000000000000000000000000"

def notification_handler(sender, data):
    """Handle incoming notifications from the kettle."""
    print(f"Notification: {data.hex()}")
    
    # Parse kettle state
    if len(data) >= 4:
        # Parse temperature mode
        if data[0:2] == bytes.fromhex("F717"):
            print("Mode: Celsius")
            if len(data) >= 5:
                print(f"Current temperature: {data[4]/2}°C")
        elif data[0:2] == bytes.fromhex("F715"):
            print("Mode: Fahrenheit")
            if len(data) >= 5:
                print(f"Current temperature: {data[4]}°F")
        
        # Parse power state (if present)
        if len(data) >= 7:
            if data[6] == 0xD0:
                print("Kettle is HEATING")
            elif data[6] == 0x80:
                print("Kettle is in STANDBY")

async def set_temperature(client, temp, use_fahrenheit=False):
    """Set kettle temperature."""
    if use_fahrenheit:
        template = FAHRENHEIT_TEMPLATE
        temp_raw = temp  # Direct value for Fahrenheit
    else:
        template = CELSIUS_TEMPLATE
        temp_raw = int(temp * 2)  # 2× for Celsius
    
    # Create command
    cmd = bytearray.fromhex(template)
    cmd[4] = temp_raw  # Set temperature byte
    cmd[6] = 0xD0      # Set to heating mode
    
    # Add checksum (optional)
    checksum = sum(cmd) % 256
    cmd.append(checksum)
    
    # Send command
    print(f"Setting temperature to {temp}{'°F' if use_fahrenheit else '°C'}")
    await client.write_gatt_char(MULTI_CHAR, cmd)
    print("Temperature set!")

async def toggle_standby(client, standby=True):
    """Toggle between heating and standby mode."""
    # Create command
    cmd = bytearray.fromhex(CELSIUS_TEMPLATE)
    cmd[6] = 0x80 if standby else 0xD0  # 0x80 for standby, 0xD0 for heating
    
    # Add checksum (optional)
    checksum = sum(cmd) % 256
    cmd.append(checksum)
    
    # Send command
    print(f"Setting kettle to {'STANDBY' if standby else 'HEATING'} mode")
    await client.write_gatt_char(MULTI_CHAR, cmd)
    print("Mode changed!")

async def control_kettle():
    """Connect to kettle and provide control interface."""
    # Scan for kettle
    devices = await BleakScanner.discover()
    kettle = next((d for d in devices if d.name and "EKG" in d.name), None)
    
    if not kettle:
        print("No kettle found!")
        return
    
    print(f"Connecting to {kettle.name} at {kettle.address}...")
    async with BleakClient(kettle.address) as client:
        print("Connected!")
        
        # Enable notifications
        await client.start_notify(STATUS_CHAR, notification_handler)
        print("Notifications enabled")
        
        # Main control loop
        while True:
            print("\n=== KETTLE CONTROL ===")
            print("1. Set temperature (Celsius)")
            print("2. Set temperature (Fahrenheit)")
            print("3. Enter heating mode")
            print("4. Enter standby mode")
            print("5. Exit")
            
            choice = input("Select option: ")
            
            if choice == "1":
                temp = float(input("Enter temperature (°C): "))
                await set_temperature(client, temp, False)
            elif choice == "2":
                temp = float(input("Enter temperature (°F): "))
                await set_temperature(client, temp, True)
            elif choice == "3":
                await toggle_standby(client, False)  # Heating mode
            elif choice == "4":
                await toggle_standby(client, True)   # Standby mode
            elif choice == "5":
                print("Exiting...")
                break
            else:
                print("Invalid option")
            
            # Wait for notifications
            await asyncio.sleep(1)

if __name__ == "__main__":
    asyncio.run(control_kettle())
```

## Notes and Limitations

1. **Command Length**: All commands to the MULTI_CHAR must be exactly 17 bytes.
2. **BLE Connection**: The kettle must be powered on (not completely off) for BLE to connect.
3. **Temperature Encoding**: Remember to multiply Celsius values by 2 when sending.
4. **Checksum**: Some commands may require a checksum byte (simple sum modulo 256).
5. **Timing**: Some commands may require delays between them to take effect.

## Further Exploration

Areas that may warrant further investigation:

1. The full meaning of bytes 10-14 in command packets
2. Additional text commands beyond the WiFi-related ones
3. Other control characteristics that might offer additional functionality
4. Hold temperature mode control
5. Firmware update mechanism

## References

This guide was compiled through reverse engineering and Wireshark packet capture analysis. Commands and structures were determined through systematic testing and observation of kettle behavior.

# Fellow Stagg EKG Pro BLE Protocol Findings

This document summarizes the findings from reverse engineering the Fellow Stagg EKG Pro kettle's Bluetooth Low Energy (BLE) protocol, with specific focus on standby mode control and command formats.

## BLE Architecture Overview

The Fellow Stagg EKG Pro kettle uses a Bluetooth Low Energy protocol with a custom service and several characteristics for different control functions.

### Service and Characteristic UUIDs

- **Primary Service UUID**: `7AEBF330-6CB1-46E4-B23B-7CC2262C605E`

- **Key Characteristics**:
  - Status Notifications: `2291C4B1-5D7F-4477-A88B-B266EDB97142` (Handle 0x0027)
  - Secondary Status: `2291C4B2-5D7F-4477-A88B-B266EDB97142` 
  - Tertiary Status: `2291C4B3-5D7F-4477-A88B-B266EDB97142`
  - Control Characteristic: `2291C4B4-5D7F-4477-A88B-B266EDB97142` 
  - Multi-purpose Control: `2291C4B5-5D7F-4477-A88B-B266EDB97142`
  - Command Channel: `2291C4B6-5D7F-4477-A88B-B266EDB97142`
  - Extra Control: `2291C4B7-5D7F-4477-A88B-B266EDB97142`
  - Read-only Info: `2291C4B8-5D7F-4477-A88B-B266EDB97142`
  - Read-only Info: `2291C4B9-5D7F-4477-A88B-B266EDB97142`

## Command Length Requirements

A critical discovery is that each characteristic has specific length requirements for commands:

- **MULTI_CHAR (2291C4B5)**: Requires exactly 17 bytes
- **CONTROL_CHAR (2291C4B4)**: Requires exactly 8 bytes
- **EXTRA_CONTROL (2291C4B7)**: Accepts commands of length 2-20 bytes
- **COMMAND_CHAR (2291C4B6)**: Used for text commands, variable length

Sending commands with incorrect lengths results in error: `Error Domain=CBATTErrorDomain Code=13 "The value's length is invalid."`

## Command Formats and Effects

### MULTI_CHAR Commands (17 bytes)

Format: `f7 [mode] 00 00 [temp] 00 [power] 00 00 00 00 [control bytes] 00 00 [checksum]`

- Used for temperature settings and primary kettle control
- **CRITICAL DISTINCTION**: Commands to this characteristic affect the clock display, NOT power/standby
- **WARNING**: The byte at index 6 (7th byte) with values 0xD0/0x80 is NOT a power/standby control as originally documented - it only affects the clock display
- **DO NOT USE** these commands for power/standby control as they only turn off the clock display, not the kettle
- Changing bytes 7-8 (index 6-7) with values like C0/E0 or C8/80 similarly only affects the clock display

This characteristic should not be used for standby control as it does not control power functions.

### CONTROL_CHAR Commands (8 bytes)

- Requires exactly 8 bytes
- Purpose of this characteristic is still being explored
- May be related to specialized control functions
- No specific command format has been confirmed yet

### EXTRA_CONTROL Commands (2-20 bytes)

- Accepts variable length commands (2-20 bytes)
- Highly flexible in terms of length
- Purpose and command structure still being investigated

### Text Commands via COMMAND_CHAR

Known text commands from documentation:
- `wifion` - Enable WiFi
- `wifioff` - Disable WiFi
- `wifista` - Enable WiFi station mode
- `httpfw` - Firmware-related function

Attempted text commands for standby control:
- `power:on` / `power:off`
- `standby` / `heat`
- `toggle`
- `start` / `stop`
- `boil` / `hold`

None of these text commands have been confirmed to control standby mode.

## Comparison with Older EKG+ Model

The older Fellow Stagg EKG+ model uses a completely different BLE protocol:

- **Service UUID**: `00001820-0000-1000-8000-00805f9b34fb`
- **Characteristic UUID**: `00002A80-0000-1000-8000-00805f9b34fb`
- **Command Structure**: 
  ```
  0xEF, 0xDD,       # Magic header
  0x0A,             # Command flag
  sequence_number,  # Incremented with each command
  command_type,     # 0=power, 1=temperature
  value,           # For power: 1=on, 0=off
  checksum1,        # (sequence + value) & 0xFF
  checksum2         # command_type
  ```
- **Power Control**: Simple command with command_type=0, value=1 (on) or 0 (off)

Attempts to use this command structure on the Pro model have not been successful.

## Known Issues and Limitations

1. **Clock Display vs. Power Control - Important Distinction**:
   - Commands to the MULTI_CHAR that were initially thought to control power (D0/80 values) actually only control the clock display
   - **DO NOT CONFUSE** clock display commands with power/standby commands - they are entirely different functions
   - The original protocol documentation incorrectly suggests that the D0/80 bytes control heating/standby, when they only affect the display
   - True standby control requires a different command and possibly a different characteristic entirely

2. **Standby Control Challenges**:
   - The exact command to toggle standby mode without affecting the display has not been identified
   - Multiple command structures and byte positions have been tested

3. **Command Length Validation**:
   - Strict length requirements for each characteristic
   - Commands must be padded to the exact required length

## Tested Commands and Results

Extensive testing with various command patterns has yielded the following results:

### CONTROL_CHAR (8-byte) Command Testing Results

The following command patterns were tested on the CONTROL_CHAR characteristic and did NOT affect standby state:

- Individual bit toggles in byte 0 (bits 0-7)
- Individual bit toggles in byte 1 (bits 0,7)
- Traditional power values in byte 0 (0xD0, 0x80)
- EKG+ style commands
- All bits ON pattern (0xFF in all bytes)
- First half ON pattern (0xFF in first 4 bytes)
- Second half ON pattern (0xFF in last 4 bytes)

Notably, one command pattern (EKG+ style on CONTROL_CHAR) resulted in a different error message: "The request is not supported," suggesting this command pattern may be recognized but rejected.

### EXTRA_CONTROL Command Testing Results

The following command patterns were tested on the EXTRA_CONTROL characteristic and did NOT affect standby state:

- Simple ON/OFF patterns (0x01/0x00 in first byte) with 2-byte length
- Simple ON/OFF patterns (0x01/0x00 in first byte) with 4-byte length
- Simple ON/OFF patterns (0x01/0x00 in first byte) with 8-byte length
- Traditional power values (0xD0, 0x80) with 8-byte length
- EKG+ style command with 8-byte length

## Recommendations for Further Exploration

Based on our extensive testing results, here are updated recommendations for further investigation:

1. **Try Other Characteristics**:
   - None of the tested patterns on CONTROL_CHAR or EXTRA_CONTROL affected standby
   - Consider exploring the COMMAND_CHAR with more complex text commands
   - Try investigating any undocumented characteristics or services

2. **Different Command Structure Approaches**:
   - The command structures we've tested (bit toggles, EKG+ style) don't control standby
   - Consider a completely different command structure or format
   - Try exploring command sequences (multiple commands in sequence) rather than single commands

3. **Deeper BLE Analysis**:
   - Use a dedicated BLE sniffer with the official app if possible
   - Focus on capturing the exact moment when standby is toggled
   - Look for commands or notifications that differ from what we've tested

4. **Hardware Analysis**:
   - Consider if there might be a hardware button that directly controls standby
   - Investigate if standby might be controlled by a different mechanism entirely

5. **Alternative Control Methods**:
   - Explore if there are other control methods beyond BLE (e.g., WiFi connectivity)
   - Investigate if power control might be part of a larger command sequence or state machine

## Next Steps

The most promising approach is to continue systematic testing of various command patterns on the CONTROL_CHAR (8-byte commands) and EXTRA_CONTROL characteristic. The script `kettle-standby-tester.py` provides a framework for this testing.

# Packet Capture Analysis for Stagg EKG Pro BLE Protocol

## Connection Sequence Breakdown

### Initial Connection Stages
1. **MTU Exchange**
   - Client MTU: 527 or 517
   - Server MTU: 256
   - Standardized BLE connection initialization

2. **Service Discovery**
   - Primary Services:
     * GAP (Generic Access Profile)
     * GATT (Generic Attribute Profile)
     * Custom Services

3. **Characteristic Exploration**
   - Multiple read and write requests to different handle ranges
   - Probing characteristics across handles 0x0001-0xffff

## Interesting Observations

### Connectivity Patterns
- Multiple connection attempts observed
- Different source MAC addresses
  * `0xaf9ab527`
  * `0x50654393`
  * `0x50654b92`
  * `0x50655caf`

### Device Identification
- Device Name: `EKG-2d-25-b0`
- Consistent across multiple connection attempts

### Command Patterns
- Detected text-based commands:
  1. `wifion`
  2. `wifista`
  3. `httpfw`

### Characteristic Interactions
- Frequent writes to handles:
  * `0x002a`
  * `0x000f`
  * `0x0012`

### Observed Write Sequence
1. Write to handle
2. Immediate read request to same handle
3. Server responds with varying payload

## Potential Protocol Insights

### Authentication Hints
- Complex multi-step connection process
- Requires specific sequence of writes and reads
- No direct password/token visible in captured logs

### Characteristic Write Patterns
```
Handle   Purpose                   Observed Commands
0x002a   WiFi-related commands     wifion, wifista, httpfw
0x000f   Unknown setup             Varied byte sequences
0x0012   Unknown configuration     Multiple write/read cycles
```

### Provisioning Hints
- Characteristics related to:
  1. Provisioning
  2. WiFi configuration
  3. Possible firmware update mechanism

## Recommended Next Steps
1. Capture logs during:
   - Initial kettle power-on
   - Temperature setting
   - WiFi configuration
2. Focus on handles `0x002a`, `0x000f`, `0x0012`
3. Document exact byte sequences for writes
4. Look for correlation between writes and subsequent reads

## Notable Payload Observations
- Mostly short text commands
- Some complex byte sequences
- Possible encryption or encoding

## Potential Clues
- Detected text in payloads:
  * `prov-scan`
  * `prov-session`
  * `prov-config`
  * Indicates provisioning/configuration mechanism

## Warning
- Protocol appears deliberately obfuscated
- Requires careful, methodical investigation
- Avoid rapid or aggressive probing

# Fellow Stagg EKG Pro BLE Protocol Testing Guide

## Current Service Characteristics

### Primary Service UUIDs
- `021A`
- `7AEBF330-6CB1-46E4-B23B-7CC2262C605E`

### Key Characteristics
| UUID | Permissions | Notes |
|------|-------------|-------|
| `2291C4B1-5D7F-4477-A88B-B266EDB97142` | Read, Notify | Status notification characteristic |
| `2291C4B2-5D7F-4477-A88B-B266EDB97142` | Read, Notify | Additional status channel |
| `2291C4B3-5D7F-4477-A88B-B266EDB97142` | Read, Notify | Additional status channel |
| `2291C4B4-5D7F-4477-A88B-B266EDB97142` | Read, Write | Control characteristic |
| `2291C4B5-5D7F-4477-A88B-B266EDB97142` | Read, Write, Notify | Multi-purpose characteristic |
| `2291C4B6-5D7F-4477-A88B-B266EDB97142` | Write, Notify | Primary command characteristic |
| `2291C4B7-5D7F-4477-A88B-B266EDB97142` | Read, Write | Additional control characteristic |
| `2291C4B8-5D7F-4477-A88B-B266EDB97142` | Read | Read-only characteristic |
| `2291C4B9-5D7F-4477-A88B-B266EDB97142` | Read | Read-only characteristic |

## Systematic Testing Procedure

### Step 1: Notification Setup
**Target Characteristic**: `2291C4B1-5D7F-4477-A88B-B266EDB97142`
- **Action**: Enable notifications
- **Method**: 
  - Use notification/bell icon in BLE tool
- **Expected Outcome**: 
  - Receive real-time status updates
  - Observe temperature and kettle status changes

### Step 2: Command Characteristic Exploration
**Target Characteristic**: `2291C4B6-5D7F-4477-A88B-B266EDB97142`
**Test Commands**:
1. `wifion`
   - **Purpose**: Enable WiFi
   - **Expected Response**: WiFi activation confirmation

2. `wifista`
   - **Purpose**: Enter WiFi station mode
   - **Expected Response**: WiFi mode change confirmation

3. `httpfw`
   - **Purpose**: Firmware-related functions
   - **Expected Response**: Firmware information or mode change

### Step 3: Additional Characteristic Testing
**Recommended Approach**:
- Test writing to `2291C4B4`, `2291C4B5`, and `2291C4B7`
- Attempt reading from `2291C4B2`, `2291C4B3`, `2291C4B8`, `2291C4B9`
- Document each characteristic's response

### Temperature Command Exploration
**Encoding Observations**:
- Temperature encoded as 2× Celsius value
- Modes:
  - `F717 0000`: Celsius mode
  - `F715 0000`: Fahrenheit mode

**Potential Temperature Command Formats**:
- Verify accepted formats for temperature setting
- Test various temperature values
- Observe encoding and response mechanisms

## Documentation Recommendations
For each test:
- Record exact command sent
- Note characteristic used
- Capture full response details
- Log any error messages
- Document unexpected behaviors

## Key Investigation Points
1. Authentication mechanism
2. Command validation process
3. Temperature control methods
4. WiFi configuration procedures
5. Firmware interaction capabilities

## Caution Notes
- Proceed methodically
- Avoid rapid, repeated commands
- Respect device's communication protocol
- Do not force or spam commands

## Next Steps
- Systematically work through test procedures
- Analyze and document each interaction
- Build comprehensive understanding of protocol

**Note**: This testing is for research and educational purposes, focusing on understanding the device's communication protocol responsibly.

# BLE Protocol Access Troubleshooting Approach

## Context
When encountering "reading is not permitted" errors while attempting to enable notifications on the 2291C4B1 characteristic, a systematic investigation is required to understand the device's authentication and access mechanisms.

## Troubleshooting Strategy

### 1. Command Characteristic Exploration
**Target Characteristic**: `2291C4B6-5D7F-4477-A88B-B266EDB97142`
- **Action**: Write to command characteristic
- **Method**: 
  - Use TEXT format (not HEX)
  - Send command: `wifion`
- **Objective**: Determine if direct command writing is possible without prior authentication

### 2. Alternative Characteristic Writing
Attempt writing to additional characteristics with Write permissions:

| Characteristic UUID | Permissions | Action |
|---------------------|-------------|--------|
| `2291C4B4-5D7F-4477-A88B-B266EDB97142` | Read and Write | Attempt write |
| `2291C4B5-5D7F-4477-A88B-B266EDB97142` | Read, Write, Notify | Attempt write and notification |
| `2291C4B7-5D7F-4477-A88B-B266EDB97142` | Read and Write | Attempt write |

### 3. Read Permission Investigation
**Targets**:
- `2291C4B2`
- `2291C4B3`

**Objectives**:
- Verify read access
- Identify if restrictions are characteristic-specific
- Gather potential authentication or initialization information

### 4. CCCD (Client Characteristic Configuration Descriptor) Approach
**Technique**: Enable notifications via descriptor
- **Descriptor**: 0x2902
- **Notification Enable Value**: `0100` (hex)
- **Rationale**: Bypass direct notification enabling
- **Approach**: 
  1. Locate descriptors for each characteristic
  2. Write `0100` to enable notifications
  3. Observe device response

## Underlying Principles
- BLE devices often require specific initialization sequences
- Authentication may involve:
  - Specific command order
  - Preparatory write operations
  - Descriptor configuration
- Error messages provide clues about required setup steps

## Recommended Documentation
For each attempted approach, carefully document:
- Exact characteristic accessed
- Command or value written
- Device response
- Any error messages
- Observed behavior

## Potential Scenarios
1. **Authentication Required**
   - Specific sequence of writes needed
   - Potential challenge-response mechanism

2. **Staged Access**
   - Initial commands may unlock subsequent operations
   - Gradual permission expansion

3. **Security Mechanism**
   - Prevents direct access without proper initialization
   - Protects device functionality

## Next Steps
- Systematically work through the troubleshooting steps
- Analyze responses carefully
- Look for patterns in successful interactions
- Compare with original app's connection sequence

## Caution
- Proceed methodically
- Avoid rapid, repeated attempts that might trigger device lockout
- Respect device security mechanisms

**Note**: This approach is for research and educational purposes, focusing on understanding the BLE communication protocol.

# Fellow Stagg EKG Pro Kettle BLE Protocol Analysis

## Project Overview

### Original Goal
We set out to reverse-engineer the Bluetooth Low Energy (BLE) protocol of the Fellow Stagg EKG Pro kettle to enable direct control without relying on the official app. This was inspired by similar work done on the older Fellow Stagg EKG model, which had been successfully reverse-engineered to allow integration with home automation systems.

### Approach Methodology
1. **Packet Capture**: Utilized Wireshark to capture BLE packets during kettle operations
2. **Traffic Analysis**: Identified key characteristics and command patterns
3. **Command Testing**: Sent test commands to verify understanding
4. **Response Observation**: Analyzed kettle's responses to various commands and operations

## Key Discoveries

### 1. BLE Service Characteristics

#### Status Notification Channel
- **Handle**: 0x0027
- **UUID**: 2291C4B1-5D7F-4477-A88B-B266EDB97142
- **Function**: Provides real-time status updates from the kettle

#### Command Channel
- **UUID**: 2291C4B6-5D7F-4477-A88B-B266EDB97142
- **Function**: Receives text-based commands

#### Full Characteristic Family
- **2291C4B1**: Status notifications (Read, Notify)
- **2291C4B2**: Additional control/status channel (Read, Notify)
- **2291C4B6**: Command channel (Write, Notify)
- **2291C4B7**: Additional characteristic (Read, Write)

### 2. Temperature Encoding

#### Encoding Format
- Temperature is encoded as 2× the Celsius value
- Example: 41°C = 0x52 (82 decimal)

#### Mode Indicators
- **F717 0000**: Celsius mode
- **F715 0000**: Fahrenheit mode

#### Notification Format Examples
```
Celsius:   F7170000 5280C080 00020000 11e000091  (41°C)
Fahrenheit: F7150000 5680CD00 00021000 11e000092
```

### 3. Command Structure
- Text-based commands for WiFi setup
- Commands include: 
  - "wifion"
  - "wifista"
  - "httpfw"

### 4. Authentication Mechanism
- More complex than the older model's simple password
- Requires:
  - Proper service discovery
  - Notification/indication configuration
  - Specific command sequence

## Challenges and Limitations

### Protocol Complexity
- Significantly more sophisticated than previous model
- Requires precise authentication sequence
- Limited functionality in official app restricted full command capture

## Technical Implications

### Differences from Previous Model
- More advanced BLE protocol
- Multi-step authentication
- Text-based command interface
- More robust communication mechanism

## Recommended Next Steps

### Protocol Mapping
1. Capture complete authentication sequence
2. Identify full set of temperature control commands
3. Locate power on/off commands

### Potential Development Paths
- Custom BLE application development
- Home Assistant integration
- Open-source control library creation

## Conclusion
The Fellow Stagg EKG Pro represents a significant advancement in smart kettle technology, with a more secure and complex BLE communication protocol that demands a sophisticated approach to reverse engineering and control.

**Note**: This analysis is for educational and research purposes, respecting the intellectual property of Fellow Products.

---

# Packet Capture Analysis for Stagg EKG Pro BLE Protocol

## Connection Sequence Breakdown

### Initial Connection Stages
1. **MTU Exchange**
   - Client MTU: 527 or 517
   - Server MTU: 256
   - Standardized BLE connection initialization

2. **Service Discovery**
   - Primary Services:
     * GAP (Generic Access Profile)
     * GATT (Generic Attribute Profile)
     * Custom Services

3. **Characteristic Exploration**
   - Multiple read and write requests to different handle ranges
   - Probing characteristics across handles 0x0001-0xffff

## Interesting Observations

### Connectivity Patterns
- Multiple connection attempts observed
- Different source MAC addresses
  * `0xaf9ab527`
  * `0x50654393`
  * `0x50654b92`
  * `0x50655caf`

### Device Identification
- Device Name: `EKG-2d-25-b0`
- Consistent across multiple connection attempts

### Command Patterns
- Detected text-based commands:
  1. `wifion`
  2. `wifista`
  3. `httpfw`

### Characteristic Interactions
- Frequent writes to handles:
  * `0x002a`
  * `0x000f`
  * `0x0012`

### Observed Write Sequence
1. Write to handle
2. Immediate read request to same handle
3. Server responds with varying payload

## Potential Protocol Insights

### Authentication Hints
- Complex multi-step connection process
- Requires specific sequence of writes and reads
- No direct password/token visible in captured logs

### Characteristic Write Patterns
```
Handle   Purpose                   Observed Commands
0x002a   WiFi-related commands     wifion, wifista, httpfw
0x000f   Unknown setup             Varied byte sequences
0x0012   Unknown configuration     Multiple write/read cycles
```

### Provisioning Hints
- Characteristics related to:
  1. Provisioning
  2. WiFi configuration
  3. Possible firmware update mechanism

## Recommended Next Steps
1. Capture logs during:
   - Initial kettle power-on
   - Temperature setting
   - WiFi configuration
2. Focus on handles `0x002a`, `0x000f`, `0x0012`
3. Document exact byte sequences for writes
4. Look for correlation between writes and subsequent reads

## Notable Payload Observations
- Mostly short text commands
- Some complex byte sequences
- Possible encryption or encoding

## Potential Clues
- Detected text in payloads:
  * `prov-scan`
  * `prov-session`
  * `prov-config`
  * Indicates provisioning/configuration mechanism

## Warning
- Protocol appears deliberately obfuscated
- Requires careful, methodical investigation
- Avoid rapid or aggressive probing

---

# Fellow Stagg EKG Pro BLE Protocol Testing Guide

## Current Service Characteristics

### Primary Service UUIDs
- `021A`
- `7AEBF330-6CB1-46E4-B23B-7CC2262C605E`

### Key Characteristics
| UUID | Permissions | Notes |
|------|-------------|-------|
| `2291C4B1-5D7F-4477-A88B-B266EDB97142` | Read, Notify | Status notification characteristic |
| `2291C4B2-5D7F-4477-A88B-B266EDB97142` | Read, Notify | Additional status channel |
| `2291C4B3-5D7F-4477-A88B-B266EDB97142` | Read, Notify | Additional status channel |
| `2291C4B4-5D7F-4477-A88B-B266EDB97142` | Read, Write | Control characteristic |
| `2291C4B5-5D7F-4477-A88B-B266EDB97142` | Read, Write, Notify | Multi-purpose characteristic |
| `2291C4B6-5D7F-4477-A88B-B266EDB97142` | Write, Notify | Primary command characteristic |
| `2291C4B7-5D7F-4477-A88B-B266EDB97142` | Read, Write | Additional control characteristic |
| `2291C4B8-5D7F-4477-A88B-B266EDB97142` | Read | Read-only characteristic |
| `2291C4B9-5D7F-4477-A88B-B266EDB97142` | Read | Read-only characteristic |

## Systematic Testing Procedure

### Step 1: Notification Setup
**Target Characteristic**: `2291C4B1-5D7F-4477-A88B-B266EDB97142`
- **Action**: Enable notifications
- **Method**: 
  - Use notification/bell icon in BLE tool
- **Expected Outcome**: 
  - Receive real-time status updates
  - Observe temperature and kettle status changes

### Step 2: Command Characteristic Exploration
**Target Characteristic**: `2291C4B6-5D7F-4477-A88B-B266EDB97142`
**Test Commands**:
1. `wifion`
   - **Purpose**: Enable WiFi
   - **Expected Response**: WiFi activation confirmation

2. `wifista`
   - **Purpose**: Enter WiFi station mode
   - **Expected Response**: WiFi mode change confirmation

3. `httpfw`
   - **Purpose**: Firmware-related functions
   - **Expected Response**: Firmware information or mode change

### Step 3: Additional Characteristic Testing
**Recommended Approach**:
- Test writing to `2291C4B4`, `2291C4B5`, and `2291C4B7`
- Attempt reading from `2291C4B2`, `2291C4B3`, `2291C4B8`, `2291C4B9`
- Document each characteristic's response

### Temperature Command Exploration
**Encoding Observations**:
- Temperature encoded as 2× Celsius value
- Modes:
  - `F717 0000`: Celsius mode
  - `F715 0000`: Fahrenheit mode

**Potential Temperature Command Formats**:
- Verify accepted formats for temperature setting
- Test various temperature values
- Observe encoding and response mechanisms

## Documentation Recommendations
For each test:
- Record exact command sent
- Note characteristic used
- Capture full response details
- Log any error messages
- Document unexpected behaviors

## Key Investigation Points
1. Authentication mechanism
2. Command validation process
3. Temperature control methods
4. WiFi configuration procedures
5. Firmware interaction capabilities

## Caution Notes
- Proceed methodically
- Avoid rapid, repeated commands
- Respect device's communication protocol
- Do not force or spam commands

## Next Steps
- Systematically work through test procedures
- Analyze and document each interaction
- Build comprehensive understanding of protocol

**Note**: This testing is for research and educational purposes, focusing on understanding the device's communication protocol responsibly.

---

# BLE Protocol Access Troubleshooting Approach

## Context
When encountering "reading is not permitted" errors while attempting to enable notifications on the 2291C4B1 characteristic, a systematic investigation is required to understand the device's authentication and access mechanisms.

## Troubleshooting Strategy

### 1. Command Characteristic Exploration
**Target Characteristic**: `2291C4B6-5D7F-4477-A88B-B266EDB97142`
- **Action**: Write to command characteristic
- **Method**: 
  - Use TEXT format (not HEX)
  - Send command: `wifion`
- **Objective**: Determine if direct command writing is possible without prior authentication

### 2. Alternative Characteristic Writing
Attempt writing to additional characteristics with Write permissions:

| Characteristic UUID | Permissions | Action |
|---------------------|-------------|--------|
| `2291C4B4-5D7F-4477-A88B-B266EDB97142` | Read and Write | Attempt write |
| `2291C4B5-5D7F-4477-A88B-B266EDB97142` | Read, Write, Notify | Attempt write and notification |
| `2291C4B7-5D7F-4477-A88B-B266EDB97142` | Read and Write | Attempt write |

### 3. Read Permission Investigation
**Targets**:
- `2291C4B2`
- `2291C4B3`

**Objectives**:
- Verify read access
- Identify if restrictions are characteristic-specific
- Gather potential authentication or initialization information

### 4. CCCD (Client Characteristic Configuration Descriptor) Approach
**Technique**: Enable notifications via descriptor
- **Descriptor**: 0x2902
- **Notification Enable Value**: `0100` (hex)
- **Rationale**: Bypass direct notification enabling
- **Approach**: 
  1. Locate descriptors for each characteristic
  2. Write `0100` to enable notifications
  3. Observe device response

## Underlying Principles
- BLE devices often require specific initialization sequences
- Authentication may involve:
  - Specific command order
  - Preparatory write operations
  - Descriptor configuration
- Error messages provide clues about required setup steps

## Recommended Documentation
For each attempted approach, carefully document:
- Exact characteristic accessed
- Command or value written
- Device response
- Any error messages
- Observed behavior

## Potential Scenarios
1. **Authentication Required**
   - Specific sequence of writes needed
   - Potential challenge-response mechanism

2. **Staged Access**
   - Initial commands may unlock subsequent operations
   - Gradual permission expansion

3. **Security Mechanism**
   - Prevents direct access without proper initialization
   - Protects device functionality

## Next Steps
- Systematically work through the troubleshooting steps
- Analyze responses carefully
- Look for patterns in successful interactions
- Compare with original app's connection sequence

## Caution
- Proceed methodically
- Avoid rapid, repeated attempts that might trigger device lockout
- Respect device security mechanisms

**Note**: This approach is for research and educational purposes, focusing on understanding the BLE communication protocol.

---

# Fellow Stagg EKG Pro Kettle BLE Protocol Analysis

## Project Overview

### Original Goal
We set out to reverse-engineer the Bluetooth Low Energy (BLE) protocol of the Fellow Stagg EKG Pro kettle to enable direct control without relying on the official app. This was inspired by similar work done on the older Fellow Stagg EKG model, which had been successfully reverse-engineered to allow integration with home automation systems.

### Approach Methodology
1. **Packet Capture**: Utilized Wireshark to capture BLE packets during kettle operations
2. **Traffic Analysis**: Identified key characteristics and command patterns
3. **Command Testing**: Sent test commands to verify understanding
4. **Response Observation**: Analyzed kettle's responses to various commands and operations

## Key Discoveries

### 1. BLE Service Characteristics

#### Status Notification Channel
- **Handle**: 0x0027
- **UUID**: 2291C4B1-5D7F-4477-A88B-B266EDB97142
- **Function**: Provides real-time status updates from the kettle

#### Command Channel
- **UUID**: 2291C4B6-5D7F-4477-A88B-B266EDB97142
- **Function**: Receives text-based commands

#### Full Characteristic Family
- **2291C4B1**: Status notifications (Read, Notify)
- **2291C4B2**: Additional control/status channel (Read, Notify)
- **2291C4B6**: Command channel (Write, Notify)
- **2291C4B7**: Additional characteristic (Read, Write)

### 2. Temperature Encoding

#### Encoding Format
- Temperature is encoded as 2× the Celsius value
- Example: 41°C = 0x52 (82 decimal)

#### Mode Indicators
- **F717 0000**: Celsius mode
- **F715 0000**: Fahrenheit mode

#### Notification Format Examples
```
Celsius:   F7170000 5280C080 00020000 11e000091  (41°C)
Fahrenheit: F7150000 5680CD00 00021000 11e000092
```

### 3. Command Structure
- Text-based commands for WiFi setup
- Commands include: 
  - "wifion"
  - "wifista"
  - "httpfw"

### 4. Authentication Mechanism
- More complex than the older model's simple password
- Requires:
  - Proper service discovery
  - Notification/indication configuration
  - Specific command sequence

## Challenges and Limitations

### Protocol Complexity
- Significantly more sophisticated than previous model
- Requires precise authentication sequence
- Limited functionality in official app restricted full command capture

## Technical Implications

### Differences from Previous Model
- More advanced BLE protocol
- Multi-step authentication
- Text-based command interface
- More robust communication mechanism

## Recommended Next Steps

### Protocol Mapping
1. Capture complete authentication sequence
2. Identify full set of temperature control commands
3. Locate power on/off commands

### Potential Development Paths
- Custom BLE application development
- Home Assistant integration
- Open-source control library creation

## Conclusion
The Fellow Stagg EKG Pro represents a significant advancement in smart kettle technology, with a more secure and complex BLE communication protocol that demands a sophisticated approach to reverse engineering and control.

**Note**: This analysis is for educational and research purposes, respecting the intellectual property of Fellow Products.

---

