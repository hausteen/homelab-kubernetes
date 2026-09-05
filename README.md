Welcome! This is my "homelab".

## Background

As I have learned Kubernetes over time, who knows how many times I have restructured my homelab, broken it, fixed it, renamed it, merged it, split it, the list goes on. Its all part of the learning process as I have got my hands dirty.

This is my best version so far and it will keep becoming better.

## Repository structure

### Clusters

This is a monorepo.

There are many clusters at play here and all are found under the "clusters/overlays" folder.

By default, each cluster is designed to be fully independent from the other clusters. In addition, I can create a shared "base" of applications for multiple clusters if I want/need to. Normally, I'll avoid it since I test with a bunch of different clusters, with wildly different software and configurations. Kustomize allows you base off any kustomization folder; so I can base off any cluster in the clusters/overlays/ or clusters/base folder.

### File / folder structure

```
clusters
    base
    overlays
        <cluster name>
            kustomization.yaml
            patches
            resources
                kustomization.yaml
manifests
    <software name>
        preinstall<optional number>
            base
            overlays
                <cluster name>
                    kustomization.yaml
                    patches
                    resources
        install
            base
            overlays
                <cluster name>
                    kustomization.yaml
                    patches
                    resources
        postinstall<optional number>
            base
            overlays
                <cluster name>
                    kustomization.yaml
                    patches
                    resources
```

Question: Where to put files?
Answer: Try and keep all code for an application in the same manifest's folder so that way its easy to remove any piece of code. For example: Keycloak needs a postgress database, a certificate, a realm file, a httproute, etc. All go in the manifests/keycloak folder.

Question: Why are there so many base and overlay folders?
Answer: This is strategic. It allows me to:
- set up fully independent clusters.
- control software dependency installation order easily by using software name and time (the preinstall, install, postinstall folders)
- set up shared bases when needed (a cluster doesn't have to use the shared base if it doesn't want to).
- base off other overlays for quick testing
- it allows me to keep installation and configuration in the same "mainfests" folder, split by software name. It's easier for me to mentally reason through.

### Naming Conventions

- Files: kind-application-component-subcomponent-purpose-version
- Manifests: application-component-subcomponent-purpose-version

Go from left to right only as far as needed to make it clear and unique. I try to follow this convention as best I can, but sometimes it doesn't fit cleanly so I do the best I can.

### Gitops

