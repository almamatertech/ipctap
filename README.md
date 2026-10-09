# ipctap

`ipctap` shows the messages Safari's processes send each other (CoreIPC), live and by name. It also
shows the messages they send to daemons and XPC services, with the process at the other end and the
service's name. It uses DTrace, so it works on the Safari that ships with macOS, with nothing
injected. It traces any other app that uses the system's WebKit the same way.

```
16:19:00.950596  WebContent 21476  -> UI 13285          WebPageProxy_DecidePolicyForNavigationActionAsync  dest 14534  1880 bytes
16:19:00.950700  UI 13285          <- WebContent 21476  WebPageProxy_DecidePolicyForNavigationActionAsync  dest 14534  1880 bytes
16:19:00.951161  UI 13285          -> Networking 13296  NetworkProcess_HasLocalStorageOrCookies  dest 0  48 bytes
16:19:00.951295  Networking 13296  <- UI 13285          NetworkProcess_HasLocalStorageOrCookies  dest 0  48 bytes
16:19:00.951322  UI 13285          -> com.apple.Safari.SafeBrowsing.Service 1352 0x10000000  port 0x22f03  888 bytes
16:19:00.952778  UI 13285          -> akd 570           com.apple.ak.authorizationservices.xpc 0x10000000  port 0x28363  256 bytes
16:19:00.953426  UI 13285          -> notifyd 165       msgh_id 0x3f9  port 0xe0f  40 bytes
16:19:00.953437  Networking 13296  -> UI 13285          NetworkProcess_HasLocalStorageOrCookiesReply  dest 96613  17 bytes
```

Safari deciding whether to load a page. The WebContent process asks the UI process, which asks the
Networking process whether the site has cookies or local storage, and sends XPC messages to Safe
Browsing and to akd while it waits.

## Quickstart

```
git clone https://github.com/almamatertech/ipctap.git
cd ipctap
sudo ./ipctap trace
```

The first trace reads this Mac's WebKit for a few seconds, and DTrace can take a minute to get ready
the first time. Then it traces Safari until Ctrl-C. It needs root and System Integrity Protection
off, as Requirements explains.

## What's different

Watching Safari's messages has meant patching WebKit and running a build of your own, or injecting
code into Safari's processes, for example with Frida. `ipctap` does neither.

- The names come from the shipped WebKit's own message table. For each WebKit build, `setup` finds
  the fields it reads in WebKit's machine code and checks them against each other.
- It shows both ends of each message, matched across processes, and the GPU process's stream
  messages, which go through shared memory, not Mach.
- On an M6, memory tagging checks DTrace's reads like the process's own, so `ipctap` reads only the
  fields it has found and never past a message.

## Requirements

- An Apple silicon Mac with the Safari that comes with macOS.
- Xcode or the Command Line Tools, for `python3` (3.9 or later) and `dyld_info`.
- Root, and System Integrity Protection (SIP) off, so DTrace can trace Safari.

Tested on macOS 27.0.1 with Safari 27.0.1, on Mac minis with an M6 and an M2 Pro. It may work
elsewhere, but that isn't promised.

## Usage

```
sudo ./ipctap trace [-a APP] [-p PROCESS] [-m MESSAGE] [-o FILE] [-t SECONDS] [--bytes [N]]
./ipctap show FILE [-p PROCESS] [-m MESSAGE]
./ipctap setup
```

`trace` follows Safari's processes, including tabs opened during the trace, until Ctrl-C or for `-t`
seconds of tracing. It also writes every message to `-o FILE` as ndjson, by default
`ipctap-<date>-<time>.ndjson`. `-a` traces another app instead of Safari, by the name of its
executable or its pid, with the WebKit processes macOS counts as that app's.

`-p` keeps only a process, by role (`UI`, `WebContent`, `GPU` or `Networking`) or pid. `-m` keeps
only messages or services whose name matches, with `*` and `?`. Both can be repeated. They change
what's printed and saved, not what's traced, so the other end of each message is still found. For
example, the WebContent processes' page messages and their XPC traffic:

```
sudo ./ipctap trace -p WebContent -m 'WebPage*' -m 'com.apple.*'
```

`show` prints a saved trace, with the same filters, without root. `setup` reads this Mac's WebKit
and libdispatch and writes what `trace` needs for that build. `trace` runs it when the build is new
to it, for example after a macOS or Safari update.

## Reading a trace

Safari loading a local page in a new tab. The WebContent process asks the Networking process for the
page, gets the response and asks the UI process what to do with it:

