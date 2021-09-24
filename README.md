# Monero Buildah Image

Goal is to have a Dockerfile that would provide the ability for everyone to cross compile Monero  for their target architecture and lower the entry point for self hosted node while aiming for maximum security and privacy along the way.
## Requirements 

- [buildah](https://github.com/containers/buildah/blob/main/install.md) on build host
- [podman](https://podman.io/getting-started/installation#linux-distributions) on target host
- git on build host
- [distroless image](https://console.cloud.google.com/gcr/images/distroless/global/base) sha256 value
- [monero](https://github.com/monero-project/monero/releases) release tag and sha commit ID
- [Tor](https://github.com/Far1za/tor-buildah.git) image

## Build target image

**NOTE:** all commands presented are being executed on a Ubuntu build host

```
git clone https://github.com/Far1za/monero-buildah.git
cd monero-buildah
```
Edit **Dockerfile.arm64** and update the file to target the right architecture and use the latest Monero version available

ENV HOST=**aarch64**
ENV ARCH=**arm64**

ARG MONERO_VERSION=**v0.17.2.3**
ARG MONERO_HASH=**2222bea92fdeef7e6449d2d784cdfc3012641ee1**

FROM gcr.io/distroless/base@sha256:gcr.io/distroless/base@sha256:**703a4726aedc9ec7a7e32251087565246db117bb9a141a7993d1c4bb4036660d**

**NOTE:** we obtain the sha256 image value by searching for **nonroot-arm64** and using the latest version available before starting the image build process

`buildah bud --timestamp 0 -f Dockerfile.arm64`

Depending on the build host performance we should get our Monero image in ~ 1 hour or less.

## Putting it all together

Now before proceeding further make sure that you have followed the guide on creating a [Tor](https://github.com/Far1za/tor-buildah.git) image first. We need to make our new images available on our target host where we plan to run our Monero node.

```
# This is needed if the host where our Monero pod is going to run is different from our build host
buildah push imageID oci-archive:/path/to/archive:image_name:tag
scp archive username@IP_ADDR:/home/username

# also scp the config files as well
scp {torrc,monerod.conf} username@IP_ADDR:/home/username

podman load -i archive
```

**NOTE:** it is possible to push the image to some repository and later pull from there,but to keep it simple and private we would use the approach explained above.

Now that we have our image on the target host we can start with the deployment process

```
mkdir ~/monero # this is where our v3 onion address data is going to be stored
chmod 700 ~/monero  #tor fails to start otherwise

podman unshare chown -R 65532:65532 ~/monero
podman unshare chown -R 65532:65532 ~/.bitmonero
```

It is expected that **.bitmonero** exists in the users **$HOME** path and has enough storage or attached to external drive to store the  Monero blockchain.

```
podman pod create -p 18081:18081 -p 18080:18080 --name monero  # we expose the ports to be reachable from the outside
podman create -v ~/monero:/tor/share/monero -v ~/torrc:/tor/etc/tor/torrc --pod monero torImageID/Tag

# execute this step to generate onion address (skip if one already exists)
podman pod start monero
sudo cat monero/hostname # get the newly generated onion address

#  using you onion address update monerod.conf file on line anonymous-inbound

podman create -v ~/.bitmonero:/home/nonroot/.monero -v ~/monerod.conf:/home/nonroot/monerod.conf --pod monero moneroImageID/Tag
podman start moneroContainerID # start the newly generated Monero container or issue the pod start command
```

If everything is configured correctly we should have our Monero pod running and our node bootstrapping to become part of the Monero network. 
Otherwise we would need to parse the logs generated to determine what is wrong.

```
podman ps --all # get container status
podman logs ContainerID
```
## Pod Auto start

To save ourselves from manual start of the pod every time we reboot our host we can generate a systemd scripts that are going to start the pod for us

	podman generate systemd --files --name monero

## Support

If you have any issues or suggestions for improvements please open a new issue.
Otherwise if this has helped you to realize your project or saved you from extra work please express your appreciation here

	monero:88nYxA5xZEfLDuTPiBXZuzMRKFzHsR6JJSnBoNkJb9rF16KZxtYzFHJcZoaFKAbeUxXtPUQgjZ6zj7y5WBiP5c8vCXP5r8N

Thank you.
