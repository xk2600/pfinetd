
# pfinetd
A protocol super-server.

## Purpose
 * FastCGI type interface for inetd.
 * Sockets (Streams/Datagrams) are bound and forwarded by pfinetd. (Future support for sctp.)
 * pfinetd starts child process(es).
 * Socket/Datagram RX -> child stdin.
 * Socket/Datagram TX <- child stdout.
 * event logging <- child stderr.
 
## HOW IT WORKS

### pfinetd startup
 1. On startup, pfinetd reads /etc/inetd.conf and/or pfinetd.conf.
 2. pfinetd starts the process(es) in inetd-mode (using options defined in conf file).
 3. pfinetd waits for the child to write data to stderr for the startup-time (default:5 sec)
    *This gives the client process the opportunity to signal promotion to a persistent pfinetd child server.*
    *NOTE: This only happens the first time pfinetd is started, not everytime the child process is started*
 4. If pfinetd receives 'PFM?\n' on the child's stderr, the child's interaction switches to pf-mode.
 5. If the startup-time expires, the child's continues with the inetd-mode interaction.

### inetd-mode interaction
 1. The child stdin is blocked until a socket request comes in from the AF_INET stack.
 2. pfinetd blindly forwards packets between child stdin/stdout and RX/TX accordingly logging stderr to appropriate log.
 3. When the Child Process/Socket closes, inetd closes the Socket/Child Process in kind.
 4. pfinetd starts another process, immediately moving to inetd-mode interaction.

### pf-mode
 0. The child has sent the 'PF00\n' blurb to the server.
 1. The server immediately sends 'PFM/1.0 200 OK' to the child with supported options which can be set as named-value pairs.
 2. The child can choose to either ignore or set any attributes with an option response.
 3. The server acknowledges any options set in the next message to the child.
 4. The child then waits for incoming connection information to be sent over the stdin channel.
 
## Syntax

#### Abbreviations

|code|abbreviation             |    |code|abbreviation
|----|-------------------------|----|----|----------------------
|0x00|NUL: null                |    |0x10|DLE: data-link-escape
|0x01|SOH: start-of-header     |    |0x11|DC1/XON
|0x02|STX: start-of-text       |    |0x12|DC2
|0x03|ETX: end-of-text         |    |0x13|DC3/XOFF
|0x04|EOT: end-of-transmission |    |0x14|DC4
|0x05|ENQ: enquiry             |    |0x15|NAK: negative-acknowledgement
|0x06|ACK: acknowledgement     |    |0x16|SYN: synchronous idle
|0x07|BEL: bell-alert          |    |0x17|ETB: end-transmission-block
|0x08|BS:  backspace           |    |0x18|CAN: cancel
|0x09|HT:  horizontal-tab      |    |0x19|EM:  end-of-medium
|0x0A|LF:  line-feed           |    |0x1A|SUB: substitute
|0x0B|VT:  vertical-tab        |    |0x1B|ESC: escape
|0x0C|FF:  form-feed           |    |0x1C|FS:  file-separator
|0x0D|CR:  carriage-return     |    |0x1D|GS:  group-separator
|0x0E|SO:  shift-out           |    |0x1E|RS:  record-separator
|0x0F|SI:  shift-in            |    |0x1F|US:  unit separator


#### Control Packet

> | `SYN`  `SOH` | `tlvcount` | ...  *`tlv-packets`* |
> |---------------|--------------|---------------------|

* The packet MUST contain at least 3 bytes.
* A control packet always starts with the same two bytes. The ANSI control character's SYN and SOH, bytes 0x16 and 0x01 respectively. If this check fails, the child process sends a to be determined TLV to terminate the connection and log to syslog.
* The `tlvcount` field provides the quantity of TLVs to expect in this control packet. (between 0 and 255)
* The `tlvpackets` field represents the actual TLVs.

_NOTE: It is completely reasonable to have tlvcount of 0. This is discussed further under the watchdog timer and keepalive functions of the protocol._


#### Type-Length-Value Packet
|type |length|value
|-----|------|-----
|8-bit|8-bit |variable

* _pfd: pseudo file descriptor - identifies the session. As attributes become available a pfd tlv will be transmitted with attributes about the remote client._


#### TLV Types

|c&leftrightarrow;s|code|name|description
|:----------------:|:---|:---|:----------
|&leftarrow;       |0x00|accept|accept the socket connection.
|&rightarrow;      |0x01|pfd|pseudo-file-descriptor
|&leftrightarrow;  |0x02|data|data-buffer-tlv
|&rightarrow;      |0x03|connect|connection request from connection-oriented (stream) based client.
|&rightarrow;      |0x04|message|message received from connectionless (datagram) based client.
|&leftarrow;       |0x10|filter|configure server to filter specific endpoints for a service.
|&leftrightarrow;  |0x11|preprocessor|specify preprocessor execution stack for inbound data.
|&leftarrow;       |0x12|altbind|allow termporal binding to an additional port.
|&leftrightarrow;  |0x81|failure|general failure.
|&leftrightarrow;  |0x82|malformed|notice that a preprocessing directive failed.
|&leftarrow;       |0x83|reject|tell Server to reject the connect.
|&leftrightarrow;  |0xFE|close|tell Server/Child to close the connection gracefully.

