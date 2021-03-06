APP_NAME = eventing-controller
APP_PATH = components/$(APP_NAME)
BUILDPACK = eu.gcr.io/kyma-project/test-infra/buildpack-golang-kubebuilder2:v20201009-5d8104e0
SCRIPTS_DIR = $(realpath $(shell pwd)/../..)/common/makefiles

override ENTRYPOINT = main.go

include $(SCRIPTS_DIR)/generic-make-go.mk

# DEV image is used during development workflow only
DEV_IMAGE = ko://github.com/kyma-project/kyma/components/eventing-controller/cmd/eventing-controller

NAMESPACE = "eventing"
VERIFY_IGNORE := /vendor\|/mocks

# Produce CRDs that work back to Kubernetes 1.11 (no version conversion)
CRD_OPTIONS ?= "crd:crdVersions=v1"

# Get the currently used golang install path (in GOPATH/bin, unless GOBIN is set)
ifneq (,$(shell which go))
ifeq (,$(shell go env GOBIN))
GOBIN=$(shell go env GOPATH)/bin
else
GOBIN=$(shell go env GOBIN)
endif
endif

all: manager

# Build controller binary
manager: generate-local fmt-local vet-local
	go build -o bin/controller cmd/eventing-controller/main.go

# Run against the configured Kubernetes cluster in ~/.kube/config
# TODO Useful default values have to be found.
run: export NATS_URL ?= nats://127.0.0.1:4222
run: export EVENT_TYPE_PREFIX ?= sap.kyma.custom
run: export WEBHOOK_TOKEN_ENDPOINT ?= WEBHOOK_TOKEN_ENDPOINT
run: export DOMAIN ?= example.com
run: generate-local fmt-local vet-local manifests
	go run cmd/eventing-controller/main.go

# Install CRDs into a cluster
install: manifests
	kustomize build config/crd | ko apply -f -

# Install sample CRs into a cluster
install-samples: manifests
	ko apply -f config/samples

# Uninstall sample CRs into a cluster
uninstall-samples: manifests
	ko delete -f config/samples

# Uninstall CRDs from a cluster
uninstall: manifests
	kustomize build config/crd | ko delete -f -

# Deploy eventing-controller in the configured Kubernetes cluster in ~/.kube/config for local development
deploy-local: manifests deploy-local-dry-run
	kubectl create ns ${NAMESPACE} || true
	kustomize build config/default | ko apply -f -

# Undeploy the eventing-controller
undeploy-local: manifests
	kustomize build config/default | kubectl delete -f -

# Dry run for deploying eventing-controller for local development
deploy-local-dry-run: manifests
	cd config/manager && kustomize edit set image controller=${DEV_IMAGE} && kustomize edit set namespace ${NAMESPACE}
	cd config/rbac && kustomize edit set namespace ${NAMESPACE}
	kustomize build config/default

# Generate manifests e.g. CRD, RBAC etc.
manifests-local: controller-gen-local
	$(CONTROLLER_GEN) $(CRD_OPTIONS) rbac:roleName=manager-role webhook paths="./..." output:crd:artifacts:config=config/crd/bases

# Generate manifests e.g. CRD, RBAC etc.
manifests:
	$(CONTROLLER_GEN) $(CRD_OPTIONS) rbac:roleName=manager-role webhook paths="./..." output:crd:artifacts:config=config/crd/bases

# Generate code
generate-local: manifests-local controller-gen-local
	$(CONTROLLER_GEN) object:headerFile="hack/boilerplate.go.txt" paths="./..."

# Generate code
generate: manifests
	$(CONTROLLER_GEN) object:headerFile="hack/boilerplate.go.txt" paths="./..."

verify:: mod-verify

resolve-local:
	GO111MODULE=on go mod vendor -v

ensure-local:
	@echo "Go modules present in component - omitting."

dep-status:
	@echo "Go modules present in component - omitting."

dep-status-local:
	@echo "Go modules present in component - omitting."

mod-verify-local:
	GO111MODULE=on go mod verify

test-local: generate-local fmt-local vet-local
	go test ./... -coverprofile cover.out

release: resolve generate verify build-image push-image copy-external-crds

path-to-referenced-charts:
	@echo "resources/eventing-manager/eventing-controller"

