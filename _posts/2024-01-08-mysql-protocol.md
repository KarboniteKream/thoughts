---
layout: post
title: Exploring the MySQL protocol
date: 2024-01-08 21:00:00 +0900
---

Early last year, I was looking to replace the deprecated [`r2dbc-mysql`][r2dbc-mysql] driver in one
of our codebases, when I stumbled upon [`jasync-sql`][jasync-sql]. While working on support for
additional authentication methods, I decided to investigate what the MySQL protocol looks like, and
what bytes get transferred over the network.

Inspired by [The Illustrated TLS 1.2 Connection][tls-exchange], this blog post describes the entire
connection phase to a MySQL server, and the execution of a simple `SELECT` query, documenting
individual bytes of each network packet. The following packets were analyzed while running MySQL
server version 8.0.32.

The [MySQL Source Code Documentation][mysql-protocol] visualizes the connection lifecycle as
follows:

{:.center}
![MySQL connection flow](/assets/images/mysql-connection.svg)

Since the replication mode is not relevant to us, we'll only focus on the connection and command
phases. Let's proceed!

ℹ️ While they have been omitted in the following examples for brevity, each MySQL packet has a
4-byte header, which includes the payload length and a sequence ID. You can find the details
[here][mysql-packet].

### 🤝 Connection phase
During the connection phase, we have to exchange the information about supported features and 
character encoding, so future packets can be adjusted. For example, the capability flags indicate
which MySQL protocol versions are supported. A detailed description of this phase can be found
[here][connection-phase].

After the connection has been established, the MySQL server initiates the communication by sending a
[`HandshakeV10` request][handshake-v10]:
```
0a                      | Protocol version (10)
38 2e 30 2e 33 32 00    | Server version ("8.0.32")
08 00 00 00             | Connection/thread ID (8)
1c 46 19 46 59 76 40 4b | Scramble (first 8 bytes)
00                      | Filler (NULL)
ff ff                   | Server capability flags (lower 2 bytes)
ff                      | Character set (lower 1 byte, "utf8mb4_0900_ai_ci")
02 00                   | Status flags
ff df                   | Server capability flags (upper 2 bytes)
15                      | Length of combined scramble (total_length)
00 00 00 00 00 00 00 00 | Reserved (10 bytes)
00 00                   ¦
3f 71 34 71 53 45 5e 5d | Remaining bytes of the scramble.
22 7a 32 3d 00          ¦ Length must equal max(13, total_length - 8)
63 61 63 68 69 6e 67 5f | Authentication method ("caching_sha2_password")
73 68 61 32 5f 70 61 73 ¦
73 77 6f 72 64 00       ¦
```

With this request, the server announces which features it supports with the
[capability flags][capability-flags] field, so it's now the client's turn to do the same. Since the
server supports SSL (the `CLIENT_SSL` capability flag is set), the client can upgrade to a secure
connection by sending an [`SSLRequest`][ssl-request]:
```
08 aa 0a 00             | Client capability flags
ff ff ff 00             | Maximum packet size (16,777,215)
e0                      | Character set (lower 1 byte, "utf8mb4_unicode_ci")
00 00 00 00 00 00 00 00 | Filler (23 bytes)
00 00 00 00 00 00 00 00 ¦
00 00 00 00 00 00 00    ¦
```

Once the connection has been upgraded (including a full [TLS exchange][tls-exchange]), the client
proceeds with sending a [`HandshakeResponse41`][handshake-response-41]:
```
08 aa 0a 00             | Client capability flags
ff ff ff 00             | Maximum packet size (16,777,215)
e0                      | Character set (lower 1 byte, "utf8mb4_unicode_ci")
00 00 00 00 00 00 00 00 | Filler (23 bytes)
00 00 00 00 00 00 00 00 ¦
00 00 00 00 00 00 00    ¦
72 6f 6f 74 00          | Username ("root")
20                      | Authentication response length (32)
38 24 94 b7 75 30 09 4f | Authentication response.
7a a0 35 1e ee a1 3e b2 ¦ Contains the password scrambled with SHA-256
e5 fe 45 7f 1b b4 d9 40 ¦
72 54 d3 a9 93 da eb 55 ¦
74 65 73 74 00          | Database name ("test")
63 61 63 68 69 6e 67 5f | Authentication method ("caching_sha2_password")
73 68 61 32 5f 70 61 73 ¦
73 77 6f 72 64 00       ¦
```

This packet starts the same as the `SSLRequest` and is followed by plugin-specific authentication
data. The various supported authentication methods will be described in the next blog post, but for
now, we're focusing on the `caching_sha2_password` method, which is the default since MySQL 8.0.

