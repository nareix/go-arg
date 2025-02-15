[![GoDoc](https://godoc.org/github.com/alexflint/go-arg?status.svg)](https://godoc.org/github.com/alexflint/go-arg)
[![Build Status](https://travis-ci.org/alexflint/go-arg.svg?branch=master)](https://travis-ci.org/alexflint/go-arg)
[![GolangCI](https://golangci.com/badges/github.com/alexflint/go-arg.svg)](https://golangci.com/r/github.com/alexflint/go-arg)
[![Coverage Status](https://coveralls.io/repos/alexflint/go-arg/badge.svg?branch=master&service=github)](https://coveralls.io/github/alexflint/go-arg?branch=master)
[![Report Card](https://goreportcard.com/badge/github.com/alexflint/go-arg)](https://goreportcard.com/badge/github.com/alexflint/go-arg)

## Structured argument parsing for Go

```shell
go get github.com/alexflint/go-arg
```

Declare command line arguments for your program by defining a struct.

```go
var args struct {
	Foo string
	Bar bool
}
arg.MustParse(&args)
fmt.Println(args.Foo, args.Bar)
```

```shell
$ ./example --foo=hello --bar
hello true
```

### Required arguments

```go
var args struct {
	ID      int `arg:"required"`
	Timeout time.Duration
}
arg.MustParse(&args)
```

```shell
$ ./example
Usage: example --id ID [--timeout TIMEOUT]
error: --id is required
```

### Positional arguments

```go
var args struct {
	Input   string   `arg:"positional"`
	Output  []string `arg:"positional"`
}
arg.MustParse(&args)
fmt.Println("Input:", args.Input)
fmt.Println("Output:", args.Output)
```

```
$ ./example src.txt x.out y.out z.out
Input: src.txt
Output: [x.out y.out z.out]
```

### Environment variables

```go
var args struct {
	Workers int `arg:"env"`
}
arg.MustParse(&args)
fmt.Println("Workers:", args.Workers)
```

```
$ WORKERS=4 ./example
Workers: 4
```

```
$ WORKERS=4 ./example --workers=6
Workers: 6
```

You can also override the name of the environment variable:

```go
var args struct {
	Workers int `arg:"env:NUM_WORKERS"`
}
arg.MustParse(&args)
fmt.Println("Workers:", args.Workers)
```

```
$ NUM_WORKERS=4 ./example
Workers: 4
```

You can provide multiple values using the CSV (RFC 4180) format:

```go
var args struct {
    Workers []int `arg:"env"`
}
arg.MustParse(&args)
fmt.Println("Workers:", args.Workers)
```

```
$ WORKERS='1,99' ./example
Workers: [1 99]
```

### Usage strings
```go
var args struct {
	Input    string   `arg:"positional"`
	Output   []string `arg:"positional"`
	Verbose  bool     `arg:"-v" help:"verbosity level"`
	Dataset  string   `help:"dataset to use"`
	Optimize int      `arg:"-O" help:"optimization level"`
}
arg.MustParse(&args)
```

```shell
$ ./example -h
Usage: [--verbose] [--dataset DATASET] [--optimize OPTIMIZE] [--help] INPUT [OUTPUT [OUTPUT ...]] 

Positional arguments:
  INPUT 
  OUTPUT

Options:
  --verbose, -v            verbosity level
  --dataset DATASET        dataset to use
  --optimize OPTIMIZE, -O OPTIMIZE
                           optimization level
  --help, -h               print this help message
```

### Default values

```go
var args struct {
	Foo string `default:"abc"`
	Bar bool
}
arg.MustParse(&args)
```

### Default values (before v1.2)

```go
var args struct {
	Foo string
	Bar bool
}
arg.Foo = "abc"
arg.MustParse(&args)
```

### Arguments with multiple values
```go
var args struct {
	Database string
	IDs      []int64
}
arg.MustParse(&args)
fmt.Printf("Fetching the following IDs from %s: %q", args.Database, args.IDs)
```

```shell
./example -database foo -ids 1 2 3
Fetching the following IDs from foo: [1 2 3]
```

### Arguments that can be specified multiple times, mixed with positionals
```go
var args struct {
    Commands  []string `arg:"-c,separate"`
    Files     []string `arg:"-f,separate"`
    Databases []string `arg:"positional"`
}
```

```shell
./example -c cmd1 db1 -f file1 db2 -c cmd2 -f file2 -f file3 db3 -c cmd3
Commands: [cmd1 cmd2 cmd3]
Files [file1 file2 file3]
Databases [db1 db2 db3]
```

### Custom validation
```go
var args struct {
	Foo string
	Bar string
}
p := arg.MustParse(&args)
if args.Foo == "" && args.Bar == "" {
	p.Fail("you must provide either --foo or --bar")
}
```

```shell
./example
Usage: samples [--foo FOO] [--bar BAR]
error: you must provide either --foo or --bar
```

### Version strings

```go
type args struct {
	...
}

func (args) Version() string {
	return "someprogram 4.3.0"
}

func main() {
	var args args
	arg.MustParse(&args)
}
```

```shell
$ ./example --version
someprogram 4.3.0
```

### Embedded structs

The fields of embedded structs are treated just like regular fields:

```go

type DatabaseOptions struct {
	Host     string
	Username string
	Password string
}

type LogOptions struct {
	LogFile string
	Verbose bool
}

func main() {
	var args struct {
		DatabaseOptions
		LogOptions
	}
	arg.MustParse(&args)
}
```

As usual, any field tagged with `arg:"-"` is ignored.

### Custom parsing

Implement `encoding.TextUnmarshaler` to define your own parsing logic.

```go
// Accepts command line arguments of the form "head.tail"
type NameDotName struct {
	Head, Tail string
}

func (n *NameDotName) UnmarshalText(b []byte) error {
	s := string(b)
	pos := strings.Index(s, ".")
	if pos == -1 {
		return fmt.Errorf("missing period in %s", s)
	}
	n.Head = s[:pos]
	n.Tail = s[pos+1:]
	return nil
}

func main() {
	var args struct {
		Name NameDotName
	}
	arg.MustParse(&args)
	fmt.Printf("%#v\n", args.Name)
}
```
```shell
$ ./example --name=foo.bar
main.NameDotName{Head:"foo", Tail:"bar"}

$ ./example --name=oops
Usage: example [--name NAME]
error: error processing --name: missing period in "oops"
```

### Custom parsing with default values

Implement `encoding.TextMarshaler` to define your own default value strings:

```go
// Accepts command line arguments of the form "head.tail"
type NameDotName struct {
	Head, Tail string
}

func (n *NameDotName) UnmarshalText(b []byte) error {
	// same as previous example
}

// this is only needed if you want to display a default value in the usage string
func (n *NameDotName) MarshalText() ([]byte, error) {
	return []byte(fmt.Sprintf("%s.%s", n.Head, n.Tail)), nil
}

func main() {
	var args struct {
		Name NameDotName `default:"file.txt"`
	}
	arg.MustParse(&args)
	fmt.Printf("%#v\n", args.Name)
}
```
```shell
$ ./example --help
Usage: test [--name NAME]

Options:
  --name NAME [default: file.txt]
  --help, -h             display this help and exit

$ ./example
main.NameDotName{Head:"file", Tail:"txt"}
```

### Description strings

```go
type args struct {
	Foo string
}

func (args) Description() string {
	return "this program does this and that"
}

func main() {
	var args args
	arg.MustParse(&args)
}
```

```shell
$ ./example -h
this program does this and that
Usage: example [--foo FOO]

Options:
  --foo FOO
  --help, -h             display this help and exit
```

### Subcommands

*Introduced in `v1.1.0`*

Subcommands are commonly used in tools that wish to group multiple functions into a single program. An example is the `git` tool:
```shell
$ git checkout [arguments specific to checking out code]
$ git commit [arguments specific to committing]
$ git push [arguments specific to pushing]
```

The strings "checkout", "commit", and "push" are different from simple positional arguments because the options available to the user change depending on which subcommand they choose.

This can be implemented with `go-arg` as follows:

```go
type CheckoutCmd struct {
	Branch string `arg:"positional"`
	Track  bool   `arg:"-t"`
}
type CommitCmd struct {
	All     bool   `arg:"-a"`
	Message string `arg:"-m"`
}
type PushCmd struct {
	Remote      string `arg:"positional"`
	Branch      string `arg:"positional"`
	SetUpstream bool   `arg:"-u"`
}
var args struct {
	Checkout *CheckoutCmd `arg:"subcommand:checkout"`
	Commit   *CommitCmd   `arg:"subcommand:commit"`
	Push     *PushCmd     `arg:"subcommand:push"`
	Quiet    bool         `arg:"-q"` // this flag is global to all subcommands
}

arg.MustParse(&args)

switch {
case args.Checkout != nil:
	fmt.Printf("checkout requested for branch %s\n", args.Checkout.Branch)
case args.Commit != nil:
	fmt.Printf("commit requested with message \"%s\"\n", args.Commit.Message)
case args.Push != nil:
	fmt.Printf("push requested from %s to %s\n", args.Push.Branch, args.Push.Remote)
}
```

Some additional rules apply when working with subcommands:
* The `subcommand` tag can only be used with fields that are pointers to structs
* Any struct that contains a subcommand must not contain any positionals


### API Documentation

https://godoc.org/github.com/alexflint/go-arg

### Rationale

There are many command line argument parsing libraries for Go, including one in the standard library, so why build another?

The `flag` library that ships in the standard library seems awkward to me. Positional arguments must preceed options, so `./prog x --foo=1` does what you expect but `./prog --foo=1 x` does not. It also does not allow arguments to have both long (`--foo`) and short (`-f`) forms.

Many third-party argument parsing libraries are great for writing sophisticated command line interfaces, but feel to me like overkill for a simple script with a few flags.

The idea behind `go-arg` is that Go already has an excellent way to describe data structures using structs, so there is no need to develop additional levels of abstraction. Instead of one API to specify which arguments your program accepts, and then another API to get the values of those arguments, `go-arg` replaces both with a single struct.

### Backward compatibility notes

Earlier versions of this library required the help text to be part of the `arg` tag. This is still supported but is now deprecated. Instead, you should use a separate `help` tag, described above, which removes most of the limits on the text you can write. In particular, you will need to use the new `help` tag if your help text includes any commas.
