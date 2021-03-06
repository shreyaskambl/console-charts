# Using ?= cuz enterprise-suite-latest overrides this
CHART ?= enterprise-suite

RELEASE_NAME := es
NAMESPACE := lightbend

define banner
	$(info === $@)
endef

# Note: We may want to require explicit lists.  As implemented, we may end
# up with random cruft in the packages...
COMPONENTS := $(wildcard */*)
SUBCOMPONENTS := $(wildcard */*/*)

HELM_CHARTS_DIR := ..
SCRIPTS_DIR := $(HELM_CHARTS_DIR)/scripts

#####
# Note:  These are tweaked by the hacky enterprise-suite-latest Makefile...
VERSION ?= $(shell $(SCRIPTS_DIR)/export-chart-version.sh $(CHART))
RELEASE = $(CHART)-$(VERSION)
CHART_DIR = .
ALL_YAML ?= all.yaml
#####

.PHONY: all build
all: lint build  ## lint then build chart
build: init package allyaml  ## Build chart package (and all.yaml)

# This target required by top-level helm-charts Makefile.  Will create chart tarball in docs directory
.PHONY: package
package: $(HELM_CHARTS_DIR)/docs/$(RELEASE).tgz  ## Build chart tarball

.PHONY: script-tests
script-tests: ## Run installer tests
	$(CHART_DIR)/tests/lib/bats/bats $(CHART_DIR)/tests
	python $(CHART_DIR)/scripts/lbc_test.py

.PHONY: minikube-backend-tests
minikube-backend-tests: ## Run the backend tests against minikube
	@echo Running minikube tests
	( cd $(CHART_DIR)/tests ; ./smoketest_minikube.sh ; )

.PHONY: openshift-backend-tests
openshift-backend-tests: ## Run the backend tests against openshift
	@echo Running openshift tests
	( cd $(CHART_DIR)/tests ; ./smoketest_openshift.sh ; )

.PHONY: frontend-tests
frontend-tests: ## Run the frontend tests against minikube
	$(CHART_DIR)/scripts/run-e2e-tests.sh

# This target required by top-level helm-charts Makefile. It's not used by travis.
# It only performs lightweight checks.
.PHONY: test
test: lint script-tests ## Run unit tests

# Note: This works for enterprise-suite and allows us to define a latest-specific version in enterprise-suite-latest
$(HELM_CHARTS_DIR)/docs/$(filter-out %-latest,$(CHART))-$(VERSION).tgz: $(COMPONENTS) $(SUBCOMPONENTS)
	$(call banner)
	helm package -d $(HELM_CHARTS_DIR)/docs $(CHART_DIR)

.PHONY: lint
lint: lint-helm lint-json lint-promql  ## Run all lint tests on chart files

.PHONY: lint-helm
lint-helm:  ## Run helm lint on chart files
	helm lint $(CHART_DIR)

.PHONY: lint-json
lint-json:  ## Test json files are well-formed
	find $(CHART_DIR) -name \*.json | xargs -tn 1 jq . >/dev/null

.PHONY: lint-promql
lint-promql:  ## Test promql bits are well-formed
	$(CHART_DIR)/scripts/validate-promql.sh

.PHONY: lint-allyaml
lint-allyaml: allyaml  ## Check that all.yaml is valid (Need Minikube running)
	kubectl --context=minikube create --dry-run --validate -f $(HELM_CHARTS_DIR)/docs/$(RELEASE_NAME)/$(ALL_YAML)

.PHONY: clean
clean::  ## Delete make artifacts
	rm -f $(HELM_CHARTS_DIR)/docs/$(RELEASE_NAME)/$(ALL_YAML)

.PHONY: allyaml
allyaml: $(HELM_CHARTS_DIR)/docs/$(RELEASE_NAME)/$(ALL_YAML)  ## Build all.yaml file

$(HELM_CHARTS_DIR)/docs/$(RELEASE_NAME)/$(ALL_YAML): $(HELM_CHARTS_DIR)/docs/$(RELEASE).tgz
	$(call banner)
	mkdir -p $(HELM_CHARTS_DIR)/docs/$(RELEASE_NAME)
	helm --namespace=$(NAMESPACE) template $< --set exposeServices=NodePort > $@

.PHONY: install-dev
install-dev: ## Install local chart directory to a local minikube.
	-kubectl create namespace lightbend
	-kubectl create serviceaccount --namespace lightbend tiller
	-kubectl create clusterrolebinding lightbend:tiller --clusterrole=cluster-admin \
		--serviceaccount=lightbend:tiller
	helm init --wait --service-account tiller --upgrade --tiller-namespace=lightbend
	TILLER_NAMESPACE=lightbend scripts/lbc.py install --local-chart=. -- --set exposeServices=NodePort --wait

.PHONY: help
help:  ## Print help for targets
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(lastword $(MAKEFILE_LIST)) | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'
