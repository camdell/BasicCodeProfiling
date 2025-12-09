# Basic Profiling for Research Code

Measuring code speed means determining how long a program or a portion of code
takes to run, often using timers, benchmarks, or profiling tools. It helps you
understand where your code spends time and whether certain parts are
inefficient. However, it’s important to remember that you should only care
about optimizing code performance when it actually wastes your time or the time
of others. Constantly chasing small speedups for fun, especially without
guidance from measurement, is usually unproductive. There are often more
valuable ways to spend your effort than micro-optimizing code that already runs
fast enough.

```python
print("Let's take a look!")
```

## Measuring Performance

**What do we measure?**
- *Wall time*: Real elapsed time; includes waiting, I/O, and scheduler delays.
- *CPU time*: Time the CPU actually spent running your process (user + system).
    - *User time*: Time running your code in user space (Python + C extensions)
    - *System time*: Time the kernel spent doing work on your behalf (I/O, syscalls, memory management)

**How do we measure it?**
- *date/time clocks*: reflect the current calendar time; can jump forward or backwards due to timezone adjustments.
  - please don’t use these to measure performance
- *monotonic clocks*: measure elapsed time only, never goes backwards and unaffected by system clock adjustments.

### Script-level Timing

Tools:
- `time` super simple and available nearly everywhere
- `hyperfine` times repetitions & calculates statistics, can easily compare multiple scripts. Needs to be installed.

```zsh
# time sleep 1

# hyperfine 'sleep .1'
hyperfine 'sleep .2' 'sleep .1' 'sleep .3'
```

### Language-level Timing

What if we want more granularity? Instead of timing entire programs, we can time subsets of programs.
This is how we can begin to identify what portions of programs are "slow" and which are not.

#### Block-level Timing

```python
from time import perf_counter # high performance monotonic clock
from time import now                          # date clock
from datetime import datetime; datetime.now() # date clock
from time import sleep
from random import Random

rnd = Random(0)

start = perf_counter()
total = (
    sum(rnd.randint(0, 100) for _ in range(10_000_000)) ** (1/2)
)
stop = perf_counter()
print(f'{stop - start:.6f}s for our slow computation')
```

We can turn this into a reusable context manager to time any arbitrary snippet of Python code.

```python
from time import perf_counter
from contextlib import contextmanager
from random import Random

@contextmanager
def timed(msg=''):
    start = perf_counter()
    stop = None
    try:
        yield lambda: stop - start
    finally:
        stop = perf_counter()
        print(f'{msg: <24} {(stop - start):,.6f}s'.strip())

rnd = Random(0)
with timed('sqrt(sum(10M random integers))') as t:
    total = (
        sum(rnd.randint(0, 100) for _ in range(10_000_000)) ** (1/2)
    )

print(f'{t() = }')
```

#### Function-level Timing

```python
from time import sleep, perf_counter
from random import Random
from functools import wraps

def timed(f):
    @wraps(f)
    def wrapper(*args, **kwargs):
        start = perf_counter()
        result = f(*args, **kwargs)
        stop = perf_counter()
        print(f'{stop - start:.6f}s')
        return result
    return wrapper

# @timed
def func(n, rnd=Random(0)):
    return sum(rnd.randint(0, 100) for _ in range(n)) ** (1/2)

# timed(func, 10)
# timed(func, 10_000)
# timed(func, 10_000_000)

func(10)
func(10_000)
func(10_000_000)
```

This is great, but what if I don’t want to rewrite my program in order to time its
subroutines/functions?

```python
from contextlib import contextmanager
from sys import setprofile
from time import perf_counter, sleep
from inspect import signature

def slow():
    sleep(.5)
    return True

def fast():
    sleep(.1)
    return True

@contextmanager
def timer(*funcs):
    towatch = {f.__code__: f.__qualname__ for f in funcs}
    active = {}
    def _profile(frame, event, arg):
        if event == 'call' and frame.f_code in towatch:
            active[frame.f_code] = perf_counter()
        elif event == 'return':
            try:
                start = active.pop(frame.f_code)
            except KeyError:
                pass
            else:
                print(f"func {towatch[frame.f_code]!r} took {perf_counter() - start:6f}s")
    setprofile(_profile)

    try:
        yield
    finally:
        setprofile(None)


def main():
    slow()
    slow()
    fast()

with timer(slow):
    main()
print('done')
```

