---
layout: post
categories: post
author: Stefan Gränitz
date: 2023-02-04 00:00:00 +0200
image: https://weliveindetail.github.io/blog/res/ez-clang-pycfg-banner.png
preview: summary_large_image
title: "ez-clang Python Device Configuration Layer"
description: "In release 0.0.6 ez-clang gained a Python Device Configuration Layer that allows users to connect their own devices."
source: https://github.com/weliveindetail/blog/blob/main/_posts/2023-02-04-ez-clang-pycfg.md
comments: https://news.ycombinator.com/item?id=34649661
---

<style>
  #banner-image {
    max-width: min(100%, 500px);
  }
  #large-image {
    max-width: min(100%, 800px);
  }
  .center {
    display: block;
    margin: 0 auto;
  }
</style>

![ez-clang-pycfg-banner](https://weliveindetail.github.io/blog/res/ez-clang-pycfg-banner.png){: #banner-image}{: .center}

<br>
[ez-clang](http://echtzeit.dev/ez-clang/){:target="_blank"} is a C++ REPL for bare-metal embedded devices. In [release 0.0.6](https://github.com/echtzeit-dev/ez-clang/releases/tag/v0.0.6){:target="_blank"} it gained a [Python Device Configuration Layer (pycfg)](https://github.com/echtzeit-dev/ez-clang-pycfg){:target="_blank"} that allows users to connect their own devices.

### Elements of a device script

As before we specify the target device in the `--connect` parameter, e.g.:

```
ez-clang --connect=teensylc@/dev/ttyACM0
ez-clang --connect=raspi32@192.168.0.100:20000
ez-clang --connect=lm3s811@qemu
```

The [`scan()`](https://github.com/echtzeit-dev/ez-clang-pycfg/blob/v0.0.6/.share/ez/scan.py#L7){:target="_blank"} function will parse this string and load the respective device script. The [`Script` class](https://github.com/echtzeit-dev/ez-clang-pycfg/blob/v0.0.6/.share/ez/util/script.py#L27){:target="_blank"} implements the interface between ez-clang and Python. Compatible device scripts define the following freestanding functions.

`accept()` checks whether the provided info matches the device. The first candidate script that returns a non-Null value here will be choosen. The type of the `info` parameter depends on the selected transport type. For serial transport we get a `serial.tools.list_ports_linux.SysFS` object and will probably check the `hwid` field ([example for TeensyLC](https://github.com/echtzeit-dev/ez-clang-pycfg/blob/v0.0.6/teensylc/serial.py#L115){:target="_blank"}).
```
def accept(info: Any) -> Any
```
`connect()` establishes the raw connection to the device and returns a serializer for it. The serializer has a [standard implementation](https://github.com/echtzeit-dev/ez-clang-pycfg/blob/v0.0.6/.share/ez/repl/serialize.py){:target="_blank"}, that can typically be reused ([example for TeensyLC](https://github.com/echtzeit-dev/ez-clang-pycfg/blob/v0.0.6/teensylc/serial.py#L122){:target="_blank"}).
```
def connect(info: Any, host: ez_clang_api.Host, device: ez_clang_api.Device) -> ez.repl.Serializer
```

`setup()` configures the `ez_clang_api.Device` and adds it to the `ez_clang_api.Host`. This is the core function for device configuration. We read the setup message from the device, initialize endpoints, set CPU, target triple and code buffer (memory range available for JITed code) as well as header search paths and compiler flags. Examples: [TeensyLC](https://github.com/echtzeit-dev/ez-clang-pycfg/blob/v0.0.6/teensylc/serial.py#L134){:target="_blank"}, [Arduino Due](https://github.com/echtzeit-dev/ez-clang-pycfg/blob/v0.0.6/due/serial.py#L84){:target="_blank"}, [Raspberry Pi (Socket)](https://github.com/echtzeit-dev/ez-clang-pycfg/blob/v0.0.6/raspi32/socket.py#L111){:target="_blank"}, [LM3S811 (QEMU)](https://github.com/echtzeit-dev/ez-clang-pycfg/blob/v0.0.6/lm3s811/qemu.py#L95){:target="_blank"}.
```
def setup(stream: ez.repl.Serializer, host: ez_clang_api.Host, device: ez_clang_api.Device) -> bool
```

`call()` allows to invoke the RPC function `endpoint` on the device with the parameters in `data`. As in the last release, [builtin endpoints](https://github.com/echtzeit-dev/ez-clang-pycfg/blob/v0.0.6/.share/ez/repl/__init__.py#L223-L228){:target="_blank"} are `lookup`, `commit`, `execute` and `memory.read.cstr` (see [binary interface docs](https://github.com/echtzeit-dev/ez-clang/blob/v0.0.5/release/0.0.5/docs/rpc.md){:target="_blank"}).
```
def call(endpoint: str, data: dict) -> dict
```

`disconnect()` shuts down the session and closes the device connection.
```
def disconnect() -> bool
```

### Host and Device interfaces

These interfaces describe specific properties for the host and the device respectively. They are both implemented in C++ inside ez-clang and not formally documented yet. For testing, however, we [mock these types](https://github.com/echtzeit-dev/ez-clang-pycfg/blob/main/.share/ez_clang_api/__init__.py){:target="_blank"} and they can be used as an informal documentation for now.

### Install

Clone the repository and install dependencies:
```
> git clone https://github.com/echtzeit-dev/ez-clang-pycfg
> python3 --version
Python 3.8.10
> cd ez-clang-pycfg
> python3 -m pip install -e .share
> python3 -m pip install -r requirements.txt
```

### Testing

One major benefit of pycfg is that connectivity testing and debugging can be done in Python. This is a lot easier and quicker than C++ and it allows for better isolation. Each target device implementation comes with a test suite that covers all relevant connectivity features (example for [Arduino Due](https://github.com/echtzeit-dev/ez-clang-pycfg/tree/v0.0.6/due/test){:target="_blank"}).

The `run_all.py` script is for batch testing. It uploads a fresh firmware and runs all tests in sequence:
```
> python3 due/test/run_all.py
Found compatible device at /dev/ttyACM2
Device unique identifier: 7503130343135130F0A0
Uploading firmware: /usr/lib/ez-clang/0.0.6/firmware/due/firmware.bin
Running tests from /usr/lib/ez-clang/0.0.6/pycfg/due/test
Selecting 8 out of 9 discovered tests
  [basics] connect ........................................ 4.51s
  [basics] setup .......................................... 4.52s
  [basics] connect_setup_repeat ........................... 9.02s
  [endpoints] ez.rpc.lookup ............................... 9.07s
  [endpoints] ez.rpc.commit ............................... 5.46s
  [endpoints] ez.rpc.execute .............................. 5.02s
  [endpoints] memory.read.cstr ............................ 5.43s
  [recovery] replace_firmware ............................. 19.96s

Testing Time: 88.65s
  Disabled: 1
  Excluded: 0
  Failed  : 0
  Passed  : 8

SUCCESS
```

The inidividual tests are self-contained and can be executed as standalone Python scripts:
```
> python3 due/test/00-basics/01-connect.py
Found compatible device at /dev/ttyACM2
Connect <-
  6a 00 00 00 00 00 00 00
  00 00 00 00 00 00 00 00
  00 00 00 00 00 00 00 00
  00 00 00 00 00 00 00 00
  05 00 00 00 00 00 00 00 30 2e 30 2e 35 50 1a 07 20 00 00 00 00
  b0 61 01 00 00 00 00 00 01 00 00 00 00 00 00 00 15 00 00 00 00
  00 00 00 5f 5f 65 7a 5f 63 6c 61 6e 67 5f 72 70 63 5f 6c 6f 6f
  6b 75 70 19 05 08 00 00 00 00 00
Disconnect ->
  20 00 00 00 00 00 00 00
  01 00 00 00 00 00 00 00
  01 00 00 00 00 00 00 00
  00 00 00 00 00 00 00 00
Disconnect <-
  21 00 00 00 00 00 00 00
  01 00 00 00 00 00 00 00
  00 00 00 00 00 00 00 00
  00 00 00 00 00 00 00 00
  00
```

### Run in ez-clang

In order to use new or modified scripts with ez-clang, just mount the repo folder in docker with the `-v` parameter:
```
> docker run -it -v $(pwd)/ez-clang-pycfg:/lib/ez-clang/0.0.6/pycfg echtzeit/ez-clang:0.0.6 --connect=raspi32@192.168.1.105:10819
```

### Debugging in ez-clang

We can also debug scripts when they run in ez-clang. For that we have [debugpy](https://github.com/microsoft/debugpy){:target="_blank"} hooks at the start of each device script:
```py
if ez_clang_api.Host.debugPython(__debug__):
    import debugpy
    debugpy.listen(('0.0.0.0', 5678))
    ez.io.note("Python API waiting for debugger. Attach to 0.0.0.0:5678 to proceed.")
    debugpy.wait_for_client()
    debugpy.breakpoint()
```

They get enabled by passing the `--rpc-debug-python` flag to ez-clang. Additionally we have to forward the debug port to the host with the `-p` parameter for docker:
```
> docker run --rm -p 5678:5678 -it echtzeit/ez-clang:0.0.6 --connect=raspi32@192.168.1.105:10819 --rpc-debug-python
Welcome to ez-clang, your friendly remote C++ REPL. Type `.q` to exit.
Python API waiting for debugger. Attach to 0.0.0.0:5678 to proceed.
```

Now any appropriate debugger should be able to attach, e.g. vscode with a launch configuration like this:
```
{
  "name": "Python: Remote Attach",
  "type": "python",
  "request": "attach",
  "connect": { "host": "localhost", "port": 5678 },
  "pathMappings": [{
    "localRoot": "${workspaceFolder}/ez-clang-pycfg",
    "remoteRoot": "/usr/lib/ez-clang/0.0.6/pycfg"
  }],
},
```

### Voilà!

![ez-clang-pycfg-debug](https://weliveindetail.github.io/blog/res/ez-clang-pycfg-debug.png){: #large-image}{: .center}
