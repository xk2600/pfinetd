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
 0. The child has sent the 'PFM?\n' blurb to the server.
 1. The server immediately sends 'PFM/1.0 200 OK' to the child with supported options which can be set as named-value pairs.
 2. The child can choose to either ignore or set any attributes with an option response.
 3. The server acknowledges any options set in the next message to the child.
 4. The child then waits for incoming connection information to be sent over the stdin channel.
 
## Syntax

### Child Methods
 * PFM?: Promote to pfinetd mode.
 * 

### Server Methods
 * 

### Options
 * NAME: Server Name (Prepended to log entries)
 * AF_INET: List of Internet Socket Types Supported. [ SOCK_STREAM | SOCK_DGRAM | SOCK_RAW ]
 * SERVICES: Service names from /etc/services to accept connections on. (Port)
 * EOM: End of message character sequence for overriding mechanics.
 * BUFFER: Maximum buffer length.
 * BINARY: Sets Binary Mode.
 * 
 
*As the connection state is negotiated, any pf-mode enabled server still works with a legacy implementation of inetd as the exchange is ignored and results in a single syslog message containing 'PFM?' at startup.*
