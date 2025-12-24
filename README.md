# Multicast DNS (mDNS)

[![Hex.pm](https://img.shields.io/hexpm/v/mdns.svg)](https://hex.pm/packages/mdns)
[![Hex Docs](https://img.shields.io/badge/hex-docs-blue.svg)](https://hexdocs.pm/mdns)

**Zero-configuration networking for Erlang/OTP applications using Multicast DNS.**

This is a rebar3-compatible fork of [shortishly/mdns](https://github.com/shortishly/mdns), maintained by [macula-io](https://github.com/macula-io) for use in the Macula HTTP/3 mesh platform.

## Attribution

This library was originally created by **Peter Morgan** ([shortishly](https://github.com/shortishly)).

Peter Morgan's implementation provides an elegant, production-ready mDNS solution that enables automatic node discovery and mesh formation in local networks. All credit for the design, protocol implementation, and core functionality goes to Peter Morgan. This fork adds rebar3/hex.pm packaging support while preserving the original functionality.

**Original Repository:** [github.com/shortishly/mdns](https://github.com/shortishly/mdns)

## Overview

mDNS is an implementation of the [Multicast DNS](http://files.multicastdns.org/draft-cheshire-dnsext-multicastdns.txt) discovery protocol written in Erlang/OTP. It enables two or more Erlang nodes to:

- **Self-discover** other nodes on the local network
- **Automatically form** a mesh network
- **Register and discover** services using [DNS-Based Service Discovery (RFC 6763)](http://www.ietf.org/rfc/rfc6763.txt)

## Installation

Add `mdns` to your `rebar.config` dependencies:

```erlang
{deps, [
    {mdns, "0.1.0"}
]}.
```

## Quick Start

### Automatic Node Discovery

By default, mDNS advertises a service of type `_erlang._tcp`. You can view registered services using standard tools:

**Linux (Avahi):**
```shell
avahi-browse _erlang._tcp
```

**macOS:**
```shell
dns-sd -B _erlang._tcp
```

### Automatic Mesh Formation

mDNS can automatically form an Erlang/OTP mesh network when `MDNS_CAN_MESH` is enabled.

**On dev001.local:**
```shell
MDNS_CAN_MESH=true rebar3 shell
```

**On dev002.local:**
```shell
MDNS_CAN_MESH=true rebar3 shell
```

After a short period, both machines will have automatically formed a mesh:

```erlang
(mdns@dev001.local)1> nodes().
['mdns@dev002.local']
```

```erlang
(mdns@dev002.local)1> nodes().
['mdns@dev001.local']
```

## Configuration

mDNS can be configured via environment variables or application environment:

| Application Config | Environment Variable | Default | Description |
|-------------------|---------------------|---------|-------------|
| `can_advertise` | `MDNS_CAN_ADVERTISE` | `true` | Advertise this node's services |
| `can_discover` | `MDNS_CAN_DISCOVER` | `true` | Discover other nodes |
| `can_mesh` | `MDNS_CAN_MESH` | `false` | Auto-form Erlang cluster |
| `environment` | `MDNS_ENVIRONMENT` | `dev` | Environment tag (nodes must match to mesh) |
| `multicast_address` | `MDNS_MULTICAST_ADDRESS` | `224.0.0.251` | mDNS multicast address |
| `udp_port` | `MDNS_UDP_PORT` | `5353` | mDNS UDP port |
| `domain` | `MDNS_DOMAIN` | `.local` | mDNS domain |
| `service` | `MDNS_SERVICE` | `_erlang._tcp` | Service type to advertise |
| `ttl` | `MDNS_TTL` | `120` | DNS record TTL (seconds) |

### Environment-Based Isolation

Only nodes that share the same `environment` value can be automatically meshed together. This allows multiple development environments on the same network:

```shell
# Team A's development cluster
MDNS_ENVIRONMENT=team_a MDNS_CAN_MESH=true rebar3 shell

# Team B's development cluster (won't mesh with Team A)
MDNS_ENVIRONMENT=team_b MDNS_CAN_MESH=true rebar3 shell
```

## Subscribing to Advertisements

mDNS uses [gproc](https://github.com/uwiger/gproc)'s [pub/sub](https://github.com/uwiger/gproc#use-case-pubsub-patterns) pattern. Applications can subscribe to mDNS advertisements:

```erlang
%% Subscribe to advertisement events
mdns:subscribe(advertisement).

%% In your gen_server handle_info/2:
handle_info({gproc_ps_event, advertisement, Info}, State) ->
    #{host := Host,
      node := Node,
      port := Port,
      env := Env} = Info,
    %% Handle the discovered node...
    {noreply, State}.
```

### Advertisement Map Structure

| Key | Type | Description |
|-----|------|-------------|
| `host` | `binary()` | Hostname of the advertised node |
| `node` | `atom()` | Node name of the advertised node |
| `port` | `integer()` | Distribution protocol port |
| `env` | `binary()` | Environment of this node |

## Integration with Macula

In the Macula HTTP/3 mesh platform, mDNS provides optional local network discovery. When available, Macula uses mDNS to discover seed nodes on the local network, reducing the need for static seed configuration.

```erlang
%% Macula checks for mDNS availability
case whereis(mdns_advertise_sup) of
    undefined ->
        %% mDNS not available, use DHT discovery
        ok;
    _Pid ->
        %% Subscribe to local node advertisements
        mdns:subscribe(advertisement)
end.
```

See [Macula mDNS Setup Guide](https://hexdocs.pm/macula/mdns_setup.html) for integration details.

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                    mdns_sup                         │
│                  (supervisor)                       │
└────────────────┬────────────────────────────────────┘
                 │
     ┌───────────┴───────────┐
     │                       │
┌────▼─────────────┐  ┌──────▼────────────┐
│ mdns_advertise_  │  │ mdns_discover_    │
│ sup (supervisor) │  │ sup (supervisor)  │
└────────┬─────────┘  └────────┬──────────┘
         │                     │
    ┌────▼────┐           ┌────▼────┐
    │ mdns_   │           │ mdns_   │
    │advertise│           │discover │
    │ (worker)│           │ (worker)│
    └─────────┘           └─────────┘
```

## Dependencies

- [gproc](https://hex.pm/packages/gproc) - Extended process registry with pub/sub
- [envy](https://hex.pm/packages/envy) - Environment configuration

## Requirements

- **Network**: Multicast must be enabled on your network
- **Firewall**: UDP port 5353 must be open for mDNS traffic
- **Same subnet**: Nodes must be on the same local network segment

## Troubleshooting

### No nodes discovered

1. **Check multicast**: Ensure multicast is enabled on your network
2. **Check firewall**: Allow UDP 5353 inbound and outbound
3. **Check environment**: Nodes must share the same `MDNS_ENVIRONMENT`
4. **Check cookie**: Nodes must share the same Erlang cookie for meshing

### Verify mDNS traffic

```shell
# Linux - capture mDNS packets
sudo tcpdump -i any port 5353

# Verify service is advertised
avahi-browse -a
```

## License

Apache-2.0

## Links

- [Original Repository](https://github.com/shortishly/mdns) - Peter Morgan's original implementation
- [Macula Platform](https://github.com/macula-io/macula) - HTTP/3 mesh networking platform
- [mDNS RFC](http://files.multicastdns.org/draft-cheshire-dnsext-multicastdns.txt) - Protocol specification
- [DNS-SD RFC 6763](http://www.ietf.org/rfc/rfc6763.txt) - Service discovery specification
- [Hex.pm Package](https://hex.pm/packages/mdns)
- [Documentation](https://hexdocs.pm/mdns)
