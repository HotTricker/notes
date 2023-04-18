# Nginx

- [Nginx](#nginx)
  - [基本HTTP服务器功能](#基本http服务器功能)
  - [TCP/UDP代理服务器功能](#tcpudp代理服务器功能)
    - [Generic proxying of TCP and UDP](#generic-proxying-of-tcp-and-udp)
    - [SSL and TLS SNI support for TCP](#ssl-and-tls-sni-support-for-tcp)
    - [Load balancing and fault tolerance](#load-balancing-and-fault-tolerance)

## 基本HTTP服务器功能

pass

## TCP/UDP代理服务器功能

### Generic proxying of TCP and UDP

`proxy_bind` address \[transparent\] | off

- Makes outgoing connections to a proxied server originate from the specified local ip address
- Parameter value can contain variables
- "off" cancels the effect of the porxy_bind directive inherited from the pervious configuration level, which allows the system to auto-assign the local ip address
- "transparent" allows connections from a non-local ip address
- Usually necessary to run niginx with superuser privileges, but not required in linux
- Configure routing table to intercept network traffic

`proxy_buffer_size` size

- Default: 16k

`proxy_connect_timeout` time

- Default: 60s

`proxy_download_rate` rate

- Default: 0
- Limits the speed of reading the data from the proxied server(b/s)
- The zero value disables rate limiting
- The limit is set per a connection, if opens two connection, the overall rate will be twice as much as the specified limit
- Parameter value can contain variables
  
`proxy_half_close` on|off

- Default: off
- Enables of disables closing each direction of a TCP connection independently. If enabled, proxying over TCP will be kept until both sides close the connection

`proxy_next_upstream` on|off

- Default: on
- When a connection to the proxied server cannot be established, determines whether a client connection will be passed to the next server
- can be limited by the number of tries `proxy_next_upstream_tries` and by time `proxy_next_upstream_timeout`

`proxy_pass` address

- Uses domain name or IP address and a port: `proxy_pass localhost:12345;`
  - If a domain name resolves to several addresses, all of them will be used in a round-robin fashion. In addition, an address can be specified as a server group
- Uses a UNIX-domain socket path: `proxy_pass unix:/tmp/stream.socket;`
- Uses variables
  - The server name is searched among the described server groups, and if not found,it determined using a resolver.

`proxy_protocol` on|off

- Default: off
- Enables the proxy protocol for connections to a proxied server

`proxy_requests` number

- Default: 0
- Sets the number of client datagrams at which binding between a client and existing UDP stream session is dropped. After receiving the specified number of datagrams, next datagram from the same client starts a new session. The session terminates when all client datagrams are transmitted to a proxied server and the expected number of responses is received, or when it reaches a timeout.

`proxy_responses` number

- Sets the number of datagrams expected from the proxied server in response to a client datagram if the UDP protocol is used. The number serves as a hint for session termination.By default, the number of datagrams is not limited.
- If zero value is specified, no response is expected. However, if a response is received and the session is still not finished, the response will be handled.

`proxy_session_drop` on|off

- Default: 0
- Enables terminating all sessions to a proxied server after it was removed from the group or marked as permanently unavailable. This can occur because of re-resolve or with the API DELETE command. A server can be marked as permanently unavailable if it is considered unhealthy or with the API PATCH command. Each session is terminated when the next read or write event is processed for the client or proxied server.

`proxy_socket_keepalive` on|off

- Default: off
- Configures the "TCP keepalive" behavior for outgoing connections to a proxied server. By default, the operating system's settings are in effect for the socket. If the directive is set to the value "on", the SO_KEEPALIVE socket option is turned on for the socket

`proxy_timeout` timeout

- Default: 10m
- Sets the timeout between two successive read or write operations on client or proxied server connections. If no data is transmitted within this time, the connection is closed.

`proxy_upload_rate` rate

- Default: 0
- Limits the speed of reading the data from the client. The rate is specified in bytes per second. The zero value disables rate limiting. The limit is set per a connection, so if the client simultaneously opens two connections, the overall rate will be twice as much as the specified limit.
- Parameter value can contain variables. It may be useful in caseswhere rate should be limited depending on a certain condition.

### SSL and TLS SNI support for TCP

`proxy_ssl` on|off

- Default: off
- Enables the SSL/TLS protocol for connections to a proxied server.

`proxy_ssl_certificate` file

- Specifiles a file with the certificate in the PEM format used for authentication to a proxied server.
- Since version 1.21.0, variables can be used in the file name.

`proxy_ssl_certificate_key` file

- Specifies a file with the secret key in the PEM format used for authentication to a proxied server.
- Since version 1.21.0, variables can be used in the file name.

`proxy_ssl_ciphers` ciphers

- specifiles the enabled ciphers for connections to a proxied server.The ciphers are specified in the format understood by the OpenSSL library.

`proxy_ssl_conf_command` name value

- Sets arbitrary OpenSSL configuration commands when establishing a connection with the proxied server.
- Serveral proxy_ssl_conf_command directives can be specified on the same level. These directives are inherited from the previous configuration level if and only if ther are no proxy_ssl_conf_command directives defined on the current level.

`proxy_ssl_crl` file

- Specifies a file with revoked certificates in the PEM format used to verify the certificate of the proxied server.

`proxy_ssl_name` name

- Allows overriding the server name used to verify the certificate of the proxied server and to be passed through SNI when establishing a connection with the proxied server. The server name can also be specified using variables.
- By default, the host part of the proxy_pass address is used.

`proxy_ssl_password_file` file

- Specifies a file with passphrases for secret keys where each passphrase is specified on a separate line.Passphrases are tried in turn when loading the key.

`proxy_ssl_protocols`

- Enables the specified protocols for connections to a proxied server.

`proxy_ssl_server_name` on|off

- Enables or disables passing of the server name throush TLS Server Name Indication extension when establishing a connection with the proxied server.

`proxy_ssl_session_reuse` on|off

- Default: on
- Determines whether SSL sessions can be reused when working with the proxied server. If the errors appear in the logs, try disabling session reuse

`proxy_ssl_trusted_certificate` file

- Specifies a file with trusted CA certificates in the PEM format used to verify the certificate of the proxied server

`proxy_ssl_verify` on|off

- Default: off
- Enables of disables verification of the proxied server certificate

`proxy_ssl_verify_depth` number

- Default: 1
- Sets the verification depth in the proxied server certificates chain

### Load balancing and fault tolerance

`upsteram`