#### pfd tlv: (server &rightarrow; child)
> | `0x01` | `byte-length` | `pfd-data` |
> |--------|---------------|------------|
>
>  #### pfd-data:
>>  | `pfd` | `transmission-type` | `variable-data` |
>>  |-------|---------------------|-----------------|
>>  * pfd: psuedo-file-descriptor, index for specific remote endpoint. (4-bytes, signed 32-bit integer)
>>  * transmission-type: determines how messages will be exchanged.
>>  
>>    | code | transmission-type | description
>>    |------|-------------------|------------
>>    | 0x01 | stream            | sock_stream style interface, connection-oriented, in-order delivery.
>>    | 0x02 | datagram          | sock_dgram style interface, connectionless, may be limited on MTU unless willingness to use IP fragmentation is acceptable.
>>  
>> #### variable-data:
>>>  | `variable-name-length` | `variable-name` | `variable-content-length` | `variable-content` |
>>>  |------------------------|-----------------|---------------------------|--------------------|
>>>  * variable-name-length: length of variable name in bytes.
>>>  * variable-name: character-data representing variable-name.
>>>  * variable-content-length: length of variable content in bytes.
>>>  * variable-content: bytes representing variable content.
>>  
>>  #### Default Variables:
>>   
>>     RPORT: Remote Port
>>     RADDR: Remote Address
>>     LPORT: Local Port
>>     LADDR: Local Address
>>     
>>     SERVICE: Service Name (in /etc/services)
>>     

#### data tlv: (server &rightarrow; child)
> | `0x02` | `pfd` | `buffer-length` | `buffer-content` |
> |--------|-------|-----------------|-------------------|
> * pfd: pseudo-file-descriptor, index for specific remote endpoint. (4-bytes, signed 32-bit integer)
> * buffer-length: length in bytes of buffer-content.
> * buffer-contents: payload destined to/from client.


#### Preprocessor Operations

|OPCODE|OPERATION      |ALGORITHM 
|------|---------------|-----------
|0x0001|SCAN           |Pop argc, pop argv(argnames), pop scanstring, exec and push.
|0x0002|STRUCT         |pop argc, pop argv(argnames), apply struct, exec, and push.
|      |EQ, NE         |pop 2 elements, bottom element (equal to|notequal to) top element, 1 for true, 0 for false, push.
|      |GT, GE         |pop 2 elements, bottom element greater than (or equal to) top element, 1 for true, 0 for false, push.
|      |LT, LE         |pop 2 elements, bottom element less than (or equal to) top element, 1 for true, 0 for false, push.
|      |<<, >>         |pop `element`, pop `shiftlength`, shift `element` `shiftlength` bits, push.
|      |SUM            |pop `count`, pop `count` elements, sum elements, push.
|      |TOSTRING       |pop `formatstring`, search formatstring for `argc`, pop `argc` args, apply args to formatstring, push.
|      |STORE          |pop `varname`, pop `varvalue`, add variable `varname` and `varvalue` to varcache.
|      |               |


PF_CONNECT [LOCAL] [REMOTE] : Begin SOCK_STREAM connection oriented session
MESSAGE [LOCAL] [REMOTE] [LEN] [CONTENT] : SOCK_DGRAM notice.
MALFORMED []
ACK [

### Child Methods
 * PF : 'Magic' Promote to pfinetd mode.
 * SCAN [SCAN-STRING] {'ARG1','ARG2',: Submit scanf string to preprocess inbound header information. 
 * COMPLEX [COMPLEX] - Predefined iterative header processes.
 * ALTBIND : Ask for Ephemeral port for side-channel.
 * ACCEPT [ORIGIN] [REMOTE]

    __Method Arguments__
    
    __Complexes__
    METHOD-LINE [: 
    KEY-VALUE
    DOUBLE-NEWLINE
    NULL


### Options
 * NAME: Server Name (Prepended to log entries)
 * AF_INET: List of Internet Socket Types Supported. [ SOCK_STREAM | SOCK_DGRAM | SOCK_RAW ]
 * SERVICES: Service names from /etc/services to accept connections on. (Port)
 * EOM: End of message character sequence for overriding mechanics.
 * BUFFER: Maximum buffer length.
 * BINARY: Sets Binary Mode.


## Protocol

#### Rules:
 * Watchdog timer : If the server/child has not transmitted a packet at halfway to the watchdog timer expiring, transmit a null payload to keep the server/child aware of each other's state . If either does not receive a packet by the time the watchdog timer exires log a WARNING message direct to syslog. When the server finds that it has lost the ability to service requests through the child, terminate the child. When the Child finds that it has lost the ability to communicate with the server, it should close all file descriptors (stdin,stdout,stderr) and terminate itself.
 
*As the connection state is negotiated, any pf-mode enabled server still works with a legacy implementation of inetd as the exchange is ignored and results in a single syslog message containing 'PFM?' at startup.*



>    `**watchdog packet:** _(SYN, SOH, tvlcount: 0 tlvs)_
>    
>    `[0x16|0x01|0x00]`
>    
>* **data only:** _(SYN, STX, pfd: 1, datalen: 16 bytes, data, ENQ)_
>    
>    `[0x16|0x02|0x00000001|0x0010|...data...|0x05]`
>    
>`* **acknowledgement only:** _(SYN, ACK, pfd: 1)_
>    
>    `[0x16|0x06|0x00000001]`


