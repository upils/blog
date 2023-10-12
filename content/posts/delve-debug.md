---
title: "Debug a debugger while debugging"
date: 2023-10-12T23:35:07+02:00
draft: false
tags: ["ubuntu", "debugger"]
---

## Failing to build from source

Recently I worked with schopin to solve a bug in the [delve](https://github.com/go-delve/delve) Debian package failing to build from source in the newly released Mantic Minotaur Ubuntu (https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=1052030).

This bug appeared following a rebuild of the package, without any change to the package. The build failed because tests were failing.

The error message was not very useful:

```
=== RUN   TestRecursiveStructure
    support.go:246: enabling recording for TestRecursiveStructure
    proc_test.go:1215: proc_test.go:1473: EvalVariable("aas"): could not find symbol value for aas
```

We quickly agree that we needed to determine at what point the code used an external piece of code (library, runtime, etc.) that could have been updated in the new Ubuntu release.

## Reproduce the fail

To reproduce locally the fail I used [sbuild](https://wiki.ubuntu.com/SimpleSbuild) in the following way:

```bash
pull-lp-source delve
cd delve-1.21.0
sbuild --no-clean
```

## Debug

After confirming a bunch of tests were failing the same way, I picked `TestRecursiveStructure` randomly and decided to debug it. To so so, I used... delve.

```bash
schroot -c mantic-amd64 -u root
apt install golang
go install github.com/go-delve/delve/cmd/dlv@latest
puill-lp-source delve
cd delve-1.21.0/pkg/proc && go test -c -gcflags='-N -l'
~/go/bin/dlv exec proc.test -- -test.run TestRecursiveStructure
```

I learned several interesting things here:
- you can use `go test` with the `-c` option to compile tests in a specific package and get the resulting test binary
- you can give `-gcflags='-N -l'` flags to avoid any optimizations. At first I did not know this, and while debugging the debugger jump to weird location or missed skipped some sections (due to optimizations and inlining).
- you can give parameters to `dlv` that should be given to the debugged binary. This way I could specifically target the test I wanted (even though I later added a breakpoint on this test).

Note: To compare with a previously working situation I did the same in my jammy-based host machine. I reproduce each `dlv` command in both environments to compare results. 

Playing around in the debugger and using a simple "divide and conquer" strategy I tried to pinpoint where the test failed. Example: 

```bash
~/go/bin/dlv exec proc.test -- -test.run TestRecursiveStructure
(dlv) b github.com/go-delve/delve/pkg/proc_test.TestRecursiveStructure
(dlv) c
(dlv) s
(dlv) s
...
(dlv) n
...
# damn, missed it
(dlv) r
(dlv) c
(dlv) s
...
(dlv) b evalVariableOrError
(dlv) cond 2 symbol == "aas"
(dlv) r
(dlv) c
(dlv) s
```

By also reading the code I learned that first the test build a target binary selected in the `_fixtures` folder and start a debugging session.
Then a `EvalScope` object containing information on the stack of the target is built. This object is derived, with `FrameToScope`, from `StackFrames`.
By comparison with the working test, I noticed the first frame referenced `/usr/lib/go-1.21/src/runtime/panic.go`:

```
(dlv) b github.com/go-delve/delve/pkg/proc.FrameToScope
...
(dlv) n
> github.com/go-delve/delve/pkg/proc.FrameToScope() ./eval.go:142 (PC: 0x7ac401)
   137:		if maxaddr > minaddr && maxaddr-minaddr < maxFramePrefetchSize {
   138:			thread = cacheMemory(thread, minaddr, int(maxaddr-minaddr))
   139:		}
   140:	
   141:		s := &EvalScope{Location: frames[0].Call, Regs: frames[0].Regs, Mem: thread, g: g, BinInfo: t.BinInfo(), target: t, frameOffset: frames[0].FrameOffset()}
=> 142:		s.PC = frames[0].lastpc
   143:		return s
   144:	}
   145:	
   146:	// ThreadScope returns an EvalScope for the given thread.
   147:	func ThreadScope(t *Target, thread Thread) (*EvalScope, error) {
(dlv) p frames[0]
github.com/go-delve/delve/pkg/proc.Stackframe {
	Current: github.com/go-delve/delve/pkg/proc.Location {
		PC: 4403204,
		File: "/usr/lib/go-1.21/src/runtime/panic.go",
		Line: 1188,
		Fn: *(*"github.com/go-delve/delve/pkg/proc.Function")(0xc000928b30),},
	Call: github.com/go-delve/delve/pkg/proc.Location {
		PC: 4403204,
		File: "/usr/lib/go-1.21/src/runtime/panic.go",
		Line: 1188,
		Fn: *(*"github.com/go-delve/delve/pkg/proc.Function")(0xc000928b30),},
	Regs: github.com/go-delve/delve/pkg/dwarf/op.DwarfRegisters {
		StaticBase: 0,
		CFA: 824634785272,
		FrameBase: 824634785272,
		ObjBase: 0,
...
```

First I thought the test did not instantiated the binary properly and made it panic. But then I thought "what if the tested binary is just panicking all by itself". To confirm, I restarted the process, step through the program until the binary was compiled, exited the debugger and executed the binary.

```bash
(mantic-amd64)root@t14:/root/delve-1.21.0/pkg/proc# /tmp/testvariables2.42d9a519 
panic: time: missing Location in call to Date

goroutine 1 [running]:
time.Date(0x4bea91?, 0x4bea80?, 0x4bea7d?, 0x0?, 0x0?, 0x0?, 0x0?, 0x0?)
	/usr/lib/go-1.21/src/time/time.go:1469 +0x4c5
time.parse({0x4bea6d, 0x13}, {0x4bea80, 0x13}, 0x3?, 0x0)
	/usr/lib/go-1.21/src/time/format.go:1398 +0x2111
time.ParseInLocation({0x4bea6d, 0x13}, {0x4bea80, 0x13}, 0x0?)
	/usr/lib/go-1.21/src/time/format.go:1029 +0xe6
main.main()
	/root/delve-1.21.0/_fixtures/testvariables2.go:376 +0x24c5
```

Here it was. The tests were fine, but the tested binary was broken. Here is the relevant code.

```go
loc, _ := time.LoadLocation("Mexico/BajaSur")
tim2, _ := time.ParseInLocation("2006-01-02 15:04:05", "2022-06-07 02:03:04", loc)
```

The call to `time.ParseInLocation()` panicked because `loc` was nil.

This can traced back to some changes in `tzdata` not providing some "uncommon" time zones. They are now available in `tzdata-legacy`.

## The one-line fix

To fix, I simply replaced the timezone with `America/Mazatlan` (see https://github.com/go-delve/delve/pull/3527).
