# btsync.docker
sync service built for persist data on clusters

## Motivation

#### CLUSTER PERSISTENT STORAGE

At [findhit.com](https://findhit.com), we have to share data between our cluster
nodes. After testing out some solutions such as: cloud provider's storage ones,
some well known cluster-targeted file systems and among other less valuable
solutions to our architecture (such as NAS sharing), we've got always the same
opinion. None of them could be fast, scalable and easy as plug-n-play.

As so, we looked for p2p syncing solutions. BTSync was our first choice, but it
lacks some important features such as cluster-based configuration storages
(`etcd` for example).

I took a time to plan how I would structure configurations on a cluster, but it
always ends on configurations per node, the only difference with current BTSync
approach would be configuration sharing, and thats not useful.

So, instead of creating an image that relied on `confd` (which is great btw),
I've decided to work on btsync cli before going to sleep.

## How it works?

A cli will allow us to place a btsync service per cluster node (making use out of
the ephemeral disk they mostly have) and control that service via `docker exec`.

### Service structure example
NOTE: Examples presented  above will rely on a CoreOS environment.

- Submitting a btsync global service (with fleet) will allow us to share one
path (my choice was `/mnt/resource`, which is an ephemeral storage on CoreOS on MS Azure)
- Services could easily ensure or remove a folder by running:
  - `docker exec btsync ctl add gitlab` on `ExecStartPre`
  - `docker exec btsync ctl del gitlab` on `ExecStopPost`
- Storage could be mounted by sharing `/data` volume of `btsync` container, but I really advise you to don't do so. `/data` could have other containers data
and that fact creates a big SECURITY WARNING ON MY HEAD!!! Instead we could use
a path getter such as:
`docker run -v $(docker exec btsync ctl path gitlab):/home/gitlab/data gitlab:latest`

## Usage

### Launching btsync container

```bash
docker run \
    --name btsync \
    -v /path/to/some/disk:/data \
    cusspvz/btsync:latest
```

NOTE: ATM it only supports one volume, since I'm still working on it, don't know
how multiple volumes could affect it.

### Running commands on container
```bash
docker exec btsync ctl [commands]
```

### Commands

#### add [--secret=""] namespace
Adds a folder for a determined namespace

```bash
docker exec btsync ctl add --secret="e3hryu35qegqery4w5y164u5u" "gitlab"
```

NOTE: Secret provided on example isn't a valid one, it was generated by a random
pseudo-geek key typing. I'm on a coffee, now people here think I write fast.

#### del namespace
Deletes folder related with that namespace

```bash
docker exec btsync ctl del "gitlab"
```

#### path [--scope=(host|container)] namespace
Returns host's or container's path to created namespace.
By default, scope is host.

```bash
docker exec btsync ctl path "gitlab" # /mnt/resources/sync/gitlab
```

#### has namespace
Exits with 0 (has) or 1 (hasn't) indicating if namespace exists [or not]

```bash
if docker exec btsync ctl has "gitlab"; then
    echo "yeah, we already have that namespace"
fi
```

#### list
Prints a spaced-separated list of namespaces

```bash
docker exec btsync ctl add gitlab
docker exec btsync ctl add star-trek
docker exec btsync ctl list # gitlab star-trek
```

### Environment Variables

#### HOST_DATA_PATH
Defaults to: /mnt/resources

#### DATA_PATH
Defaults to: /data

#### CONFIG_PATH
Defaults to: /data/btsync.conf

#### CONFIG_INTERVAL_CHECK
Defaults to: 10

#### PID_PATH
Defaults to: /var/run/btsync.pid

#### DEBUG
Defaults to: "btsync:ctl"

#### UNAME
Defaults to: btysnc

#### UID
Defaults to: 1000

#### GID
Defaults to: 1000
