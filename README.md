## Top Level actions
   1. add connection
      Used to open an new connection to a serial port. The detailed parameter description is further below. Each connection needs to have a unique idetifying name. The connection will show up as a metric below Links root node. The action will return a boolean result and an additional message in case of an error. The connection node will only be created, if the action was sucessfull and the result value is true.
      Right after the connection is created the Link will try to read incoming data accoring the given parameter description. The value of the link is the last recevied incoming data. Each connection node has two additional sub metrics describing the current state of the connection. The Status submetrics gives information, wether the connection to the serial device could be established or is closed.The LastError submetric gives information about the last error, which occured on this connection. In case of read error the connection normally get closed. In case of write errors the connection might not be closed.
   1. add control port
      This action is used to open a kind of special handling for a serial device. In this mode the port may not be used to read or write data, but directly read and modify the input/output pins of the serial port. This way the serial port might be used as a kind of poor man GPIO. The parameter description for this action method is further below. The return types and submetrices  of this action are the same as with the 'add connection' action.

## Add connection parameter description
  * Name: The unique identifying name of the connection. The connection will show up with a node of this name in the link. If a connection with the same name already exists, the action will return an error and no node will be created.
  * Serial Port: The serial port device to be used for this connection (e.g. /dev/ttyS0). If the device does not exist the node will still be created. But the Status of this connection will be "Failed to connect". This way the behavior is consistent with the behavior on link restart and the device was unplugged in between
  * Baud Rate: The Baud rate to be used on this serial connection. Has to match the device baud rate.
  * Data Bits: The number of data bits to be used for this connection. 
  * Szop Bits: The number of stop bits to use for this connection
  * Parity: The parity to be used for this connection
  * Message Type: Select, if the received message is expected to be a binary value or as a text value. In case the type is binary value the optional parameter 'Message Format'may be used to decode complex message into JSON objects. Binary messages without a message format will be published as binary data. Message of type text will be published as string values.
  * Message Size: If the sender send fixed size message you might configure the size of the incoming messages here. If no start and end code is configured the DSLink will try to read the given number of bytes as one incoming message. This parameter might be combined with the 'Start Code' parameter. In this case the 'Message Size' should include the size of the start code. The DSLink will scan the incoming data for the start code first. Once the DSLink found the start code the remaining bytes of the message will be read.
  * Message Format: For parsing binary data the DSLink contains a simple binary data decode. This optional parameter describes how to parse the incoming binary data. If no message format description is provided the DSLink will publish the raw binary data blob. If a message format description is specified the DSLink will decode the incoming data and publish a JSON object. the  The format description itself is a JSON object. The details for format description can be found in a chapter further below.
  * Start Code: If a start code is configured the DSLink will always scan for the start code in the incoming data. 
    All data prior to the start code will be ignored. This parameter has to be combined either with the 'End Code' parameter or with the 'Message Size' parameter. Depending on the configuration the DSLink will eihter try to read until he finds the end code or will try to read a fixed message size. For sending of binary data see the string parameter description below.
  * End Code: If an end code is configured the messages may have variable size. This parameter has to be combined with the 'Start Code' parameter. For sending of binary data see the string parameter description below.

## Send Message action

On each connection a message may be send using the 'send message' action associated with this action. The send message has just a single parameter:
  * Message: The message to be send to be send. To send binary data or other non printable data please see the String parameter description below.  

## Remove action

Each connection has a 'remove' action used to close and remove the defined connection.

## String parameter description

The string parameters 'Start Code', 'End Code' and the 'Message' parameter are by defaout interpreted as string text messages. But depending on the first characters of the parameter non printable (e.g. binary) data may be used. The currently supported modifiers are:
  * If the parameter starts with a 0x modifier the remaining parameter is a hex dump of the search sequence (e.g. 0x0a0d will scan for a LF followed by a CR)
  * If the parameter starts with a ' (a single quote) modifier the remaining parameter is used literally (e.g. '0x will scan for the character sequence 0x)

##  Message Format description

The message format description is itself a JSON object. With this object the mapping from the incoming binary data to the 
structured result data is described. A simple example description might lokk loke this:
```
{
    "byteorder": "lowhigh",
    "type": "struct",
    "elements": [
        {
            "name": "evt_num",
            "type": "uint16",
            "offset": 0
        },
        {
            "name": "status",
            "type": "struct",
            "offset": 2,
            "elements": [
                {
                    "name": "blasting",
                    "type": "boolean",
                    "bits": 0
                },
                {
                    "name": "unloading",
                    "type": "boolean",
                    "bits": 1
                },
                {
                    "name": "conv",
                    "type": "enum",
                    "bits": "5-6",
                    "values": {
                        "Closed Loop": 0,
                        "Open Loop": 1,
                        "Manual": 2,
                        "Wt/Area": 3
                    }
                }
        },
        {      
            "name": "PavTemp",
            "type": "int8",
            "offset": 3
        }
   ]
}
```
Currently the following result datatypes are supported:
* struct
* uint8
* uint16
* int8
* boolean 
* enum
* string

There are some object memeber, which are common to all datatypes:
* name: Name of the element in the result structure. For struct datatypes the names of the direct children have to be unique.
* type: The datatype of this element
* offset: The offset of this element is relative to start the surrounding struct. For the top level struct the offset is relative to the beginning of the message. 
* byteorder: This member is used in all multibyte numerical values. The parameter describes the byte order of the incoming bytes on the connection. 'lowhigh' means on the wire the low bytes is transferred first and the high byte is the second byte to be received. 'highlow' means the high byte is recieved as the first byte and the low bytes is received as the scond byte on the connection. This setting will be propagate to all nested data elements.  If the whole message uses the same byte order (which normally is the case) its sufficient to specify the byte order once in the top level struct descripion.

### struct
The struct datatype is the basic type to build up complex data structures. The top level data structure does not need to have a name. The main object member inside the struct is the elements member:
* elements: Array of json objects describing the elements of the struct.

### uint8, uint16, int8
Numeric datatypes, which will be decoded at the given offset with the given byte order.

### boolean
By default the byte at the offset position will be decoded. In case the byte is equal to zero, the boolean result will be 
false. In all other cases the boolean result will be true. To support compact storing of boolean values in the binary stream the boolean data types supports an optional description member:
* bits: Specify a single bit number or a range of bits to use as a filter mask on the byte. Only in case all bits in the given range (resp. the single bit) are equal to zero the result boolean will be false.

### enum
With the enum data type binary values may be mapped to string values in the result object. To achieve this the datatype supports the following description member:
* values: This is a json object describing the values of the enum. The member names are the text values used in the result structure. The numerical value of the member are the binary values in the incoming data.
* bits: Specify the range of bits to be used to fetch the values of the incoming data. The numeric values are shifted to the value range of the number of bits. E.g. with 'bits':'3-4' the range of the numeric value will be 0..3.

### string
Fixed length string. If the string contains a zeor byte the string will be cut off at this position. The maximal length of this string is specified as:
* length: The number of bytes to fetch from the input data stream

# NPDESI
# NPDESI
