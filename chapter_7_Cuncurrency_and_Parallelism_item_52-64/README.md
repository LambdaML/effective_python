# Chapter 7. Concurrency and Parallelism

***Concurrency*** enables a computer to do many different things ***seemingly*** at the same time. For example, operating system on a single CPU computer rapidly changes which program is running. In doing so, it interleaves execution of the program, providing the illusion that the programs are running simultaneously.

***Parallelism***, on the other hand, involves ***actually*** doing many different things at the same time. A computer with multiple CPUs can execute multiple programs simultaneously. Each CPU running instructions of different programs, allowing computational progress during the same instance.

Within a single program, concurrency is a tool to make it easier to solve certain types of problems. Concurrent programs enable many distinct paths of execution, including separate streams of I/O, to make forward process in a way that seems to be both simultaneous and independent.

The key difference between *concurrency* and *parallelism* is ***speedup***. When two distinct paths of execution of the program make forward progress in parallel, the time it takes to do the total work is cut by *half*; speed of execution is faster by factor of two. In contrast, concurrent programs may run thousands of separate paths of execution seemingly in parallel bu provide no speed for total work. 

Python provides variety of ways to write a concurrent program. Threads support relatively small amount of concurrency, when coroutines enables a vast number of concurrent functions. Python also enables to do parallel works through system calls, subprocesses, and C extension. But it can be very difficult to make concurrent Python code truly run in parallel. It is important to understand how to best utilize Python in these different situations.

## Item 52: Use `subprocess` to Manage Child Process

Python is grate and has many ways to run subprocesses, and the best choice for managing child processes is to use the `subprocess` built-in module. For example, here is a function to start a process, read its output, and verify that it terminated cleanly:
```python
import os
import subprocess


os.environ["COMSPEC"] = "powershell"

result = subprocess.run(
    ["Write-Host", "Hello from the child"],
    capture_output=True,
    encoding="utf-8",
    shell=True)

result.check_returncode() # No exception means clean exit
print(result.stdout)
```
    >>>
    Hello from the child

* Child processes run independently from their parent process, the Python interpreter. If we use `popen` to create a subprocess, we can periodically poll child process status:
```python
proc = subprocess.Popen(["sleep", "1"], shell=True)
while proc.poll() is None:
    print("Working...")
    import time
    time.sleep(0.4)

print("Exit status", proc.poll())
```
    >>>
    Working...
    Working...
    Working...
    Working...
    Exit status 0

* Decoupling the child process from the parent frees up the parent process to run child processes in parallel. Here, we do this by starting all the child processes together with `Popen` upfront:
```python
import time


start = time.time()
sleep_procs = []
for _ in range(10):
    proc = subprocess.Popen(["sleep", "1"], shell=True)
    sleep_procs.append(proc)
```
Later, we wait them to finish their I/O and terminate with `communicate` method:
```python
for proc in sleep_procs:
    proc.communicate

end = time.time()
delta = end - start

print(f"Finished in {delta:.3} seconds")
```
    >>>
    Finished in 2.09 seconds

If these processes run in sequence, the total delay would be 10 seconds and more, not ~ 2 seconds.

* We can also pipe data from Python program into a subprocess and retrieve its output.This allows us to utilize many program to run in parallel.For example, we want to encrypt our data with `openssl`. Starting the child process with commend-line argument and I/O pipe is easy:
```python
import os


def run_encrypt(data):
    env = os.environ.copy()
    env["password"] = "4(;QlJ?mVXv?^|+q@UmR%eQaq|Aqh):?"
    proc = subprocess.Popen(
        ["openssl", "enc", "-des3", "-pass", "env:password"],
        env=env,
        stdin=subprocess.PIPE,
        stdout=subprocess.PIPE)
    proc.stdin.write(data)
    proc.stdin.flush() # Ensure that the child gets input
    return proc
```
Here we encrypt random numbers
```python
procs = []
for _ in range(3):
    data = os.urandom(10)
    proc = run_encrypt(data)
    procs.append(proc)
```
The child processes run in parallel and consume their input, and retrieve their final output:
```python
for proc in procs:
    out, _ = proc.communicate()
    print(out[-10:])
```
    >>>
    b'\x96\x82\x82\x9e\xde\xfa\x90\xe21t'
    b'&\nRG\xab\x04\x99\xd7\xc1"'
    b'x\xe3\xb8\xb8\xd7\xe5t\x03@\x8e'
