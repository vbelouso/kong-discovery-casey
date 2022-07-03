.PHONY: helm-repo
helm-repo:
	helm repo add hashicorp https://helm.releases.hashicorp.com
	helm repo update

.PHONY: deploy-vault
deploy-vault:
	helm -n vault install vault hashicorp/vault \
  	--set="global.openshift=true" \
  	--set="server.ha.enabled=true" \
  	--create-namespace
	oc project vault
	oc get pods

.PHONY: deploy-consul
deploy-consul:
	helm upgrade --install consul consul --values consul/values-custom.yaml -n consul --create-namespace
	oc project consul
	oc wait --for=condition=ready pod -l app=consul -n consul --timeout=240s
	oc get pods
	oc rsh consul-consul-server-0 consul members

.PHONY: deploy-all
deploy-all: deploy-consul deploy-vault

.PHONY: clean-vault
clean-vault:
	- helm uninstall vault -n vault
	oc delete pvc --all -n vault
.PHONY: clean-consul
clean-consul:
	- helm uninstall consul -n consul
	oc delete pvc --all -n consul
.PHONY: clean-full
clean-full: clean-vault clean-consul