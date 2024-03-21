# DOTS-Manual

## Sync points

### What is the sync point?

```csharp
JobHandle.Complete();
```

*This method may only be called on main thread.*

Codewise sync point is just calling `Complete()`on job handle.

We don't know exact implementation of it, but we can assume that under the hood it looks somewhat like:

```csharp
public void Complete()
{
    while (true) 
    {
        if (JobHandle.IsComplete())
        {
            break;
        }   
    }
}
```

So basically, main thread just wait for jobs to finish. 

*Technically, main thread also starts to participate in finishing those jobs, but it's not always possible.*

### Why sync points should be avoided?

Assuming you have multiple sync points in your project, here's what's going to happen:

1. Main thread schedules job A.
2. Next system, makes a sync point and main thread waits for job A to finish. 
3. Main thread schedules job B.
4. Next system, makes another sync point and main thread wait for job B to finish.

You may guess that so far, main thread had spent all it's time waiting for jobs to finish, so it's as good as 
singlethreaded application.

What would be much better:

1. Main thread schedules job A.
2. Main thread schedules job B, C, D...
3. After all jobs in project are scheduled - some system makes a sync point.

What makes it different from first example is that jobs are already being executed, while main thread schedules next jobs.
Then, by the time sync point happens - all jobs are already scheduled and worker threads can be efficiently utilized.


### How sync points happen?

There are many places where sync points are placed for safety reasons. Here's several examples:
1. `EntityManager.GetComponentData` - this method reads data that is potentially being written to in a job, that's why
it needs to sync all jobs that write to that component (but not jobs that only read it).
2. `EntityManager.SetComponentData` - this method writes data that is potentially read/written by a job, so all jobs that
either read or write to that component must be synced.
3. `EntityManager.AddComponent`, `EntityManager.CreateEntity`, `EntityManager.DestroyEntity` - and any other method
that causes structural changes can modify chunks that are used by jobs, that's why they will cause sync point on 
every job scheduled in this world.
4. `EntityCommandBuffer.Playback` - any ecb playback will cause a sync point, since all operations that it does are
executed on main thread. So that means, simply using `EndSimulationEntityCommandBufferSystem` will cause you a sync point
during that system's update.

There are many more places where sync points happen and it's best to look at source of method to see if it has one.
But as general rule of thumb - any direct ECS data access on main thread = sync point.

### How to avoid sync points?

Schedule a job! So far it's the easiest way to avoid sync point. But since it's not always possible - you may need
to organize your system order in a way, so you can avoid them.

One of the ways I personally find most efficient:

1. Main thread systems
2. Job scheduling systems

Basically it means, after first job scheduling system ran - no sync points should happen until next frame.
In reality, by the time next frame starts - all jobs will be finished already, so syncing wouldn't waste any time
actually waiting for jobs to finish.
That also means - you may not use any `EntityCommandBufferSystem` except for `BeginSimulationEntityCommandBufferSystem`.