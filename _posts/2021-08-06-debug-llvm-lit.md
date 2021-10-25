---
layout: post
categories: post
author: Stefan Gränitz
date: 2021-08-06 17:00:00 +0200
image: https://weliveindetail.github.io/blog/res/debug-llvm-lit.png
title: "Debugging llvm-lit in vscode"
description: "LLVM's test driver LIT contains a mix of Python modules and configuration scripts that can be a little tricky to debug"
source: https://github.com/weliveindetail/blog/blob/main/_posts/2021-08-06-debug-llvm-lit.md
comments: https://www.reddit.com/r/LLVM/comments/oz8q5w/debugging_llvmlit_in_vscode/
---

![Inspect lit.local.cfg in vscode](https://weliveindetail.github.io/blog/res/debug-llvm-lit.png)

The [LLVM Integrated Tester `llvm-lit`](https://llvm.org/docs/CommandGuide/lit.html){:target="_blank"} provides the test infrastructure for LLVM and its subprojects. It features automatic test exploration, fine-grained hierarchical configration and a flexible notation for `RUN` and `CHECK` lines (in combination with [FileCheck](https://llvm.org/docs/CommandGuide/FileCheck.html){:target="_blank"}). LIT also exists as a [standalone package](https://pypi.org/project/lit/){:target="_blank"}. If you are looking for a tool to run input/output tests, it's certainly worth considering LIT.

All in all LIT consists of a pretty complex mix of Python modules and configuration scripts that makes it a little tricky to debug. Especially, for those who use Python only occasionally. The way it works best for me is to run LIT standalone from the command line with [debugpy](https://github.com/microsoft/debugpy/){:target="_blank"} and then attach the Visual Studio Code (vscode) debugger to it.

In the following I assume that you have a LLVM build at hand and installed [vscode](https://code.visualstudio.com/){:target="_blank"}, [Python3](https://www.python.org/downloads/){:target="_blank"} and the [official Python plugin](https://marketplace.visualstudio.com/items?itemName=ms-python.python){:target="_blank"}. The only thing we still need is the [debugpy package from pip](https://pypi.org/project/debugpy/){:target="_blank"}:
```
> apt install python3-pip
> pip3 install debugpy
```

Now navigate to your LLVM build root and run the LIT driver with debugpy. The extra `--wait-for-client` argument will cause Python to defer execution until we connect a debugger:
```
> cd /path/to/llvm-build
> python3 -m debugpy --listen localhost:5678 --wait-for-client bin/llvm-lit
```

Next, open vscode, create a `launch.json` and add a `Python` debug configuration. The default settings of the `Remote attach` template should do:
```json
{
  "name": "Python: Remote Attach",
  "type": "python",
  "request": "attach",
  "connect": {
    "host": "localhost",
    "port": 5678
  },
  "pathMappings": [
    {
      "localRoot": "${workspaceFolder}",
      "remoteRoot": "."
    }
  ]
}
```

Launching this configuration triggers the Python process to run the test suite. If a test fails, execution gets interrupted with an exception and vscode points you to the code location. In all LIT modules and scripts with a `.py` file extension you can now set breakpoints as usual.

However, a lot of configuration scripts in LLVM still have no `.py` extension. Instead they are interpreted as Python code only through `exec()` commands from the driver. vscode won't let you set breakpoints here. This is where the [explicit breakpoint feature in `debugpy`](https://github.com/microsoft/debugpy/#breakpoint-function){:target="_blank"} comes handy! Just add these two lines in your `lit.local.cfg` file:
```py
import debugpy
debugpy.breakpoint()
```

Execution will be interrupted at the subsequent line during test exploration and voila, you can properly debug your configuration script.