When the server does not have the password cached, it asks the client to send the actual password
(full authentication mode) by sending an [`AuthMoreData` packet][auth-more-data]:
```
01                      | Status tag (1)
04                      | Payload (4 means "perform full authentication")
```

Since we're connected over SSL, the client can send an [`AuthSwitchResponse`][auth-switch-response]
with the plaintext password. If the SSL connection was not established, the client would have to
encrypt the password.
```
74 65 73 74 00          | Password ("test")
```

Once the server verifies the password and the client host's permissions, it responds with an
[`OK_Packet`][ok-packet] to complete the connection phase:
```
00                      | OK
00                      | Number of affected rows (0)
00                      | Last inserted ID (0)
02 00                   | Status flags (AUTO_COMMIT)
00 00                   | Warnings (none)
```

### 🗣️ Command phase
With the connection and protocol capabilities fully established, the server transitions into the
command phase, awaiting client requests. A detailed description of this phase can be found
[here][command-phase].

While the server supports various commands (see [this page][command-types]), we'll focus on the
query command, to send the following simple SQL query:
```sql
SELECT user, plugin
FROM mysql.user
WHERE CONCAT(user, '@', host) = CURRENT_USER();
```

The process is initiated by the client sending a [`COM_QUERY` command][com-query]:
```
03                      | Command type (COM_QUERY)
53 45 4c 45 43 54 20 75 | Query in plaintext:
73 65 72 2c 20 70 6c 75 ¦ SELECT user, plugin
67 69 6e 20 46 52 4f 4d ¦ FROM mysql.user
20 6d 79 73 71 6c 2e 75 ¦
73 65 72 20 57 48 45 52 ¦ WHERE CONCAT(user, '@', host)
45 20 43 4f 4e 43 41 54 ¦
28 75 73 65 72 2c 20 27 ¦
40 27 2c 20 68 6f 73 74 ¦
29 20 3d 20 43 55 52 52 ¦ = CURRENT_USER();
45 4e 54 5f 55 53 45 52 ¦
28 29 3b                ¦
```

Once the query execution is complete, the server sends a [`Text Resultset` response][result-set],
containing the following sequence of packets:
- a packet containing the number of columns
- packets with a definition for each column
- an EOF packet
- packets for each row in the query result
- an EOF packet

#### Columns
The first packet contains the column count, which is also the number of upcoming packets:
```
02                      | Number of columns (2)
```

Next are the [`ColumnDefinition41` packets][column-definition-41] for each column in the query
result:
```
03 64 65 66             | Catalog (always "def")
05 6d 79 73 71 6c       | Schema name ("mysql")
04 75 73 65 72          | Virtual table name ("user")
04 75 73 65 72          | Physical table name ("user")
04 75 73 65 72          | Virtual column name ("user")
04 55 73 65 72          | Physical column name ("User")
0c                      | Length of the fixed-length fields (always 12)
e0 00                   | Character set ("utf8m4_unicode_ci")
80 00 00 00             | Maximum column length (128)
fe                      | Column type (MYSQL_TYPE_STRING)
83 40                   | Column flags (PRIMARY_KEY, NOT_NULL, ...)
00                      | Number of decimal places (0)
00 00                   | Unknown, not documented
```

```
03 64 65 66             | Catalog (always "def")
05 6d 79 73 71 6c       | Schema name ("mysql")
04 75 73 65 72          | Virtual table name ("user")
04 75 73 65 72          | Physical table name ("user")
06 70 6c 75 67 69 6e    | Virtual column name ("plugin")
06 70 6c 75 67 69 6e    | Physical column name ("plugin")
0c                      | Length of the fixed-length fields (always 12)
e0 00                   | Character set ("utf8m4_unicode_ci")
00 01 00 00             | Maximum column length (256)
fe                      | Column type (MYSQL_TYPE_STRING)
81 00                   | Column flags (NOT_NULL, BINARY)
00                      | Number of decimal places (0)
00 00                   | Unknown, not documented
```

The column definitions are finalized by an [`EOF_Packet`][eof-packet]:
```
fe                      | EOF
00 00                   | Warnings (none)
22 00                   | Status flags (AUTO_COMMIT, QUERY_NO_INDEX_USED)
```

ℹ️ Please note that as of MySQL 5.7.5, `EOF_Packet` has been deprecated in favor of `OK_Packet`. For
the server to send an `OK_Packet`, the client needs to set the `CLIENT_DEPRECATE_EOF` capability
flag. At the time of writing, `jasync-sql` does not support this flag.

