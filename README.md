threadgroup
-----------

Threadgroup is a utility to facilitate clean and quick shutdown of related,
long-running threads or resources. Threads or resources can call `Add` to signal
that shutdown should be blocked until they have finished, and then can call
`Done` when they have finished. Calling `Stop` will block until all resources
have called `Done`, and will return an error if future resources attempt to call
`Add`.

Threadgroup has two helper functions, `OnStop` and `AfterStop`, which can help
to clean up resources which are intended to run for the life of the group.
Functions added to the threadgroup with `OnStop` will be called immediately
after `Stop` is called, before waiting for all existing threads to return.
`OnStop` is frequently called with resources like a net.Listener, where you want
to halt new connections immediately. `AfterStop` will be called after waiting
for all resources to return. `AfterStop` is frequently used for resources like
loggers, which need to be closed but not until they are not needed anymore.

Finally, `IsStopped` returns a channel that gets closed when `Stop` is called,
which can be passed as a cancel channel to things like `net.Dial` to facilitate
shutting down quickly when `Stop` is called.

Example:
```go
var tg threadgroup.ThreadGroup

// Create the logger and set it to shutdown upon closing.
log := NewLogger()
tg.AfterStop(func() {
	log.Close()
}

// Create a thread to repeatedly dial a remote address with quick shutdown.
go func() {
	for {
		// Perform the dial, but abort quickly if 'Stop' is called.
		dialer := &net.Dialer{
			Cancel:  tg.StopChan(),
			Timeout: time.Minute,
		}
		conn, err := dialer.Dial("tcp", 8.8.8.8)
		if err == nil {
			conn.Close()
		}

		// Sleep for an hour, but abort quickly if 'Stop' is called.
		select {
		case <-time.After(time.Hour):
			continue
		case <-tg.StopChan():
			return
		}
	}
}()

// Calling Stop will result in a quick, organized shutdown that closes all
// long-running resources.
```
