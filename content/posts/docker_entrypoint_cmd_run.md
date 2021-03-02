---
title: "Docker Entrypoint vs CMD"
date: 2021-02-09T19:53:48Z
author: "Carlos Castro"
tags: ["Docker"]
categories: ["Docker"]
draft: false
showToc: false
TocOpen: false
hidemeta: false
comments: false
disableHLJS: true 
disableShare: true
disableHLJS: true
searchHidden: true
---

Docker has a default entrypoint which is `/bin/sh -c` but does not have a default command. `ENTRYPOINT` and `--entrypoint` were introduced to allow to override `/bin/sh -c`.

`CMD` specify a default command that executes when the container is starting.

Taking `docker run -i -t ubuntu bash` as example, with the entrypoint `/bin/sh -c`, the command is the actual thing that gets executed: `docker run -i -t ubuntu <cmd>`. C

If we run `docker run -i -t ubuntu`, it will start `bash` shell and that's because the Ubuntu Dockerfile has `CMD ["bash"]`
