# Exploring different ways to managed bundles

Here are three different bundle formats for the application [test](https://github.com/hgibsonqb/test)

## Test1
- Application image version is built into bundle base
- Image/Bundle deploy process
 1. Merge PR to test, build new image
 2. Merge PR to root-gitops/bundle/test1/manifests/base, this will roll out new image to all envs where revision is not pinned
 3. Merge PR to root-gitops/bundle/test1/flux/cluster/<production or env where revision is pinned>

## Test2 
- Application image version and bundle base are separate
- Image deploy process
 1. Merge PR to test, build new image
 2. Merge PR to root-gitops/bundle/test2/manifests/cluster/revision/<target nev> to deploy to that env
- Bundle deploy process
 1. Merge PR to root-gitops/bundle/test2/manifests/base, root-gitops/bundle/test2/manifests/cluster/config/* this will roll out new image to all envs where root-gitops/bundle/test2/manifests/cluster/revision does not have a pinned config
 2. Merge PR to root-gitops/bundle/test1/manifests/cluster/revision/<production or env where revision is pinned> to bump config version

## Test3
- Bundle base lives in [test repo](https://github.com/hgibsonqb/test/tree/main/k8s)
- Image/Bundle deploy process
 1. Merge PR to test, build new image
 2. Merge PR to root-gitops/bundle/test2/manifests/cluster/<target nev> to deploy to that env