```
13:07:13.758200  WebContent 18254  -> Networking 13296  NetworkConnectionToWebProcess_ScheduleResourceLoad  dest 0  3488 bytes
13:07:13.758240  Networking 13296  <- WebContent 18254  NetworkConnectionToWebProcess_ScheduleResourceLoad  dest 0  3488 bytes
13:07:13.760712  Networking 13296  -> WebContent 18254  WebResourceLoader_DidReceiveResponse  dest 31  577 bytes
13:07:13.760837  WebContent 18254  <- Networking 13296  WebResourceLoader_DidReceiveResponse  dest 31  580 bytes
13:07:13.760995  WebContent 18254  -> UI 13285          WebPageProxy_DecidePolicyForResponse  dest 9448  1440 bytes
13:07:13.761059  UI 13285          <- WebContent 18254  WebPageProxy_DecidePolicyForResponse  dest 9448  1440 bytes
```

A moment earlier, the GPU process drawing for a page from stream messages:

```
13:07:13.701114  GPU 13297         <- WebContent        RemoteGraphicsContext_Save  dest 350  (stream)
13:07:13.701129  GPU 13297         <- WebContent        RemoteGraphicsContext_Clip  dest 350  20 bytes (stream)
13:07:13.701152  GPU 13297         <- WebContent        RemoteGraphicsContext_ConcatCTM  dest 350  56 bytes (stream)
```

Each line has the time, the process, `->` for a send or `<-` for a receive, and the process at the
other end. `ipctap` finds it by matching each send with its receive in another traced process. When
no single process matches, it shows the kind of process from WebKit's message table, or `?` when the
table doesn't say. Then the message, the object it's for (0 for the process itself) and its size.
`(stream)` marks a message the GPU process reads from shared memory. The ndjson has the same, with
the other end's pid as `peer_pid`, plus the thread and the message's number in this build:

```
{"time": 1791544033546549215, "pid": 13285, "process": "UI", "thread": 178182, "direction": "send", "peer": "WebContent", "peer_pid": 18254, "message": "WebPage_LoadRequest", "id": 3514, "destination": 9448, "size": 266}
```

## Mach and XPC

`trace` also prints every other Mach message the processes send, such as the XPC messages to Safe
Browsing and akd and the message to notifyd in the lines at the top. It leaves out calls to the
kernel.

A Mach line has the process that received the message, or `?` when `ipctap` can't tell. Then the
XPC service's name and the message's msgh_id, or only the msgh_id when there's no service name or
the receiving process has the same name. Then the port it was sent to, as the sending process names
it, and its size. XPC uses msgh_id 0x10000000 for a message, 0x20000000 for a reply, 0x30000000 for
a notification and 0x40000000 for a message that waits for its reply. Other IDs are the sender's
own, such as a MIG routine's number. In the ndjson these lines have the direction `mach`, with the
service or null and the port:

```
{"time": 1791555567543696834, "pid": 13296, "process": "Networking", "thread": 178339, "direction": "mach", "peer": "launchservicesd", "peer_pid": 144, "service": "com.apple.coreservices.launchservicesd", "id": 1073741824, "port": 9739, "size": 168}
{"time": 1791555567545231467, "pid": 13285, "process": "UI", "thread": 178182, "direction": "mach", "peer": "WindowServer", "peer_pid": 171, "service": null, "id": 30333, "port": 16899, "size": 40}
```

## Message bytes

`--bytes` also copies the first bytes of each CoreIPC message, 256 unless you give a number, and
prints them under its line. Here the WebContent process asks the Networking process to load an
image:

```
13:27:26.906883  Networking 13296  <- WebContent 19270  NetworkConnectionToWebProcess_ScheduleResourceLoad  dest 0  1224 bytes
                 0000  00 00 73 02 00 00 00 00  00 00 00 00 00 00 00 00  |..s.............|
                 0010  01 25 00 00 00 00 00 00  02 25 00 00 00 00 00 00  |.%.......%......|
                 0020  58 00 00 00 01 00 00 00  00 00 00 00 26 00 00 00  |X...........&...|
                 0030  01 68 74 74 70 3a 2f 2f  31 32 37 2e 30 2e 30 2e  |.http://127.0.0.|
                 0040  31 3a 38 37 36 35 2f 69  6d 67 2f 30 2e 73 76 67  |1:8765/img/0.svg|
                 0050  3f 69 3d 31 33 38 35 00  43 00 00 00 01 68 74 74  |?i=1385.C....htt|
                 0060  70 3a 2f 2f 31 32 37 2e  30 2e 30 2e 31 3a 38 37  |p://127.0.0.1:87|
                 0070  36 35 2f 62 65 6e 63 68  2e 68 74 6d 6c 3f 74 6f  |65/bench.html?to|
                 ...
                       256 of 1224 bytes
```

Each dump starts with the message's header, which is its flags, then its number at offset 2 (`73 02`
here, which is `0x273`) and its destination at offset 8. `ipctap` copies a message's bytes only when
its buffer starts with that number, and warns when messages come without them, because that would
mean `setup` found the wrong buffer.

Rows that repeat, such as a run of zeros, fold into a single `*`.

In the ndjson, `bytes` holds the copied bytes as hex and `size` the whole message's size. When
`bytes` is shorter than `size`, the copy was cut at the limit.

The bytes can include page content, URLs and cookies, so keep traces made with `--bytes` private.

