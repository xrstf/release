# Copyright 2017 The Kubernetes Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

all: all-build

REGISTRY ?= gcr.io/k8s-staging-build-image
IMAGE ?= $(REGISTRY)/debian-base
BUILD_IMAGE ?= debian-build

TAG ?= $(shell git describe --tags --always --dirty)
IMAGE_VERSION ?= buster-v1.4.0
CONFIG ?= buster

TAR_FILE ?= rootfs.tar
ARCH ?= amd64
ALL_ARCH = amd64 arm arm64 ppc64le s390x

TEMP_DIR:=$(shell mktemp -d)
QEMUVERSION=5.2.0-2

SUDO=$(if $(filter 0,$(shell id -u)),,sudo)

# This option is for running docker manifest command
export DOCKER_CLI_EXPERIMENTAL := enabled

ifeq ($(ARCH),amd64)
	BASEIMAGE?=debian:$(CONFIG)-slim
endif
ifeq ($(ARCH),arm)
	BASEIMAGE?=arm32v7/debian:$(CONFIG)-slim
	QEMUARCH=arm
endif
ifeq ($(ARCH),arm64)
	BASEIMAGE?=arm64v8/debian:$(CONFIG)-slim
	QEMUARCH=aarch64
endif
ifeq ($(ARCH),ppc64le)
	BASEIMAGE?=ppc64le/debian:$(CONFIG)-slim
	QEMUARCH=ppc64le
endif
ifeq ($(ARCH),s390x)
	BASEIMAGE?=s390x/debian:$(CONFIG)-slim
	QEMUARCH=s390x
endif

sub-build-%:
	$(MAKE) ARCH=$* build

all-build: $(addprefix sub-build-,$(ALL_ARCH))

sub-push-image-%:
	$(MAKE) ARCH=$* push

all-push-images: $(addprefix sub-push-image-,$(ALL_ARCH))

all-push: all-push-images push-manifest

push-manifest:
	docker manifest create --amend $(IMAGE):$(IMAGE_VERSION) $(shell echo $(ALL_ARCH) | sed -e "s~[^ ]*~$(IMAGE)\-&:$(IMAGE_VERSION)~g")
	@for arch in $(ALL_ARCH); do docker manifest annotate --arch $${arch} ${IMAGE}:${IMAGE_VERSION} ${IMAGE}-$${arch}:${IMAGE_VERSION}; done
	docker manifest push --purge ${IMAGE}:${IMAGE_VERSION}

build: clean
	cp -r ./$(CONFIG) $(TEMP_DIR)/
	cat $(CONFIG)/Dockerfile.build \
		| sed "s|BASEIMAGE|$(BASEIMAGE)|g" \
		| sed "s|ARCH|$(QEMUARCH)|g" \
		> $(TEMP_DIR)/$(CONFIG)/Dockerfile.build

ifeq ($(ARCH),amd64)
	# When building "normally" for amd64, remove the whole line, it has no part in the amd64 image
	sed "/CROSS_BUILD_/d" $(TEMP_DIR)/$(CONFIG)/Dockerfile.build > $(TEMP_DIR)/$(CONFIG)/Dockerfile.build.tmp
else
	# Register /usr/bin/qemu-ARCH-static as the handler for non-x86 binaries in the kernel
	docker run --rm --privileged multiarch/qemu-user-static:$(QEMUVERSION) --reset -p yes
	docker buildx version
	docker buildx create --use
	docker buildx inspect --bootstrap

	curl -sSL https://github.com/multiarch/qemu-user-static/releases/download/v$(QEMUVERSION)/x86_64_qemu-$(QEMUARCH)-static.tar.gz | tar -xz -C $(TEMP_DIR)/$(CONFIG)
	# Ensure we don't get surprised by umask settings
	chmod 0755 $(TEMP_DIR)/$(CONFIG)/qemu-$(QEMUARCH)-static

	# When cross-building, only the placeholder "CROSS_BUILD_" should be removed
	sed "s/CROSS_BUILD_//g" $(TEMP_DIR)/$(CONFIG)/Dockerfile.build > $(TEMP_DIR)/$(CONFIG)/Dockerfile.build.tmp
endif
	mv $(TEMP_DIR)/$(CONFIG)/Dockerfile.build.tmp $(TEMP_DIR)/$(CONFIG)/Dockerfile.build

	docker buildx build \
		--pull \
		--load \
		--platform linux/$(ARCH) \
		-t $(BUILD_IMAGE) \
		-f $(TEMP_DIR)/$(CONFIG)/Dockerfile.build $(TEMP_DIR)/$(CONFIG)
	docker create --name $(BUILD_IMAGE) $(BUILD_IMAGE)
	docker export $(BUILD_IMAGE) > $(TEMP_DIR)/$(CONFIG)/$(TAR_FILE)
	docker buildx build \
		--load \
		--platform linux/$(ARCH) \
		-t $(IMAGE)-$(ARCH):$(IMAGE_VERSION) \
		-t $(IMAGE)-$(ARCH):$(TAG)-$(CONFIG) \
		-t $(IMAGE)-$(ARCH):latest-$(CONFIG) \
		$(TEMP_DIR)/$(CONFIG)
	rm -rf $(TEMP_DIR)

push: build
	docker push $(IMAGE)-$(ARCH):$(IMAGE_VERSION)
	docker push $(IMAGE)-$(ARCH):$(TAG)-$(CONFIG)
	docker push $(IMAGE)-$(ARCH):latest-$(CONFIG)

clean:
	docker rmi -f $(IMAGE)-$(ARCH):$(IMAGE_VERSION) || true
	docker rmi -f $(BUILD_IMAGE)   || true
	docker rm  -f $(BUILD_IMAGE)   || true
