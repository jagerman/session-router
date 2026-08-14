# Tunnelled TCP

Session Router carries TCP for embedded clients by tunnelling it inside a QUIC connection nested in
the session's data channel.

An embedded client (one linking `libsessionrouter-core`, with no permission or ability to create a TUN
device) cannot put raw IP packets on the wire the way a full client does, and cannot obtain raw IP
packets for its own application's loopback socket either.  Session path data is also deliberately
unreliable — encapsulated packets carried as QUIC datagrams between hops, with no acknowledgement,
sequencing or retransmission anywhere along the way — so something has to supply reliability in
process.  Rather than implement a TCP stack to do it, we nest a QUIC connection inside the session and
let it provide retransmission, ordering, flow control, congestion control and per-stream error codes.

## Scope

Supported: **an embedded client connecting out to a full client.**  The embedded side initiates; the
full (tun) side accepts and terminates each stream into a real TCP connection to its own tun address.

Not supported, deliberately:

- **Inbound TCP to an embedded client.**  An embedded client cannot accept tunnelled connections.
  Earlier drafts of this document proposed a registration callback for this, plus SYN interception on
  full clients wanting to reach an embedded one; none of that is built.
- **Relays.**  A relay is reachable by QUIC natively, so `establish_udp` already covers everything
  needed there.

## Data path

    ┌ embedded client ─────────┐                              ┌ full client ───────────┐
    │ application              │                              │ service                │
    │ TCP -> [::1]:47165 ──────│─┐                            │ TCP 10.0.0.1:80 <──────│─┐
    ├──────────────────────────┤ │                            ╞════════════════════════╡ │
    │ session router (in app)  │ │                            │ session router         │ │
    │ one QUIC stream per conn │ └─ ... session router path ─┐ │                        │ │
    │ inside one QUIC conn ────│────  (unreliable datagrams) └─│─> stream -> TCP connect┘ │
    │ [::1]:47165 <────────────│─┐                            │ to its own tun address   │
    └──────────────────────────┘ └────────────────────────────│──────────────────────────┘

The application connects to a port on `[::1]` that `establish_tcp` hands back.  Each connection to
that port opens one stream on the tunnel's QUIC connection; the far end reads the destination port
from the head of the stream and makes a matching TCP connection locally.  Both directions are then
just data on the stream.

## Wire details

**Stream preamble.**  The first two bytes of each stream are the destination port, big-endian.  The
accepting side buffers until it has both, then connects; anything already received beyond the preamble
is written to the socket ahead of whatever the socket subsequently carries.  Port 0 is refused.

**Capability advertisement.**  A client that can accept tunnelled TCP sets `protocol_flag::TCP_TUNNEL`
(`1 << 5`) in its client contact, which in practice means any client with a tun device.  An initiator
that already holds a contact without that flag refuses to map a port at all, rather than mapping one
that could never carry anything; if the contact only arrives later and lacks the flag, the failure is
reported as `tunnel_failure::no_tcp`.

Note this is a *new* flag rather than 1.0.x's `QUIC_TUNNEL` (`1 << 1`), which advertised the opposite
role — "I am embedded, reach me via a tunnel" — and was therefore set by exactly the clients that
cannot accept one.

**No encryption of its own (yet).**  The inner connection needs neither authentication nor encryption:
the session layer already encrypts end to end and the path layer onion-encrypts.  It currently uses a
fixed, well-known keypair as a stand-in, which costs a TLS handshake and a second AEAD pass.  Both go
away when libquic gains null crypto.

**Packet sizing.**  The inner connection is capped at the QUIC minimum (1200 bytes).  An inner packet
becomes a 1278-byte session message once session and path overhead (78 bytes) are added, which rides in
a single path datagram as long as the outer hop's payload is at least 1324 bytes; below that it is
split in two and both halves must survive.  Dropping the inner cap to 1076 would make splitting
impossible on any path, but libquic currently refuses a cap below 1200.

**Idle handling.**  QUIC closes a connection with no activity at all on it, which would kill a TCP
connection that is merely idle, so the inner connection sends keep-alive pings.  To avoid paying for
those on a tunnel nobody is using, the inner connection is torn down a minute after its last stream
goes away, and rebuilt on demand.

## Backpressure

Both directions are flow-controlled, since either end can be the slow one:

- application → tunnel: stream watermarks stop reading from the local socket when the tunnel falls
  behind, and resume when it drains.
- tunnel → application: a bufferevent's output buffer grows without limit and so signals nothing on
  its own, so the stream is paused once the queued output passes a threshold and resumed from the
  write callback once it drains.

## Close semantics

Distinguishing an orderly close from a failure matters: conflating them truncates transfers silently,
which is exactly what went wrong in the previous (lokinet 0.9.x, ngtcp2) implementation of this idea.

| Event | Action |
|---|---|
| Local TCP EOF (clean) | send FIN on the stream; keep receiving (a real half-close) |
| Local TCP error | close the stream with `TCP_FAILURE` |
| Stream FIN received | shut down the socket's write side, but only once queued output has drained |
| Stream closed with an error | drop the local connection without a clean FIN |
| Accepting side cannot connect | close the stream with `CONNECT_FAILED` |
| Accepting side refuses the port | close the stream with `REFUSED` |

## Using it

```c++
// Holding the claim is what keeps the tunnel up.
session::router::tcp_tunnel tcp = router.establish_tcp(
        "kcpyawm9se7trdbzncimdi5t7st4p5mh9i1mg7gkpuubi4k4ku1y.sesh", 80,
        [](auto info) { /* ready: connect to [::1]:info.local_port */ },
        [](auto failure) { /* unreachable, unsupported, or timeout */ });
```

The application then makes ordinary TCP connections to `[::1]:tcp->local_port`, as many as it likes;
each becomes a separate connection to port 80 on the remote.

Releasing the claim (destroying it, or calling `reset()`) closes the listening port *and* the
connections established through it.  Asking for a remote and port that is already mapped hands back
another claim on the same mapping rather than a new one, so releasing one claim never pulls the tunnel
out from under another holder.

## Acceptor policy

The accepting side connects to **its own tun address** on whatever port the stream asks for.  Any port
is allowed, which is parity with how tun mode already behaves: a remote sending raw IP packets to a
full client's tun address can likewise reach anything listening there.  If that ever needs narrowing,
an allow-list belongs in config rather than in the tunnel.

## Known gaps

- Null crypto in libquic, which removes the inner handshake round trip (via unconditional early data)
  and the second AEAD pass.
- A sub-1200 inner packet cap, so an inner packet never splits across two path datagrams.
- Nothing here handles IPv4: the accepting side connects to the tun's IPv6 address only, deliberately,
  as IPv4 is on its way out.  A tun client with no IPv6 address cannot accept tunnelled TCP.
- Nested congestion control (the inner connection's BBR inside each hop's BBR, over a channel whose
  congestion shows up as deep buffering rather than loss) is unmeasured.
