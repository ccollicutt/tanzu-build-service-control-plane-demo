# Tanzu Build Service Control Plane Demo

The Tanzu Build Service (TBS) automatically creates container images without Dockerfiles. TBS uses buildpacks which, unlike Dockerfiles, don't smash together the operating system, dependencies, and application code. Buildpacks keep these pieces separate, which means we can easily "patch" the operating system without having to worry about affecting the application.

What's more, because by using TBS we standardize the container images, it becomes very easy to fix every single container running, should there be some kind of security issue.

Finally, with these things in mind, TBS becomes a control plane for your container images.

## What will we do?

In this demo we will simulate "patching" a container image, or set of images, with a newer underying operating system. The idea being that the "older" images are insecure simply because they have CVEs that haven't been fixed, where the newer images have had the CVEs fixed.

Usually it is quite difficult to patch every single image in a Kubernetes cluster, because they are typically all built by different teams, have different Dockerfiles, etc, etc.

## Requirements 

* TBS is already installed and configured
* `pivnet` is installed and can access the Tanzu network

## Clone this repository

```
git clone https://github.com/ccollicutt/tanzu-build-service-control-plane-demo
cd tanzu-build-service-control-plane-demo
```

## Using TBS as a Container Image Control Plane

### Download the descriptor files.

First, let's get a quiet old descriptor.

```
pivnet download-product-files --product-slug='tbs-dependencies' --release-version='100.0.55' --product-file-id=853492
```

