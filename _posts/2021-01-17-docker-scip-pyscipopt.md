---
layout: post
title: Docker with the SCIP Optimization Suite and PySCIPOpt. 
  Solving optimization problem (0-1 Knapsack) with Python.
slug: docker-scip-pyscipopt
description: How to solve optimization problem with Python. 
  How to build docker with SCIP Optimization Suite and PySCIPOpt.
keywords: docker docker-compose scip pyscipopt scip-optimization-suite knapsack-problem python optimization 
---

In this tutorial, we demonstrate how to solve a small-scale optimization problem (0-1 Knapsack)
with Python, and explain how to build a docker container with SCIP and PySCIPOpt, to solve 
the optimization problem inside the container.

<br/>
<div class="blog-card">
<h7 class="m-1">source code repository: <a href="https://github.com/viktorsapozhok/docker-scip">viktorsapozhok/docker-scip</a></h7><br/><br/>
<a class="github-button" href="https://github.com/viktorsapozhok/docker-scip" data-icon="octicon-star" data-size="large" data-show-count="true" aria-label="Star viktorsapozhok/docker-scip on GitHub">Star</a>
<a class="github-button" href="https://github.com/viktorsapozhok/docker-scip/fork" data-icon="octicon-repo-forked" data-size="large" data-show-count="true" aria-label="Fork viktorsapozhok/docker-scip on GitHub">Fork</a>
<a class="github-button" href="https://github.com/viktorsapozhok" data-size="large" data-show-count="true" aria-label="Follow @viktorsapozhok on GitHub">Follow @viktorsapozhok</a>
</div>

## Why to choose SCIP

SCIP is currently one of the fastest non-commercial solvers for mixed integer programming (MIP) and
mixed integer nonlinear programming (MINLP). It's regularly updated releasing a new version several times a year. 
In addition, it provides an easy-to-use Python API (PySCIPOpt) to the SCIP optimization software.
PySCIPOpt is implemented in Cython, what gets a good speedup when building an optimization problem in comparison
with raw Python code.

## How to build a docker with SCIP

To build a docker container with SCIP Optimization Suite and PySCIPOpt installed, you need to clone this 
[repository][1]. It contains only the root directory with the following
files:

    .
    ├── Dockerfile                    
    │── docker-compose.yml
    ├── knapsack.py             # demo example implemented small-scale knapsack problem 
    └── ...

To clone the repo, use following:

```shell
$ git clone https://github.com/viktorsapozhok/docker-scip.git
```

Before building a container, you need to download the latest version of the SCIP Optimization Suite, currently 
it's 7.0.2. SCIP is distributed under the Academic License, and you can download it from the [official website](https://www.scipopt.org/index.php#download).

Note, that you need to download deb installer. Copy it to the root directory (where the Dockerfile is located).

Then to build a docker image, you can issue `docker-compose build` from the root directory:

```shell
$ docker-compose build
```

When building process is over, you can check that a new image is now in the local image store.

```shell
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
scip                v0.1                78791bbde634        14 hours ago        519MB
```

Now let's take a look at the [Dockerfile](https://github.com/viktorsapozhok/docker-scip/blob/master/Dockerfile) to understand 
in details what happens when you run the building process.

The first instruction specifies the base image on which we add new layers. Here we use the slim variant of
the official Docker Python image.

```dockerfile
FROM python:3.9-slim
```

As a next step, we install the basic GCC/g++ compilers and libraries included into `build-essential`
package. It's needed to compile debian packages. Along with it, we install all the SCIP Optimization Suite dependencies.

```dockerfile
RUN apt-get update \
    && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
        build-essential \
        libgfortran4 \
        libcliquer1 \
        libopenblas-dev \
        libgsl23 \
        libtbb2 \
        wget \
    && wget -O libboost.deb "http://archive.ubuntu.com/ubuntu/pool/main/b/boost1.65.1/libboost-program-options1.65.1_1.65.1+dfsg-0ubuntu5_amd64.deb" \
    && dpkg -i libboost.deb \
    && rm libboost.deb
```

Now everything is ready to install the SCIP Optimization Suite. We copy the debian package 
to docker, install it and remove the installer.

```dockerfile
ADD SCIPOptSuite-7.0.2-Linux-ubuntu.deb /

RUN dpkg -i SCIPOptSuite-7.0.2-Linux-ubuntu.deb \
    && rm SCIPOptSuite-7.0.2-Linux-ubuntu.deb
```

As a last step, we create a new user, copy the demo script `knapsack.py` to
user's home directory, and install PySCIPOpt. 

```dockerfile
RUN groupadd --gid 1000 user \
    && useradd --uid 1000 --gid 1000 --create-home --shell /bin/bash user \
    && chown -R "1000:1000" /home/user

RUN mkdir /home/user/scripts
ADD knapsack.py /home/user/scripts
USER user
RUN pip install pyscipopt
```

To verify, that SCIP was installed correctly, we can issue `scip --version` in the running container.

