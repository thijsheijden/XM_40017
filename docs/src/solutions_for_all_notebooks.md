# Solutions

## Julia Basics

### NB1-Q1

In the first, line we assign a variable to a value. In the second line, we assign another variable to the same value. Thus,we have 2 variables associated with the same value. In line 3, we associate `y` to a new value (re-assignment). Thus, we have 2 variables associated with 2 different values. Variable `x` is still associated with its original value. Thus, the value at the final line is `x=1`.

### NB1-Q2

It will be `1` for very similar reasons as in the previous questions: we are reassigning a local variable, not the global variable defined outside the function.

### NB1-Q3

It will be `6`. In the returned function `f2`, `x` is equal to `2`. Thus, when calling `f2(3)` we compute `2*3`.


### Exercise 1

```julia
function ex1(a)
    j = 1
    m = a[j]
    for (i,ai) in enumerate(a)
        if m < ai
            m = ai
            j = i
        end
    end
    (m,j)
end
```

### Exercise 2

```julia
ex2(f,g) = x -> f(x) + g(x)
```

### Exercise 3

```julia
using GLMakie
max_iters = 100
n = 1000
x = LinRange(-1.7,0.7,n)
y = LinRange(-1.2,1.2,n)
heatmap(x,y,(i,j)->mandel(i,j,max_iters))
```

## Asynchronous programming in Julia

### NB2-Q1

Evaluating `compute_π(100_000_000)` takes about 0.25 seconds. Thus, the loop would take about 2.5 seconds since we are calling the function 10 times.

### NB2-Q2

The time in doing the loop will be almost zero since the loop just schedules 10 tasks, which should be very fast.

### NB2-Q3

It will take 2.5 seconds, like in question 1. The `@sync` macro forces to wait for all tasks we have generated with the `@async` macro. Since we have created 10 tasks and each of them takes about 0.25 seconds, the total time will be about 2.5 seconds.

### NB2-Q4

It will take about 3 seconds. The channel has buffer size 4, thus the call to `put!`will not block. The call to `take!` will not block neither since there is a value stored in the channel. The taken value is 3 and therefore we will wait for 3 seconds. 

### NB2-Q5

The channel is not buffered and therefore the call to `put!` will block. The cell will run forever, since there is no other task that calls `take!` on this channel. 

## Distributed computing in Julia

### NB3-Q1

We send the matrix (16 entries) and then we receive back the result (1 extra integer). Thus, the total number of transferred integers in 17.

### NB3-Q2

Even though we only use a single entry of the matrix in the remote worker, the entire matrix is captured and sent to the worker. Thus, we will transfer 17 integers like in Question 1.

### NB3-Q3

The value of `x` will still be zero since the worker receives a copy of the matrix and it modifies this copy, not the original one.

### NB3-Q4

In this case, the code `a[2]=2` is executed in the main process. Since the matrix is already in the main process, it is not needed to create and send a copy of it. Thus, the code modifies the original matrix and the value of `x` will be 2. 

## Distributed computing with MPI

### Exercise 1

```julia
using MPI
MPI.Init()
comm = MPI.Comm_dup(MPI.COMM_WORLD)
rank = MPI.Comm_rank(comm)
nranks = MPI.Comm_size(comm)
buffer = Ref(0)
if rank == 0
    msg = 2
    buffer[] = msg
    println("msg = $(buffer[])")
    MPI.Send(buffer,comm;dest=rank+1,tag=0)
    MPI.Recv!(buffer,comm;source=nranks-1,tag=0)
    println("msg = $(buffer[])")
else
    dest = if (rank != nranks-1)
        rank+1
    else
        0
    end
    MPI.Recv!(buffer,comm;source=rank-1,tag=0)
    buffer[] += 1
    println("msg = $(buffer[])")
    MPI.Send(buffer,comm;dest,tag=0)
end
```

### Exercise 2

```julia
f = () -> Channel{Int}(1)
chnls = [ RemoteChannel(f,w) for w in workers() ]
@sync for (iw,w) in enumerate(workers())
    @spawnat w begin
        chnl_snd = chnls[iw]
        if w == 2
            chnl_rcv = chnls[end]
            msg = 2
            println("msg = $msg")
            put!(chnl_snd,msg)
            msg = take!(chnl_rcv)
            println("msg = $msg")
        else
           chnl_rcv = chnls[iw-1]
           msg = take!(chnl_rcv)
           msg += 1
           println("msg = $msg")
           put!(chnl_snd,msg)
        end
    end
end
```

This is another possible solution.

```julia
@everywhere function work(msg)
    println("msg = $msg")
    if myid() != nprocs()
        next = myid() + 1
        @fetchfrom next work(msg+1)
    else
        @fetchfrom 2 println("msg = $msg")
    end
end
msg = 2
@fetchfrom 2 work(msg)
```

