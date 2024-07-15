# Management of configurations in TAO

## Changing the configuration of a server

Any connected client is allowed to change the configuration of a TAO server via the
`config` command. Clients are supposed to behave in collaboration, so, in general, a
dedicated client takes the responsibility of applying given settings to a given server. A
server configuration may however not be changed while the server is running (i.e., after a
successful `start` command and until a `stop` or an `abort` command). Hence,
re-configuring a server generally involves first sending a `stop` (or an `abort`) command,
followed by `config` commands, and finally a `start` command.


## Tracking server configurations

To keep track of server configurations, each server saves its configuration as a TOML file
when a `start` command is received. These configuration files are registered into a Git
repository (common to all servers) and the Git identifier of the configuration is written
in each frame of the telemetry published by the server. In that way, clients may easily
check whether the configuration has changed and off-line or non-real-time processing tasks
can retrieve the exact configuration for a given frame of telemetry. The current
configuration is also available in the shared resources owned by the server but may have
changed if another telemetry frame than the last one is considered or if the shared
resources are not read-locked by the client. To avoid locking the shared resources and,
thus, possibly blocking the server, it is recommended to use the Git identifier written in
the telemetry.

To avoid collisions when servers register their current configuration before starting,
registering the configuration files is done by a dedicated *Configuration Server*. The
*Configuration Server* can also be asked by clients to retrieve a registered configuration
content given its identifier.


## Parameters changing in real-time

The parameters which may change while a real-time server is running (i.e., after a `start`
command and until a `stop` or an `abort` command) shall not be considered as being part of
a registered configuration. Their initial value may however be part of a registered
configuration. If relevant for the clients, the values of changing parameters shall be
part of the published telemetry.


## Suggestions/ideas

* To simplify the management of a running AO system, it is recommended that the directory
  where servers write their ShmId's and the Git repository storing the different
  configurations be the same.

* To not delay the starting of the real-time servers, the Git identifier of the
  configuration can be directly computed and the configuration content sent asynchronously
  (i.e., without waiting for the Configuration Server to acknowledge for the success).


## Synopsis

To illustrate how configuration can be managed, we use the
[`YakMessenger`](https://github.com/emmt/YakMessenger.jl) protocol where messages consist
in two parts: the *type* (a single ASCII character) and the *content* (any text or binary
data). Depending on the message type, an answer form the server is expected by the client.
As an answer, a type-`E` message indicates an error (the error message being provided by
the message content).

To register its configuration, a real-time TAO server executes the following steps:

1. Write its configuration in a TOML file named `$TAO_CONFIG_DIR/$owner/config.toml`.
2. Ask the configuration server to register its configuration. For example, in Julia:
   `YakMessenger.send_message(conn, 'C', owner)`.
3. Receive acknowledgment from the configuration server. For example, in Julia: `(type,
   data) = YakMessenger.recv_message(conn)` where, in case of success, `type` is `'H'`
   with blob hash code given by `data`; otherwise, in case of error, `type` is `'E'` with
   error message given by `data`.

To register a client's configuration in the Git repository, the configuration server
executes the following steps:

1. Receive a type-`C` message whose content is the *owner* name of the client.
2. Optionally, load the TOML configuration file in `$TAO_CONFIG_DIR/$owner/config.toml`
   to check that it is a valid TOML file.
3. Copy the TOML configuration file to the Git repository as `$repo/$owner.toml`.
4. Commit file `$repo/$owner.toml` with a message like `"Update $owner configuration"`.
5. Retrieve the hash code of the resulting Git blob.
6. Send an answer to the client.  For example, in Julia:
   `YakMessenger.send_message(conn, 'H', hash)`.

Remarks:

- To avoid blocking (reading sockets is a blocking operation in Julia), the protocol can
  be implemented using TAO's shared objects. This would however prevent accessing
  configurations from outside the local host. The risk of blocking is mitigated in the
  [`YakMessenger`](https://github.com/emmt/YakMessenger.jl) protocol because the size of
  the message is known after reading a few bytes and because the convention is to close
  the connection whenever a malformed or corrupted message is received.

- With the proposed synopsis, there are 2 instances of a TOML configuration file for a
  given server: one in `$TAO_CONFIG_DIR/$owner/config.toml`, the other in the Git
  repository. This may be a drawback in terms of overheads (we need to benchmark a typical
  scenario). An advantage is however that it is sufficient that the different real-time
  tasks have read/write access to `$TAO_CONFIG_DIR/$owner` while the configuration server
  has read-only access to `$TAO_CONFIG_DIR/$owner/config.toml` and read/write access to
  the Git repository. To avoid this double copy, a possibility is to have the
  configuration sent as the message content.

- The Git repository with the configurations could simply be specific to each adaptive
  optics system based on TAO (e.g. `ThemisAO` for the Themis solar telescope). This
  repository could be a valid Julia package whose dependencies are listed in the
  `Project.toml` file.


Changes with respect to current TAO:

- The ShmId of a TAO real-time server is saved in file `$TAO_CONFIG_DIR/$owner/shmid`
  instead of `$TAO_CONFIG_DIR/$owner`.

- The *telemetry* of a camera server is provided by published images (as TAO shared
  arrays). The hash of the blob of the registered camera server configuration should be
  part of the shared array structure.


## Implementation

The configuration server can be implemented in Julia using `@async` tasks to deal with
blocking reads while serving multiple clients. The server may also manage the settings
(real-time priorities, CPU affinities, etc.) of the real-time tasks.
