
# Motor controller I2C protocol

Version 2.0

---
---

# Table of contents

- [(1) Quick overview](#1-quick-overview)
- [(2) Commands](#2-commands)
    - [(2.1) Overview](#21-overview)
    - [(2.2) SMBus protocols](#22-smbus-protocols)
	    - [(2.2.1) Write word](#221-write-word)
	    - [(2.2.2) Read word](#222-read-word)
	    - [(2.2.3) Read block](#223-read-block)
    - [(2.3) List of commands](#23-list-of-commands)
    - [(2.4) Command descriptions](#24-command-descriptions)
        - [(2.4.1) PARAMETER_n (n = 0...15)](#241-parameter_n-n--015)
        - [(2.4.2) CONTROL_MODE](#242-control_mode)
        - [(2.4.3) CONTROL_UPDATE](#243-control_update)
        - [(2.4.4) SAMPLES_EN_VALUES](#244-samples_en_values)
        - [(2.4.5) SAMPLES_RATE_DIV](#245-samples_rate_div)
        - [(2.4.6) SAMPLES_READ_BUFFER](#246-samples_read_buffer)
- [(3) Parameters and control modes](#3-parameters-and-control-modes)
    - [(3.1) Overview](#31-overview)
    - [(3.2) List of parameters](#32-list-of-parameters)
    - [(3.3) List of control modes](#33-list-of-control-modes)
    - [(3.4) Control mode descriptions](#34-control-mode-descriptions)
        - [(3.4.1) MODE_DISABLED](#341-mode_disabled)
        - [(3.4.2) MODE_CALIBRATE](#342-mode_calibrate)
        - [(3.4.3) MODE_VELOCITY](#343-mode_velocity)
        - [(3.4.4) MODE_POSITION](#344-mode_position)
- [(4) The sample buffer](#4-the-sample-buffer)
    - [(4.1) Overview](#41-overview)
    - [(4.2) Samples](#42-samples)
    - [(4.3) List of sample values](#43-list-of-sample-values)
    - [(4.4) Buffer overflows](#44-buffer-overflows)
- [(5) List of data types](#5-list-of-data-types)

---
---

# (1) Quick overview

This protocol is designed for communicating with a motor controller over an I2C bus. Version 2.0 has been designed to work with SMBus masters, and adds some improvements over version 1.0.

The device presents multiple control modes (listed in [section 3.3](#33-list-of-control-modes), each of which control the motor in a different way. In order to configure the control loops, a set of up to sixteen 16-bit parameters are available. These are listed in [section 3.2](#32-list-of-parameters). For more information about parameters and control modes, see [section 3](#3-parameters-and-control-modes).

In order to facilitate debugging, as well as enable data logging (among other possible use cases), the device includes a sample buffer for storing samples. Information about this can be found in [section 4](#4-the-sample-buffer).

All communication is done using SMBus commands, as listed in [section 2.3](#23-list-of-commands). For more information about the commands, see [section 2](#2-commands).

# (2) Commands

## (2.1) Overview

All communication is done using commands over an SMBus interface. Each command supports one or more SMBus protocols, listed in [section 2.2](#22-smbus-protocols). Calling a command with a protocol it doesn't support will result in undefined behaviour (including possibly crashing the device).

Some commands may be aborted by the device, by responding with a NACK to any byte it receives. After receiving a NACK, the master must generate a STOP condition (as required by the SMBus specification). The master must never abort a command, and must transfer all the data as specified by the SMBus protocol used (unless the command is aborted by the device). Note that protocols that end with a master-receiver require the master to send a NACK on the last byte to end the transaction; this does not count as aborting the command. Also note that it is only possible for the device to abort a command when it is acting as a slave-receiver; when acting as a slave-transmitter it cannot abort a command since it doesn't send the ACK/NACK bit. The device will abort a command in the following cases:

- The device will respond with a NACK to any invalid command code.
- Valid command codes may get a NACK response. See the command's description for more information.
- A command may return a NACK after any data byte. See the command's description for more information.

The device will always respond with an ACK to its SMBus address (as required by the SMBus specification).

## (2.2) SMBus protocols

The commands make use of the following SMBus protocols. This list doesn't contain all the SMBus protocols, only those that are used in this document.

- Write word
- Read word
- Read block

The following subsections illustrate how each protocol is carried out using diagrams like this:

```
  1         7         1    1       8       1   1
| S | Slave address | Wr |=A=| Data byte |=A=| P |
```

Fields are separated by a vertical bar and the number of bits in each field is indicated above them. Fields that are send from master to slave are written as normal; fields sent from slave to master have spaces replaced by equals signs. A value shown below a field is a mandatory value for that field. Long messages may be slit across multiple lines. The diagrams also make use of the following symbols:

| Symbol | Meaning                  |
|:-----: | ------------------------ |
| S      | Start condition          |
| Sr     | Repeated start condition |
| Rd     | Read (bit value of 1)    |
| Wr     | Write (bit value of 0)   |
| A      | Acknowledge bit          |
| P      | Stop condition           |
| Text   | Master to slave          |
| =Text= | Slave to master          |
| ...    | Repeating pattern        |

### (2.2.1) Write word

The "write word" protocol writes a 16-bit word of data after sending a command code. The least significant byte is sent first. If the device responds with a NACK to either the command byte or either of the data bytes, the master must immediately generate a STOP condition. If all bytes get an ACK response, the master must generate a STOP condition after the last byte.

```
  1         7         1    1        8         1
| S | Slave address | Wr |=A=| Command code |=A=|

            8         1         8          1   1
    | Data byte low |=A=| Data byte high |=A=| P |
```

### (2.2.2) Read word

The "read word" protocol reads a 16-bit word of data after sending a command code. The master must respond with a NACK to the last (and only the last) data byte and then generate a STOP condition. The master must also generate a STOP condition if the device responds to the command code with a NACK.

```
  1         7         1    1        8         1   1          7         1
| S | Slave address | Wr |=A=| Command code |=A=| Sr | Slave address | Rd |

             8          1          8           1   1
    |=Data=word=[7:0]=| A |=Data=word=[15:8]=| A | P |
                                                   1
```

### (2.2.3) Read block

The "read block" protocol reads a block of N bytes ($0 \lt N \le 32$) after sending a command code. The master must respond with a NACK to the last (and only the last) data byte and then generate a STOP condition. The master must also generate a STOP condition if the device responds to the command code with a NACK.

```
  1         7         1    1        8         1   1          7         1
| S | Slave address | Wr |=A=| Command code |=A=| Sr | Slave address | Rd |

            8          1        8        1        8        1              8        1   1
    |=Byte=count=(N)=| A |=Data=byte=0=| A |=Data=byte=1=| A | ... |=Data=byte=N=| A | P |
                                                                                       1
```

## (2.3) List of commands

The following table lists all the available commands for communicating with the device. Any command codes not on this list will respond with a NACK.

| Command code | Name                 | SMBus protocols | Description                               |
|:------------:| -------------------- | --------------- | ----------------------------------------- |
|     0x00     | PARAMETER_0          | read/write word | Gets or sets controller parameter 0       |
|     0x01     | PARAMETER_1          | read/write word | Gets or sets controller parameter 1       |
|     0x02     | PARAMETER_2          | read/write word | Gets or sets controller parameter 2       |
|     0x03     | PARAMETER_3          | read/write word | Gets or sets controller parameter 3       |
|     0x04     | PARAMETER_4          | read/write word | Gets or sets controller parameter 4       |
|     0x05     | PARAMETER_5          | read/write word | Gets or sets controller parameter 5       |
|     0x06     | PARAMETER_6          | read/write word | Gets or sets controller parameter 6       |
|     0x07     | PARAMETER_7          | read/write word | Gets or sets controller parameter 7       |
|     0x08     | PARAMETER_8          | read/write word | Gets or sets controller parameter 8       |
|     0x09     | PARAMETER_9          | read/write word | Gets or sets controller parameter 9       |
|     0x0A     | PARAMETER_10         | read/write word | Gets or sets controller parameter 10      |
|     0x0B     | PARAMETER_11         | read/write word | Gets or sets controller parameter 11      |
|     0x0C     | PARAMETER_12         | read/write word | Gets or sets controller parameter 12      |
|     0x0D     | PARAMETER_13         | read/write word | Gets or sets controller parameter 13      |
|     0x0E     | PARAMETER_14         | read/write word | Gets or sets controller parameter 14      |
|     0x0F     | PARAMETER_15         | read/write word | Gets or sets controller parameter 15      |
|     0x10     | CONTROL_MODE         | read/write word | Gets or sets the control mode             |
|     0x11     | CONTROL_UPDATE       | write word      | Sends updates to the controller           |
|     0x20     | SAMPLES_EN_VALUES    | read/write word | Gets or sets the enabled sample values    |
|     0x21     | SAMPLES_RATE_DIV     | read/write word | Gets or sets the sample rate divider      |
|     0x22     | SAMPLES_READ_BUFFER  | read block      | Extracts data from the sample buffer      |

## (2.4) Command descriptions

### (2.4.1) PARAMETER_n (n = 0...15)

- **Command code:** 0x00 + n
- **SMBus protocols:** read/write word

Gets or sets one of the controller's parameters. Each parameter uses one of the 16-bit data types defined in [section 5](#5-list-of-data-types). Some parameters may allow only certain values to be written; if an invalid value is written, the device will respond with a NACK after either of the data bytes, aborting the command. The device will always respond with an ACK to the command code. For a list of parameters, their data types and their valid values see [section 3.2](#32-list-of-parameters).

Note that not all 16 parameters will necessarily be used. Used parameters will always return the last value successfully written to them (with the exception of a hardware reset); the device will never change their value automatically. Unused parameters can be written to but their values may not be saved; they can also be read from but will return undefined values. Whether or not unused parameters reject certain values is not defined.

See [section 3](#3-parameters-and-control-modes) for more information about parameters.

### (2.4.2) CONTROL_MODE

- **Command code:** 0x10
- **SMBus protocols:** read/write word

Gets or sets the device's control mode. Only the control modes listed in [section 3.3](#33-list-of-control-modes) are allowed; attempting to write any other value will result in a NACK response after either of the data bytes, aborting the command. Reading the control mode will always return the last value succesfully set (with the exception of a hardware reset); the device will never chage the control mode automatically.

See [section 3](#3-parameters-and-control-modes) for more information about control modes.

### (2.4.3) CONTROL_UPDATE

- **Command code:** 0x11
- **SMBus protocols:** write word

Used for sending dynamic update values to the controller. The way this data is interpreted depends on the current control mode. Some control modes may allow only certain values to be written; if an invalid value is written, the device will respond with a NACK after either of the data bytes, aborting the command. Some control modes may not support this command and will respond with a NACK to the command code.

See [section 3](#3-parameters-and-control-modes) for more information about how this command's data is used.

### (2.4.4) SAMPLES_EN_VALUES

- **Command code:** 0x20
- **SMBus protocols:** read/write word

Controls which values are included in samples saved to the sample buffer. Each bit corresponds to a value ID (least significant bit corresponds to value ID 0). If the bit is set, that value will be included in samples; if the bit is cleared, it won't. Note that not all 16 IDs will be used; those that are not used won't add any values to samples, even if their bit is set.

Writing any non-zero value to this register will immediately (after receiving the whole command) clear the sample buffer and restart it with the specified sample values enabled. Writing a value of zero will stop any more samples from being written to the buffer; this won't clear the buffer. The device will also automatically set this register to zero if the buffer ever gets full, to prevent the buffer from overflowing. Reading this register will always return the last value successfully set, or zero; the only value the device is allowed to automatically write is a value of zero.

See [section 4.2](#42-samples) for more information about sample values.

### (2.4.5) SAMPLES_RATE_DIV

- **Command code:** 0x21
- **SMBus protocols:** read/write word

Gets or sets the sample rate divider, interpreted as an unsigned 16-bit integer. The value of the divider determines how often samples are taken and saved to the sample buffer. The sample rate is calculated by

$$F_\text{sample} = \frac{ F_\text{update} }{ \text{SAMPLE\\_RATE\\_DIV} + 1 }$$

where $F_\text{sample}$ is the sample rate in Hz and $F_\text{update}$ is the device's update rate in Hz. Any 16-bit value is allowed to be written by this command.

Note that the sample buffer has no way of indicating when the sample rate changes; clear the sample buffer after changing the sample rate to ensure that all samples in the buffer have a known sample rate, if this is desired.

Reading this register will always return the last value written to it (with the exception of a hardware reset); the device will never change its value automatically.

### (2.4.6) SAMPLES_READ_BUFFER

- **Command code:** 0x22
- **SMBus protocols:** read block

Pops and returns data directly from the sample buffer. This command reads as many bytes as are available from the sample buffer, up to 32 bytes. Since a block size of zero is not allowed, if the buffer is empty the device will respond to the command code with a NACK. The command code will always get an ACK response if there is data available in the buffer.

See [section 4](#4-the-sample-buffer) for more information about the sample buffer.

# (3) Parameters and control modes

## (3.1) Overview

The device provides a set of control modes for controlling the attached motor; each mode controls the motor in a different way. Only a single mode can be used at a time, and is selected using the CONTROL_MODE command. The CONTROL_UPDATE command allows the master to send dynamic update data to the controller and is interpreted differently depending on the control mode.

Each control mode can make use of any of the 16 parameters for tuning control loops or for other things that require constant data.

## (3.2) List of parameters

The following table lists all the used parameters, their data types and valid values. Any IDs not in this list are not used.

#TODO This table needs filling with actual parameters.

| Parameter ID | Name           | Data type | Valid values | Description         |
|:------------:| -------------- | --------- | ------------ | ------------------- |
|       0      | MY_PARAMETER_0 | int16     | Any value    | My first parameter  |
|       1      | MY_PARAMETER_1 | uint16    | 1 < x < 100  | My second parameter |
|       6      | MY_PARAMETER_2 | uint16    | 0, 1, 5, 6   | My third parameter  |
|       8      | MY_PARAMETER_3 | float16   | Any value    | My fourth parameter |

## (3.3) List of control modes

The following table lists all the control modes. Any IDs not in this list are invalid and will be rejected by the CONTROL_MODE command.

| Mode ID | Name           | Description                   |
|:-------:| -------------- | ----------------------------- |
|    0    | MODE_DISABLED  | Motor disabled and not driven |
|    1    | MODE_CALIBRATE | Calibrates the motor position |
|    2    | MODE_VELOCITY  | Controls the motor's velocity |
|    3    | MODE_POSITION  | Controls the motor's position |

## (3.4) Control mode descriptions

### (3.4.1) MODE_DISABLED

- **Mode ID:** 0
- **Parameters used:** None

While in this mode, the controller doesn't drive the motor and lets it move freely. This mode doesn't use the CONTROL_UPDATE command and will always abort any calls to it.

### (3.4.2) MODE_CALIBRATE

- **Mode ID:** 1
- **Parameters used:** None

While in this mode, the controller doesn't drive the motor and lets it move freely. Calls to the CONTROL_UPDATE command will calibrate the motor's absolute position; the motor's current position will be set equal to the value written. The value is interpreted in the same way as [MODE_POSITION](#344-mode_position).

### (3.4.3) MODE_VELOCITY

- **Mode ID:** 2
- **Parameters used:** #TODO List the parameters used

While in this mode, the controller drives the motor at a user-set velocity. Calls to the CONTROL_UPDATE command set the target velocity, and the controller will try to match it. The value is interpreted as a 16-bit signed integer (2's complement), with the least-significant bit being equal to one revolution division per second. The number of divisions per revolution is not currently defined. Positive and negative numbers spin the motor in opposite directions.

#TODO Maybe some information about the control algorithm and how the parameters are used.

### (3.4.4) MODE_POSITION

- **Mode ID:** 3
- **Parameters used:** #TODO List the parameters used

While in this mode, the controller drives the motor to a user-set position. Calls to the CONTROL_UPDATE command set the target position, and the controller will try to match it. The value is interpreted as a 16-bit signed integer, with the least-significant bit being equal to one revolution division. The number of divisions per revolution is not currently defined.

#TODO Maybe some information about the control algorithm and how the parameters are used.

# (4) The sample buffer

## (4.1) Overview

The sample buffer is a FIFO buffer which the device can save sample to. Samples contain values configured by the SAMPLES_EN_VALUES and are saved at a rate configured by the SAMPLES_RATE_DIV command. They can then be read back from the device using the SAMPLES_READ_BUFFER command.

## (4.2) Samples

The device defines a set of values that can be included in each sample, as listed in [section 4.3](#43-list-of-sample-values). These values are intended for things like tracking internal variables for debugging, tracking performance metrics, or logging data.

Each sample is made up of up to 16 different values, configurable using the SAMPLES_EN_VALUES command. Enabled values are pushed to the buffer in ascending order (value ID 0 first, then value ID 1 and so on) with each value in little-endian order (least significant byte is pushed first). The datatype of each value is defined in [section 4.3](#43-list-of-sample-values), but must be one of the datatypes defined in [section 5](#5-list-of-data-types). Note that not all 16 sample value indices are used, and those that are used don't have to be contiguous.

As an example, lets imagine the device implements four sample values:

- 0: MY_VALUE_0 (uint16)
- 1: MY_VALUE_1 (int16)
- 4: MY_VALUE_4 (uint8)
- 5: MY_VALUE_5 (float32)

Setting SAMPLES_EN_VALUES to 0x0021 would enable VALUE_0 and VALUE_5, so each sample would look like this:

```
| MY_VALUE_0[7:0]   | < first byte pushed to FIFO
| MY_VALUE_0[15:8]  |
| MY_VALUE_5[7:0]   |
| MY_VALUE_5[15:8]  |
| MY_VALUE_5[23:16] |
| MY_VALUE_5[31:24] | < last byte pushed to FIFO
```

## (4.3) List of sample values

The following table lists all the implemented sample values. IDs which are not listed are not used.

#TODO This table needs filling with actual sample values.

| Value ID | Name       | Data type | Description            |
|:--------:| ---------- | --------- | ---------------------- |
|    0     | MY_VALUE_0 | uint16    | My first sample value  |
|    1     | MY_VALUE_1 | int16     | My second sample value |
|    4     | MY_VALUE_4 | uint8     | My third sample value  |
|    5     | MY_VALUE_5 | float32   | My fourth sample value |

## (4.4) Buffer overflows

The sample buffer has a limited size and may therfore fill up if it's not emptied fast enough. A sample is only written to the buffer if there is enough space for the whole sample; partial samples will not be written to the buffer. If the buffer is too full to save a pending sample, the value of SAMPLES_EN_VALUES is automacitcally set to zero by the device and no more samples are written to the buffer. The master must write a new value to SAMPLES_EN_VALUES to clear the buffer and start writting samples again.

When the master reads data from the sample buffer using the SAMPLES_READ_BUFFER command, it knows that it has emptied the buffer when the command returns less than 32 bytes or when the command code gets a NACK response. When this happens it may be useful to check the SAMPLES_EN_VALUES register to see if samples are still being written to the buffer.

# (5) List of data types

The following table lists all the data types that might be used by the protocol. Not all of them will necessarily be used.

| Name    | Description                                      |
| ------- | ------------------------------------------------ |
| uint8   | 8-bit unsigned integer                           |
| uint16  | 16-bit unsigned integer                          |
| uint24  | 24-bit unsigned integer                          |
| uint32  | 32-bit unsigned integer                          |
| uint40  | 40-bit unsigned integer                          |
| uint48  | 48-bit unsigned integer                          |
| uint64  | 64-bit unsigned integer                          |
| int8    | 8-bit signed integer                             |
| int16   | 16-bit signed integer                            |
| int24   | 24-bit signed integer                            |
| int32   | 32-bit signed integer                            |
| int40   | 40-bit signed integer                            |
| int48   | 48-bit signed integer                            |
| int64   | 64-bit signed integer                            |
| float16 | 16-bit floating point (IEE 754 half-precision)   |
| float32 | 32-bit floating point (IEE 754 single-precision) |
| float64 | 64-bit floating point (IEE 754 double-precision) |

