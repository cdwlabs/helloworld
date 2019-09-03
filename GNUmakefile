# -*- Mode: makefile -*-

MKFILE_PATH := $(abspath $(lastword $(MAKEFILE_LIST)))
CURRENT_DIR := $(shell dirname $(MKFILE_PATH))

# Project info
NAME=helloworld
ORG_PATH=github.com/cdwlabs
REPO_PATH=$(ORG_PATH)/$(NAME)
DOCKER_BIN := $(shell which docker)

SHELL := /bin/bash

# Populate version variables
# Add to compile time flags
VERSION := $(shell cat VERSION)
GITCOMMIT := $(shell git rev-parse --short HEAD)
GITUNTRACKEDCHANGES := $(shell git status --porcelain --untracked-files=no)
ifneq ($(GITUNTRACKEDCHANGES),)
	GITCOMMIT := $(GITCOMMIT)-dirty
endif
CTIMEVAR=-X $(REPO_PATH)/pkg/version.GITCOMMIT=$(GITCOMMIT) -X $(REPO_PATH)/pkg/version.VERSION=$(VERSION)
LDFLAGS=-ldflags '-w -extldflags "-static"'

# Docker registries
ARTIFACTORY_REG=cdwlabs-docker-local.jfrog.io/$(NAME)
DOCKER_IMAGE=$(ARTIFACTORY_REG):$(VERSION)

# If you wanna run a containerized version of this.
CNAME?=$(NAME)

# List of packages that will be built and tested
PKGS=$(shell go list ./... | grep -v /vendor)
FMT_PKGS=$(shell go list -f {{.Dir}} ./... | grep -v vendor | grep -v test | tail -n +2)

# Test options passed in via cli
TEST_PATTERN?=.
TEST_OPTIONS?=

.PHONY: compile
compile:  ## Compile for local development
	go build -a -tags netgo ${LDFLAGS} \
		-mod vendor \
		-o helloworld-api

.PHONY: docker-image
docker-image:  ## Build docker image
	@docker build \
		-t $(DOCKER_IMAGE) .

.PHONY: docker-start
docker-start: ## Run the docker image with some sane defaults
	@echo " "
	@echo "Starting '$(NAME)' image..."
	@echo " "
	$(DOCKER_BIN) run --name $(CNAME) \
		--rm -d \
    -p 8080:8080 \
	  $(DOCKER_IMAGE)

.PHONY: docker-stop
docker-stop: ## Stop the running container
	@echo " "
	@echo "Stopping '$(NAME)' image..."
	@echo " "
	@$(DOCKER_BIN) stop $(CNAME)

.PHONY: docker-ip
docker-ip: ## ip address of running container
	@docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' $(CNAME)

clean: ## Clean the project tree from binary files
	@rm -rf bin/*
	@rm -f helloworld
	@rm -f helloworld-api

.PHONY: clean-docker
clean-docker: ## Remove crusty docker images
	@echo " "
	@echo "Delete all stopped containers"
	@$(DOCKER_BIN) ps -q -f status=exited | xargs --no-run-if-empty $(DOCKER_BIN) rm || true
	@echo " "
	@echo "Prune containers"
	@$(DOCKER_BIN) image prune -f
	@echo " "
	@echo "Delete all dangling (unused) images"
	@$(DOCKER_BIN) ps -q -f dangling=true | xargs --no-run-if-empty $(DOCKER_BIN) rmi || true

.PHONY: kubectl-status
kubectl-status: ## Get the current status of our pods, services, secrets...
	@echo ""
	@echo ""
	@kubectl get secrets,pods,deployments,svc,ingress -o wide --namespace gha
	@echo ""
	@echo ""

.PHONY: check-go-version
check-go-version:
	@./scripts/check-go-version

.PHONY: bump-version
BUMP := patch
bump-version: ## Bump the version in the version file. Set BUMP to [ major | minor | patch ]
	@go get -u github.com/jessfraz/junk/sembump # update sembump tool
	$(eval NEW_VERSION = $(shell sembump --kind $(BUMP) $(VERSION)))
	@echo "Bumping VERSION.txt from $(VERSION) to $(NEW_VERSION)"
	echo $(NEW_VERSION) > VERSION.txt
	@echo "Updating links to download binaries in README.md"
	sed -i s/$(VERSION)/$(NEW_VERSION)/g README.md
	git add VERSION.txt README.md
	git commit -vsam "Bump version to $(NEW_VERSION)"
	@echo "Run 'make tag' to create and push the tag for new version $(NEW_VERSION)"

.PHONY: tag
tag: ## Create a new git tag to prepare to build a release
	git tag -sa $(VERSION) -m "$(VERSION)"
	@echo "Run git push origin $(VERSION) to push your new tag to GitHub and trigger any CI/CD build."


.PHONY: help
help:  ## Show help messages for make targets
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[32m%-30s\033[0m %s\n", $$1, $$2}'

