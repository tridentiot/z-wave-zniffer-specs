# Zniffer protocol specification

This specification describes the serial protocol between a Z-Wave zniffer and a host.

## Physical interface

The zniffer communicates with the host over a UART, configured as:

| Setting      | Value                                                  |
| ------------ | ------------------------------------------------------ |
| Baud rate    | Implementation defined (see [Set Baud Rate](#set-baud-rate-command)) |
| Data bits    | 8                                                      |
| Parity       | None                                                   |
| Stop bits    | 1                                                      |
| Flow control | None                                                   |

## Byte order

Multi-byte fields do not use the same byte order:

| Field                                | Byte order    |
| ------------------------------------ | ------------- |
| Timestamp                            | Big endian    |
| Beam stop counter                    | Big endian    |
| Start of data (fixed value `0x0321`) | Little endian |

## Frame format

The Zniffer protocol specifies two different frame formats: one for commands and one for radio frames.

The host sends only commands. The zniffer sends command replies and radio frames, and the host tells them apart by the SOF byte.

### Commands

| 0   | 1          | 2      | 3..(2+N) |
| --- | ---------- | ------ | -------- |
| SOF | Command ID | Length | Payload  |

#### Start Of Frame (SOF) (8 bits)

For commands, the SOF must be set to 0x23 (`'#'`).

The zniffer discards any bytes received while it is waiting for a SOF.

#### Command (8 bits)

Specifies the ID of the command. See [Command ID's](#command-ids).

#### Length (8 bits)

The length specifies the length of the command payload. It can be 0.

#### Payload (Length * 8 bits)

The payload depends on the command.

#### Reception rules

- A zniffer can limit the size of commands it accepts. If Length exceeds that limit, the zniffer discards the command and starts waiting for a new SOF. None of the commands in this specification has a payload longer than 1 byte.
- If too much time passes between two bytes of a command, the zniffer discards the partial command and starts waiting for a new SOF. The timeout is implementation defined, so the host should send each command without pauses.
- Commands have no checksum and are not acknowledged at the framing level.
- The zniffer silently ignores unknown command IDs and sends no reply.

### Radio frames

| 0   | 1    | 2..3      | 4..     |
| --- | ---- | --------- | ------- |
| SOF | Type | Timestamp | Payload |

#### Start of Frame (SOF) (8 bits)

For frames, the SOF must be set to 0x21 (`'!'`).

#### Type (8 bits)

The type specifies the type of radio frame:

| Type | Description                                                                |
| :--: | :------------------------------------------------------------------------- |
|  1   | Normal frame                                                               |
|  2   | Beam frame (Deprecated. Replaced by Beam start and Beam stop. Never sent.) |
|  3   | Reserved (not used)                                                        |
|  4   | Beam start                                                                 |
|  5   | Beam stop                                                                  |

#### Timestamp (16 bits)

Big endian.

For normal frames, the timestamp is the lower 16 bits of the zniffer's system tick counter in milliseconds, sampled when the frame is processed. It wraps around every 65.536 seconds.

Beam start and Beam stop frames do not carry a timestamp. The field is set to 0x0000.

#### Payload

The payload depends on the frame type.

For normal frames the payload is specified as:

| 4                 | 5      | 6    | 7..8          | 9      | 10..(9+N)     |
| ----------------- | ------ | ---- | ------------- | ------ | ------------- |
| Channel and speed | Region | RSSI | Start of data | Length | Radio payload |

For Beam start frames the payload is specified as:

| 4                 | 5      | 6    | 7..10     |
| ----------------- | ------ | ---- | --------- |
| Channel and speed | Region | RSSI | Beam data |

A Beam start frame is always 11 bytes long.

For Beam stop frames the payload is specified as:

| 4    | 5..6    |
| ---- | ------- |
| RSSI | Counter |

A Beam stop frame is always 7 bytes long. It has no Channel and speed or Region fields.

#### Channel and speed

Defined as a bitmask with channel in the three most significant bits and speed in
the five least significant bits:

| Bits 7..5 | Bits 4..0 |
| --------- | --------- |
| Channel   | Speed     |

Speed:

| Value | Speed   |
| :---: | :------ |
|   0   | 9.6k    |
|   1   | 40k     |
|   2   | 100k    |
|   3   | 100k LR |

Channel (0 - 4) depends on the region and speed:

| Region                                    | 9.6k / 40k | 100k | 100k LR            |
| ----------------------------------------- | :--------: | :--: | :----------------: |
| 2-channel regions (EU, US, ANZ, HK, IN, IL, RU, CN) | 1 | 0 | -                  |
| LR regions (US_LR, EU_LR)                 | 1          | 0    | Radio channel (3 or 4) |
| 3-channel regions (JP, KR)                | -          | Radio channel (0 - 2) | -     |

For Beam start frames, the zniffer sets the speed based on the region and channel the beam was received on:

| Region            | Channel | Reported speed |
| ----------------- | :-----: | :------------- |
| 2-channel regions | Any     | 40k            |
| LR regions        | 0 - 2   | 40k            |
| LR regions        | 3 - 4   | 100k LR        |
| 3-channel regions | Any     | 100k           |

#### Region

The Z-Wave region the zniffer is listening on. See [Region identifiers](#region-identifiers).

This field always holds a standard Z-Wave region identifier (0 - 33). It never holds a zniffer-specific region identifier (100 - 103). For example, a zniffer set to `US_LR_920MHz` (100) reports `US_LR` (9).

#### RSSI

In signed dBm (int8).

For Beam start and Beam stop frames, this is the RSSI of the most recently received beam.

#### Start of data

The Start of data is the 16-bit value 0x0321 in little endian, so it is sent as the bytes `0x21 0x03`.

#### Length

The length of the radio payload.

#### Radio payload

The received Z-Wave radio frame.

#### Beam data

The first bytes of the received wake-up beam, always 4 bytes in total:

- For beams on channel 3 or 4 (Long Range), the 4 bytes of beam data received.
- For beams on channel 0 - 2, the radio provides 3 bytes, `B0 B1 B2`. The zniffer inserts the constant 0x01 as the third byte and sends `B0 B1 0x01 B2`.

#### Counter

Big endian.

Represents the number of beam frames received in a beam train that started with the Beam start frame type.

The zniffer ends a beam train and sends a Beam stop frame when any of the following happens:

- A normal frame is received. The Beam stop frame is sent before the normal frame.
- The radio reports a receive timeout.
- No new beam is received within a short, implementation-defined time.

## Region identifiers

The region identifiers below are used by the Set Region, Get Regions and Get Region String commands.

Identifiers 100 - 103 are specific to the zniffer. They select alternative Long Range channel configurations of the US_LR and EU_LR regions.

| ID  | Name               | Supported channels | Description                                       |
| :-: | :----------------- | :----------------: | :------------------------------------------------ |
|  0  | `EU`               | 2                  | Europe                                            |
|  1  | `US`               | 2                  | USA                                               |
|  2  | `ANZ`              | 2                  | Australia / New Zealand                           |
|  3  | `HK`               | 2                  | Hong Kong                                         |
|  5  | `IN`               | 2                  | India                                             |
|  6  | `IL`               | 2                  | Israel                                            |
|  7  | `RU`               | 2                  | Russia                                            |
|  8  | `CN`               | 2                  | China                                             |
|  9  | `US_LR_912MHz`     | 4                  | US Long Range, LR channel configuration 1         |
| 11  | `EU_LR_864MHz`     | 4                  | EU Long Range, LR channel configuration 1         |
| 32  | `JP`               | 3                  | Japan                                             |
| 33  | `KR`               | 3                  | Korea                                             |
| 100 | `US_LR_920MHz`     | 4                  | US Long Range, LR channel configuration 2         |
| 101 | `EU_LR_866MHz`     | 4                  | EU Long Range, LR channel configuration 2         |
| 102 | `US_LR_END_DEVICE` | 2                  | US Long Range end device (LR channel configuration 3) |
| 103 | `EU_LR_END_DEVICE` | 2                  | EU Long Range end device (LR channel configuration 3) |

The region a zniffer uses before the host sends a Set Region command is implementation defined.

## Command ID's

| ID   | Command                                         |
| :--: | :---------------------------------------------- |
| 0x01 | [Get Version](#get-version-command)             |
| 0x02 | [Set Region](#set-region-command)               |
| 0x03 | [Get Regions](#get-regions-command)             |
| 0x04 | [Start](#start-command)                         |
| 0x05 | [Stop](#stop-command)                           |
| 0x0E | [Set Baud Rate](#set-baud-rate-command)         |
| 0x13 | [Get Region String](#get-region-string-command) |

### Get Version command

The Get Version command is used for getting the version of the zniffer firmware and the type and version of the chip it is running on.

#### Command format

| 0    | 1          | 2   |
| :--: | :--------: | :-: |
| 0x23 |    0x01    |  0  |

#### Command reply

| 0    | 1          | 2   | 3         | 4             | 5     | 6     |
| :--: | :--------: | :-: | :-------- | :------------ | :---- | :---- |
| 0x23 |    0x01    |  4  | Chip Type | Chip Revision | Major | Minor |

##### Chip Type

An identifier of the chip the zniffer runs on. The values are assigned by the chip vendor.

##### Chip Revision

The revision of the chip the zniffer runs on.

##### Major and Minor

The major and minor version of the zniffer firmware.

### Set Region command

The Set Region command is used to set the region the zniffer should listen on.

#### Command format

| 0    | 1          | 2   | 3      |
| :--: | :--------: | :-: | :----- |
| 0x23 |    0x02    |  1  | Region |

##### Region

The region to use. See [Region identifiers](#region-identifiers). A list of supported regions can be fetched using the Get Regions command.

A zniffer may store the region in non-volatile memory, so it persists across resets.

The host must only send region identifiers returned by the Get Regions command. The behavior for any other value is undefined.

#### Command reply

None

### Get Regions command

The Get Regions command is used to get the current region and the list of regions supported by the zniffer.

#### Command format

| 0    | 1          | 2   |
| :--: | :--------: | :-: |
| 0x23 |    0x03    |  0  |

#### Command reply

| 0    | 1          | 2     | 3              | 4..(2+N)          |
| :--: | :--------: | :---: | :------------- | :---------------- |
| 0x23 |    0x03    | N     | Current Region | Supported Regions |

##### Current Region

The region the radio is currently running. Like the Region field in radio frames, this is a standard Z-Wave region identifier (0 - 33), so a zniffer set to region 100 - 103 reports 9 or 11 here.

##### Supported Regions

One byte per supported region identifier, so the number of supported regions is N - 1. The order is implementation defined.

### Start command

The Start command is used to start receiving and forwarding radio frames.

#### Command format

| 0    | 1          | 2   |
| :--: | :--------: | :-: |
| 0x23 |    0x04    |  0  |

#### Command reply

| 0    | 1          | 2   |
| :--: | :--------: | :-: |
| 0x23 |    0x04    |  0  |

The zniffer sends the reply before it starts the radio receiver.

### Stop command

The Stop command is used to stop receiving radio frames.

#### Command format

| 0    | 1          | 2   |
| :--: | :--------: | :-: |
| 0x23 |    0x05    |  0  |

#### Command reply

| 0    | 1          | 2   |
| :--: | :--------: | :-: |
| 0x23 |    0x05    |  0  |

The zniffer sends the reply, then powers down the radio.

### Set Baud Rate command

The Set Baud Rate command is used to set the UART baud rate.

#### Command format

| 0    | 1          | 2   | 3         |
| :--: | :--------: | :-: | :-------- |
| 0x23 |    0x0E    |  1  | Baud Rate |

##### Baud Rate

| Value | Baud rate |
| :---: | :-------- |
|   0   | 115200    |
|   1   | 230400    |

#### Command reply

If Baud Rate is 0 or 1:

| 0    | 1          | 2   |
| :--: | :--------: | :-: |
| 0x23 |    0x0E    |  0  |

For any other value, the zniffer sends no reply.

### Get Region String command

The Get Region String command is used to get the name and number of channels of a region.

#### Command format

| 0    | 1          | 2   | 3      |
| :--: | :--------: | :-: | :----- |
| 0x23 |    0x13    |  1  | Region |

##### Region

The region identifier to describe. See [Region identifiers](#region-identifiers).

#### Command reply

If the region is supported:

| 0    | 1          | 2   | 3      | 4                  | 5..(2+N)    |
| :--: | :--------: | :-: | :----- | :----------------- | :---------- |
| 0x23 |    0x13    | N   | Region | Supported Channels | Region Name |

If the region is not supported:

| 0    | 1          | 2   |
| :--: | :--------: | :-: |
| 0x23 |    0x13    |  0  |

##### Supported Channels

The number of channels in the region.

##### Region Name

The name of the region as ASCII characters, for example `US_LR_912MHz`. The name is not null terminated; its length is N - 2.

# Resources

- Conversion of Markdown to Sphinx: pandoc input.md -f markdown -t rst -o output.rst
