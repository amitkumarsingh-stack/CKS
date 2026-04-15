**Question 1**
A Dockerfile at ``/opt/course/image/api-server.Dockerfile`` is currently using a full Ubuntu base image with unnecessary packages that increase the attack surface.
Convert the Dockerfile to use a minimal distroless base image by:

1. Changing the base image from ``ubuntu:20.04`` to ``gcr.io/distroless/base``
2. Removing package managers ``(apt-get, dpkg)`` and shell ``(bash)``
3. Copying only the necessary application binary
4. Setting the non-root user nonroot ``(UID: 65532)``
5. Using the exec form for ``ENTRYPOINT``

Do not add any new lines - only modify existing ones. The application binary is a Go binary that listens on port ``8080``.

## Solution

**Step 1: Inspect the existing dockerfile**
```
cat /opt/course/image/api-server.Dockerfile

FROM ubuntu:20.04

RUN apt-get update && apt-get install -y bash

COPY api-server /app/api-server

RUN useradd -u 65532 nonroot

USER nonroot

ENTRYPOINT /app/api-server
```

**Step 2: Modify the Dockerfile as per the question
```
FROM gcr.io/distroless/base

COPY ./app-server /app/server

USER nonroot:nonroot

ENTRYPOINT ["/app/server"]
```
