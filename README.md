# presetshare-project-verificator

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

```dockerfile
FROM ubuntu:18.04 AS build

ENV DEBIAN_FRONTEND=noninteractive
ENV TZ=Europe/Moscow

COPY ./comments-service /comments-service
WORKDIR /comments-service/build

SHELL ["/bin/bash", "-c"]

RUN apt-get update && \
    apt-get install -y \
    software-properties-common \
    python3-pip \
    libuv1 \
    openssl \
    libssl-dev \
    zlib1g \
    wget && \
    wget https://github.com/scylladb/cpp-driver/releases/download/2.15.2-1/scylla-cpp-driver_2.15.2-1_amd64.deb \
    https://github.com/scylladb/cpp-driver/releases/download/2.15.2-1/scylla-cpp-driver-dev_2.15.2-1_amd64.deb && \
    wget -O - https://apt.kitware.com/keys/kitware-archive-latest.asc 2>/dev/null | gpg --dearmor - | tee /etc/apt/trusted.gpg.d/kitware.gpg >/dev/null && \
    apt-add-repository 'deb https://apt.kitware.com/ubuntu/ bionic main' && \
    apt-get update && \
    apt-get install -y \
    cmake \
    ./scylla-cpp-driver_2.15.2-1_amd64.deb \
    ./scylla-cpp-driver-dev_2.15.2-1_amd64.deb
RUN ls
RUN pip3 install conan --upgrade && \
    conan profile detect && \
    conan install .. --output-folder=. --build=missing && \ 
    source conanbuild.sh && \
    cmake -DCMAKE_TOOLCHAIN_FILE=conan_toolchain.cmake -DCMAKE_BUILD_TYPE=Release .. && \
    cmake --build . && \
    source deactivate_conanbuild.sh

FROM ubuntu:18.04

RUN groupadd dev && useradd -g dev dev
USER dev
COPY --chown=dev:dev --from=build /comments-service/build/comments-service /app/comments-service

CMD ["/app/comments-service", "0.0.0.0", "8080"]
EXPOSE 8080

```

```conanfile
[requires]
boost/1.84.0
nlohmann_json/3.11.3

[tool_requires]
cmake/3.28.1

[generators]
CMakeDeps
CMakeToolchain
```

```
FROM ubuntu:22.04 AS build

ENV DEBIAN_FRONTEND=noninteractive
ENV TZ=Europe/Moscow

COPY ./comments-service /comments-service
WORKDIR /comments-service/build

SHELL ["/bin/bash", "-c"]

RUN apt-get update && \
    apt-get install -y \
    python3-pip \
    cmake
RUN ls
RUN pip3 install conan --upgrade && \
    conan profile detect && \
    conan install .. --output-folder=. --build=missing && \ 
    source conanbuild.sh && \
    cmake -DCMAKE_BUILD_TYPE=Release .. && \
    cmake --build . && \
    source deactivate_conanbuild.sh

FROM ubuntu:22.04

RUN groupadd dev && useradd -g dev dev
USER dev
COPY --chown=dev:dev --from=build /comments-service/build/comments-service /app/comments-service

CMD ["/app/comments-service", "0.0.0.0", "8080"]
EXPOSE 8080
```