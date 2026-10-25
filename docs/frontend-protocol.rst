
.. _frontend-protocol:

The ``fowld`` Frontend Protocol
================================

**Target Audience**: users of ``fowld`` (integrators, GUI authors, experimenters)

The ``fowld`` program speaks a line-based protocol on stdin and stdout.

Our intent is to allow any language to use and experiment with Dilation via local streaming interfaces; if a feature for your use-case is missing please `open a new Issue <https://github.com/meejah/fowl/issues/new>`_.

Every message is a single line, terminated with a single newline (``\n``) character.
Every message is a complete and valid JSON message, encoded in UTF8.
Every message is a ``dict`` (aka "JSON object").

Thus, controller programs can deserialize each line as a ``dict`` (or mapping, or your language's equivalent).

Since ``fowl`` itself uses ``fowld`` under the hood, this project has examples of parsing and producing these messages.

We now go over the keys that particular messages have.


The ``"kind"`` Key
--------------------

Every single message MUST have a ``"kind"`` key.
This tells the parser what sort of message it is (and hence what other keys may be present).

It is a protocol error to emit a message without a ``"kind"`` key.
Hereafter, we refer to all messages by their "kind".

An "input" message is one that ``fowld`` accepts on stdin.

An "output" message is one that ``fowld`` may produce on stdout.

To follow along in the code, see the ``fowl.messages`` package.
Types derived from ``FowlOutputMessage`` are produced on stdout and ``FowlCommandMessage`` derived types are accepted on stdin.


Input: ``kind: allocate-code``
------------------------------

This message tells ``fowld`` to allocate a fresh code.
These codes look like ``1-foo-bar`` (you can control how many words).
Note that we interact with the server to reserve the "nameplate" (the short number) but the words are chosen locally (and randomly).

Keys allowed:

- ``code-length`` (optional): how many words to use

After issuing this command, at some point in the future a ``kind: code-allocated`` message will be emitted.


Input: ``kind: set-code``
-------------------------

Tell ``fowld`` a specific code to use.
This likely came from a corresponding use of ``kind: allocate-code`` (in another instance of ``fowld`` on another computer).

- ``code`` (required): the secret code to use


Input: ``kind: local``
----------------------

This message asks ``fowld`` to listen locally, forwarding local connections to the other peer.
That is, the other peer is running the daemon-style software. The other peer must enable the same service-name.If a ``remote_connect_port`` is specified, the invocation on the other peer must agree.Therefore, it is best to ONLY choose ports on your side, unless the protocol requires otherwise.If you can avoid choosing at all, a random port is assigned -- this is the most likely to succeed.

Keys allowed in this message:

- ``name`` (required): unique name for this service
- ``local_listen_port``: port to listen locally (or select randomly)
- ``remote_connect_port``: port to connect to
- ``bind_interface``: interface to bind the ``local_listen_port`` (defaut to ``localhost``)

For example:

.. code-block:: json

    {
        "kind": "local",
        "name": "myapp",
        "local_listen_port": 8000,
        "bind_interface": "localhost",
    }

In this example, we will open a listener on our machine on TCP port ``8000`` and interface ``localhost``. This port is associated with a service name ``myapp``.
Every request to ``localhost:8000`` will be forwarded to the other peer.

.. You are only limited to streaming endpoints (i.e. no UDP) but they do not have to be TCP.
.. For example, one side could use ``unix:/tmp/socket`` to open a listener (or connection) on Unix-domain socket.
.. With `txtorcon <https://meejah.ca/projects/txtorcon>`_ one could have ``onion:...`` Tor endpoints on either end.

Once the listener is established, we'll issue a ``kind: listening`` output.


Input: ``kind: remote``
-----------------------

This message asks ``fowld`` to listen on the other peer, so the server-style software runs here.
The other peer must enable the same service ``name``.Ports must agree.Therefore, it is best to ONLY choose ports on your side, unless the protocol requires otherwise.If you can avoid choosing at all, a random port is assigned -- this is the most likely to succeed.

Keys allowed in this message:

- ``name`` (required): unique name for this service
- ``remote_listen_port``: port to listen on (or let peer select)
- ``local_connect_port``: port to connect here on
- ``connect_address``: address to connect here on

For example, to peer with the example above:

.. code-block:: json

    {
        "kind": "remote",
        "name": "myapp",
        "local_connect_port": 80,
        "connect_address": "localhost",
    }

Once the two peers are connected, any TCP request to ``localhost:8000`` **on the local side** will be forwarded to ``localhost:80`` **on the remote side**.

The far-side ``fowld`` will issue a ``kind: listening`` message (on its side) when it has started listening.


Input: ``kind: session-close``
------------------------------

Keys allowed:

- ``timeout`` (optional): an integer number of seconds

Initiate an orderly shutdown.
We ask our peer to close their conection; if they do, we also close ours and exit.
If we do not hear back from the peer, we give up after ``"timeout"`` seconds and exit.

That is, regardless of what our peer does, we exit.
If we did not hear back from the peer we exit with exit-status 1, otherwise exit-status 0.



Output: ``kind: listening``
---------------------------

This message is issued by ``fowld`` when it has opened a listening socket on the local side.

An example message:

.. code-block:: json

    {
        "kind": "listening",
        "name": "myapp",
        "listening_port": 8000,
    }

Guidance for UX: the user should be made aware their machine is listening on a particular port / interface.


Output: ``kind: error``
-----------------------

Some sort of error has happened.

This message MUST have a ``message`` key containing a free-form error message.

An example message:

.. code-block:: json

    {
        "kind": "error",
        "message": "Unknown control command: foo"
    }

Guidance for UX: most errors are probably interesting to the user.


Output: ``kind: welcome``
-------------------------

This message is emitted to both sides once per session when we connect to the Mailbox Server.

- ``"welcome"``: a ``dict`` containing whatever the Mailbox server sent in its Welcome message.

Guidance for UX: the user should be informed that progress has been made (e.g. the Mailbox Server is available).


Output: ``kind: code-allocated``
--------------------------------

This is emitted once ``fowld`` has a secret code.
We could have been given one with ``kind: set-code`` or created a new one with ``kind: allocate-code``.
In either case, this message is emitted.

- ``"code"``: the secret code


Output: ``kind: peer-connected``
--------------------------------

The ``fowld`` process has successfully communicated with the other peer.

  - ``"verifier"``: a string containing 32 hex-encoded bytes which are a hash of the session key
  - ``"versions"``: an object containing application-specific versioning information

Guidance for UX: advanced users may wish to compare the verifiers for extra security (they should match; if they don't, it may be a "Machine in the Middle" attack).

Guidance for integration: the "versions" metadata is intended to allow your application to determine information about the peer.
This could be use for capability discovery, protocol selection, or anything else.


Output: ``kind: bytes-in``
--------------------------

The ``fowld`` process received some forwarded bytes successfully.

Keys present:

- ``id`` (required): the sub-connection id, a unique number
- ``bytes`` (required): how many bytes are forwarded recently

Guidance for UX: the user may be curious to know if a connection is alive, what its throughput is, etc.


Output: ``kind: bytes-out``
---------------------------

The ``fowld`` process forwarded some bytes to the other peer successfully.

Keys present:

- ``id`` (required): the sub-connection id, a unique number
- ``bytes`` (required): how many bytes are forwarded recently

Guidance for UX: the user may be curious to know if a connection is alive, what its throughput is, etc.


Output: ``kind: local-connection``
----------------------------------

We have received a connection on one of our local listeners.

Keys present:

- ``id`` (required): the sub-connection id, a unique number

Guidance for UX: the user should be informed that something is interacting with our listener.


Output: ``kind: incoming-connection``
-------------------------------------

The other side has asked us to make a local connection.

Keys present:

- ``id`` (required): the sub-connection id, a unique number
- ``endpoint`` (required): the Twisted client-style endpoint we will attempt a connection to

Guidance for UX: the user should be informed that something is interacting with our listener.
