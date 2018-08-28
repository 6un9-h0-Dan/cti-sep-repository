# SEP Name
Modbus

## SEP Identifier
`x-newcontext-modbus-ext`

## SEP Version
1

## SEP Description
Allows for characterizing SCADA protocol [Modbus](https://en.wikipedia.org/wiki/Modbus) Network Traffic.

## SEP Use Cases
* Support common ICS/SCADA Protocol.
* Allow utilities to describe and detect possible attacks.

## SEP Extension Context
This is an extension to the `network-traffic` SCO object.

## SEP Definition
This is designed for Modbus/TCP.  It does not address Modbus over other transports, such as serial.

The properties below are extracted from the standard.  If the packet is non-standard, then it is expected that an artifact of the payload will be provided, and these properies will not be used.

The is_query property is optional, but it is highly recommended it be used.  In Modbus/TCP, this may be difficult, but it is likely easier that the decoder will have a better idea of which side is which.

Currently, only the most common function codes are fully broken out.  This does not mean that support for the more rare codes should be added.

In the case of descrete inputs or read/write coils, the nregs will be 8 times the length.

XXX - (do we support short nregs for when the decoder can tie the response to the query that has the actual count?)

## SEP Sponsors
Org | Primary Contact
--- | ---------------
New Context | John-Mark Gurney

## POC Implementations
Org | GitHub Repository
--- | -----------------
New Context | TBD

## Properties
| Property Name               | Type                | Description |
| -------------               | ----                | ----------- |
| **excpt_code** (required)   | `integer`           | The exception code if the function code is >= 128. |
| **func_code** (required)    | `integer`           | The function code from the frame. |
| **is_query** (optional)     | `boolean`           | Is this frame a query for data (true), or a response to a query (false).  It is recommended that this be present. |
| **prot_id** (required)      | `integer`           | The protocol identifier for Modbus. |
| **nregs** (required)        | `integer`           | The number of registers queried, or the number of registers requested.  If this is a response, it contains the length of the list of registers. |
| **registers** (optional)    | `list` of `integer` | If the frame is a response (**is_query** is false), then this property **MUST** be present, and contain **nregs** `integer`s. |
| **start_addr** (optional)   | `integer`           | If the frame is a request (**is_query** is true), then this property **MUST** be present, and contain the address of the request. |
| **tx_id** (required)        | `integer`           | The transaction identifier. |
| **unit_id** (required)      | `integer`           | The unit identifier from the frame. |
