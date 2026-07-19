# Multi-port reverse-proxy VPS trial

This procedure deploys the focused multi-port fork with the normal NetBird
self-hosted installer. It uses three custom images: the combined server, the
reverse proxy, and the dashboard. Standard NetBird peer clients do not need a
fork.

Treat the first deployment as a trial. The focused branches are based on the
current upstream development trees rather than a stable release tag. Pin every
custom image to an immutable tag or digest and keep the server and proxy images
on the same backend commit.

## Prerequisites

The build machine needs Git, Node.js 20.9 or newer, npm, Docker Buildx, and the
GitHub CLI (or an equivalent way to download the public IronRDP v0.0.2 release
assets). The VPS needs Docker Engine, Docker Compose v2, `jq`, `curl`, and
`openssl`; the installer checks or invokes those tools.

Also prepare:

- A public DNS name for management, for example `netbird.example.net`.
- A wildcard `*.netbird.example.net` record pointing to the same VPS. The
  installer's free reverse-proxy cluster uses the management domain, so an
  initial service can use `game.netbird.example.net`. A different service
  domain must first be registered as a custom reverse-proxy domain.
- A container registry accessible from the VPS. The examples use the GitLab
  registry, but any OCI registry works.
- TCP 80 and 443, UDP 3478, and UDP 51820 open for the normal installation,
  plus every raw TCP/UDP listener that will be exposed.

Use DNS-only records for arbitrary raw TCP/UDP services. A conventional HTTP
CDN proxy does not forward those transports or ports.

## Build and publish the images

Set registry and platform values. Use `linux/arm64` instead for an ARM VPS.

```bash
export REGISTRY=registry.gitlab.com/example/netbird-multiport
export PLATFORM=linux/amd64
docker login "${REGISTRY%%/*}"
```

Build the server and reverse proxy from the backend repository. The
multistage Dockerfiles compile source; the non-multistage Dockerfiles are
release-packaging inputs that expect prebuilt binaries.

Push both focused branches to repositories you control and check out immutable
commit IDs on the build machine. Before assigning a tag, confirm that the
checkout has no local source changes:

```bash
cd /path/to/netbird
export BACKEND_COMMIT=REPLACE_WITH_FULL_BACKEND_COMMIT
git checkout "$BACKEND_COMMIT"
test -z "$(git status --porcelain)"
export BACKEND_TAG="multiport-$(git rev-parse --short HEAD)"

docker buildx build \
  --platform "$PLATFORM" \
  --file combined/Dockerfile.multistage \
  --tag "$REGISTRY/netbird-server:$BACKEND_TAG" \
  --push .

docker buildx build \
  --platform "$PLATFORM" \
  --file proxy/Dockerfile.multistage \
  --tag "$REGISTRY/reverse-proxy:$BACKEND_TAG" \
  --push .
```

Build the dashboard from the matching focused dashboard branch. The IronRDP
downloads reproduce the current upstream release workflow; they are not part
of the multi-port feature.

```bash
cd /path/to/netbird-dashboard
export DASHBOARD_COMMIT=REPLACE_WITH_FULL_DASHBOARD_COMMIT
git checkout "$DASHBOARD_COMMIT"
test -z "$(git status --porcelain)"
export DASHBOARD_TAG="multiport-$(git rev-parse --short HEAD)"

npm ci
printf '{}\n' > .local-config.json
mkdir -p public/ironrdp-pkg

gh release download v0.0.2 --repo netbirdio/IronRDP \
  --pattern '*.ts' --dir public/ironrdp-pkg --clobber
gh release download v0.0.2 --repo netbirdio/IronRDP \
  --pattern '*.js' --dir public/ironrdp-pkg --clobber
gh release download v0.0.2 --repo netbirdio/IronRDP \
  --pattern 'ironrdp_web_bg.wasm' --dir public/ironrdp-pkg --clobber

NEXT_PUBLIC_DASHBOARD_VERSION="$DASHBOARD_TAG" npm run build
test -s out/index.html

# Match the upstream build.sh workflow: stage its Docker ignore file only for
# the image build. It excludes node_modules/.next but deliberately keeps out/.
cp docker/.dockerignore .dockerignore
trap 'rm -f .dockerignore' EXIT
docker buildx build \
  --platform "$PLATFORM" \
  --file docker/Dockerfile \
  --tag "$REGISTRY/dashboard:$DASHBOARD_TAG" \
  --push .
rm -f .dockerignore
trap - EXIT
```

The dashboard Docker context must include `out/`. Do not add `out/` to the
staged ignore file: `docker/Dockerfile` copies that static export into nginx.
Record the resulting manifests (and preferably deploy by digest) before moving
to the VPS:

```bash
docker buildx imagetools inspect "$REGISTRY/netbird-server:$BACKEND_TAG"
docker buildx imagetools inspect "$REGISTRY/reverse-proxy:$BACKEND_TAG"
docker buildx imagetools inspect "$REGISTRY/dashboard:$DASHBOARD_TAG"
```