* It is also possible to create a chain of parallel processes, connecting the output of one child process to the input of another, and so on. Here is the command-line tool to output Whirlpool hash of the input stream:
```python
def run_hash(input_stdin):
    return subprocess.Popen(
        ["openssl", "dgst", "-whirlpool", "-binary"],
        stdin=input_stdin,
        stdout=subprocess.PIPE)
```
Now we can create set of processes to encrypt some data and hash the output:
```python
encrypt_procs = []
hash_procs = []

for _ in range(3):
    data = os.urandom(100)
    encrypt_proc = run_encrypt(data)
    encrypt_procs.append(encrypt_proc)
    hash_proc = run_hash(encrypt_proc.stdout)
    hash_procs.append(hash_proc)
    # Ensure that the child consumes the input stream and 
    # the communicate() method doesn't inadvertently steal
    # input from the child. Also lets SIGPIPE propagate to 
    # upstream process if the downstream process dies.
    encrypt_proc.stdout.close()
    encrypt_proc.stdout = None
```
THe I/O between the child processes happens automatically once they are started:
```python
for proc in encrypt_procs:
    proc.communicate()
    assert proc.returncode == 0

for proc in hash_procs:
    out, _ = proc.communicate()
    print(out[-10:])
    assert proc.returncode == 0
```
    >>>
    b'\xe5{.\x8fu\xa5\x06\x9d@P'
    b'\xd8\xce\xba\x84\x9d\xcf\xc5\xfe\x86\x95'
    b'[\x02\x9a\xb5M\x0e\xbd\x99)\x86'

* If we worry that child process never terminate or somehow blocking on input or output pipes, we can use `timeout` parameter to the `communicate` method. THis causes an exception to be raised of the process hasn't finished in time:
```python
proc = subprocess.Popen(['sleep', '10'], shell=True)

try:
    proc.communicate(timeout=0.1)
except subprocess.TimeoutExpired:
    proc.terminate()
    proc.wait()

print('Exit status', proc.poll())
```
    >>>
    Exit status 1

## Item 53: Use Treads for Blocking I/O, Avoid Parallelism

The standard implementation of Python called CPython. CPython runs a Python program in two steps. First, it parses and compiles the source text into `bytecode`, which is low-level representation of the program as 8-bit instructions. Then, CPython runs bytecode using stack-based interpreter. The bytecode interpreter has state that must be maintained and coherent while Python program executes. CPython enforces the coherence with a mechanism called `global interpreter lock` (GIL).

Essentially, the GIL is a mutual-execution lock (mutex) that prevents CPython being affected by preemptive multithreading, where one thread takes control of a program by interrupting another thread. Such an interruption could corrupt the interpreter state (e.g. garbage collection reference count) if it comes at an unexpected time. The GIL prevents these interruptions and ensures that every bytecode instruction work correctly with the CPython implementation and its C-extension modules. 

THe GIL has an important negative side effect. With programs written in languages like `C++` or `Java`, having multiple threads of execution means that a program can utilize multiple CPU cores at the same time. Although Python supports multiple threads of execution, GIL causes only one of them to be executed at a time. WHich mean that when you reach for threads for parallel computation and speed up your Python program, yu will be sorely disappointed.

* For example, we need to do something computationally intensive in Python, like number factorization algorithm:
```python
def factorize(number):
    for i in range(1, number + 1):
        if number % i ==0:
            yield i
```
Factoring a set of numbers would take sometime:
```python
import time 


numbers = [2139079, 1214759, 1516637, 1852285]
start = time.time()


for number in numbers:
    list(factorize(number))

end = time.time()
delta = end - start
print(f"Took {delta:.3} seconds")


```
    >>>
    Took 0.756 seconds

* Using multiple threads to do this would make sense. Let's us try to implement it:
```python
from threading import Thread


class FactorizeThread(Thread):
    def __init__(self, number):
        super().__init__()
        self.number = number
    def run(self):
        self.factors = list(factorize(self.number))
```
Then, we start start a thread for each number to factorize in parallel:
```python
start = time.time()


threads = []
for number in numbers:
    thread = FactorizeThread(number)
    thread.start()
    threads.append(thread)


```
We wait them to calculate and print the result:
```python
for thread in threads:
    thread.join()


end = time.time()
delta = end - start
print(f"Took {delta:.3} seconds")
```
    >>>
    Took 1.16 seconds

As we can see it took even longer to do the same calculations. 



# 
* [Back to repo](https://github.com/almazkun/effective_python#effective_python)