## Profiling

```python
from cProfile import Profile
from time import sleep

def summation():
    total = 0
    for _ in range(10_000_000):
        total += 1
    return total

def exponentiation():
    start = 10_000_000
    for _ in range(10_000_000):
        start = pow(start, 0.5) # should be **= 0.5
    return start

with Profile() as pr:
    summation()
    exponentiation()

pr.print_stats(sort="cumtime")
```

You can even do this in a jupyter notebook!

*timing*:
    - prefix any line with `%timeit` to time a single line
    - prefix any cell with `%%timeit` to time a single cell

*profiling*:
    - prefix any line with `%prun` to time a single line
    - prefix any cell with `%%prun` to time a single cell
    - jupyter line_profiler()

## (In)Efficiently Loading Data from Lots of Files

```python
from pathlib import Path
from datetime import date, timedelta
from random import Random

def date_range(start, stop, step=timedelta(days=1)):
    while start <= stop:
        yield start
        start += step

rnd = Random(0)
data_dir = Path("data", "dates")
data_dir.mkdir(exist_ok=True, parents=True)

for dt in date_range(date(2000, 1, 1), date(2000, 12, 31)):
    with open(data_dir / f"{dt:%Y-%m-%d}.txt", "w") as f:
        for _ in range(100):
            value = rnd.randint(0, 100)
            f.write(f"{value}\n")
    print(f'wrote {f.name}!')
print("done")
```

```sh
find data/dates -name '*.txt' | xargs -n1 wc -l | sort -h
```

```python
from numpy import array, int64
from pathlib import Path
from numpy import append as np_append, loadtxt, concat
from fileinput import FileInput

data_dir = Path("data", "dates")

def a(): # I.
    xs = array([], dtype=int64)
    for path in data_dir.glob("*.txt"):
        with open(path, "r") as f:
            for ln in f:
                xs = np_append(xs, int(ln))
    return xs

def b(): # II.
    xs = []
    for path in data_dir.glob("*.txt"):
        values = loadtxt(path, dtype=int64)
        xs.append(values)
    return concat(xs, dtype=int64)

def c(): # III.
    with FileInput(files=data_dir.glob("*.txt")) as f:
        return loadtxt(f)


from cProfile import Profile
import pstats
from contextlib import redirect_stdout
from io import StringIO

# profile each function and collect results & profile stats
results = {}
for func in [a, b, c]:
    with Profile() as pr:
        res = func()

    with redirect_stdout(StringIO()) as buffer:
        pr.print_stats("cumtime")
    results[func] = (res, buffer.getvalue())

for func, (_, perf) in results.items():
    print(
        f' Func {func.__qualname__!r} '.center(80, '='),
        *perf.splitlines()[:1],
        *perf.splitlines()[3:12],
        sep='\n'
    )

# verify results are accurate/comparable to one another
from itertools import pairwise
numeric_results = (res for res, _ in results.values())
assert all((a == b).all() for a, b in pairwise(numeric_results))
```

## Wrap Up

Basic measurement tells you how long your code takes overall; profiling tells
you which parts of the code are responsible for that time.

Remember, your code is only slow if it begins to waste your time or the time of others.
If you just need a script to run one time and one time only, it is okay if it takes a couple hours.

| Aspect     | Basic Measurement                   | Profiling                                                   |
| ---------- | ----------------------------------- | ----------------------------------------------------------- |
| Goal       | How long or how much overall        | Detailed insight into performance distribution              |
| Scope      | Entire program, single metric       | Function-by-function, line-by-line, or resource-specific    |
| Tools      | `time`, `hyperfine`, `timeit`       | `cProfile`, `line_profiler`, `pyinstrument`, `setprofile`   |
| Output     | Single number (seconds, CPU time)   | Detailed report: calls, time per call, percentage of total  |
| Use Case   | Quick estimation, benchmarking      | Optimization, understanding bottlenecks, performance tuning |

