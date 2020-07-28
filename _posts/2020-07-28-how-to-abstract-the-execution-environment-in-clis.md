---
layout: post
title: How to abstract the execution environment in CLIs
date: 2020-07-28
---

Whenever I write CLIs in Go I find useful to create an abstaction between the
logic of the CLI from the details of the execution environment. This post
describes how to create such an abstraction to improve the testability of the
CLI code.

The most straightforward CLI is a program that embeds the entirety of its logic
in the `main()` function. This kind of program looks very similar to this:

```
func main() {
    if len(os.Args) < 2 {
        fmt.Fprintf(os.Stderr, "usage: %s NAME\n", os.Args[0])
        os.Exit(1)
    }
    if err := updateUserName(os.Args[1]); err != nil {
        fmt.Fprintf(os.Stderr,  "error: update user name: %v\n", err)
        os.Exit(2)
    }
    name, err := readUserName()
    if err != nil {
        fmt.Fprintf(os.Stderr, "error: read user name: %v\n", err)
        os.Exit(2)
    }
    fmt.Printf("the user name is: '%s'\n", name)
}
```

The first problem with the `main()` function is that it is very difficult to
invoke from a test. Moreover, even if I managed to invoke `main()` from a test,
the calls to `os.Exit()` would be a big problem. The first step towards
testability is to move the bulk of the code in a new function that returns the
exit code.

```
func main() {
    os.Exit(run())
}

func run() int {
    if len(os.Args) < 2 {
        fmt.Fprintf(os.Stderr, "usage: %s NAME\n", os.Args[0])
        return 1
    }
    if err := updateUserName(os.Args[1]); err != nil {
        fmt.Fprintf(os.Stderr, "error: update user name: %v\n", err)
        return 2
    }
    name, err := readUserName()
    if err != nil {
        fmt.Fprintf(os.Stderr, "error: read user name: %v\n", err)
        return 2
    }
    fmt.Printf("the user name is: '%s'\n", name)
    return 0
}
```

I could easily invoke `run()` from a test, but passing arguments to it is still
difficult. This is due to the fact that the input to `run()` are the command
line parameters stored in the global variable `os.Args`. The next logical step
is to pass the command line parameters as an argument to `run()`.

```
func main() {
    os.Exit(run(os.Args))
}

func run(args []string) int {
    if len(args) < 2 {
        fmt.Fprintf(os.Stderr, "usage: %s NAME\n", args[0])
        return 1
    }
    if err := updateUserName(args[1]); err != nil {
        fmt.Fprintf(os.Stderr, "error: update user name: %v\n", err)
        return 2
    }
    name, err := readUserName()
    if err != nil {
        fmt.Fprintf(os.Stderr, "error: read user name: %v\n", err)
        return 2
    }
    fmt.Printf("the user name is: '%s'\n", name)
    return 0
}
```

`run()` is now much easier to test. I can simulate different combinations of
command line parameters by passing a different `args` slice to it. The
observable behaviour of the function is now limited to the return code. What if
I want to assert whether specific messages are printed under certain
circumstances? Surely I can replace `os.Stdin` and `os.Stderr` with something
that captures the output of `run()`, but changing global state can introduce
unwanted side effects while the tests runs. It would be much easier if `run()`
accepted its output streams as parameters instead.

```
func main() {
    os.Exit(run(os.Stdout, os.Stderr, os.Args))
}

func run(stdout, stderr io.Writer, args []string) int {
    if len(args) < 2 {
        fmt.Fprintf(stderr, "usage: %s NAME\n", args[0])
        return 1
    }
    if err := updateUserName(args[1]); err != nil {
        fmt.Fprintf(stderr, "error: update user name: %v\n", err)
        return 2
    }
    name, err := readUserName()
    if err != nil {
        fmt.Fprintf(stderr, "error: read user name: %v\n", err)
        return 2
    }
    fmt.Fprintf(stdout, "the user name is: '%s'\n", name)
    return 0
}
```

Now `run()` doesn't use any global variable and is much easier to test. In fact,
I can now easily test that when not enough parameters are passed to `run()`, it
should print the usage information.

```
func TestNotEnoughParameters(t *testing.T) {
    var stderr strings.Builder

    if exit := run(ioutil.Discard, &stderr, []string{"cmd"}); exit == 0 {
        t.Fatalf("invalid exit code: %d", exit)
    }

    if output := stderr.String(); output != "usage: cmd NAME\n" {
        t.Fatalf("invalid output: %s", output)
    }
}
```

This test invokes `run()` with a command line that contains only the executable
name and no other parameters, and asserts that `run()` returns with a non-zero
exit code. Since I'm not interested in what is printed to stdout, I pass
`ioutil.Discard` to `run()` in place of the `stdout` parameter. The usage should
be printed on `stderr`, so I create a `strings.Builder` (which implementes the
`io.Writer` interface) and pass it in place of the `stderr` parameter. At the
end of the test, I assert that the output caputred by the `strings.Builder` is
the one I expect.

Looking closely at the `run()` function, I created a small abstraction over the
execution environment. In fact, every program:

- receives command line arguments from the shell, abstracted by the parameter
  `args []string`;
- is attached to stdout and stderr, abstracted by the parameters `stdout, stderr
  io.Writer`;
- returns with an exit code, abstracted by the `int` return value.

This simple abstraction over the operating system and the shell allows me to
write more testable code, since I can mock a big part of the real execution
environment.
