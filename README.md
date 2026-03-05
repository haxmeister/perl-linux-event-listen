# Linux::Event::Listen

[![CI](https://github.com/haxmeister/perl-linux-event-listen/actions/workflows/ci.yml/badge.svg)](https://github.com/haxmeister/perl-linux-event-listen/actions/workflows/ci.yml)

Nonblocking bind + accept for the Linux::Event ecosystem.

## Linux::Event Ecosystem

The Linux::Event modules are designed as a composable stack of small,
explicit components rather than a framework.

Each module has a narrow responsibility and can be combined with the others
to build event-driven applications.

Core layers:

Linux::Event
    The event loop. Linux-native readiness engine using epoll and related
    kernel facilities. Provides watchers and the dispatch loop.

Linux::Event::Listen
    Server-side socket acquisition (bind + listen + accept). Produces accepted
    nonblocking filehandles.

Linux::Event::Connect
    Client-side socket acquisition (nonblocking connect). Produces connected
    nonblocking filehandles.

Linux::Event::Stream
    Buffered I/O and backpressure management for an established filehandle.

Linux::Event::Fork
    Asynchronous child process management integrated with the event loop.

Linux::Event::Clock
    High resolution monotonic time utilities used for scheduling and deadlines.

Canonical network composition:

Listen / Connect
        ↓
      Stream
        ↓
  Application protocol

Example stack:

Linux::Event::Listen → Linux::Event::Stream → your protocol

or

Linux::Event::Connect → Linux::Event::Stream → your protocol

The core loop intentionally remains a primitive layer and does not grow
into a framework. Higher-level behavior is composed from small modules.

## Synopsis

use v5.36;
use Linux::Event;
use Linux::Event::Listen;

my $loop = Linux::Event->new;

Linux::Event::Listen->new(
  loop => $loop,
  host => '127.0.0.1',
  port => 3000,

  on_accept => sub ($loop, $fh, $peer, $listen) {

    # You own $fh
    $loop->watch($fh,
      read => sub ($loop, $fh, $w) {

        my $buf;
        my $n = sysread($fh, $buf, 8192);

        if (!defined $n || $n == 0) {
          $w->cancel;
          close $fh;
          return;
        }

        # handle $buf
      },
    );
  },
);

$loop->run;

## Canonical integration with Stream

use Linux::Event::Stream;

Linux::Event::Listen->new(
  loop => $loop,
  host => '127.0.0.1',
  port => 3000,

  on_accept => sub ($loop, $fh, $peer, $listen) {

    Linux::Event::Stream->new(
      loop => $loop,
      fh   => $fh,

      codec      => 'line',

      on_message => sub ($stream, $line, $data) {
        $stream->write_message("echo: $line");
        $stream->close_after_drain if $line eq 'quit';
      },
    );
  },
);

$loop->run;

## Notes

Accepted sockets are nonblocking (best-effort enforced).

You own accepted filehandles.

Listener teardown is explicit via $listen->cancel.

## License

Same terms as Perl itself.