#### Rows
Once all column definitions have been sent to the client, the server starts sending the rows in
the query result as [`Text Resultset Row` packets][result-row]:
```
06 6b 6c 65 6d 65 6e    | Column 1 ("klemen")
15 63 61 63 68 69 6e 67 | Column 2 ("caching_sha2_password")
5f 73 68 61 32 5f 70 61 ¦
73 73 77 6f 72 64       ¦
```

The row packets are finalized by another `EOF_Packet`:
```
fe                      | EOF
00 00                   | Warnings (none)
22 00                   | Status flags (AUTO_COMMIT, QUERY_NO_INDEX_USED)
```

### 🚫 Closing connections and thoughts
Once the client is done executing commands, it can send a simple [`COM_QUIT` command][com-quit] to
tell the server to close the connection:
```
01                      | Command type (COM_QUIT)
```

🎉 And that's it! While the MySQL server and its protocol are complex beasts, it should be fairly
simple to implement a basic client to execute simple queries on the database. The only difficult
part of the above connection flow is the authentication method, but that can be implemented by
referencing the existing drivers for your desired programming language.

Of course, this blog post would not be possible without the excellent (albeit at times a bit hard to
read) [source code documentation][mysql-dev-docs]. As always, referencing the Java driver
implementations helped me fill in the gaps.

### 📚 References
- [Client/Server Protocol][mysql-protocol]
- [Connection Phase][connection-phase]
- [Command Phase][command-phase]
- [Understanding the MySQL Client/Server Protocol using Wireshark][wireshark-mysql]

[r2dbc-mysql]: https://github.com/mirromutth/r2dbc-mysql
[jasync-sql]: https://github.com/jasync-sql/jasync-sql
[tls-exchange]: https://tls12.xargs.org
[mysql-dev-docs]: https://dev.mysql.com/doc/dev/mysql-server/latest/
[mysql-protocol]: https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_PROTOCOL.html
[mysql-packet]: https://dev.mysql.com/doc/dev/mysql-server/latest/page_protocol_basic_packets.html
[connection-phase]: https://dev.mysql.com/doc/dev/mysql-server/latest/page_protocol_connection_phase.html
[handshake-v10]: https://dev.mysql.com/doc/dev/mysql-server/latest/page_protocol_connection_phase_packets_protocol_handshake_v10.html
[capability-flags]: https://dev.mysql.com/doc/dev/mysql-server/latest/group__group__cs__capabilities__flags.html
[ssl-request]: https://dev.mysql.com/doc/dev/mysql-server/latest/page_protocol_connection_phase_packets_protocol_ssl_request.html 
[handshake-response-41]: https://dev.mysql.com/doc/dev/mysql-server/latest/page_protocol_connection_phase_packets_protocol_handshake_response.html#sect_protocol_connection_phase_packets_protocol_handshake_response41
[auth-more-data]: https://dev.mysql.com/doc/dev/mysql-server/latest/page_protocol_connection_phase_packets_protocol_auth_more_data.html
[auth-switch-response]: https://dev.mysql.com/doc/dev/mysql-server/latest/page_protocol_connection_phase_packets_protocol_auth_switch_response.html
[ok-packet]: https://dev.mysql.com/doc/dev/mysql-server/latest/page_protocol_basic_ok_packet.html
[command-phase]: https://dev.mysql.com/doc/dev/mysql-server/latest/page_protocol_command_phase.html
[command-types]: https://dev.mysql.com/doc/dev/mysql-server/latest/my__command_8h.html
[com-query]: https://dev.mysql.com/doc/dev/mysql-server/latest/page_protocol_com_query.html
[result-set]: https://dev.mysql.com/doc/dev/mysql-server/latest/page_protocol_com_query_response_text_resultset.html
[column-definition-41]: https://dev.mysql.com/doc/dev/mysql-server/latest/page_protocol_com_query_response_text_resultset_column_definition.html#sect_protocol_com_query_response_text_resultset_column_definition_41
[eof-packet]: https://dev.mysql.com/doc/dev/mysql-server/latest/page_protocol_basic_eof_packet.html
[result-row]: https://dev.mysql.com/doc/dev/mysql-server/latest/page_protocol_com_query_response_text_resultset_row.html
[com-quit]: https://dev.mysql.com/doc/dev/mysql-server/latest/page_protocol_com_quit.html
[wireshark-mysql]: https://www.turing.com/blog/understanding-mysql-client-server-protocol-using-python-and-wireshark-part-1