copy-crds: manifests-local
	@echo "copying Eventing CRDs to cluster-essentials"
	cp ./config/crd/bases/eventing.kyma-project.io_subscriptions.yaml ./../../resources/cluster-essentials/files/subscriptions.eventing.kyma-project.io.crd.yaml
	cp ./config/crd/bases/eventing.kyma-project.io_eventingbackends.yaml ./../../resources/cluster-essentials/files/eventingbackends.eventing.kyma-project.io.crd.yaml
	@echo "copying Eventing CRDs to installation/eventing"
	cp ./config/crd/bases/eventing.kyma-project.io_subscriptions.yaml ./../../installation/resources/crds/eventing/subscriptions.eventing.kyma-project.io.crd.yaml
	cp ./config/crd/bases/eventing.kyma-project.io_eventingbackends.yaml ./../../installation/resources/crds/eventing/eventingbackends.eventing.kyma-project.io.crd.yaml

copy-external-crds:
	@echo "copying external CRDs to config/crd/external"
	cp ../../installation/resources/crds/api-gateway/apirules.gateway.crd.yaml config/crd/external/apirules-gateway-kyma-project-io.yaml
	cp ../../installation/resources/crds/ory/oauth2clients.crd.yaml config/crd/external/hydra.ory.sh_oauth2clients.yaml

# find or download controller-gen
# download controller-gen if necessary
.PHONY: controller-gen-local
controller-gen-local:
ifeq (, $(shell which controller-gen))
	@{ \
	set -e ;\
	CONTROLLER_GEN_TMP_DIR=$$(mktemp -d) ;\
	cd $$CONTROLLER_GEN_TMP_DIR ;\
	if [ ! $(which go) ]; then \
		go mod init tmp ; \
		go get sigs.k8s.io/controller-tools/cmd/controller-gen@v0.2.5 ; \
		rm -rf $$CONTROLLER_GEN_TMP_DIR ; \
	fi; \
	}
CONTROLLER_GEN=$(GOBIN)/controller-gen
else
CONTROLLER_GEN=$(shell which controller-gen)
endif

$(eval $(call buildpack-cp-ro,resolve))
$(eval $(call buildpack-mount,mod-verify))
$(eval $(call buildpack-mount,test))
$(eval $(call buildpack-mount,controller-gen))
$(eval $(call buildpack-mount,generate))
$(eval $(call buildpack-mount,manifests))

# produce goals for each cmd program ("bin1 bin2 ...")
cmds := $(foreach dir,$(wildcard cmd/*),$(notdir $(dir)))
# produce goals to build images for each cmd program ("bin1.image bin2.image ...")
cmds_images = $(foreach cmd,$(cmds),$(cmd).image)
cmds_clean = $(foreach cmd,$(cmds),$(cmd).clean)
# produce goals to push images for each cmd program ("bin1.image.push bin2.image.push ...")
cmds_images_push = $(foreach img,$(cmds_images),$(img).push)

MOUNT_TARGETS = $(cmds)
$(foreach t,$(MOUNT_TARGETS),$(eval $(call buildpack-mount,$(t))))

.PHONY: clean
clean: $(cmds_clean) resolve_clean licenses_clean

$(cmds_clean): %.clean:
	rm -f $*

resolve_clean:
	rm -rf vendor

licenses_clean:
	rm -rf licenses

build-local: $(cmds) test-local ;

.PHONY: $(cmds_images) $(cmds_images_push)

# override image goals from common Makefile because we need to build several images
build-image: $(cmds_images) ;
push-image: $(cmds_images_push) ;

# Example:
#   make eventing-controller
GOBUILD_FLAGS := -v
PKG := github.com/kyma-project/kyma/components/eventing-controller
$(cmds): %: resolve-local
	@echo "+ Building $*"
	@CGO_ENABLED=0 go build -o $* \
		$(GOBUILD_FLAGS) \
		$(PKG)/cmd/$*

# Example:
#   make eventing-controller.image
$(cmds_images): %.image: build pull-licenses
	$(eval $@_img_name := $*)
	@echo "+ Building container image $($@_img_name)"
	docker image build -f cmd/$*/Dockerfile -t $($@_img_name) .

# Example:
#   make eventing-controller.image.push
$(cmds_images_push): %.push: %
	$(eval $@_img_name := $(subst .image,,$*))
	@echo "+ Pushing container image $($@_img_name)"
	docker image tag $($@_img_name) $(DOCKER_PUSH_REPOSITORY)$(DOCKER_PUSH_DIRECTORY)/$($@_img_name):$(TAG)
	docker image push $(DOCKER_PUSH_REPOSITORY)$(DOCKER_PUSH_DIRECTORY)/$($@_img_name):$(TAG)
