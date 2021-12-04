# RIPE Atlas Docker Image

This is the [RIPE Atlas software probe](https://atlas.ripe.net/docs/software-probe/) packaged as a Docker image.

This is a fork from [github.com/Jamesits/docker-ripe-atlas](https://github.com/Jamesits/docker-ripe-atlas) with following changes:

- Added arm64 support (this image now supports amd64, arm64 and armv7)
- All architecture versions share the same set of tags so you don't need to use different tags on different platforms
- Build pipeline changed to Github actions, and the images are available on [ghcr.io](https://ghcr.io/scrin/docker-ripe-atlas) instead of Dockerhub
- Minor documentation changes and other tweaks

## Requirements

- 1 CPU core (of course)
- 20MiB memory
- 100MiB HDD
- A Linux installation with Docker installed
- Internet access

## Tags

The following prebuilt tags are available at [Github packages](https://github.com/Scrin/docker-ripe-atlas/pkgs/container/docker-ripe-atlas):

- `ghcr.io/scrin/docker-ripe-atlas:latest`: amd64, arm64 and armv7

## Running

First we start the container:

```shell
docker run --detach --restart=always --log-opt max-size=10m \
	--cpus=1 --memory=64m --memory-reservation=64m \
	--cap-add=SYS_ADMIN --cap-add=NET_RAW --cap-add=CHOWN \
	-v /var/atlas-probe/etc:/var/atlas-probe/etc \
	-v /var/atlas-probe/status:/var/atlas-probe/status \
	-e RXTXRPT=yes \
	--name ripe-atlas --hostname "$(hostname --fqdn)" \
	ghcr.io/scrin/docker-ripe-atlas
```

Then we fetch the generated public key:

```shell
cat /var/atlas-probe/etc/probe_key.pub
```

[Register](https://atlas.ripe.net/apply/swprobe/) the probe with your public key. After the registration being manually processed, you'll see your new probe in your account.

## Caveats

### IPv6

At the time of writing, Docker IPv6 support with ip6tables is experimental, so you need to set some experimental settings in `daemon.json` for docker:

```json
{
  "experimental": true,
  "ipv6": true,
  "ip6tables": true,
  "fixed-cidr-v6": "fdd0::/64"
}
```

And if running with docker-compose, following is needed at the time of writing as current versions of docker-compose don't set up ipv6 properly for the container network:

```yaml
networks:
  default:
    enable_ipv6: true
    ipam:
      driver: default
      config:
        - subnet: fdd0:f0f0::/80
```

Alternative workaround without needing to enable experimental mode for docker, you can use IPv6 NAT like this:

```shell
cat > /etc/sysctl.d/50-docker-ipv6.conf <<EOF
net.ipv6.conf.eth0.accept_ra=2
net.ipv6.conf.all.forwarding=1
net.ipv6.conf.default.forwarding=1
EOF
sysctl -p /etc/sysctl.d/50-docker-ipv6.conf
docker network create --ipv6 --subnet=fd00:a1a3::/48 ripe-atlas-network
docker run -d --restart=always -v /var/run/docker.sock:/var/run/docker.sock:ro -v /lib/modules:/lib/modules:ro --cap-drop=ALL --cap-add=NET_RAW --cap-add=NET_ADMIN --cap-add=SYS_MODULE --net=host --name=ipv6nat robbertkl/ipv6nat:latest
```

Then start the RIPE Atlas container with argument `--net=ripe-atlas-network`.

Note this might break your network and your mileage may vary. You should swap `eth0` with your primary network adapter name, and if you use static IPv6 assignment instead of SLAAC, change `accept_ra` to `0`.

### Auto Update

Use this recipe for auto updating the docker container.

```shell
docker run -d -v /var/run/docker.sock:/var/run/docker.sock --name watchtower containrrr/watchtower --cleanup --label-enable
```

Then start the RIPE Atlas container with argument `--label=com.centurylinklabs.watchtower.enable=true`.

### Backup

All the config files are stored at `/var/atlas-probe`. Just backup it.
