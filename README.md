# Photon Feeder Bus Protocol

## Packet Framing

The photon bus does not use any explicit framing protocol with recovery
mechanisms like [HDLC](https://en.wikipedia.org/wiki/High-Level_Data_Link_Control).
Instead participants should maintain a buffer of received bytes and sucessivly 
attempt to decocde packets with each byte.

The Marlin firmware the implements the `M485` command has a maximum buffer
size of `32` in the [pull request](https://github.com/MarlinFirmware/Marlin/pull/25680/files#diff-63080d1f1d3cd04a0f8e73e6402f3123060692cae1e0be874c901ab90576c1cfR32)

## Packet Header and Format


| Offset | Field         | Description                                              |
| ------ | ------------- | -------------------------------------------------------- |
|   0    | `src_addr`    | Source Address                                           |
|   1    | `dest_addr`   | Destination Address                                      |
|   2    | `packet_id`   | Arbitray packet assigned by reqestor.  Repsonses should copy this ID. |
|   3    | `payload_len` | Payload length excluding this header                     |
|   4    | `crc`         | CRC of the payload entire packet excluding the crc field |

## CRC

The CRC is an 8 bit CRC with the polinomial `0x1070`.  It is calcuated with 
the following C code:

```c
uint8_t crc8(uint8_t data, size_t len) {
    uint32_t crc = 0;

    for (size_t i = 0; i < len; i++) {
        crc ^= (data << 8);
        for (size_t bit_n = 0; bit_n < 8; bit_n++) {
            if (crc & 0x8000) {
                crc ^= (0x1070 << 3);
            }
            crc <<= 1;
        }
    }

    return (uint8_t)(crc >> 8);
}
```

## Status Codes

([source](https://github.com/photonfirmware/photon/blob/main/photon/src/PhotonFeederProtocol.cpp#L8))

| Value  | Name                   | Notes                                  |
| ------ | ---------------------- | -------------------------------------- |
| `0x00` | `OK`                   |                                        |
| `0x01` | `WRONG_FEEDER_ID`      |                                        |
| `0x02` | `COULDNT_REACH`        | Reach What?                            |
| `0x03` | `UNINITIALIZED_FEEDER` |                                        |
| `0x04` | `FEEDING_IN_PROGRESS`  |                                        |
| `0x05` | `FAIL`                 | Failure of what?                       |
| `0xFE` | `TIMEOUT`              | Does this ever get sent over the wire? |
| `0xFF` | `UNKNOWN`              |                                        |

## Commands

| Value  | Addressing | Description                   |
| ------ | ---------- | ----------------------------- |
| `0x01` | Unicast    | Get Feeder ID                 |
| `0x02` | Unicast    | Initialize Feeder             |
| `0x03` | Unicast    | Get Version                   |
| `0x04` | Unicast    | Move Feed Forward             |
| `0x05` | Unicast    | Move Feed Backward            |
| `0x06` | Unicast    | Move Feed Status              |
| `0xbf` | Unicast    | Vendor Options                |
| `0xc0` | Broadcast  | Get Feeder Address            |
| `0xc1` | Broadcast  | Identify Feeder               |
| `0xc2` | Boradcast  | Program Feeder Floor          |
| `0xc3` | Broadcast  | Uninitialized Feeders Respond |

### `0x01` - Get Feeder ID

#### Addressing
Unicast

#### Payload
None

#### Response
| Offset | Field    | Description |
| ------ | -------- | ----------- |
| 0      | `status` | `OK` |
| 1..12  | `uuid`   | 12 byte UUID |

#### Description

Feeder responds with its unique ID.  On the lumen feeders this comes from the
unique ID build into the STM32 chip.  There is no provision to divide this space
to avoid colisions with other unique ID sources.

### `0x02` - Initialize Feeder

#### Addressing
Unicast

#### Payload
| Offset | Field  | Description  |
| ------ | ------ | ------------ |
| 0..11  | `uuid` | 12 byte UUID |

#### Response
| Offset | Field    | Description |
| ------ | -------- | ----------- |
| 0      | `status` | `OK` or `WRONG_FEEDER_ID` |
| 1..12  | `uuid`   | 12 byte UUID |

#### Description

In the Photon firmware, this checks that the UUID is correct and sets its
internal `initialized` flag to `true`.

### `0x03` - Get Version

#### Addressing
Unicast

#### Payload
None

#### Response
| Offset | Field     | Description                              |
| ------ | --------- | ---------------------------------------- |
| 0      | `status`  | `OK`                                     |
| 1      | `version` | Max protocol version supported by feeder |

#### Description

Query the protocol version supported by this feeder.  At time of writing this
is `1`.

### `0x04` - Move Feed Forward

#### Addressing
Unicast

#### Payload
| Offset | Field      | Description               |
| ------ | ---------- | ------------------------- |
| 0      | `distance` | Distance to move (units?) |

#### Response
| Offset | Field                | Description       |
| ------ | -------------------- | ----------------- |
| 0      | `status`             | `OK`              |
| 1..2   | `expected_feed_time` | units? Big Endian |

#### Description

Advance the feeder forward by `distance` (units?) and respond with the expected
time needed to advance (units?).

### `0x05` - Move Feed Backwared

#### Addressing
Unicast

#### Payload
| Offset | Field      | Description               |
| ------ | ---------- | ------------------------- |
| 0      | `distance` | Distance to move (units?) |

#### Response
| Offset | Field                | Description       |
| ------ | -------------------- | ----------------- |
| 0      | `status`             | `OK`              |
| 1..2   | `expected_feed_time` | units? Big Endian |

#### Description

Advance the feeder backward by `distance` (units?) and respond with the expected
time needed to advance (units?).

### `0x06` - Move Feed Status

#### Addressing
Unicast

#### Payload
None

#### Response
| Offset | Field                | Description                         |
| ------ | -------------------- | ----------------------------------- |
| 0      | `status`             | `OK`, `COULDNT_REACH`, or `UNKNOWN` |

#### Description

Query the state of the last feed request.  This may return the following 
statuses:

| Status | Meaning                                                          |
| ------ | ---------------------------------------------------------------- |
| `OK`   | Feed completed sucessfully                                       |
| `COULDNT_REACH` | Either the feeder failed to move by the requested `distance` or the `distance` was invalid |
| `UNKNOWN` | Either the feeder is still feeding or an unkown error occured |

### `0xbf` - Vendor Options

#### Addressing
Unicast

#### Payload
Implementation Dependent

#### Response
Implementation Dependent?  Photon sends no response

#### Description

Vendor specific command.   Photon expects 20 bytes and does nothing with them
and does not respond.

### `0xc0` - Get Feeder Address

#### Addressing
Broadcast

#### Payload
| Offset | Field  | Description  |
| ------ | ------ | ------------ |
| 0..11  | `uuid` | 12 byte UUID |

#### Response
| Offset | Field    | Description  |
| ------ | -------- | ------------ |
| 0      | `status` | `OK`         |


#### Description

If the feeder's UUID matches the payload's it will respond with `OK`.  Otherwise
the packet is ignored.  Since the feeder will respond with it's own `src_addr`,
This is a way of querying into which slot feeder with a given UUID is installed.

### `0xc1` - Identify Feeder

#### Addressing
Broadcast

#### Payload
| Offset | Field  | Description  |
| ------ | ------ | ------------ |
| 0..11  | `uuid` | 12 byte UUID |

#### Response
| Offset | Field    | Description  |
| ------ | -------- | ------------ |
| 0      | `status` | `OK`         |

#### Description

Ask the feeder with the matching UUID to identify itself.  The Photon feeer
does this visually by flashing it's RGB LED which 3 times in rapid sucession.

### `0xc2` - Program Feeder Floor

#### Addressing
Broadcast

#### Payload
| Offset | Field  | Description                            |
| ------ | ------ | -------------------------------------- |
| 0..11  | `uuid` | 12 byte UUID                           |
| 12     | `addr` | Address to program to the feeder floor |

#### Response
| Offset | Field    | Description  |
| ------ | -------- | ------------ |
| 0      | `status` | `OK`, `FAIL` |

#### Description

Programs the feeder floor slot at that the matching feeder is plugged into with
the specified address.  This should only need to be done during initial feeder
floor setup.

### `0xc3` - Program Feeder Floor

#### Addressing
Broadcast

#### Payload
None

#### Response
| Offset | Field  | Description                            |
| ------ | ------ | -------------------------------------- |
| 0..11  | `uuid` | 12 byte UUID                           |

#### Description

All feeders which have not been initialized through the `Initialize Feeder`
command respond with their UUID.  It is unclear how the bus is arbitrated when
multiple uninitialized feeders are connected.

## References

* [RS485 Packet Library Photon Protocol](https://github.com/Jnesselr/RS485/blob/main/src/rs485/protocols/photon.cpp)
* [Marlin M485 Pull Request](https://github.com/MarlinFirmware/Marlin/pull/25680)
* [OpenPnP Photon Support in test branch](https://github.com/openpnp/openpnp/tree/test/src/main/java/org/openpnp/machine/photon/protocol)
* [Photon Feeder Firmware](https://github.com/photonfirmware/photon)
