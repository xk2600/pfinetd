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

#### Control Packet

|SYN |SOH | tlvcount | ...tlvs... | datalen | data
|----|----|----------|-----/ /----|---------|----------
|0x16|0x01| uint8    | ...Var...  | 16-bit  | var.

_NOTE: Maximum buffer length is 256 TLVs + 65KB _

#### Type-Length-Value Packet
|type |length|value
|-----|------|-----
|8-bit|8-bit |8-bit


#### Acknowledgement

| SYN  | SOH  |
|------|------|
| 0x16 | 0x01 |


#### 

_Note: Tokens are signed and should increase to 255 and then return to one. This provides a sanity check. If a token ever goes negative we should bail out._

| SINT16 | SERVER METHODS | Description
|--------|----------------|------------
| 0x0000 | NULL           | Used as a watchdog timer. 

| 0x0016 | SYN            | 
| 0x0002 | CONNECT        |
| 0x0003 | MESSAGE        |
| 0x0082 | ACCEPT         |
| 0x0006 | ACKNOWLEDGE    | Acknowledge request defined by token.
| 0x     |                |
| 0x5046 | PF             |
| 0x8001 | FAILURE        |
| 0x     | MALFORMED      | Notice that a preprocessing directive failed.
| 0x     |                |
| 0x     | ALTBIND        | Allow termporal binding to an additional port.
| 0x     | ACCEPT         | Accept the socket connection.
| 0x     | REJECT         | Tell Server to reject the connect
| 0x     | CLOSE          | Tell Server/Child to close the connection
| 0x     |                |
| 0x     | PREPROCESSOR   | Specify preprocessor execution stack for inbound data

#### Preprocessor Operations

| OPCODE | OPERATION      | ALGORITHM 
|--------|----------------|-----------
| 0x0001 | SCAN           | Pop argc, pop argv(argnames), pop scanstring, exec and push to stack.
| 0x0002 | STRUCT         | pop argc, pop argv(argnames), apply struct, exec, and push to stack.


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