## How it works

```
        Safari's processes, talking over CoreIPC, and to daemons over Mach and XPC
    +--------+    +--------------------------+    +-------+    +------------+
    |   UI   |    | WebContent (one or more) |    |  GPU  |    | Networking |
    +--------+    +--------------------------+    +-------+    +------------+
        |                      |                      |              |
      DTrace                 DTrace                 DTrace         DTrace
        |                      |                      |              |
        +----------------------+-----------+----------+--------------+
                                           |
                     message number, destination, size and bytes,
                  or a Mach message's port, msgh_id, size and service
                                           |
                                           v
    ./ipctap setup  -------------->  ./ipctap trace  -------------->  terminal and ndjson
    finds the fields and the         names each message,
    message names in this            finds the other end,
    Mac's WebKit and libdispatch     with lsmp for Mach ports,
                                     sorts by time
```

`trace` starts one DTrace per process, because all of WebKit's helper processes have the same
name in DTrace. It traces `IPC::Connection::sendMessageImpl` for sends,
`IPC::Connection::processIncomingMessage` for receives, and the GPU process's 38
`didReceiveStreamMessage` functions for messages that come through shared memory. Each time it reads
three fields, the message's number, the object it's for and its size, and turns the number into a
name with WebKit's own message table. A stream message's size is where the GPU process stopped
reading it when its function returns.

Each DTrace also traces the `mach_msg2_trap` system call, which the other Mach messages go through.
Its registers hold the message's header, so DTrace gets the port, msgh_id and size without reading
the message, and leaves out calls to the kernel and CoreIPC. libdispatch sends XPC's messages in
`_dispatch_mach_msg_send`, whose first argument is the connection's channel. When the trap comes
inside that call, DTrace reads the channel's label, which is the service's name. Port names mean
something only inside their process, so `trace` looks each port up in a snapshot of every process's
ports from `lsmp -a`, which names the process that receives it. When a port isn't in the last
snapshot, a thread takes a new one, about 0.7 s of work, and the message waits for it. Messages a
process sends to itself, mostly run-loop wakeups, are left out.

`setup` finds where those fields are in WebKit's and libdispatch's machine code, and the message
names in WebKit's table, then checks what it found:

```
$ ./ipctap setup
WebKit 988EABF6-FF35-382E-A31A-428FD4FE16BA, libdispatch 3EFC0FCD-6B49-3299-B370-65ADA0802100
Encoder  name +0x0  destination +0x8  size +0x220  buffer +0x10 or inline +0x20
Decoder  name +0x32  destination +0x38  size +0x8  position +0x10
Dispatch channel  label +0x48
4492 message names, 38 stream receivers
ok    the Encoder constructor and sendOutgoingMessage read the name at the same offset
ok    the Encoder's 512-byte inline buffer ends where its size field starts
ok    Decoder::create and processIncomingMessage read the name at the same offset
ok    Decoder::create and decode<MessageName> keep the read position at the same offset
ok    the reply IDs processIncomingMessage tests are SyncMessageReply and CancelSyncMessageReply
ok    the table ends right after the last ID processIncomingMessage looks up
ok    WebPage, WebPageProxy, NetworkConnectionToWebProcess and GPUConnectionToWebProcess messages go to their own process
ok    a dispatch channel's label field holds the label dispatch_queue_get_label gives
ok    the label field is inside the channel
ok    _dispatch_mach_msg_send, which sends a channel's messages, leaves only through ret
wrote builds/988EABF6-FF35-382E-A31A-428FD4FE16BA in 2.4 s
```

The checks compare what different WebKit functions say about the same fields, and test libdispatch's
fields on a channel `setup` makes and never connects. If one fails, `setup` stops rather than write
offsets it can't trust.

On an M6, DTrace's reads are checked against memory tags like the process's own, and a bad read can
crash the process. So `ipctap` reads only those fields. With `--bytes`, it first reads the 2 bytes
where the buffer should hold the message's number, and copies the message's own bytes, up to the
limit, only when they match. For a Mach message it reads only the size of a vector send, inside the
vector, and the channel's label while libdispatch is still sending on the channel. `setup` checks
that `_dispatch_mach_msg_send` leaves only through `ret`, so DTrace always sees it return and never
reads a channel after that.

## Performance

On an M6 Mac mini, with Safari reloading a heavy test page, page loads took 6 to 24% longer while
traced, depending on the run and on what was traced. `ipctap` used about a quarter of one core, and
DTrace less.

## Limits

- Messages from daemons to Safari's processes aren't seen, only the ones Safari's processes send.
- A Mach message's other end can be `?` or, rarely, wrong. XPC messages that skip libdispatch, such
  as those to launchd, have no service name.
- Stream messages show only on the GPU side, some without a size, and `--bytes` doesn't copy them.
- A process that starts during a trace is missed for its first fraction of a second.
- A macOS or Safari update can break `setup`. It then stops rather than guess.

Licensed under [MIT](LICENSE).
