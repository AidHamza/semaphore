> # semaphore
>
> Semaphore pattern implementation with a timeout of lock/unlock operations based on channels.

[![Awesome](https://cdn.rawgit.com/sindresorhus/awesome/d7305f38d29fed78fa85652e3a63e154dd8e8829/media/badge.svg)](https://github.com/avelino/awesome-go#goroutines)
[![Build Status](https://travis-ci.org/kamilsk/semaphore.svg?branch=master)](https://travis-ci.org/kamilsk/semaphore)
[![Coverage Status](https://coveralls.io/repos/github/kamilsk/semaphore/badge.svg)](https://coveralls.io/github/kamilsk/semaphore)
[![Go Report Card](https://goreportcard.com/badge/github.com/kamilsk/semaphore)](https://goreportcard.com/report/github.com/kamilsk/semaphore)
[![Exago](https://api.exago.io/badge/rank/github.com/kamilsk/semaphore)](https://www.exago.io/project/github.com/kamilsk/semaphore)
[![GoDoc](https://godoc.org/github.com/kamilsk/semaphore?status.svg)](https://godoc.org/github.com/kamilsk/semaphore)
[![License](https://img.shields.io/github/license/mashape/apistatus.svg?maxAge=2592000)](LICENSE)

## Code of Conduct

The project team follows [Contributor Covenant v1.4](http://contributor-covenant.org/version/1/4/).
Instances of abusive, harassing or otherwise unacceptable behavior may be reported by contacting
the project team at feedback@octolab.org.

---

## Usage

### Console tool for command execution in parallel

```bash
$ semaphore create 2
$ semaphore add -- docker build
$ semaphore add -- vagrant up
$ semaphore add -- ansible-playbook
$ semaphore wait --notify --timeout=1m
```

See more details [here](cmd#semaphore).

### HTTP response' time limitation

This example shows how to follow SLA.

```go
sla := 100 * time.Millisecond
sem := semaphore.New(1000)

http.Handle("/do-with-timeout", http.HandlerFunc(func(rw http.ResponseWriter, req *http.Request) {
    done := make(chan struct{})

    ctx, cancel := context.WithTimeout(req.Context(), sla)
    defer cancel()

    release, err := sem.Acquire(ctx.Done())
    if err != nil {
        http.Error(rw, err.Error(), http.StatusGatewayTimeout)
        return
    }
    defer release()

    go func() {
        defer close(done)

        // do some heavy work
    }()

    // wait what happens before
    select {
    case <-ctx.Done():
        http.Error(rw, err.Error(), http.StatusGatewayTimeout)
    case <-done:
        // send success response
    }
}))
```

### HTTP request' throughput limitation

This example shows how to limit request' throughput.

```go
limiter := func(limit int, timeout time.Duration, handler http.Handler) http.Handler {
	throughput := semaphore.New(limit)
	return http.HandlerFunc(func(rw http.ResponseWriter, req *http.Request) {
		ctx, cancel := context.WithTimeout(req.Context(), timeout)
		defer cancel()

		release, err := throughput.Acquire(ctx.Done())
		if err != nil {
			http.Error(rw, err.Error(), http.StatusTooManyRequests)
			return
		}
		defer release()

		handler.ServeHTTP(rw, req)
	})
}

http.Handle("/do-limited", limiter(1000, time.Minute, http.HandlerFunc(func(rw http.ResponseWriter, req *http.Request) {
	// do some limited work
})))
```

## Installation

```bash
$ go get github.com/kamilsk/semaphore
```

### Mirror

```bash
$ egg bitbucket.org/kamilsk/semaphore
```

> [egg](https://github.com/kamilsk/egg) is an `extended go get`.

### Update

This library is using [SemVer](http://semver.org) for versioning and it is not
[BC](https://en.wikipedia.org/wiki/Backward_compatibility)-safe.
Therefore, do not use `go get -u` to update it, use [Glide](https://glide.sh) or something similar for this purpose.

## Contributing workflow

Read first [Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments).

### Code quality checking

```bash
$ make docker-pull-tools
$ make check-code-quality
```

### Testing

#### Local

```bash
$ make install-deps
$ make test # or test-with-coverage
$ make bench
```

#### Docker

```bash
$ make docker-pull
$ make complex-tests # or complex-tests-with-coverage
$ make complex-bench
```

## Feedback

[![Gitter](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/kamilsk/semaphore)
[![@ikamilsk](https://img.shields.io/badge/author-%40ikamilsk-blue.svg)](https://twitter.com/ikamilsk)

## Notes

- tested on Go 1.5, 1.6, 1.7 and 1.8
- [research](research)
