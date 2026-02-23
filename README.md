# Linux::Event::Listen

[![CI](https://github.com/haxmeister/perl-linux-event-listen/actions/workflows/ci.yml/badge.svg)](https://github.com/haxmeister/perl-linux-event-listen/actions/workflows/ci.yml)


Listening sockets for **Linux::Event**, without injecting any methods into the loop.

## Install

```bash
cpanm Linux::Event::Listen
```

## Usage

```perl
use v5.36;
use Linux::Event;
use Linux::Event::Listen;

my $loop = Linux::Event->new;

my $listen = Linux::Event::Listen->new(
  loop => $loop,
  host => '127.0.0.1',
  port => 3000,

  on_accept => sub ($loop, $client_fh, $peer, $listen) {
    # You own $client_fh (already non-blocking).
    $loop->watch($client_fh,
      read => sub ($loop, $fh, $w) {
        my $buf;
        my $n = sysread($fh, $buf, 8192);
        if (!defined $n || $n == 0) {
          $w->cancel;
          close $fh;
          return;
        }
        # ... handle $buf ...
      },
    );
  },
);

$loop->run;
```

## Semantics

- Accept loop drains the accept queue until `EAGAIN` (required for edge-triggered readiness).
- A fairness cap (`max_accept_per_tick`, default 128) prevents a hot listener from starving other watchers.
- Accepted client sockets are set non-blocking and handed to user code; the user owns them.

See the POD in `Linux::Event::Listen` for full details.

### Optional: EMFILE/ENFILE handling

You can provide `on_emfile` to implement a reserve-FD mitigation strategy when the process runs out of file descriptors.

## UNIX domain sockets

```perl
my $listen = Linux::Event::Listen->new(
  loop => $loop,
  path => '/tmp/app.sock',
  unlink => 1,
  on_accept => sub ($loop, $client_fh, $peer, $listen) {
    ...
  },
);
```

## UNIX socket lifecycle

When using `path => ...`, `unlink => 1` removes an existing filesystem entry before binding.
`unlink_on_cancel` defaults true for internally-created UNIX sockets.
In wrap mode (`fh => ...`), Listen does not know the filesystem path and will not unlink it.
