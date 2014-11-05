flashlight [![Travis CI Status](https://travis-ci.org/getlantern/flashlight.svg?branch=master)](https://travis-ci.org/getlantern/flashlight)&nbsp;[![Coverage Status](https://coveralls.io/repos/getlantern/flashlight/badge.png)](https://coveralls.io/r/getlantern/flashlight)&nbsp;[![GoDoc](https://godoc.org/github.com/getlantern/flashlight?status.png)](http://godoc.org/github.com/getlantern/flashlight)
==========

Lightweight host-spoofing web proxy written in go.

flashlight runs in one of two modes:

client - meant to run locally to wherever the browser is running, forwards
requests to the server

server - handles requests from a flashlight client proxy and actually proxies
them to the final destination

Using CloudFlare (and other CDNS), flashlight has the ability to masquerade as
running on a different domain than it is.  The client simply specifies the
"masquerade" flag with a value like "thehackernews.com".  flashlight will then
use that masquerade host for the DNS lookup and will also specify it as the
ServerName for SNI (though this is not actually necessary on CloudFlare). The
Host header of the HTTP request will actually contain the correct host
(e.g. getiantem.org), which causes CloudFlare to route the request to the
correct host.

Flashlight uses [enproxy](https://github.com/getlantern/enproxy) to encapsulate
data from/to the client as http request/response pairs.  This allows it to
tunnel regular HTTP as well as HTTPS traffic over CloudFlare.  In fact, it can
tunnel any TCP traffic.

### Usage

```bash
Usage of flashlight:
  -addr="": ip:port on which to listen for requests. When running as a client proxy, we'll listen with http, when running as a server proxy we'll listen with https (required)
  -cloudconfig="": optional http(s) URL to a cloud-based source for configuration updates
  -cloudconfigca="": optional PEM encoded certificate used to verify TLS connections to fetch cloudconfig
  -configdir="": directory in which to store configuration, including flashlight.yaml (defaults to current directory)
  -country="xx": 2 digit country code under which to report stats. Defaults to xx.
  -cpuprofile="": write cpu profile to given file
  -help=false: Get usage help
  -instanceid="": instanceId under which to report stats to statshub
  -memprofile="": write heap profile to given file
  -parentpid=0: the parent process's PID, used on Windows for killing flashlight when the parent disappears
  -portmap=0: try to map this port on the firewall to the port on which flashlight is listening, using UPnP or NAT-PMP. If mapping this port fails, flashlight will exit with status code 50
  -role="": either 'client' or 'server' (required)
  -server="": FQDN of flashlight server when running in server mode (required)
  -statsaddr="": host:port at which to make detailed stats available using server-sent events (optional)
  -statshub="pure-journey-3547.herokuapp.com": address of statshub server
  -statsperiod=0: time in seconds to wait between reporting stats. If not specified, stats are not reported. If specified, statshub, instanceid and statsaddr must also be specified.
```

Example Client:

```bash 
./flashlight -addr localhost:10080 -role client
```

Example Server:

```bash
./flashlight -addr :443 -role server
```

Example Curl Test:

```bash
curl -x localhost:10080 http://www.google.com/humans.txt
Google is built by a large team of engineers, designers, researchers, robots, and others in many different sites across the globe. It is updated continuously, and built with more tools and technologies than we can shake a stick at. If you'd like to help us out, see google.com/careers.
```

On the client, you should see something like this for every request:

```bash
Handling request for: http://www.google.com/humans.txt
```

### Building

Flashlight requires [Go 1.3](http://golang.org/dl/).

It is convenient to build flashlight for multiple platforms using something like
[goxc](https://github.com/laher/goxc).

With goxc, the binaries used for Lantern can be built using the
./crosscompile.bash script. This script also sets the version of flashlight to
the most recent annotated tag in git. An annotated tag can be added like this:

`git tag -a v1.0.0 -m"Tagged 1.0.0"`

Note - ./crosscompile.bash omits debug symbols to keep the build smaller.

The binaries end up at
`$GOPATH/bin/flashlight-xc/snapshot/<platform>/flashlight`.

Note that these binaries should also be signed for use in production, at least
on OSX and Windows. On OSX the command to do this should resemble the following
(assuming you have an associated code signing certificate):

```
codesign -s "Developer ID Application: Brave New Software Project, Inc" -f install/osx/pt/flashlight/flashlight
```

### Masquerade Host Management

Masquerade host configuration is managed using utilities in the [`genconfig/`](genconfig/) subfolder.

#### Setup

You need the s3cmd tool installed and set up.  To install on
Ubuntu:

```bash
sudo apt-get install s3cmd
```

On OS X:
```bash
brew install s3cmd
```

And then run `s3cmd --configure` and follow the on-screen instructions.  You
can get AWS credentials that are good for uploading to S3 in
[too-many-secrets/lantern_aws/aws_credential](https://github.com/getlantern/too-many-secrets/blob/master/lantern_aws/aws_credential).

#### Managing masquerade hosts

The file allsites.txt contains the list of masquerade hosts we use. To add/remove domains:

1. Edit [`allsites.txt`](genconfig/allsites.txt)
2. `go run genconfig.go allsites.txt`.  You can also specify a 2nd file of blacklisted domains, which will be excluded from the configuration, for example `go run genconfig.go allsites.txt blacklist.txt`.
3. Commit the changed [`masquerades.go`](config/masquerades.go) and [`cloud.yaml`](genconfig/cloud.yaml) to git if you want
4. Upload cloud.yaml to s3 using [`udpateyaml.bash`](genconfig/updateyaml.bash) if you want
