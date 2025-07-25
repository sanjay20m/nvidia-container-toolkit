# Copyright (c) 2021, NVIDIA CORPORATION.  All rights reserved.
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

BUILD_MULTI_ARCH_IMAGES ?= false
DOCKER ?= docker
REGCTL ?= regctl

BUILDX =
ifeq ($(BUILD_MULTI_ARCH_IMAGES),true)
BUILDX = buildx
endif

MKDIR    ?= mkdir
DIST_DIR ?= $(CURDIR)/dist

##### Global variables #####
include $(CURDIR)/versions.mk

IMAGE_VERSION := $(VERSION)

IMAGE_TAG ?= $(VERSION)
IMAGE = $(IMAGE_NAME):$(IMAGE_TAG)

OUT_IMAGE_NAME ?= $(IMAGE_NAME)
OUT_IMAGE_VERSION ?= $(IMAGE_VERSION)
OUT_IMAGE_TAG = $(OUT_IMAGE_VERSION)
OUT_IMAGE = $(OUT_IMAGE_NAME):$(OUT_IMAGE_TAG)

##### Public rules #####
DEFAULT_PUSH_TARGET := application
DISTRIBUTIONS := $(DEFAULT_PUSH_TARGET)

META_TARGETS := packaging

IMAGE_TARGETS := $(patsubst %,image-%,$(DISTRIBUTIONS) $(META_TARGETS))
BUILD_TARGETS := $(patsubst %,build-%,$(DISTRIBUTIONS) $(META_TARGETS))
PUSH_TARGETS := $(patsubst %,push-%,$(DISTRIBUTIONS) $(META_TARGETS))
TEST_TARGETS := $(patsubst %,test-%,$(DISTRIBUTIONS))

.PHONY: $(DISTRIBUTIONS) $(PUSH_TARGETS) $(BUILD_TARGETS) $(TEST_TARGETS)

ifneq ($(BUILD_MULTI_ARCH_IMAGES),true)
include $(CURDIR)/deployments/container/native-only.mk
else
include $(CURDIR)/deployments/container/multi-arch.mk
endif

# Define the push targets
$(PUSH_TARGETS): push-%:
	$(CURDIR)/scripts/publish-image.sh $(IMAGE) $(OUT_IMAGE)

DOCKERFILE = $(CURDIR)/deployments/container/Dockerfile

# For packaging targets we set the output image tag to include the -packaging suffix.
%-packaging: INTERMEDIATE_TARGET := --target=packaging
%-packaging: IMAGE_TAG = $(IMAGE_VERSION)-packaging
%-packaging: OUT_IMAGE_TAG = $(IMAGE_VERSION)-packaging

ARTIFACTS_ROOT ?= $(shell realpath --relative-to=$(CURDIR) $(DIST_DIR))

# Use a generic build target to build the relevant images
$(IMAGE_TARGETS): image-%: $(ARTIFACTS_ROOT)
	DOCKER_BUILDKIT=1 \
		$(DOCKER) $(BUILDX) build --pull \
		--provenance=false --sbom=false \
		$(DOCKER_BUILD_OPTIONS) \
		$(DOCKER_BUILD_PLATFORM_OPTIONS) \
		$(INTERMEDIATE_TARGET) \
		--tag $(IMAGE) \
		--build-arg ARTIFACTS_ROOT="$(ARTIFACTS_ROOT)" \
		--build-arg GOLANG_VERSION="$(GOLANG_VERSION)" \
		--build-arg PACKAGE_DIST_DEB="$(PACKAGE_DIST_DEB)" \
		--build-arg PACKAGE_DIST_RPM="$(PACKAGE_DIST_RPM)" \
		--build-arg PACKAGE_VERSION="$(PACKAGE_VERSION)" \
		--build-arg VERSION="$(VERSION)" \
		--build-arg GIT_COMMIT="$(GIT_COMMIT)" \
		--build-arg GIT_COMMIT_SHORT="$(GIT_COMMIT_SHORT)" \
		--build-arg GIT_BRANCH="$(GIT_BRANCH)" \
		--build-arg SOURCE_DATE_EPOCH="$(SOURCE_DATE_EPOCH)" \
		-f $(DOCKERFILE) \
		$(CURDIR)


PACKAGE_DIST_DEB = ubuntu18.04
# TODO: This needs to be set to centos8 for ppc64le builds
PACKAGE_DIST_RPM = centos7

# Handle the default build target.
.PHONY: build push
build: build-$(DEFAULT_PUSH_TARGET)
push: push-$(DEFAULT_PUSH_TARGET)

# Test targets
TEST_CASES ?= docker crio containerd
$(TEST_TARGETS): test-%:
	TEST_CASES="$(TEST_CASES)" bash -x $(CURDIR)/test/container/main.sh run \
		$(CURDIR)/shared-$(*) \
		$(IMAGE) \
			--no-cleanup-on-error

.PHONY: test-packaging
test-packaging: DIST = packaging
test-packaging:
	@echo "Testing package image contents"
	@$(DOCKER) run --rm $(IMAGE) test -d "/artifacts/packages/centos7/aarch64" || echo "Missing centos7/aarch64"
	@$(DOCKER) run --rm $(IMAGE) test -d "/artifacts/packages/centos7/x86_64" || echo "Missing centos7/x86_64"
	@$(DOCKER) run --rm $(IMAGE) test -d "/artifacts/packages/centos8/ppc64le" || echo "Missing centos8/ppc64le"
	@$(DOCKER) run --rm $(IMAGE) test -d "/artifacts/packages/ubuntu18.04/amd64" || echo "Missing ubuntu18.04/amd64"
	@$(DOCKER) run --rm $(IMAGE) test -d "/artifacts/packages/ubuntu18.04/arm64" || echo "Missing ubuntu18.04/arm64"
	@$(DOCKER) run --rm $(IMAGE) test -d "/artifacts/packages/ubuntu18.04/ppc64le" || echo "Missing ubuntu18.04/ppc64le"
