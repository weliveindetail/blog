---
layout: post
categories: post
author: Stefan Gränitz
date: 2021-10-20 18:00:00 +0200
image: https://weliveindetail.github.io/blog-sandbox/res/vscode-hide-warning-markers-on.png
preview: summary_large_image
title: "Stop warning markers from polluting vscode minimap"
description: "More plugins means more warnings and not everything is configurable. Here's a quick way to remove warning markers from the minimap and make space, e.g. for search results."
source: https://github.com/weliveindetail/blog/blob/main/_posts/2021-10-20-vscode-hide-warning-markers.md
---

![vscode with clang-tidy warnings and minimap markers on](https://weliveindetail.github.io/blog-sandbox/res/vscode-hide-warning-markers-on.png)

[clangd](https://marketplace.visualstudio.com/items?itemName=llvm-vs-code-extensions.vscode-clangd){:target="_blank"} and other plugins tremendously increase efficiency when working with C++ in Visual Studio Code (vscode), but more plugins means more warnings and not everything is configurable. The above screenshot shows how a load of [clang-tidy](https://clang.llvm.org/extra/clang-tidy/){:target="_blank"} warning highlights from clangd clashes with the highlights for our search results in the minimap. This is going to slow down search-intensive tasks.

Ideally we fixed the warnings altogether or we adjusted the project's clang-tidy settings so they won't show up anymore. However, we all know these situations where we don't have the time right now to deal with such things. So what else can we do? Disable warnings in clangd? Unfortunately that doesn't seem possible:

![clangd plugin settings](https://weliveindetail.github.io/blog-sandbox/res/vscode-hide-warning-markers-clangd-settings.png)

To the rescue, there is a way to [change highlight colors globally](https://code.visualstudio.com/api/references/theme-color){:target="_blank"} in vscode! So we can make warnings transparent in the minimap and in the overview-ruler:
```
{
  "workbench.colorCustomizations": {
    "minimap.warningHighlight": "#00000000",
    "editorOverviewRuler.warningForeground": "#00000000",
  },
}
```

### Voilà!

We still see the warning squiggles in the editor, but they don't pollute the minimap anymore.

![vscode with clang-tidy warnings and minimap markers off](https://weliveindetail.github.io/blog-sandbox/res/vscode-hide-warning-markers-off.png)
