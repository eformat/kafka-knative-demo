# Image URL to use all building/pushing image targets
REGISTRY ?= quay.io
REPOSITORY ?= $(REGISTRY)/eformat/kafka-knative-aggregator

IMG := $(REPOSITORY):latest

# clean compile
compile:
	mvn clean package -DskipTests

# Podman Login
podman-login:
	@podman login -u $(DOCKER_USER) -p $(DOCKER_PASSWORD) $(REGISTRY)

# Build the oci image
podman-build: compile
	podman build . -t ${IMG} -f src/main/docker/Dockerfile.jvm

# Push the oci image
podman-push: podman-build
	podman push ${IMG}