## Run the standard installer

Copy `infrastructure_files/getting-started.sh` from the backend commit used to
build the images to the VPS. Using that exact script avoids coupling the trial
to a later installer release.

On the VPS, log in to a private registry if necessary and run:

```bash
sudo install -d -o "$USER" -g "$(id -gn)" /opt/netbird
cd /opt/netbird
docker login registry.gitlab.com

export REGISTRY=registry.gitlab.com/example/netbird-multiport
export BACKEND_TAG=multiport-REPLACE_ME
export DASHBOARD_TAG=multiport-REPLACE_ME
export NETBIRD_SERVER_IMAGE="$REGISTRY/netbird-server:$BACKEND_TAG"
export NETBIRD_PROXY_IMAGE="$REGISTRY/reverse-proxy:$BACKEND_TAG"
export DASHBOARD_IMAGE="$REGISTRY/dashboard:$DASHBOARD_TAG"

bash ./getting-started.sh
```

Leave `NETBIRD_DOMAIN` unset to use the installer's normal domain prompt, or
export it before running. For the initial trial choose the built-in Traefik
option, enter the ACME email, and answer yes when asked to enable NetBird
Proxy. Choose no for CrowdSec during the minimal feature trial unless it is
also under test. The installer creates the proxy token and writes the three
selected image references into `docker-compose.yml`.

There is no default administrator login. With a fresh `netbird_data` volume,
opening `https://<management-domain>` redirects to `/setup`; the wizard creates
the first owner using a name, email address, and password of at least eight
characters. If the wizard does not appear, the volume already contains an
account. `docker compose down --volumes` resets it, but is destructive and
should only be used for a disposable trial.

## Publish the raw listener ports

The installer publishes the normal front door and optional proxy WireGuard
port, but it cannot predict future service mappings. Docker must publish every
raw listener before it can be reached from outside the proxy container.

Copy
`infrastructure_files/docker-compose.multiport.override.yml.example` to
`/opt/netbird/docker-compose.override.yml`, edit the port set, and apply it:

```bash
docker compose config
docker compose up -d --force-recreate proxy
docker port netbird-proxy
```

Open the identical protocol/port set in the VPS host firewall and the cloud
provider's security group. TCP and UDP may use the same numeric port when both
are published separately. A range must be published as a range.

Confirm in the rendered `docker compose config` output that the base
`51820/udp` publication and every custom listener are present. Do not reuse
installer-owned host ports 80/tcp, 443/tcp, 3478/udp, or 51820/udp in the raw
override.

Port `443/tcp` is already owned by Traefik on a single-address installation;
do not add `443:443/tcp` to the proxy override. Built-in Traefik forwards
unmatched SNI on public 443 to the proxy's normal TLS/HTTPS listener on 8443.
A raw TCP fallback on 443 needs a second public address or a specialized
front-proxy design. Use high raw ports for the first test.

## Configure and verify one service

Enroll a target peer, start representative listeners on it, and use the
dashboard to create one L4 reverse-proxy service with one target and multiple
mappings, for example:

| Protocol | Public listener | Target |
| --- | ---: | ---: |
| TCP | 8080 | 8080 |
| TCP | 9000 | 9000 |
| UDP | 9001 | 9001 |
| UDP | 5000-5030 | 6000-6030 |

The source and target ranges must have equal sizes. The same listener range
cannot overlap another service on the same proxy cluster for the same
protocol. TCP and UDP ownership is independent.

Verify both the proxy container and the target application rather than relying
only on the dashboard:

```bash
docker compose logs --tail=200 proxy netbird-server
nc -vz game.netbird.example.net 8080
nc -vz game.netbird.example.net 9000
# Use a UDP echo client/server or application-specific probe for UDP; a bare
# UDP connect check does not prove that a response reached the target.
```

## Deployment limitations

- A raw TCP/UDP hostname provides DNS resolution, service identity, and proxy
  cluster selection. The hostname is not present in raw packets; routing is by
  protocol and listener port. Two domains therefore cannot own the same raw
  protocol/port on one public address.
- Adding a mapping in the dashboard does not alter Docker port publication.
  Update the override and recreate the proxy when the public listener set
  changes.
- One multi-port L4 service still has one target peer/resource. Per-mapping
  targets and L4 load balancing are outside this feature.
- Port ranges are compact in the API and database but expand to one runtime
  listener/relay per port.
- Do not deploy an upstream-only proxy with the custom server. Multi-port
  mappings require the proxy capability added by this fork.
- The dashboard does not yet expose an aggregate cluster capability for this
  new wire feature. During rolling upgrades, create multi-port services only
  after every serving proxy has been replaced with the matching custom image;
  management deliberately withholds repeated mappings from older proxies.
