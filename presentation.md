[bit.ly/3KzQD4H](https://bit.ly/3KzQD4H) <!-- .element: class="r-fit-text" -->

---

# Python (and Julia) at Scale

Examples in Earth Science

**Stefano Campanella**

---

## Motivation <!-- .element: class="r-fit-text" -->

:::

Modern scientific computing workloads include more and more scripting languages (as Python). 

They provide interactive computing environments and are the standard for AI/ML.

There is the need for using them while leveraging the HPC resources in a way transparent to the user. 

NOTE: **Modern HPC workloads** include more and more the execution of programs written in **high-level, dynamic or scripting languages**. The reason is that scientist may not be familiar with low-level languages and advanced programming techniques. There is need for replicating on large cluster the workflows scientists are used to on their **local machines**, while benefitting from **computational resources of large distributed systems**. Furthermore, the **interactivity** allowed by these languages makes possible to adopt a fast-paced development cycles, which is more suited to the exploratory nature of scientific investigations. Finally, these languages are the de-facto standard for **artificial intelligence and machine learning**, the usage of which is increasingly important in the scientific community.

:::

A clear understanding of the performance, concurrency, and of the available software libraries in these languages is required.

I will provide two examples of scientific computing using Python (and Julia) taken from Earth science.

NOTE: From the point of view of engineers and system administrators, there is the need for exposing a functioning environment to the users, as similar as possible to the one they are used to yet exploiting large computing resources, while managing **complex software dependencies and data pipelines**. In order to do that, it is crucial to understand the **performance**, the way of expressing **concurrency**, the usage of **accelerators** (such as GPUs), and the available **software libraries** in these languages. In the following I will briefly introduce some of these topics, including two examples of applications I had the occasion to work on, taken from **Earth science**. I will mainly talk about **Python**, but I'll discuss **Julia** at the end of the talk.

---

## Scientific Computing <!-- .element: class="r-fit-text" -->

:::

Python is known to be slow.

NOTE: Scripting languages, and Python in particular, have fame of being **slow**. Here, when I'll say Python, I will mean **CPython**, the reference implementation of the language.

:::

### Why? 

The execution on the virtual machine of the Python bytecode does not exploit the expedients of modern hardware (especially caching). This is apparent in tight loops.

```python
# Lamest example: don't do this

total = 0
for x in xs:
    total += x
``` 
<!-- .element: class="fragment" -->

NOTE: An analysis of the precise reasons of the poor performance of Python is difficult and rather technical. However, the general idea is that Python code is executed on a **virtual machine** (after being translated to bytecode) that is not benefitting from the **expedients of modern hardware**, especially **caching**. Indeed, this is particularly manifest when executing repeated floating point operations on contiguous memory, that is **tight loops**, which can be very fast on modern architectures.

:::

Libraries like Numpy implements routines in low-level languages, like C or Fortran. Array programming is encouraged, as in MATLAB.

```python
# Lamest example (cont'd): instead do this
import numpy

total = numpy.sum(xs) # supposing xs is a numpy.ndarray
```

NOTE: The solution to this problem, as in MATLAB, is to express these loops as function calls or operators acting on arrays, a style of programming known as **array programming**. Libraries such as **Numpy** implement these functions and operators using statically compiled languages such as C, Fortran or C++. CPython is written in C and can be relatively easily extended using this language (C++ is more complicated due to name mangling).

:::

Other approaches: 

* F2PY (automatic boilerplate generation),
* Cython (static compilation),
* Numba, Julia (JIT compilation).

_(Also, Python can run standalone applications.)_ <!-- .element: class="fragment" -->

NOTE: There are projects to integrate C++ code in Python and also limited support for FORTRAN is offerend in Numpy, with a program called F2PY which automatically generates the boilerplate code. There is also Cython in which one can write code in a typed, subset of the Python language and compile it to get performance closer to native. Finally, Numba, thanks to the LLVM framework, compiles on the fly (just in time, or JIT) Python functions to native code. This is the same approach used in Julia. Python can be used also to execute other programs and intercept the IO. This can be useful when working with other programs that were designed to work as stand-alone applications. An example of this sort will be provided in the following.

---

## Concurrency <!-- .element: class="r-fit-text" -->

:::

### Multitasking

Preemptive: multi-threading/processing.

Collaborative: aync/await, asyncIO

NOTE: Python supports natively **several types concurrency**. Among **preemptive multitasking**, it supports **multi-threading** and **multi-processing**. Cooperative multitasking is provided at language level and utilities to work with coroutines are provided both in the standard and third party libraries. When using preemptive multitasking the operating system is responsible for interrupting one thread or process and switching to another. Instead, in cooperative multi-tasking coroutines have a mechanism to suspend and resume (or, more precisely, to signal when they are available to resume) their execution. By itself, **cooperative multi-tasking does not imply parallelism**. If however the coroutines await non-blocking operations that happens in other threads or don't require the CPU (such as IO), then one can have effectively parallelism.

:::

### GIL: The Elephant in the Room

The Global Interpreter Lock is a mutex.

TLDR. Use multi-threading when not running CPython.

NOTE: One thing to keep in mind is the existence of the **Global Interpreter Lock**, or GIL. This is a **mutex** on the Python interpreter that prevents multiple threads to execute bytecode instructions in parallel. However, the **GIL may be released** and, as such, many operations work in a **non-blocking** fashion. For example, many routines in the Numpy library release the GIL. It should also be noted that Numpy can spawn new threads by itself, depending on the linear algebra packages Numpy is build against and system settings. As a rule of thumb, one should **use multi-threading when most of the execution time is spent outside the Python interpreter** and multi-processing otherwise. Multi-processing circumvents the GIL effectively by spawning new interpreter processes and serializing and deserializing the state. Hence, one should pay attention to the **overhead due to serialization and to memory consumption** due to multiple copies of the same objects. Another option is to use **MPI4Py**, which is a wrapper around MPI calls.

---

## Distributed Systems <!-- .element: class="r-fit-text" -->

:::

![Ray logo](https://github.com/ray-project/ray/raw/master/doc/source/images/ray_header_logo.png) <!-- .element width="50%" -->
![Dask logo](https://docs.dask.org/en/stable/_images/dask_horizontal.svg) <!-- .element width="35%" -->

Ray, Dask: One scheduler, many workers.

NOTE: While MPI4Py is an available option to run Python on distributed systems, it often requires a complete re-write of existing code. Libraries such as **Ray or Dask** offer other options. I will refer mainly to the latter since it is more mature.

:::
### Dask

* Same interface of Numpy, Pandas, and scikit-learn,
* decompose computation into tasks, 
* execute tasks according to dependencies DAG, and avoiding useless data transfers,
* implements the concurrent.future interface.

Notice that Dask uses `cloudpickle` for serialization.

NOTE: Dask is a large project. **It extends Numpy, Pandas and scikit-learn** to work on multi-core or distributed systems and to perform out-of-core computations. Internally, Dask **decomposes computations into tasks**, and keeps track of tasks dependencies to execute them in the right order. The execution of different tasks happens using a **mix of the multi-threading and multi-processing** paradigms discussed earlier. Each process is called a worker, which internally has a thread pool to execute tasks in parallel. The user, when **setting up a Dask cluster** (with one scheduler and many workers), one can decide how many processes and threads to use. One should note, however, that processes communicate among themselves using sockets, even if on the same machine. Hence, the proliferation of connections might be a problem on large Dask clusters. Dask is able to **avoid useless data transfers**, by savvy choice of the grouping of tasks. Dask, among the other thing, **implements the `concurrent.future` interface**, allowing to perform custom parallel computations (which can be very handy in case of embarrassingly parallel problems). Notice that Dask uses **`cloudpickle` for serialization**, making it possible to serialize Python constructs not supported by `pickle` and allowing, for example, to submit for execution **lambda functions**.

---

## GPGPU Computing <!-- .element: class="r-fit-text" -->

:::

![CuPy logo](https://raw.githubusercontent.com/cupy/cupy/master/docs/image/cupy_logo_1000px.png) <!-- .element: width="80%"-->

Numpy on the GPU

NOTE: There is however to take into account the **heterogeneous memory** of systems using accelerators and the **latency due to transferring the data back and forth between the CPU and the GPU**. Another thing to take into account is the concurrent capability of Nvidia graphics card, that can execute multiple kernels on different queues, called **CUDA streams**. In CuPy the CUDA streams are available as context managers. Since **CuPy functions are generally non-blocking**, one can easily take full advantage of GPGPUs capability of CUDA by combining CuPy with multi-threading.

:::

Things to keep in mind:

* Heterogeneous memory
* CUDA streams 
* CuPy functions are generally non-blocking

NOTE: The array **programming style allows to exploit accelerators** in a simple manner. The **CuPy library implements the Numpy routines on the GPU** using the CUDA Toolkit libraries. When using CuPy, one can work as he would do with Numpy. 

---

## GEOTop Calibration <!-- .element: class="r-fit-text" -->

:::

**GEOTop**: hydrological model, stand-alone application

**GEOtoPy**: dead simple, paper-thin Python wrapper

NOTE: GEOTop is a **hydrological model** focused on small catchments and mountain areas written in C and C++. It can work both as a distributed or lumped model. In my MHPC project, I used evolutionary algorithms to **calibrate GEOTop on a large distributed system** (up to 32 nodes equipped with Xeon E5-2683 v4, 2 sockets, 16 cores, 2 threads, 64GB RAM). To do that, I created a **Python wrapper that interacted with the model via writing and reading of files**. This was suboptimal, but since the simulations I worked with were CPU bounded and took each approximately 1 minute (that is, a time much larger then the writing and reading on disks), **the overhead was negligible**. **The wrapper was of course non-blocking**.

:::

### (Hyper)Parameter Optimization

Facebook Nevergrad, Jupyter, Papermill, Scrapbook

Interactive & batch calibrations

Run on Ulysses v2 (new SISSA cluster) using Dask, up to 32 Xeon E5-2683 v4 (1024 cpu cores)

NOTE: I used the evolutionary algorithms provided by the **Facebook Nevergrad** library and run the model on a cluster thanks to the **Dask** library. The calibration had two modes of operation. In the interactive mode, I run a single calibration using a Dask cluster were the **workers were spawn dynamically thanks to dask-jobqueue**. I also used **Papermill and Scrapbook** to execute batches of calibrations for profiling purposes and to investigate the optimization algorithms. Papermill and Scrapbook are two libraries by Nteract which respectively allows to **parametrize, execute and serialize objects within notebooks**.

---

## Template Matching <!-- .element: class="r-fit-text" -->

:::

### High-throughput data analysis using CuPy

Finding earthquakes in seismograms similar to template ones: 10 years of continuous data, 50,000 templates.

Run on Marconi 100 (CINECA): 2x16 cores IBM POWER9 AC922 at 2.6(3.1) GHz and 4 x NVIDIA Volta V100 GPUs/node, Nvlink 2.0, 16GB

NOTE: At the seismological research center at OGS, I re-wrote their Python package for **earthquake template matching**. Template matching is a technique originally developed in digital image processing to find the part of an image that resemble a given template. The idea is to compute a similarity index for every possible alignment of the template with respect to the starting image. In seismology, the **template is a known earthquake** and one looks for matches within continuous data, that is seismograms. My version of the package used **CuPy to run a large scale analysis (High-throughput data analysis) on the new CINECA Marconi 100**, nodes having 2x16 cores IBM POWER9 AC922 at 2.6(3.1) GHz and 4 x NVIDIA Volta V100 GPUs/node, Nvlink 2.0, 16GB.  

---

## Julia <!-- .element: class="r-fit-text" -->

:::

Julia: dynamic language with multiple dispatch and JIT compilation (based on LLVM)

Type-stable code near to native performance

NOTE: Julia is a **dynamic programming language** that use **JIT compilation** to reach near to **native performance**. It was designed with **scientific computing** in mind and has a convenient syntax for that. When writing **type stable code** (that is, functions where the return type is a function only of the argument types, not their values), the Julia compiler, base on the **LLVM framework**, is able to produced highly optimized code. Finally, the **type system** and **multiple dispatch** in Julia allows to write **extensible code**. 

:::

Analysis of rock fracture laboratory experiments data using template matching

Pluto: reactive, stateless notebooks

NOTE: Right now, I am working on a template matching package to **analyse data from rock fracture laboratory experiments** using Julia and **Pluto, a reactive, stateless notebook alternative to Jupyter**.

---

## Q&A <!-- .element: class="r-fit-text" -->

:::

> He who know, does not speak. He who speaks, does not know. (Lao Tzu)