# erlperf

[![Build Status](https://github.com/max-au/erlperf/actions/workflows/erlang.yml/badge.svg?branch=master)](https://github.com/max-au/erlperf/actions) [![Hex.pm](https://img.shields.io/hexpm/v/erlperf.svg)](https://hex.pm/packages/erlperf) [![Hex Docs](https://img.shields.io/badge/hex-docs-blue.svg)](https://hexdocs.pm/erlperf)

Erlang Performance & Benchmarking Suite.
Simple way to say "this code is faster than that one".

Build (tested with OTP 23, 24, 25):

```bash
    $ rebar3 as prod escriptize
```

# TL; DR

Find out how many times per sample (second) a function can be run  (beware of shell escaping your code!):

```bash
    $ ./erlperf 'rand:uniform().'
    Code                    ||        QPS       Time
    rand:uniform().          1   13942 Ki      71 ns
```

Run four processes executing `rand:uniform()` in a tight loop, and see that code is indeed
concurrent:

```bash
    $ ./erlperf 'rand:uniform().' -c 4
    Code                    ||        QPS       Time
    rand:uniform().          4   39489 Ki     100 ns
```

Benchmark one function vs another, taking average of 10 seconds and skipping first second:

```bash
    $ ./erlperf 'rand:uniform().' 'crypto:strong_rand_bytes(2).' --samples 10 --warmup 1
    Code                                 ||        QPS       Time     Rel
    rand:uniform().                       1   15073 Ki      66 ns    100%
    crypto:strong_rand_bytes(2).          1    1136 Ki     880 ns      7%
```

Run a function passing the state into the next iteration. This code demonstrates performance difference
between `rand:uniform_s` with state passed explicitly, and `rand:uniform` reading state from the process
dictionary:

```bash
    $ ./erlperf 'r(_Init, S) -> {_, NS} = rand:uniform_s(S), NS.' --init_runner 'rand:seed(exsss).' 'r() -> rand:uniform().'
    Code                                                    ||        QPS       Time     Rel
    r(_Init, S) -> {_, NS} = rand:uniform_s(S), NS.          1   20272 Ki      49 ns    100%
    r() -> rand:uniform().                                   1   15081 Ki      66 ns     74%
```

Squeeze mode: 
measure how concurrent your code is. In the example below, `code:is_loaded/1` is implemented as
`gen_server:call`, and all calculations are done in a single process. It is still possible to
squeeze a bit more from a single process by putting work into the queue from multiple runners,
therefore the example may show higher concurrency.

```bash
    $ ./erlperf 'code:is_loaded(local_udp).' --init 'code:ensure_loaded(local_udp).' --squeeze
    Code                               ||        QPS       Time
    code:is_loaded(local_udp).          5     927 Ki    5390 ns
```

Start a server (`pg` scope in this example), use it in benchmark, and shut down after:

```bash
    $ ./erlperf 'pg:join(scope, group, self()), pg:leave(scope, group, self()).' --init 'pg:start_link(scope).' --done 'gen_server:stop(scope).'
    Code                                                                   ||        QPS       Time
    pg:join(scope, group, self()), pg:leave(scope, group, self()).          1     336 Ki    2978 ns
```

Run the same code with different arguments, returned from `init_runner` function:

```bash
    $ ./erlperf 'runner(X) -> timer:sleep(X).' --init_runner '1.' 'runner(X) -> timer:sleep(X).' --init_runner '2.'
    Code                                 ||        QPS       Time     Rel
    runner(X) -> timer:sleep(X).          1        498    2008 us    100%
    runner(X) -> timer:sleep(X).          1        332    3012 us     66%
```
    
Determine how many times a process can join/leave pg2 group on a single node (use OTP 23, because pg2 is removed in
later versions):

```bash
    $ ./erlperf 'ok = pg2:join(g, self()), ok = pg2:leave(g, self()).' --init 'pg2:create(g).'
    Code                                                         ||        QPS       Time
    ok = pg2:join(g, self()), ok = pg2:leave(g, self()).          1      64021   15619 ns
```

Compare `pg` with `pg2` running two nodes (note the `-i` argument spawning an extra node to
run benchmark in):

```bash
    ./erlperf 'ok = pg2:join(g, self()), ok = pg2:leave(g, self()).' --init 'pg2:create(g).' 'ok = pg:join(g, self()), ok = pg:leave(g, self()).' --init 'pg:start(pg).' -i
    Code                                                         ||        QPS       Time     Rel
    ok = pg:join(g, self()), ok = pg:leave(g, self()).            1     241 Ki    4147 ns    100%
    ok = pg2:join(g, self()), ok = pg2:leave(g, self()).          1       1415     707 us      0%
```

Watch the progress of your test running (use -v option) with extra information: scheduler utilisation, dirty CPU & IO
schedulers, number of running processes, ports, ETS tables, and memory consumption. Last column is the job throughput.
When there are multiple jobs, multiple columns are printed.

```bash
    $ ./erlperf 'rand:uniform().' -q -v
    
    YYYY-MM-DDTHH:MM:SS-oo:oo  Sched   DCPU    DIO    Procs    Ports     ETS Mem Total  Mem Proc   Mem Bin   Mem ETS   <0.80.0>
    2022-04-08T22:42:55-07:00   3.03   0.00   0.32       42        3      20  30936 Kb   5114 Kb    185 Kb    423 Kb   13110 Ki
    2022-04-08T22:42:56-07:00   3.24   0.00   0.00       42        3      20  31829 Kb   5575 Kb    211 Kb    424 Kb   15382 Ki
    2022-04-08T22:42:57-07:00   3.14   0.00   0.00       42        3      20  32079 Kb   5849 Kb    211 Kb    424 Kb   15404 Ki
    <...>
    2022-04-08T22:43:29-07:00  37.50   0.00   0.00       53        3      20  32147 Kb   6469 Kb    212 Kb    424 Kb   49162 Ki
    2022-04-08T22:43:30-07:00  37.50   0.00   0.00       53        3      20  32677 Kb   6643 Kb    212 Kb    424 Kb   50217 Ki
    Code                    ||        QPS       Time
    rand:uniform().          8   54372 Ki     144 ns
```

Command-line benchmarking does not save results anywhere. It is designed to provide a quick answer to the question
"is that piece of code faster".

## Timed (low overhead) mode
Since 2.0, `erlperf` includes timed mode. It cannot be used for continuous benchmarking. In this mode
runner code is executed specified amount of times in a tight loop:

```bash
    ./erlperf 'rand:uniform().' 'rand:uniform(1000).' -l 10M
    Code                        ||        QPS       Time     Rel
    rand:uniform().              1   16319 Ki      61 ns    100%
    rand:uniform(1000).          1   15899 Ki      62 ns     97%
```

This mode effectively runs following code: `loop(0) -> ok; loop(Count) -> rand:uniform(), loop(Count - 1).`
Timed mode reduced benchmarking overhead (compared to continuous mode) by 1-2 ns per iteration.

# Benchmarking existing application
`erlperf` can be used to measure performance of your application running in production, or code that is stored
on disk.

## Running with existing codebase
Use `-pa` argument to add extra code path. Example:
```bash
    $ ./erlperf 'argparse:parse([], #{}).' -pa _build/test/lib/argparse/ebin
    Code                             ||        QPS       Time
    argparse:parse([], #{}).          1     955 Ki    1047 ns
```

If you need to add multiple released applications, supply `ERL_LIBS` environment variable instead:
```bash
    $ ERL_LIBS="_build/test/lib" erlperf 'argparse:parse([], #{}).'
    Code                             ||        QPS       Time
    argparse:parse([], #{}).          1     735 Ki    1361 ns
```

## Usage in production
It is possible to use `erlperf` to benchmark a running application (even in production, assuming necessary safety
precautions). To achieve this, add `erlperf` as a dependency, and use remote shell:

```bash
    # run a mock production node with `erl -sname production`
    # connect a remote shell to the production node
    erl -remsh production
    (production@max-au)3> erlperf:run(timer, sleep, [1]).
    488
```

## Continuous benchmarking
You can run a job continuously, to examine performance gains or losses while doing 
hot code reload. This process is designed to help during development and testing stages, 
allowing to quickly notice performance regressions.

Example source code:
```erlang
    -module(mymod).
    -export([do/1]).
    do(Arg) -> timer:sleep(Arg).
```
 
Example below assumes you have `erlperf` application started (e.g. in a `rebar3 shell`)

```erlang
    % start a logger that prints VM monitoring information
    > {ok, Logger} = erlperf_file_log:start_link(group_leader()).
    {ok,<0.235.0>}
    
    % start a job that will continuously benchmark mymod:do(),
    %  with initial concurrency 2.
    > JobPid = erlperf:start(#{init_runner => "rand:uniform(10).", 
        runner => "runner(Arg) -> mymod:do(Arg)."}, 2).
    {ok,<0.291.0>}
    
    % increase concurrency to 4
    > erlperf_job:set_concurrency(JobPid, 4).
    ok.
    
    % watch your job performance

    % modify your application code, 
    % set do(Arg) -> timer:sleep(2*Arg), do hot code reload
    > c(mymod).
    {module, mymod}.
    
    % see that after hot code reload throughput halved!
```

# Reference Guide

## Terms

* **runner**: code that gets continuously executed
* **init**: code that runs one when the job starts (for example, start some registered process or create an ETS table)
* **done**: code that runs when the job is about to stop (used for cleanup, e.g. stop some registered process)
* **init_runner**: code that is executed in every runner process (e.g. add something to process dictionary)
* **job**: single instance of the running benchmark (multiple runners)
* **concurrency**: how many processes are running concurrently, executing *runner* code
* **throughput**: total number of calls per sampling interval (for all concurrent processes)
* **cv**: coefficient of variation, the ratio of the standard deviation to the mean. Used to stop the concurrency 
(squeeze) test, the lower the *cv*, the longer it will take to stabilise and complete the test

## Using `erlperf` from `rebar3 shell` or `erl` REPL
Supported use-cases:
 * single run for MFA: ```erlperf:run({rand, uniform, [1000]}).``` or ```erlperf:run(rand, uniform, []).```
 * anonymous function: ```erlperf:run(fun() -> rand:uniform(100) end).```
 * anonymous function with an argument: ```erlperf:run(fun(Init) -> io_lib:format("~tp", [Init]) end).```
 * source code: ```erlperf:run("runner() -> rand:uniform(20).").```
 * (experimental) call chain: ```erlperf:run([{rand, uniform, [10]}, {erlang, node, []}]).```,
     see [recording call chain](#recording-call-chain). Call chain may contain only complete
     MFA tuples and cannot be mixed with functions.
 
Startup and teardown 
 * init, done and init_runner are supported (there is no done_runner,
 because it is never stopped in a graceful way)
 * init_runner and done may be defined with arity 0 and 1 (in the latter case,
 result of init/0 passed as an argument)
 * runner can be of arity 0, 1 (accepting init_runner return value) or 2 (first
 argument is init_runner return value, and second is state passed between runner invocations)
 
Example with mixed MFA:
```erlang
   erlperf:run(
       #{
           runner => fun(Arg) -> rand:uniform(Arg) end,
           init => 
               {pg, start_link, []},
           init_runner =>
               fun ({ok, Pid}) -> 
                   {total_heap_size, THS} = erlang:process_info(Pid, total_heap_size),
                   THS
               end,
           done => fun ({ok, Pid}) -> gen_server:stop(Pid) end
       }
   ).
``` 
 
Same example with source code:
```erlang
erlperf:run(
    #{
        runner => "runner(Max) -> rand:uniform(Max).",
        init => "init() -> pg:start_link().",
        init_runner => "init_runner({ok, Pid}) ->
            {total_heap_size, THS} = erlang:process_info(Pid, total_heap_size),
            THS.",
        done => "done({ok, Pid}) -> gen_server:stop(Pid)."
    }       
).
```

## Measurement options
Benchmarking is done by counting number of *runner* iterations done over
a specified period of time (**sample_duration**). 
By default, erlperf performs no **warmup** cycle, then takes 3 consecutive 
**samples**, using **concurrency** of 1 (single runner). It is possible 
to tune this behaviour by specifying run_options:
```erlang
    erlperf:run({erlang, node, []}, #{concurrency => 2, samples => 10, warmup => 1}).
```

Next example takes 10 samples with 100 ms duration. Note that throughput is reported
per *sample_duration*: if you shorten duration in half, throughput report will also be
halved:

```bash
    $ ./erlperf 'rand:uniform().' -d 100 -s 20
    Code                    ||        QPS       Time
    rand:uniform().          1    1480 Ki      67 ns
    $ ./erlperf 'rand:uniform().' -d 200 -s 20
    Code                    ||        QPS       Time
    rand:uniform().          1    2771 Ki      72 ns
```

## Benchmarking under lock contention
ERTS cannot guarantee precise timing when there is severe lock contention happening,
and scheduler utilisation is 100%. This often happens with ETS:
```bash
    $ ./erlperf -c 50 'ets:insert(ac_tab, {1, 2}).'
```
Running 50 concurrent processes trying to overwrite the very same key of an ETS
table leads to lock contention on a shared resource (ETS table/bucket lock).



## Concurrency test (squeeze)
Sometimes it's necessary to measure code running multiple concurrent
processes, and find out when it saturates the node. It can be used to
detect bottlenecks, e.g. lock contention, single dispatcher process
bottleneck etc.. Example (with maximum concurrency limited to 50):

```erlang
    > erlperf:run({code, is_loaded, [local_udp]}, #{warmup => 1}, #{max => 50}).
    {1284971,7}
```

In this example, 7 concurrent processes were able to squeeze 1284971 calls per second
for `code:is_loaded(local_udp)`.

## Benchmarking overhead
Benchmarking overhead varies depending on ERTS version and the way runner code is supplied. See the example: 

```erlang
    (erlperf@max-au)7> erlperf:benchmark([
            #{runner => "runner(X) -> is_float(X).", init_runner=>"2."}, 
            #{runner => {erlang, is_float, [2]}},
            #{runner => fun (X) -> is_float(X) end, init_runner => "2."}], 
        #{}, undefined).
    [105824351,66424280,5057372]
```

This difference is caused by the ERTS itself: running compiled code (first variant) with OTP 25 is 
two times faster than applying a function, and 20 times faster than repeatedly calling anonymous `fun`. Use
the same invocation method to get a relevant result.

Absolute benchmarking overhead may be significant for very fast functions taking just a few nanoseconds.
Use timed mode for such occasions.

## Experimental: recording call chain
This experimental feature allows capturing a sequence of calls as a list of
`{Module, Function, [Args]}`. The trace can be supplied as a `runner` argument
to `erlperf` for benchmarking purposes:

```erlang
    > f(Trace), Trace = erlperf:record(pg, '_', '_', 1000).
    ...
    
    % for things working with ETS, isolation is recommended
    > erlperf:run(#{runner => Trace}, #{isolation => #{}}).
    ...
    
    % Trace can be saved to file before executing:
    > file:write("pg.trace", term_to_binary(Trace)).
    
    % run the saved trace
    > {ok, Bin} = file:read_file("pg.trace"),
    > erlperf:run(#{runner => binary_to_term(Trace)}).
```

It's possible to create a Common Test testcase using recorded samples.
Just put the recorded file into xxx_SUITE_data:
```erlang
    benchmark_check(Config) ->
        {ok, Bin} = file:read_file(filename:join(?config(data_dir, Config), "pg.trace")),
        QPS = erlperf:run(#{runner => binary_to_term(Bin)}),
        ?assert(QPS > 500). % catches regression for QPS falling below 500
```

## Experimental: starting jobs in a cluster

It's possible to run a job on a separate node in the cluster.

```erlang
    % watch the entire cluster (printed to console)
    (node1@host)> {ok, _} = erlperf_history:start_link().
    {ok,<0.213.0>}
    (node1@host)> {ok, ClusterLogger} = erlperf_cluster_monitor:start_link(group_leader(), [sched_util, jobs]).
    {ok, <0.216.0>}
    
    % also log cluster-wide reports to file (jobs & sched_util)
    (node1@host)> {ok, FileLogger} = erlperf_cluster_monitor:start_link("/tmp/cluster", [time, sched_util, jobs]).
    {ok, <0.223.0>}

    % run the benchmarking process in a different node of your cluster
    (node1@host)> rpc:call('node2@host', erlperf, run, [#{runner => {rand, uniform, []}}]).
```

Cluster-wide monitoring will reflect changes accordingly.


# Implementation details
Starting with 2.0, `erlperf` uses call counting for continuous benchmarking purposes. This allows
the tightest possible loop without extra runtime calls. Running
`erlperf 'rand:uniform().' --init '1'. --done '2.' --init_runner '3.'` results in creating,
compiling and loading a module with this source code:

```erlang
    -module(unique_name).
    -export([init/0, init_runner/0, done/0, run/0]).

    init() ->
        1.

    init_runner() ->
        3.

    done() ->
        2.

    run() ->
        runner(), 
        run().

    runner() ->
        rand:uniform().
```

Number of `run/0` calls per second is reported as throughput. Before 2.0, `erlperf` 
used `atomics` to maintain a counter shared between all runner processes, introducing
unnecessary BIF call overhead. 

Timed (low-overhead) mode tightens it even further, turning runner into this function:
```erlang
runner(0) ->
    ok;
runner(Count) ->
    rand:uniform(),
    runner(Count - 1).
```