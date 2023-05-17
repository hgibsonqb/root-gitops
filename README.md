# Exploring different ways to structure bundle

Here are three different bundle formats for the application [test](https://github.com/hgibsonqb/test)

## Test1
* Application image version is built into bundle base
* Image/Bundle deploy process
    1. Merge PR to test, build new image
    2. Merge PR to `root-gitops/bundles/test1/manifests/base`, this will roll out new image to all envs where revision is not pinned
    3. Merge PR to `root-gitops/bundles/test1/flux/cluster/<production or env where revision is pinned>`

## Test2 
* Application image version and bundle base are separate
* Image deploy process
    1. Merge PR to test, build new image
    2. Merge PR to `root-gitops/bundles/test2/manifests/cluster/revision/<target env>` to deploy to that env
* Bundle deploy process
    1. Merge PR to `root-gitops/bundles/test2/manifests/base`, `root-gitops/bundles/test2/manifests/cluster/config/*` this will roll out new image to all envs where `root-gitops/bundles/test2/manifests/cluster/revision` does not have a pinned config
    2. Merge PR to `root-gitops/bundles/test2/manifests/cluster/revision/<production or env where revision is pinned>` to bump config version

## Test3
* Bundle base lives in [test repo](https://github.com/hgibsonqb/test/tree/main/k8s)
* Image/Bundle deploy process
    1. Merge PR to test, build new image
    2. Merge PR to `root-gitops/bundles/test3/manifests/cluster/<target env>` to deploy to that env
* We'll need to set up sops in the application repo as well

## Test4
* Application image version and bundle base are separate
* Image deploy process
    1. Merge PR to test, build new image
    2. Merge PR to `root-gitops/bundles/test4/manifests/cluster/<target env>` to deploy to that env
* Bundle deploy process
    1. Merge PR to `root-gitops/bundles/test4/manifests/base`, `root-gitops/bundles/test4/manifests/cluster/*` this will roll out new image to all envs where `root-gitops/bundles/test4/manifests/cluster/` does not have a pinned config. Only update clusters where config is not pinned/bundles should be updated.
    2. Merge PR to `root-gitops/bundles/test4/manifests/cluster/<production or env where revision is pinned>` to bump base version and update config

## Test5
* Use kustomize post build variable substitution to set the image version https://fluxcd.io/flux/components/kustomize/kustomization/#post-build-variable-substitution
* Application image version and bundle base are separate, but we have the ability to manage them both together by updating both flux-kustomization.patch.yaml and gitrepository.patch.yaml for an env.
* Image deploy process
    1. Merge PR to test, builds new image
    2. Merge PR to `root-gitops/bundles/test5/flux/cluster/<target env>/flux-kustomization.patch.yaml`
* Bundle deploy process
    1. Merge PR `root-gitops/bundles/test5/manifests/*` to update base and clusters for that bundle, cut a tag
    2. Merge PR to `root-gitops/bundles/test5/flux/cluster/<target env>/gitrepository.patch.yaml` to reference new tag

## IUA tabletop exercise

We keep configuration for all our environments in the same repository on the same branch, which the image update automation controller has write access to. We give devs admin access in staging/devlab including the ability to suspend the main flux controller, and create iua resources. Does this mean someone could use iua from devlab/staging to overwrite configuration in production?

We could mitigate this by moving the production config to it's own repo like in topaz.

### Scenario 1
The root flux controller is not suspended. Attacker creates new imagerepository hosting malicious image, new image policy, new iua in which points to the production flux clusteroverlay (where the image version is set) and the main branch in devlab. 

Nothing happens because there the magic comment in the production cluster overlay indicates which image policy updates should come from, and it is not the new policy.

### Scenario 2
The root flux controller is not suspended. Attacker creates new imagerepository hosting malicious image, updates the existing image policy (which is named the same in devlab as in production), new iua which points to the production flux clusteroverlay (where the image version is set) and the main branch in devlab. 

The policy updated to pull in the new image. In this case nothing happened because the environments are separated by namespace instead of by cluster, and the magic comment references the namespace as well as the policy name, however in a multi-cluster setup the namespaces are the same. Eventually the flux controller overwrote the edited registry but we can't rely on that.

We could give the policy a different name in each environment, but someone could simply create that policy in the devlab environment. Same with having a different namespace per environment, someone could simply create the namespace, create the iua resources. 

To test what would happen if the policy name was correct, I added the bad resources to the production namespace, then edited the production registry policy to reference the bad image repo, and it did update the main branch, bypassing branch protection. Eventually flux overwrote the imagepolicy, and because the bad iua referencing the main branch was still there, it was set back to the correct image on main. 

I was able to mitigate this by configuring the main branch protection rule with "Do not allow bypassing the above settings -> The above settings will apply to administrators and custom roles with the "bypass branch protections" permission." but that means human admins on the repo also can't bypass the setting. 
```
 Warning  error  9s (x7 over 61s)  image-automation-controller  command error on refs/heads/main: protected branch hook declined
``` 
This exploit relies on the attacker having admin/edit permissions in a non-prod cluster. They read access to the root-gitops repo would be useful although they could potentially infer configuration (target path, policy name, etc) from the non-prod configuration. 

