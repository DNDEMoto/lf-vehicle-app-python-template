# Copyright (c) 2022-2024 Contributors to the Eclipse Foundation
#
# This program and the accompanying materials are made available under the
# terms of the Apache License, Version 2.0 which is available at
# https://www.apache.org/licenses/LICENSE-2.0.
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
# WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
# License for the specific language governing permissions and limitations
# under the License.
#
# SPDX-License-Identifier: Apache-2.0

# syntax=docker/dockerfile:1.2

# Build stage, to create the executable
FROM --platform=$TARGETPLATFORM ghcr.io/eclipse-velocitas/devcontainer-base-images/python:v0.2 as builder
ARG TARGETARCH

RUN apt-get update && apt-get install -y python3-dev

COPY ./.velocitas.json /workspace/.velocitas_org.json
COPY ./app /workspace/app

# FIXME: For build tooling we only need "devenv-devcontainer-setup", we should be able to
# filter this without manual jq intervention...
RUN cat ./workspace/.velocitas_org.json | jq 'del(.packages[] | select(.name != "devenv-devcontainer-setup"))' > ./workspace/.velocitas.json

# Remove this installation for Arm64 once staticx has a prebuilt wheel for Arm64
RUN /bin/bash -c 'set -ex && \
    ARCH=`uname -m` && \
    if [ "$ARCH" == "aarch64" ]; then \
    echo "ARM64" && \
    apt-get install -y gcc && \
    pip3 install --no-cache-dir scons; \
    fi'

RUN pip3 install --no-cache-dir pyinstaller==5.9.0 \
    && pip3 install --no-cache-dir patchelf==0.17.0.0 \
    && pip3 install --no-cache-dir staticx \
    && pip3 install --no-cache-dir -r ./workspace/app/requirements.txt \
    && pip3 install --no-cache-dir -r ./workspace/app/requirements-links.txt

# install java for LinguaFranca
RUN apt-get update && apt-get install -y curl zip unzip
RUN curl -s "https://get.sdkman.io" | bash
RUN /bin/bash -c "source $HOME/.sdkman/bin/sdkman-init.sh"
RUN /bin/bash -c "source $HOME/.sdkman/bin/sdkman-init.sh && sdk help"
RUN /bin/bash -c "source $HOME/.sdkman/bin/sdkman-init.sh && sdk install java 17.0.7-ms"
RUN /bin/bash -c "source $HOME/.sdkman/bin/sdkman-init.sh && sdk use java 17.0.7-ms"
ENV JAVA_HOME /root/.sdkman/candidates/java/current
#####################

# install LinguaFranca
COPY ./.devcontainer/lf-scripts/* /tmp/lf-scripts/
RUN apt-get update && apt-get install -y sudo
RUN  /bin/bash /tmp/lf-scripts/setup-env.bash
RUN  /bin/bash /tmp/lf-scripts/setup-lf.bash
ENV PATH="/lingua-franca/bin:${PATH}"
#####################
WORKDIR /workspace

RUN velocitas init

WORKDIR /workspace/app

# build LinguaFranca
RUN mkdir -p /workspace/app/dist
RUN lfc -o dist src/lf_velocitas_event/lf_velocitas_e.lf
RUN mv dist/bin/lf_velocitas_e dist/bin/run-exe
# RUN pyinstaller --clean -F -s src/main.py

# WORKDIR /workspace/app/dist

# RUN staticx main run-exe

# Runner stage, to copy the executable
#####################

FROM scratch

# for LinguaFranca
COPY --from=builder ./workspace/app/dist/* /dist/
# COPY --from=builder ./workspace/app/dist/run-exe /dist/
#####################

WORKDIR /tmp
WORKDIR /dist

ENV PATH="/dist:$PATH"

LABEL org.opencontainers.image.source="https://github.com/eclipse-velocitas/vehicle-app-python-template"

CMD ["./run-exe"]