Now a much newer descriptor, which will have a newer image (ie. one with fewer vulnerabilities--simply because it's newer and the underlying images are updated every couple of days).

```
pivnet download-product-files --product-slug='tbs-dependencies' --release-version='100.0.90' --product-file-id=919092
```

eg. output:

```
$ pivnet download-product-files --product-slug='tbs-dependencies' --release-version='100.0.90' --product-file-id=919092
2021/04/01 16:07:31 Downloading 'descriptor-100.0.90.yaml' to 'descriptor-100.0.90.yaml'
 3.38 KiB / 3.38 KiB [==============================================] 100.00% 0s
2021/04/01 16:07:32 Verifying SHA256
2021/04/01 16:07:32 Successfully verified SHA256
```

## Import the Descriptors

>NOTE: This can take 10-15 minutes or more to complete.

```
kp import -f descriptor-100.0.55.yaml
kp import -f descriptor-100.0.90.yaml
```

eg. output for the `descriptor-100.0.90.yaml` descriptor file.

```
$ kp import -f descriptor-100.0.90.yaml
Importing ClusterStore 'default'...
	Uploading 'TBS_REPOSITORY/tanzu-buildpacks_go@sha256:2146fb2de550f162a49b5f4c2d6183f8f7c63a9a7c432635f65785f4e0330bff'
        Buildpackage already exists in the store
	Uploading 'TBS_REPOSITORY/tanzu-buildpacks_java@sha256:f440c19fe7742b01f253c986d801eb243690acb42505af1b7509c764c2f10bcb'
        Added Buildpackage
	Uploading 'TBS_REPOSITORY/tanzu-buildpacks_nodejs@sha256:100fc4c332c6207a8c79c33b8dab0df4adee31d0a7b21edaf5ecbf50395f56a2'
        Buildpackage already exists in the store
	Uploading 'TBS_REPOSITORY/tanzu-buildpacks_java-native-image@sha256:15497d2f1d5fb0c7a15deb695882d8b4a9b412dceadf0d0d999c1df52aed2732'
        Added Buildpackage
	Uploading 'TBS_REPOSITORY/tanzu-buildpacks_dotnet-core@sha256:024607974e03deab30b64478a41b91233bec5d6285c75dbaf1398a10bb47cd58'
        Buildpackage already exists in the store
	Uploading 'TBS_REPOSITORY/tanzu-buildpacks_php@sha256:fd5abb334f4adbcf46f42977992af145db04fb262d4c516ff4264f94e1fcd689'
        Buildpackage already exists in the store
	Uploading 'TBS_REPOSITORY/tanzu-buildpacks_nginx@sha256:e67d5cd2e5240a9eb7a899b9b5d979ad85d0cf6c8182e15424516dac9f577371'
        Buildpackage already exists in the store
	Uploading 'TBS_REPOSITORY/tanzu-buildpacks_httpd@sha256:34989fb8e264ccaea7916a9017b306d621b017920f71439fc515164ac0484cf5'
        Buildpackage already exists in the store
	Uploading 'TBS_REPOSITORY/paketo-buildpacks_procfile@sha256:bf6a4265db23ae25b34d402cd24e04c36dccdf24d6a6b9297f1d154a9d0b8062'
        Buildpackage already exists in the store
Importing ClusterStack 'tiny'...
Uploading to 'TBS_REPOSITORY'...
	Uploading 'TBS_REPOSITORY/build@sha256:b642dd2d4ec4617e683b3e510bda20e6027979f1653f84801a876658ce27ca86'
        Uploading 'TBS_REPOSITORY/run@sha256:f35345771f18118a5632def411d3c3e688384e603c35e899d7b7f78b516f3539'
Importing ClusterStack 'base'...
Uploading to 'TBS_REPOSITORY'...
	Uploading 'TBS_REPOSITORY/build@sha256:c14014504c2fd0195f27ba8ae972e60c403dd9384f15e5f4a2aed454791f8dc9'
        Uploading 'TBS_REPOSITORY/run@sha256:0bf521429c5fac06616ef542da735f9e34c4997cc5d5987242eb7199b04ac923'
Importing ClusterStack 'full'...
Uploading to 'TBS_REPOSITORY'...
	Uploading 'TBS_REPOSITORY/build@sha256:d0e8bfb686fd3bc5463bd79f33a31d1dc9528a37f88ffc26a64ff0949173223d'
        Uploading 'TBS_REPOSITORY/run@sha256:84eff69367995f990875cdbd47b2dc427c0bac27ae15f2a115090962e6969421'
Importing ClusterStack 'default'...
Uploading to 'TBS_REPOSITORY'...
	Uploading 'TBS_REPOSITORY/build@sha256:c14014504c2fd0195f27ba8ae972e60c403dd9384f15e5f4a2aed454791f8dc9'
        Uploading 'TBS_REPOSITORY/run@sha256:0bf521429c5fac06616ef542da735f9e34c4997cc5d5987242eb7199b04ac923'
Importing ClusterBuilder 'base'...
Importing ClusterBuilder 'full'...
Importing ClusterBuilder 'tiny'...
Importing ClusterBuilder 'default'...
Imported resources
```

### Build a "insecure" Clusterbuildler

Set your repository.

>NOTE: You would have a repository for TBS already because we are assuming it's already installed.

```
export TBS_REPOSITORY=<your container image registry>
```

Create clusterstack based on "insecure" (ie. older) images, which are part of the `descriptor-100.0.55.yaml` descriptor.

```
$ grep cf87e6b7e69c5394440c11d41c8d46eade57d13236e4fb79c80227cc15d33abf ./descriptor-100.0.55.yaml 
    image: registry.pivotal.io/tbs-dependencies/build-full@sha256:cf87e6b7e69c5394440c11d41c8d46eade57d13236e4fb79c80227cc15d33abf
$ grep 52a9a0002b16042b4d34382bc244f9b6bf8fd409557fe3ca8667a5a52da44608 descriptor-100.0.55.yaml 
    image: registry.pivotal.io/tbs-dependencies/run-full@sha256:52a9a0002b16042b4d34382bc244f9b6bf8fd409557fe3ca8667a5a52da44608
```

Create the stack.

```
kp clusterstack create demo-stack  \
  --build-image $TBS_REPOSITORY/build@sha256:cf87e6b7e69c5394440c11d41c8d46eade57d13236e4fb79c80227cc15d33abf \
  --run-image $TBS_REPOSITORY/run@sha256:52a9a0002b16042b4d34382bc244f9b6bf8fd409557fe3ca8667a5a52da44608
```

eg. output:

```
$ kp clusterstack create demo-stack  --build-image $TBS_REPOSITORY/build@sha256:cf87e6b7e69c5394440c11d41c8d46eade57d13236e4fb79c80227cc15d33abf   --run-image $TBS_REPOSITORY/run@sha256:52a9a0002b16042b4d34382bc244f9b6bf8fd409557fe3ca8667a5a52da44608
Creating ClusterStack...
Uploading to 'TBS_REPOSITORY'...
	Uploading 'TBS_REPOSITORY/build@sha256:cf87e6b7e69c5394440c11d41c8d46eade57d13236e4fb79c80227cc15d33abf'
        Uploading 'TBS_REPOSITORY/run@sha256:52a9a0002b16042b4d34382bc244f9b6bf8fd409557fe3ca8667a5a52da44608'
ClusterStack "demo-stack" created
```

And create a customer cluster builder that uses that stack.

```
kp clusterbuilder create demo-cluster-builder \
  --tag $TBS_REPOSITORY/demo-cluster-builder \
  --order demo-cluster-builder-order.yaml \
  --stack demo-stack \
  --store default
```

eg. output:

```
$ kp clusterbuilder create demo-cluster-builder \
>   --tag $TBS_REPOSITORY/demo-cluster-builder \
>   --order demo-cluster-builder-order.yaml \
>   --stack demo-stack \
>   --store default
ClusterBuilder "demo-cluster-builder" created
```

## Build an Image

First, identify where you are going to push the image once TBS has built it. 

>NOTE: It will probably be the same as TBS is using, but doesn't have to be.

```
export TBS_REPOSITORY=<your container image registry>
```

Now build the image.

```
kp image create demo-image \
--tag $TBS_REPOSITORY/demo-image \
--git https://github.com/ccollicutt/tbs-sample-apps/ \
--sub-path sample-apps/go \
--git-revision main \
--cluster-builder demo-cluster-builder
```

Watch the logs of the image being built.

```
$ kp build logs demo-image
===> PREPARE
Build reason(s): CONFIG
CONFIG:
	resources: {}
	- source: {}
	+ source:
	+   git:
	+     revision: 513fd440fec8e79bdc78b500b37f6b23881af951
	+     url: https://github.com/ccollicutt/tbs-sample-apps/
	+   subPath: sample-apps/go
Loading secret for "gcr.io" from secret "gcr-secret" at location "/var/build-secrets/gcr-secret"
Loading secret for "harbor.shared.demo.globalbanque.com" from secret "harbor-creds" at location "/var/build-secrets/harbor-creds"
Cloning "https://github.com/ccollicutt/tbs-sample-apps/" @ "513fd440fec8e79bdc78b500b37f6b23881af951"...
Successfully cloned "https://github.com/ccollicutt/tbs-sample-apps/" @ "513fd440fec8e79bdc78b500b37f6b23881af951" in path "/workspace"
===> DETECT
tanzu-buildpacks/go-dist  0.1.5
tanzu-buildpacks/go-build 0.1.0
===> ANALYZE
Previous image with name "TBS_REPOSITORY/demo-image" not found
===> RESTORE
===> BUILD
Tanzu Go Distribution Buildpack 0.1.5
  Resolving Go version
    Candidate version sources (in priority order):
      <unknown> -> ""

    Selected Go version (using <unknown>): 1.15.9

  Executing build process
    Installing Go 1.15.9
      Completed in 3.963s

Tanzu Go Build Buildpack 0.1.0
  Executing build process
    Running 'go build -o /layers/tanzu-buildpacks_go-build/targets/bin -buildmode pie .'
      Completed in 7.762s

  Assigning launch processes
    web: /layers/tanzu-buildpacks_go-build/targets/bin/workspace
    workspace: /layers/tanzu-buildpacks_go-build/targets/bin/workspace
===> EXPORT
Adding layer 'tanzu-buildpacks/go-build:targets'
Adding 1/1 app layer(s)
Adding layer 'launcher'
Adding layer 'config'
Adding layer 'process-types'
Adding label 'io.buildpacks.lifecycle.metadata'
Adding label 'io.buildpacks.build.metadata'
Adding label 'io.buildpacks.project.metadata'
Setting default process type 'web'
*** Images (sha256:971f872644cf6a927c8efc890001c272a8c820f4d07efd683ff2905df9d84be1):
      TBS_REPOSITORY/demo-image
      TBS_REPOSITORY/demo-image:b1.20210401.203959
Adding cache layer 'tanzu-buildpacks/go-dist:go'
Adding cache layer 'tanzu-buildpacks/go-build:gocache'
===> COMPLETION
Build successful
```

## Investigate the Image

Pull the image locally.

```
docker pull $TBS_REPOSITORY/demo-image
```

Use Trivy to scan the image for security issues, noting how many `HIGH` or `CRITICAL` there are.

>NOTE: This image was built with the older, "insecure" images which we will patch.

```
$ /tmp/trivy -q --severity=HIGH,CRITICAL $TBS_REPOSITORY/demo-image | grep Total
Total: 7 (HIGH: 7, CRITICAL: 0)
```

## Update the Image 

We'll use the build and run images from the more recent descriptor file.

```
kp clusterstack update demo-stack  \
  --build-image $TBS_REPOSITORY/build@sha256:d0e8bfb686fd3bc5463bd79f33a31d1dc9528a37f88ffc26a64ff0949173223d \
  --run-image $TBS_REPOSITORY/run@sha256:84eff69367995f990875cdbd47b2dc427c0bac27ae15f2a115090962e6969421
```

eg. output:

```
kp clusterstack update demo-stack  \
>   --build-image $TBS_REPOSITORY/build@sha256:d0e8bfb686fd3bc5463bd79f33a31d1dc9528a37f88ffc26a64ff0949173223d \
>   --run-image $TBS_REPOSITORY/run@sha256:84eff69367995f990875cdbd47b2dc427c0bac27ae15f2a115090962e6969421
Updating ClusterStack...
Uploading to 'TBS_REPOSITORY'...
	Uploading 'TBS_REPOSITORY/build@sha256:d0e8bfb686fd3bc5463bd79f33a31d1dc9528a37f88ffc26a64ff0949173223d'
        Uploading 'TBS_REPOSITORY/run@sha256:84eff69367995f990875cdbd47b2dc427c0bac27ae15f2a115090962e6969421'
ClusterStack "demo-stack" updated
```


## Trigger a Build


```
$ kp image trigger demo-image
Triggered build for Image "demo-image"
```

Watch the logs.

```
$ kp build logs demo-image
===> PREPARE
Build reason(s): TRIGGER
TRIGGER:
	+ A new build was manually triggered on Thu, 01 Apr 2021 20:52:35 +0000
Loading secret for "gcr.io" from secret "gcr-secret" at location "/var/build-secrets/gcr-secret"
Loading secret for "harbor.shared.demo.globalbanque.com" from secret "harbor-creds" at location "/var/build-secrets/harbor-creds"
Cloning "https://github.com/ccollicutt/tbs-sample-apps/" @ "513fd440fec8e79bdc78b500b37f6b23881af951"...
Successfully cloned "https://github.com/ccollicutt/tbs-sample-apps/" @ "513fd440fec8e79bdc78b500b37f6b23881af951" in path "/workspace"
===> DETECT
tanzu-buildpacks/go-dist  0.1.5
tanzu-buildpacks/go-build 0.1.0
===> ANALYZE
Restoring metadata for "tanzu-buildpacks/go-dist:go" from cache
Restoring metadata for "tanzu-buildpacks/go-build:targets" from app image
Restoring metadata for "tanzu-buildpacks/go-build:gocache" from cache
===> RESTORE
Restoring data for "tanzu-buildpacks/go-dist:go" from cache
Restoring data for "tanzu-buildpacks/go-build:gocache" from cache
===> BUILD
Tanzu Go Distribution Buildpack 0.1.5
  Resolving Go version
    Candidate version sources (in priority order):
      <unknown> -> ""

    Selected Go version (using <unknown>): 1.15.9

  Reusing cached layer /layers/tanzu-buildpacks_go-dist/go

Tanzu Go Build Buildpack 0.1.0
  Executing build process
    Running 'go build -o /layers/tanzu-buildpacks_go-build/targets/bin -buildmode pie .'
      Completed in 433ms

  Assigning launch processes
    web: /layers/tanzu-buildpacks_go-build/targets/bin/workspace
    workspace: /layers/tanzu-buildpacks_go-build/targets/bin/workspace
===> EXPORT
Reusing layers from image 'TBS_REPOSITORY/demo-image@sha256:91d93f1f8312ce0adc3a13024a3605cb5232f6baa2d6bf849ab0e83a043c78b9'
Adding layer 'tanzu-buildpacks/go-build:targets'
Reusing 1/1 app layer(s)
Reusing layer 'launcher'
Reusing layer 'config'
Reusing layer 'process-types'
Adding label 'io.buildpacks.lifecycle.metadata'
Adding label 'io.buildpacks.build.metadata'
Adding label 'io.buildpacks.project.metadata'
Setting default process type 'web'
*** Images (sha256:7b2321b4a896103a72437ae185b5b62548f655e0092f1456118c5d040e212c97):
      TBS_REPOSITORY/demo-image
      TBS_REPOSITORY/demo-image:b3.20210401.205235
Reusing cache layer 'tanzu-buildpacks/go-dist:go'
Adding cache layer 'tanzu-buildpacks/go-build:gocache'
===> COMPLETION
Build successful
```

## Pull and check the image again

Get the new version of the image that TBS built.

```
$ docker pull $TBS_REPOSITORY/demo-image
Using default tag: latest
latest: Pulling from pa-ccollicutt/build-service/demo-image
7a3dbe310959: Pull complete 
d607b3dbf740: Pull complete 
0a865ece12ec: Pull complete 
0c7f742136bd: Pull complete 
e375f738cc46: Pull complete 
0d9d63805be5: Pull complete 
c4559ef0808b: Pull complete 
737f86c7b467: Pull complete 
2877c4e0b5d1: Pull complete 
bfc572f68c83: Pull complete 
cda8ee06160e: Pull complete 
Digest: sha256:7b2321b4a896103a72437ae185b5b62548f655e0092f1456118c5d040e212c97
Status: Downloaded newer image for TBS_REPOSITORY/demo-image:latest
TBS_REPOSITORY/demo-image:latest
```

Now if we scan the image again it will show fewer, or in this case zero, `HIGH` or `CRITICAL` CVEs.

```
$ trivy -q --severity=HIGH,CRITICAL $TBS_REPOSITORY/demo-image | grep Total
Total: 0 (HIGH: 0, CRITICAL: 0)
```

## Conclusion

Now imagine easily and efficiently rebuilding all of your images with a simple command.

Also remember that because of how buildpacks work, we don't have to be scared to update the image because it'll break the application.

One the images are automagically rebuilt, we can use Kubernetes to recreate all of the pods, and therefore have updated and secure EVERY application in our portfolio.

## Thanks

Hat tip to the [e2e demo](https://github.com/doddatpivotal/tkg-lab-e2e-adaptation/blob/main/docs/02-tbs-base-install.md) team.