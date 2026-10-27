# Process-method preparation before SVM integration

Prepared branch: `prep/nommu-process-method`.
Base: `3957dbcdc4b0afbb0d557de14295bc51f8a1c074`.
Code change: `486f07c86f30374175f442e0bccdd501db89fd41`, equivalent to the
preserved `codex/lmbench-process-method` change `686653ffcd117d3408cd9ebf165d3f62b7067fb3`.
No upstream submission or old-branch rewrite.

## Meaning of the change

`lat_proc --fork` and `--vfork` select the measured operation. The harness still
uses private fork normally and a separate CLONE_VM stack with CONFIG_NOMMU.
A vfork child must take only its exec/_exit path: it must not change scheduler
or stdio state shared with its parent. Pipe writers and harness termination
retain that distinction.

## Local correctness validation, 2026-10-27

- Native glibc: fork and vfork each pass fork+exit, fork+execve and shell cases;
  64 KiB pipe transfer passes. Seven successful completions.
- Native musl CONFIG_NOMMU harness: explicit fork, explicit vfork and the pipe
  case pass on the host kernel. This is a harness reference, not a NOMMU guest.
- Plain UML NOMMU: vfork+exit, vfork+execve, shell and 64 KiB pipe pass, followed
  by completion0 and normal poweroff/host exit0. Private fork is unsupported
  in this configuration and is not claimed as passing.
- Guest kernel source7c5fbf911203ac41c9eb4a8f6931265220b0534d differs from
  the accepted ad26b8b28d7307811241cfbf1a69b9d7184da611 only in
  Documentation/mm/swmmu-kselftest-build.rst. The runtime code is identical.

These are completion checks, not publication-quality latency comparisons.
`ENOUGH=10000 TIMING_O=0 LOOP_O=0` bypasses host-dependent calibration for
bounded correctness iterations. Earlier unbounded-calibration attempts hit
short limits before clone; strace confirms this was not a fork/harness fault.

Example guest commands (musl static build with `-DCONFIG_NOMMU`):

```sh
export ENOUGH=10000 TIMING_O=0 LOOP_O=0
cp ./hello /tmp/hello
./lat_proc --vfork -N 2 fork
./lat_proc --vfork -N 2 exec
./lat_proc --vfork -N 2 shell
./bw_pipe -N 2 -m 65536 -M 1048576
```

Native commands use the ordinary build and exercise both method selectors.
Full logs, kernel config/provenance and exact build commands are stored in the
Obsidian pre-SVM execution record under nommu-swmmu/Codex-Logs, dated2026-10-27.
