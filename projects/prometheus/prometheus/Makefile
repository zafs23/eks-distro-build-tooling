REPO?=prometheus
CLONE_URL?=https://github.com/prometheus/$(REPO).git

# Using GIT_COMMIT instead of a TAG as we need recent changes made to dockerfile
# This also avoids for us having to maintain a seperate dockerfile
GIT_COMMIT?=$(shell cat GIT_COMMIT)

ifeq ("$(CLONE_URL)","")
	$(error No clone url was provided.)
endif

ifeq ("$(REPO)","")
	$(error No repository name was provided.)
endif

ifeq ("$(GIT_COMMIT)","")
	$(error No git commit was provided.)
endif


AWS_ACCOUNT_ID?=$(shell aws sts get-caller-identity --query Account --output text)
AWS_REGION?=us-west-2

IMAGE_REPO?=$(AWS_ACCOUNT_ID).dkr.ecr.$(AWS_REGION).amazonaws.com
IMAGE_NAME?=prometheus/prometheus
# This tag is overwritten in the prow job to point to the commit hash
IMAGE_TAG?=latest
IMAGE?=$(IMAGE_REPO)/$(IMAGE_NAME):$(IMAGE_TAG)
IMAGE_OS?="linux"
IMAGE_ARCH?="amd64"

MAKE_ROOT=$(shell cd "$(shell dirname "${BASH_SOURCE[0]}")" && pwd -P)

.PHONY: build-dependencies
build-dependencies:
	curl -sL https://dl.yarnpkg.com/rpm/yarn.repo -o /etc/yum.repos.d/yarn.repo
	curl -sL https://rpm.nodesource.com/setup_14.x | bash -
	yum install nodejs yarn bzip2 -y

.PHONY: clone
clone: clean
	git clone $(CLONE_URL) $(REPO)
	cd $(REPO) && git checkout $(GIT_COMMIT)

.PHONY: buildkit-check
buildkit-check:
	$(MAKE_ROOT)/../../../scripts/buildkit_check.sh

.PHONY: binaries
binaries:
	make -C $(REPO) build

.PHONY: move-binaries
move-binaries:
	mkdir -p $(REPO)/.build/$(IMAGE_OS)-$(IMAGE_ARCH)
	mv $(REPO)/prometheus $(REPO)/.build/$(IMAGE_OS)-$(IMAGE_ARCH)/
	mv $(REPO)/promtool $(REPO)/.build/$(IMAGE_OS)-$(IMAGE_ARCH)/

.PHONY: npm_licenses
npm_licenses:
	make -C $(REPO) npm_licenses

.PHONY: local-images
local-images: clone binaries move-binaries npm_licenses buildkit-check
	buildctl \
		build \
		--frontend dockerfile.v0 \
		--opt platform=linux/amd64 \
		--local dockerfile=./$(REPO) \
		--local context=./$(REPO) \
		--output type=oci,oci-mediatypes=true,name=$(IMAGE),dest=/tmp/prometheus.tar

.PHONY: images
images: clone binaries move-binaries npm_licenses buildkit-check
	buildctl \
		build \
		--frontend dockerfile.v0 \
		--opt platform=linux/amd64 \
		--local dockerfile=./$(REPO) \
		--local context=./$(REPO) \
		--output type=image,oci-mediatypes=true,name=$(IMAGE),push=true

.PHONY: build
build: build-dependencies local-images

.PHONY: release
release: build-dependencies images

.PHONY: clean
clean:
	rm -rf $(REPO)