```shell
$ docker-compose up -d
$ docker exec -it scip scip --version

SCIP version 7.0.2 [precision: 8 byte] [memory: block] [mode: optimized] [LP solver: SoPlex 5.0.2] [GitHash: e9d280a398]
Copyright (C) 2002-2020 Konrad-Zuse-Zentrum fuer Informationstechnik Berlin (ZIB)

External codes: 
  SoPlex 5.0.2         Linear Programming Solver developed at Zuse Institute Berlin (soplex.zib.de) [GitHash: e24c304e]
  CppAD 20180000.0     Algorithmic Differentiation of C++ algorithms developed by B. Bell (www.coin-or.org/CppAD)
  ZLIB 1.2.11          General purpose compression library by J. Gailly and M. Adler (zlib.net)
  GMP 6.1.2            GNU Multiple Precision Arithmetic Library developed by T. Granlund (gmplib.org)
  ZIMPL 3.4.0          Zuse Institute Mathematical Programming Language developed by T. Koch (zimpl.zib.de)
  PaPILO 1.0.2         parallel presolve for integer and linear optimization (https://github.com/lgottwald/PaPILO) [GitHash: 62d2842]
  bliss 0.73p          Computing Graph Automorphism Groups by T. Junttila and P. Kaski (http://www.tcs.hut.fi/Software/bliss/)
  Ipopt 3.13.2         Interior Point Optimizer developed by A. Waechter et.al. (www.coin-or.org/Ipopt)

Compiler: gcc 7.5.0
```

## Packing knapsack

To demonstrate how to use PySCIPOpt, we show how to solve a small-scale 
[knapsack problem](https://en.wikipedia.org/wiki/Knapsack_problem) for the case of multiple knapsacks.

Let's assume, that we have a collection of items with different weights and values, and we want to
pack a subset of items into five knapsacks (bins), where each knapsack has a maximum capacity 100, so the 
total packed value is a maximum.

Define a simple container class to store item parameters and initialize 15 items.

```python
class Item:
    def __init__(self, index, weight, value):
        self.index = index
        self.weight = weight
        self.value = value

items = [
    Item(1, 48, 10), Item(2, 30, 30), Item(3, 42, 25), Item(4, 36, 50), Item(5, 36, 35), 
    Item(6, 48, 30), Item(7, 42, 15), Item(8, 42, 40), Item(9, 36, 30), Item(10, 24, 35), 
    Item(11, 30, 45), Item(12, 30, 10), Item(13, 42, 20), Item(14, 36, 30), Item(15, 36, 25)
]
```

Introduce bins (knapsacks) in the similar fashion.

```python
class Bin:
    def __init__(self, index, capacity):
        self.index = index
        self.capacity = capacity

bins = [Bin(1, 100), Bin(2, 100), Bin(3, 100), Bin(4, 100), Bin(5, 100)]
```

As a next step, we create a solver instance.

```python
from pyscipopt import Model, quicksum

model = Model()
```

We introduce the binary variables `x[i, j]` indicating that item `i` is packed into bin `j`.

```python
x = dict()
for _item in items:
    for _bin in bins:
        x[_item.index, _bin.index] = model.addVar(vtype="B")
```

Now we add the constraints which prevent the situations when the same item is packed into multiple bins.
It says that each item can be placed in at most one bin.

```python
for _item in items:
    model.addCons(quicksum(x[_item.index, _bin.index] for _bin in bins) <= 1)
```

The following constraints require that the total weight packed in each knapsack don't exceed its maximum capacity.

```python
for _bin in bins:
    model.addCons(
        quicksum(
            _item.weight * x[_item.index, _bin.index] for _item in items
        ) <= _bin.capacity)
```

Finally, we define an objective function as a total value of the packed items and run the optimization.

```python
model.setObjective(
    quicksum(
        _item.value * x[_item.index, _bin.index]
        for _item in items for _bin in bins
    ), 
    sense="maximize")

model.optimize()
```

See [knapsack.py](https://github.com/viktorsapozhok/docker-scip/blob/master/knapsack.py) for more details.

## Running SCIP solver inside docker

We copy script `knapsack.py` to `/home/user/scripts` directory inside the container when building an image:

```dockerfile
RUN mkdir /home/user/scripts
ADD knapsack.py /home/user/scripts
```

To launch the script, we start the container in the detached mode:

```shell
$ docker-compose up -d
```

To run the script inside the container, we use `docker exec` command.
The optimization displays the following output:

```shell
$ docker exec -it scip python scripts/knapsack.py
...
Bin 1
Item 6: weight 48, value 30
Item 13: weight 42, value 20
Packed bin weight: 90
Packed bin value : 50

Bin 2
Item 3: weight 42, value 25
Item 8: weight 42, value 40
Packed bin weight: 84
Packed bin value : 65

Bin 3
Item 4: weight 36, value 50
Item 5: weight 36, value 35
Item 10: weight 24, value 35
Packed bin weight: 96
Packed bin value : 120

Bin 4
Item 2: weight 30, value 30
Item 11: weight 30, value 45
Item 14: weight 36, value 30
Packed bin weight: 96
Packed bin value : 105

Bin 5
Item 9: weight 36, value 30
Item 15: weight 36, value 25
Packed bin weight: 72
Packed bin value : 55

Total packed value: 395.0
```

To stop the running container, use `down` command:

```shell
docker-compose down
```