The repo is currently using Flux CD. I really like the mental model that Flux uses (its a beautiful design in my opinion). It solves many of my problems that I had with Argocd.
I did steal Argocd's sync phases / waves idea since that's useful (preinstall, install, postinstall is what I'm calling the phases). It allows me to define dependencies that Flux CD can enforce easily.

### How the codebase all fits together

During cluster bootstrap, I install Flux Operator. I give it a Flux Instance file and I give it my Github app credential. This Flux Instance file creates a Flux gitrepository and Flux kustomization object (both called flux-system), which point to my github repo and clusters/overlays/name folder respectively. The clusters/overlays/name/kustomization.yaml file points at the clusters/overlays/name/resources/kustomization.yaml file. That clusters/overlays/name/resources/kustomization.yaml file points at the manifests/name/* folders in the correct order to deploy everything.

Read the clusters/overlays/name/resources/kustomization.yaml file in order to see what is in each cluster. It should match the below tables for each cluster.

If the cluster installs trust-manager and sets up a trust bundle automatically, you need to perform a certificate rotation after the root certificate expires in 9.5 years. The command is in the trust bundle file found in trust-manager's folder.

## Installation

### For "clusters/overlays/lab":

| Name                                   | Path (manifests/.../overlays/lab)      | Depends On |
| -------------------------------------- | -------------------------------------- | ---------- |
| fluxoperator-install                   | fluxoperator/install                   | nothing |
| fluxoperator-postinstall               | fluxoperator/postinstall               | fluxoperator-install |
| cilium-install                         | cilium/install                         | fluxoperator-postinstall |
| cilium-postinstall1                    | cilium/postinstall1                    | cilium-install |
| cilium-postinstall2                    | cilium/postinstall2                    | cilium-install, nginxgatewayfabric-postinstall |
| certmanager-install                    | certmanager/install                    | cilium-install |
| certmanager-postinstall1               | certmanager/postinstall1               | certmanager-install |
| certmanager-postinstall2               | certmanager/postinstall2               | certmanager-postinstall1 |
| trustmanager-install                   | trustmanager/install                   | certmanager-install |
| trustmanager-postinstall               | trustmanager/postinstall               | certmanager-postinstall2, trustmanager-install |
| longhorn-install                       | longhorn/install                       | cilium-install |
| longhorn-postinstall                   | longhorn/postinstall                   | longhorn-install, nginxgatewayfabric-postinstall |
| nginxgatewayfabric-preinstall          | nginxgatewayfabric/preinstall          | fluxoperator-postinstall |
| nginxgatewayfabric-install             | nginxgatewayfabric/install             | cilium-install, nginxgatewayfabric-preinstall |
| nginxgatewayfabric-postinstall         | nginxgatewayfabric/postinstall         | certmanager-postinstall1, nginxgatewayfabric-install |
| coredns-preinstall                     | coredns/preinstall                     | cilium-install |
| coredns-install                        | coredns/install                        | coredns-preinstall |
| coredns-postinstall                    | coredns/postinstall                    | coredns-install, nginxgatewayfabric-postinstall |
| istio-install                          | istio/install                          | cilium-install |
| istio-postinstall                      | istio/postinstall                      | istio-install |
| cloudnativepg-install                  | cloudnativepg/install                  | cilium-install |
| externalsecretsoperator-install        | externalsecretsoperator/install        | cilium-install |
| externalsecretsoperator-postinstall    | externalsecretsoperator/postinstall    | externalsecretsoperator-install, openbao-install, trustmanager-postinstall |
| openbao-preinstall1                    | openbao/preinstall1                    | certmanager-postinstall1, externalsecretsoperator-install |
| openbao-preinstall2                    | openbao/preinstall2                    | openbao-preinstall1, cloudnativepg-install, longhorn-install |
| openbao-install                        | openbao/install                        | openbao-preinstall2, trustmanager-postinstall |
| openbao-postinstall                    | openbao/postinstall                    | openbao-install, nginxgatewayfabric-postinstall |
| pocketid-preinstall1                   | pocketid/preinstall1                   | certmanager-postinstall1 |
| pocketid-preinstall2                   | pocketid/preinstall2                   | pocketid-preinstall1, cloudnativepg-install, longhorn-install |
| pocketid-install                       | pocketid/install                       | pocketid-preinstall2 |
| pocketid-postinstall                   | pocketid/postinstall                   | pocketid-install, nginxgatewayfabric-postinstall |
| kubebench-install                      | kubebench/install                      | cilium-install (technically all other software. runs as daily cronjob) |

### For "clusters/overlays/home":

| Name                                   | Path (manifests/.../overlays/home)      | Depends On |
| -------------------------------------- | -------------------------------------- | ---------- |
| fluxoperator-install                   | fluxoperator/install                   | nothing |
| fluxoperator-postinstall               | fluxoperator/postinstall               | fluxoperator-install |
| cilium-install                         | cilium/install                         | fluxoperator-postinstall |
| cilium-postinstall1                    | cilium/postinstall1                    | cilium-install |
| cilium-postinstall2                    | cilium/postinstall2                    | cilium-install, nginxgatewayfabric-postinstall |
| certmanager-install                    | certmanager/install                    | cilium-install |
| certmanager-postinstall1               | certmanager/postinstall1               | certmanager-install |
| certmanager-postinstall2               | certmanager/postinstall2               | certmanager-postinstall1 |
| trustmanager-install                   | trustmanager/install                   | certmanager-install |
| trustmanager-postinstall               | trustmanager/postinstall               | certmanager-postinstall2, trustmanager-install |
| nginxgatewayfabric-preinstall          | nginxgatewayfabric/preinstall          | fluxoperator-postinstall |
| nginxgatewayfabric-install             | nginxgatewayfabric/install             | cilium-install, nginxgatewayfabric-preinstall |
| nginxgatewayfabric-postinstall         | nginxgatewayfabric/postinstall         | certmanager-postinstall1, nginxgatewayfabric-install |
| coredns-preinstall                     | coredns/preinstall                     | cilium-install |
| coredns-install                        | coredns/install                        | coredns-preinstall |
| coredns-postinstall                    | coredns/postinstall                    | coredns-install, nginxgatewayfabric-postinstall |
