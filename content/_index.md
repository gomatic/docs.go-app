---
title: go-app
---

`go-app` is the gomatic ecosystem's **[urfave/cli/v3](https://github.com/urfave/cli) application framework and run harness**, shared by every gomatic CLI. It supplies the generic, command-agnostic glue — the [`Runner`](https://pkg.go.dev/github.com/gomatic/go-app#Runner)/[`Default`](https://pkg.go.dev/github.com/gomatic/go-app#Default) action combinators, the logger-in-metadata convention, the standard global flags, and the signal-aware [`Run`](https://pkg.go.dev/github.com/gomatic/go-app#Run) harness — and nothing tied to any one command. It composes [go-log](https://github.com/gomatic/go-log) and [go-output](https://github.com/gomatic/go-output) on top of urfave/cli/v3; command-specific flags, errors, and command trees stay in their own repositories.

- **Source:** [`gomatic/go-app`](https://github.com/gomatic/go-app)
- **API reference:** [pkg.go.dev/github.com/gomatic/go-app](https://pkg.go.dev/github.com/gomatic/go-app)

## Install

```sh
go get github.com/gomatic/go-app
```

## Why a shared framework

Every gomatic CLI repeats the same plumbing: parse global logging flags, build a logger, stash it where actions can reach it, decode a result to JSON or YAML, and run the root command under signal-aware cancellation. `go-app` owns that plumbing **once** so each command repository carries only its own flags, config, and work. A command becomes a small [`Runner`](https://pkg.go.dev/github.com/gomatic/go-app#Runner) — a typed function over a config and positional arguments — wired into a `*cli.Command` with the standard flags and hooks.

The library owns the **mechanism only**: it ships no command definitions, no command-specific flags, and no error values. Each consumer declares those in its own package and composes them with the combinators here.

## Usage

### Define a command

Write the command's work as a [`Runner`](https://pkg.go.dev/github.com/gomatic/go-app#Runner), bind it with [`Default`](https://pkg.go.dev/github.com/gomatic/go-app#Default), and run the root command through [`Run`](https://pkg.go.dev/github.com/gomatic/go-app#Run) for SIGINT/SIGTERM-aware cancellation:

```go
package main

import (
	"context"
	"log/slog"
	"os"

	app "github.com/gomatic/go-app"
	"github.com/gomatic/go-log"
	"github.com/urfave/cli/v3"
)

type config struct{}

type result struct {
	Message string `json:"message" yaml:"message"`
}

// greet is the command's work: it receives the bound config and positional
// arguments and returns a result the action combinator encodes.
func greet(_ context.Context, logger *slog.Logger, _ config, args ...string) (result, error) {
	logger.Info("greeting", "args", args)
	return result{Message: "hello"}, nil
}

func main() {
	var cfg config
	var logCfg log.LoggerConfig

	cmd := &cli.Command{
		Name:     "greeter",
		Metadata: map[string]any{},
		Flags: []cli.Flag{
			app.LogLevelFlag(&logCfg, "GREETER_"),
			app.LogFormatFlag(&logCfg, "GREETER_", log.FormatText),
			app.OutputFlag("GREETER_"),
		},
		Before: app.LoggerBefore(func(*cli.Command) *slog.Logger {
			return logCfg.NewLogger(os.Stderr)
		}),
		Action: app.Default(&cfg, greet),
	}

	app.Run(context.Background(), cmd, os.Args, os.Exit)
}
```

Each global flag resolves from its `--flag`, the matching `GREETER_*` environment variable, or its default. [`OutputFlag`](https://pkg.go.dev/github.com/gomatic/go-app#OutputFlag) selects the result encoding (`json` or `yaml`), which the `Default` action applies when writing the runner's result to the root command's writer.

### The standard global flags

Three constructors return the conventional global flags, each sourced from an `<envPrefix>` environment variable so consumers share one naming scheme:

```go
app.LogLevelFlag(&logCfg, "GREETER_")               // --log-level / GREETER_LOG_LEVEL (default "info")
app.LogFormatFlag(&logCfg, "GREETER_", log.FormatText) // --log-format / GREETER_LOG_FORMAT
app.OutputFlag("GREETER_")                           // --output, -o / GREETER_OUTPUT (default "json")
```

`LogLevelFlag` and `LogFormatFlag` bind into a [`log.LoggerConfig`](https://github.com/gomatic/go-log); the `def` argument to `LogFormatFlag` lets a CLI default to text while a daemon defaults to json.

### The logger-in-metadata convention

The logger built in the `Before` hook is stored in the root command's `Metadata` under [`LoggerMetadataKey`](https://pkg.go.dev/github.com/gomatic/go-app#pkg-constants) and retrieved by every action through [`GetLogger`](https://pkg.go.dev/github.com/gomatic/go-app#GetLogger):

```go
Before: app.LoggerBefore(func(c *cli.Command) *slog.Logger {
	return logCfg.NewLogger(os.Stderr)
}),
```

[`LoggerBefore`](https://pkg.go.dev/github.com/gomatic/go-app#LoggerBefore) wraps a [`GetLoggerFunc`](https://pkg.go.dev/github.com/gomatic/go-app#GetLoggerFunc) into a cli `Before` hook; the [`Default`](https://pkg.go.dev/github.com/gomatic/go-app#Default) action then passes the stored logger to the runner. `GetLogger` falls back to [`slog.Default`](https://pkg.go.dev/log/slog#Default) when no logger is present or the command is nil, so a runner always receives a usable logger.

## Design

- **Command-agnostic.** The package ships no command definitions, command-specific flags, or error values — only the combinators, flags, and harness every CLI reuses. Each consumer owns its own command tree.
- **Generic combinators.** [`Runner[CONFIG, RESULT]`](https://pkg.go.dev/github.com/gomatic/go-app#Runner) and [`Default`](https://pkg.go.dev/github.com/gomatic/go-app#Default) are parameterized over a command's config and result types, so a command's work stays a plain typed function with no cli coupling.
- **Injected seams.** [`Run`](https://pkg.go.dev/github.com/gomatic/go-app#Run) takes the args slice and the `exit` function as parameters, and the logger source is injected through [`LoggerBefore`](https://pkg.go.dev/github.com/gomatic/go-app#LoggerBefore) — so a `main` is exercised end to end in tests.
- **Signal-aware.** `Run` derives a [`signal.NotifyContext`](https://pkg.go.dev/os/signal#NotifyContext) for SIGINT/SIGTERM, logs any non-nil error under the command's name, and exits non-zero via the injected `exit`.
- **Marked as a library.** A `library.go` build-tagged marker (never compiled) flags the repo as a library rather than a CLI for gomatic tooling and conventions.

## Who uses it

Every gomatic CLI builds on `go-app`, composing it with [go-log](https://github.com/gomatic/go-log) and [go-output](https://github.com/gomatic/go-output): [`renderizer`](https://github.com/gomatic/renderizer), [`template.cli`](https://github.com/gomatic/template.cli), and the other [`gomatic/go-*`](https://github.com/orgs/gomatic/repositories?q=go-) libraries.
