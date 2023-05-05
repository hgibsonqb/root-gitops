# Exploring different ways to structure bundle

Here are three different bundle formats for the application [test](https://github.com/hgibsonqb/test)

## Test1
- Application image version is built into bundle base
- Image/Bundle deploy process
 1. Merge PR to test, build new image
 2. Merge PR to root-gitops/bundles/test1/manifests/base, this will roll out new image to all envs where revision is not pinned
 3. Merge PR to root-gitops/bundles/test1/flux/cluster/<production or env where revision is pinned>

## Test2 
- Application image version and bundle base are separate
- Image deploy process
 1. Merge PR to test, build new image
 2. Merge PR to root-gitops/bundles/test2/manifests/cluster/revision/<target env> to deploy to that env
- Bundle deploy process
 1. Merge PR to root-gitops/bundles/test2/manifests/base, root-gitops/bundles/test2/manifests/cluster/config/* this will roll out new image to all envs where root-gitops/bundles/test2/manifests/cluster/revision does not have a pinned config
 2. Merge PR to root-gitops/bundles/test2/manifests/cluster/revision/<production or env where revision is pinned> to bump config version

## Test3
- Bundle base lives in [test repo](https://github.com/hgibsonqb/test/tree/main/k8s)
- Image/Bundle deploy process
 1. Merge PR to test, build new image
 2. Merge PR to root-gitops/bundles/test3/manifests/cluster/<target env> to deploy to that env
- We'll need to set up sops in the application repo as well

## Test4
- Application image version and bundle base are separate
- Image deploy process
 1. Merge PR to test, build new image
 2. Merge PR to root-gitops/bundles/test4/manifests/cluster/<target env> to deploy to that env
- Bundle deploy process
 1. Merge PR to root-gitops/bundles/test4/manifests/base, root-gitops/bundles/test4/manifests/cluster/* this will roll out new image to all envs where root-gitops/bundles/test4/manifests/cluster/ does not have a pinned config. Only update clusters where config is not pinned/bundles should be updated.
 2. Merge PR to root-gitops/bundles/test4/manifests/cluster/<production or env where revision is pinned> to bump base version and update config

# Test5
- Use kustomize post build variable substitution to set the image version https://fluxcd.io/flux/components/kustomize/kustomization/#post-build-variable-substitution
- Application image version and bundle base are separate, but we have the ability to manage them both together by updating both flux-kustomization.patch.yaml and gitrepository.patch.yaml for an env.
- Image deploy process
 1. Merge PR to test, builds new image
 2. Merge PR to root-gitops/bundles/test5/flux/cluster/<target env>/flux-kustomization.patch.yaml
- Bundle deploy process
 1. Merge PR root-gitops/bundles/test5/manifests/* to update base and clusters for that bundle, cut a tag
 2. Merge PR to root-gitops/bundles/test5/flux/cluster/<target env>/gitrepository.patch.yaml to reference new tag
