# piknik
Copy/paste anything over the network

## Blurb
Ever needed a copy/paste clipboard that works over the network?

Piknik seamlessly transfers URLs, code snipppets, documents, virtually anything between (possibly firewalled) hosts.
No SSH needed.

Fill in the clipboard ("copy") with whatever comes in to the standard input:

```bash
$ pkc
clipboard content
```

Magically retrieve that content from any other host having Piknik installed with the same configuration:

```bash
$ pkp
clipboard content
```

Boom.

Obviously, it can be used to transfer files as well:

```bash
$ pkc < kitten.gif
$ pkp > kittencopy.gif
```

```bash
$ tar cvf - *txt | pkc
$ pkp | tar xvf -
```

In order to bypass firewalls/NAT gatways and to provide persistence, the clipboard content transits via a staging server.

Nothing transits without end-to-end encryption and authentication; the server doesn't learn much about what the clipboard actually contains.

Data can be shared between different operating systems. Even Windows is kinda supported.

## Installation

This project is written in Go. So, a Go compiler is required, as well as the following incantation:

```bash
$ export GOPATH=${GOPATH:-~/go} ; mkdir -p $GOPATH
$ go get github.com/jedisct1/piknik
```

The `piknik` should then be available in `$GOPATH/bin`.

## Setup

Piknik requires a bunch of keys. Generate them all with

```bash
$ piknik -genkeys
```

The output of that command is all you need to build a configuration file.

Only copy the section for servers on the staging server. Only copy the section for clients on the clients.

Is a host gonna act both as a staging server and as a client? Ponder on it before copying the "hybrid" section, but it's there, just in case.

The default location for the configuration file is `~/.piknik.toml`. With the exception of Windows, where dot-files are not so common. On that platform, the file is simply called `piknik.toml`.

Sample configuration file for a staging server:
```toml
Listen = "0.0.0.0:8075"	# Edit appropriately
Psk = "bf82bab384697243fbf616d3428477a563e33268f0f2307dd14e7245dd8c995d"
SignPk = "0c41ca9b0a1b5fe4daae789534e72329a93a352a6ad73d6f1d368d8eff37271c"
```

Sample configuration file for clients:
```toml
Connect = "127.0.0.1:8075"	# Edit appropriately
EncryptSk = "2f530eb85e59c1977fce726df9f87345206f2a3d40bf91f9e0e9eeec2c59a3e4"
Psk = "bf82bab384697243fbf616d3428477a563e33268f0f2307dd14e7245dd8c995d"
SignPk = "0c41ca9b0a1b5fe4daae789534e72329a93a352a6ad73d6f1d368d8eff37271c"
SignSk = "cecf1d92052f7ba87da36ac3e4a745b64ade8f9e908e52b4f7cd41235dfe74810c41ca9b0a1b5fe4daae789534e72329a93a352a6ad73d6f1d368d8eff37271c"
```

Do not use these, uh? Get your very own keys with the `piknik -genkeys` command.
And edit the `Connect` and `Listen` properties to reflect the staging server IP and port.

Don't like the default config file location? Use the `-config` switch.

## Usage (staging server)

Run the following command on the staging server (or use `runit`, whatever):

```bash
$ piknik -server
```

## Usage (client)

```bash
$ piknik -copy
```

Copy the standard input to the clipboard.

```bash
$ piknik -paste
```

Retrieve the content of the clipboard and spit it to the standard output.
`-paste` is actually a no-op. This is the default action if `-copy` is not being used.

That's it.

Feed it anything. Text, binary data, whatever. As long as it fits in memory.

## Wait...

Wait. Where are the `pkc` and `pkp` commands describer earlier?

Sample shell aliases:

```bash
pko() { echo "$*" | piknik -copy }
pkf() { piknik -copy < $1 }
alias pkc='piknik -copy'
alias pkp='piknik -paste'
```

Use your own :)

## Protocol

Common:
```
ct: XChaCha20 ke,n (m)
Hk,s: BLAKE2b(domain="SK", key=k, salt=s, size=32)
Len(x): x encoded as a 64-bit little endian unsigned integer
n: random 192-bit nonce
r: random 256-bit nonce
Sig: Ed25519
v: 1
```

Copy:
```
-> v || r || h0
h0 := Hk,0(v || r)

<- v || h1
Hh := Hk,1(v || h0)

-> 'S' || h2 || Len(n || ct) || s || n || ct
s := Sig(n || ct)
h2 := Hk,2(h1 || s)

<- Hk,3(h2)
```

Paste:
```
-> v || r || h0
h0 := Hk,0(v || r)

<- v || h1
h1 := Hk,1(v || H0)

-> 'G' || h2
h2 := Hk,2(h1)

<- Hk,3(h2 || s) || Len(n || ct) || s || n || ct
s := Sig(n || ct)
```
