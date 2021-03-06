# VIC Product OVA Build

The build process uses a collection of bash scripts to launch a docker container on your local machine
where we provision a linux OS, install VIC dependencies, and extract the filesystem to make the OVA.

## Usage

The build process is controlled from a central script, `build.sh`. This script
launches the build docker container and controls our provisioning and ova
extraction through `make`.

### Prerequisites

The build machine must have `docker` and if using `build.sh` must have `gsutil`.

- `gsutil`: https://cloud.google.com/sdk/downloads
- `docker for Mac`: https://www.docker.com/docker-mac
- `docker for Windows`: https://www.docker.com/docker-windows

### Build bundle and OVA

#### Build script

This is the recommended way to build the OVA.


The build script pulls the desired versions of each included component into the build container.
It accepts files in the `installer/build` directory, URLs, or revisions and automatically sets
required environment variables.

*You must specify build step `ova-dev` when calling build.sh*

If called without any values, `build.sh` will get the latest build for each component
```
sudo ./build/build.sh ova-dev
```

Default values:
```
--admiral dev <vmware/admiral:vic_dev tag>
--harbor <latest in harbor-builds bucket>
--vicengine <latest in vic-engine-builds bucket>
--vicmachineserver dev <vic-machine-server:dev tag>

DCH Photon is pinned to 1.13 tag
```

If called with the values below, `build.sh` will include the Harbor version from
`installer/build/harbor.tgz`, the VIC Engine version from `installer/build/vic_XXXX.tar.gz`, and 
Admiral tag `vic_dev` (since `--admiral` was not specified it defaults to the `vic_dev` tag)
```
./build/build.sh ova-dev --harbor harbor.tgz --vicengine vic_XXXX.tar.gz
```

If called with the values below, `build.sh` will include the Harbor and VIC Engine versions
specified by their respective URLs, Admiral tag `vic_v1.1.1`, and VIC Machine Server tag `latest`.
```
./build/build.sh ova-dev --admiral v1.1.1 --harbor https://example.com/harbor.tgz --vicengine https://example.com/vic_XXXX.tar.gz --vicmachineserver latest
```

Note: the VIC Engine artifact used when building the OVA must be named following the `vic_*.tar.gz` format.
This is required by the OVA in order to automatically configure the VIC Engine UI plugins correctly for installation.

#### Upload

You can upload the ova builds to the `vic-product-ova-builds` and `vic-product-ova-releases` in google cloud.

*Personal development builds MUST be renamed with the username as a prefix before upload*

To do this, use the gsutil cli tool: `sudo gsutil cp -va public-read johndoe-vic-v1.2.1.ova gs://vic-product-ova-builds`.

## Deploy

The OVA must be deployed to a vCenter.
Deploying to ESX host is NOT supported.

The recommended method for deploying the OVA:
- Access the vCenter Web UI, click `vCenter Web Client (Flash)`
- Right click on the desired cluster or resource pool
- Click `Deploy OVF Template`
- Select the URL or file and follow the prompts
- Power on the deployed VM
- Access the `Getting Started Page` by going to `https://<appliance_ip>:9443`

Alternative, deploying with `ovftool`:
```
export VC_USER="<your vCenter user>"
export VC_PASSWORD="<your vCenter password>"
export VC_IP="<your vCenter IP address>"
export VC_COMPUTE="<your vCenter compute resource>"
docker run -it --net=host -v $GOPATH/src/github.com/vmware/vic-product/installer/bin:/test-bin \
  gcr.io/eminent-nation-87317/vic-integration-test:1.34 ovftool --acceptAllEulas --X:injectOvfEnv \
  --X:enableHiddenProperties -st=OVA --powerOn --noSSLVerify=true -ds=datastore1 -dm=thin \
  --net:Network="VM Network" \
  --prop:appliance.root_pwd="password" \
  --prop:appliance.permit_root_login=True \
  --prop:management_portal.port=8282 \
  --prop:registry.port=443 \
  /test-bin/$(ls -1t bin | grep "\.ova") \
  vi://$VC_USER:$VC_PASSWORD@$VC_IP/$VC_COMPUTE
```

## Vendor

To build the installer dependencies, ensure `GOPATH` is set, then issue the following.
``
$ make vendor
``

This will install the [dep](https://github.com/golang/dep) utility and retrieve the build dependencies via `dep ensure`.

NOTE: Dep is slow the first time you run it - it may take 10+ minutes to download all of the dependencies. This is because
dep automatically falttens the vendor folders of all dependencies. In most cases, you shouldn't need to run `make vendor`,
as our vendor directory is checked in to git.

## CI Workflow

VIC Product build is auto-triggered from the successful completion of the following CI builds:

[VIC Engine](https://ci.vcna.io/vmware/vic)

[Admiral](https://ci.vcna.io/vmware/admiral)

[Harbor](https://ci.vcna.io/vmware/harbor)

There is also a separate build for [VIC UI](https://ci.vcna.io/vmware/vic-ui) which publishes the [artifact](https://console.cloud.google.com/storage/browser/vic-ui-builds) consumed by VIC Engine builds. VIC Engine publishes vic engine artifacts and vic machine server image.
Harbor build publishes harbor installer and Admiral build publishes admiral image. All these artifacts are published to Google cloud except Admiral image which is published to Docker hub.

For every auto-triggered build, by default, VIC Product consumes the latest artifacts from [vic engine](https://storage.googleapis.com/vic-engine-builds) and [harbor](https://storage.googleapis.com/harbor-builds) buckets, recent `dev` tagged images for [admiral](https://hub.docker.com/r/vmware/admiral/) and [vic machine server](https://console.cloud.google.com/gcr/images/eminent-nation-87317/GLOBAL/vic-machine-server?project=eminent-nation-87317&gcrImageListsize=50) and publishes [OVA artifact](https://storage.googleapis.com/vic-product-ova-builds) after successful build and test.

Refer to the next section `Staging and Release` on how to build OVA with specific dependent component versions.

## Staging and Release

To perform staging and release process using Drone CI, refer to following commands. 
Please note that you cannot trigger new CI builds manually, but have to promote existing build to either staging or release.

Make sure `DRONE_SERVER` and `DRONE_TOKEN` environment variables are set before executing these commands.

To promote existing successful CI build to staging...

``
$ drone deploy --param VICENGINE=<vic_engine_version> --param VIC_MACHINE_SERVER=<vic_machine_server> --param ADMIRAL=<admiral_tag> --param HARBOR=<harbor_version> vmware/vic-product <ci_build_number_to_promote> staging
``

To promote existing successful CI build to release...

``
$ drone deploy --param VICENGINE=<vic_engine_version> --param VIC_MACHINE_SERVER=<vic_machine_server> --param ADMIRAL=<admiral_tag> --param HARBOR=<harbor_version> vmware/vic-product <ci_build_number_to_promote> release
``

`vic_engine_version` and `harbor_version` can be specified as a URL or a file in `cwd`, eg. 'https://storage.googleapis.com/vic-engine-releases/vic_1.2.1.tar.gz'

`admiral_tag` and `vic_machine_server` should be specified as docker image revision tag, eg. 'latest'

`ci_build_number_to_promote` is the drone build number which will be promoted

## Troubleshooting
