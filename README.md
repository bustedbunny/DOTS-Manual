# DOTS-Manual

## Table of contents

- [System dependencies](#system-dependencies)
- [Sync points](#sync-points)
- [Addendum for System dependencies](#addendum-for-system-dependencies)

## System dependencies

Job dependencies are fully handled by Unity.Entities code when it comes to cross system relationships.
For example: scheduling job that writes to `LocalTransform` in System A will always ensure that scheduling job
that reads/writes `LocalTransform` in System B, will always wait for each other. At the same time, if both
jobs are only reading some component - then they can run in parallel.

### How it works

During system lifetime, system keeps lists of registered Read (RO) and Write (RW)component types for this system.
Generally those are gathered in `OnCreate`, but sometimes (which is usually a mistake) it may happen even in
`OnUpdate`.
The example of logic, that causes gathering such components:

* `EntityQuery` creation
* `SystemState.GetComponentTypeHandle` or `SystemAPI.GetComponentTypeHandle`
* `SystemState.GetComponentLookup` or `SystemAPI.GetComponentLookup` (as well as buffer lookups)

*Some information may be not 100% correct, but the general concept is.*

Each `Unity.Entities.World` contains a dependency registry for each component type. By default each
component dependency is `default(JobHandle)`.
Before any system updates, it has `BeforeOnUpdate` call, which gets `JobHandle` for each component type out of
this registry and combines them all into one `JobHandle`, which then gets assigned to `SystemState.Dependency`.

Then during `OnUpdate` user code will schedule jobs, using `SystemState.Dependency` as input dependency, which
makes all jobs scheduled by system depend on all those dependencies. At the same time all jobs scheduled by
system (at least in all normal cases) return `JobHandle` which then gets assigned to same `SystemState.Dependency`.

After system `OnUpdate` there is another `AfterOnUpdate` call, which assigns final `SystemState.Dependency` back
to component dependency registry. So after system is done, all `JobHandle` for specific component types, contain
the dependency on all jobs, that system scheduled.

So let's make an example of what happens to properly understand what is going on:

Let's say we have `System X` which reads component `A` and `B`. And then we also have `System Y`
which writes to `B` and updates after `System X`.

Before `System X` runs, component dependency registry looks like:

* A - `default(JobHandle)`
* B - `default(JobHandle)`

So right before `System X`'s `OnUpdate` is called, it's `SystemState.Dependency` will result in same
`default(JobHandle)`.

After `System X` runs, it writes back it's `SystemState.Dependency` so component dependency registry looks like:

* A - `System X dependency`
* B - `System X dependency`

Right before `System Y` runs, it obtains only component `B` dependency and assigns it to it's `SystemState.Dependency`.
So now, any job scheduled by `System Y` will implicitly depend on jobs scheduled by `System X` and avoid
race conditions.

If we add another `System Z`, that updates after `System Y`, and depends only on example component `C`.
Then in this current world, all jobs scheduled by `System Z` will not depend on neither `System X` or `System Y`,
meaning they will run in parallel to each other.

*This is a very simple explanation on how it works. In actual implementation it has separation on
`ReadOnly` and `ReadWrite`which makes things more complicated,
but allows more jobs to in parallel to each other.*

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

### Types of sync points

There are multiple types of sync points and each behaves differently.

* ReadOnly sync point - completes only jobs that have `ReadWrite` type dependency
* ReadWrite sync point - completes only jobs that have type dependency
* Structural Change sync point - completes all jobs scheduled from `Unity.Entities.World`

Let's go over some examples:
let's say we have `JobA` that reads from `LocalTransform`, `JobB` which writes to `LocalTransform` and `JobC` which
reads from `PhysicsVelocity` scheduled by some systems.

If later in the frame system will call `EntityManager.GetComponentData<LocalTransform>` - this is going to cause
`ReadOnly` sync point and it will force complete `JobA`.

If system will call `EntityManager.SetComponentData<LocalTransform>` it will complete both `JobA` and `JobB` since
both are accessing `LocalTransform`.

If system will call `EntityManager.AddComponent/RemoveComponent/Instantiate/CreateEntity` - this will cause
structural change sync point, and it will complete all jobs: `JobA`, `JobB` and `JobC` regardless of what components
they access.

### Why sync points should be avoided?

Assuming you have multiple sync points in your project, here's what's going to happen:

1. Main thread schedules job A.
2. Next system, makes a sync point and main thread waits for job A to finish.
3. Main thread schedules job B.
4. Next system, makes another sync point and main thread wait for job B to finish.

You may guess that so far, main thread had spent all it's time waiting for jobs to finish, so it's as good as
single-threaded application.

What would be much better:

1. Main thread schedules job A.
2. Main thread schedules job B, C, D...
3. After all jobs in project are scheduled - some system makes a sync point.

What makes it different from first example is that jobs are already being executed, while main thread schedules next
jobs.
Then, by the time sync point happens - all jobs are already scheduled and worker threads can be efficiently utilized.

### How sync points happen?

There are many places where sync points are placed for safety reasons. Here's several examples:

1. `EntityManager.GetComponentData/SystemAPI.GetComponent` - this method reads data that is potentially
   being written to in a job, that's why it needs to sync all jobs that write to that component
   (but not jobs that only read it).
2. `EntityManager.SetComponentData/SystemAPI.SetComponent` - this method writes data that is potentially
   read/written by a job, so all jobs that either read or write to that component must be synced.
3. `EntityManager.AddComponent/RemoveComponent/Instantiate/CreateEntity` - and any other method
   that causes structural changes can modify chunks that are used by jobs, that's why they will cause sync point on
   every job scheduled in this world.
4. `EntityCommandBuffer.Playback` - any ecb playback will cause a sync point, since all operations that it does are
   executed on main thread. So that means, simply using `EndSimulationEntityCommandBufferSystem` will cause you a sync
   point during that system's update.

SystemAPI calls that cause sync points:

* `SystemAPI.GetSingletonBuffer` - causes a sync RO/RW type sync point, so any job that iterates this buffer or
  has a lookup for it will be synced.
* `SystemAPI.Query` - causes a sync point depending on whether you get `RefRO/RW` for each used component type

SystemAPI calls that do not cause sync points:

* `SystemAPI.GetSingleton` and `SystemAPI.GetSingletonRW` - they won't cause a sync point, but they still will
  throw if there is a job, scheduled that uses those types (either via iteration or via lookup).

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

### Addendum for System dependencies

It was mentioned that systems complete dependencies in `BeforeOnUpdate` and gather dependencies in `AfterOnUpdate`.
While this is true, it leads to some wrong expectations.

It's easiest to explain with an example.

Imagine 2 systems A and B. Both work on a singleton NativeList and register a write dependency. (For some reason
we don't go into detail, system A can't schedule a job but has to do it on the main thread.)

System A does main thread tasks, gets the singleton, iterates through the list, processes element and then clears the list.
System B updates after system A and schedules a job, and the job adds to the NativeList.

System A
```csharp
protected override void OnCreate()
{
    // stock method to create RW dependency in SystemState, there are other and better methods
    SystemAPI.GetSingletonRW<RequestSingleton>();
}

protected override void OnUpdate()
{
    var requests = SystemAPI.GetSingleton<RequestSingleton>().Requests;
    
    if (requests.Length == 0)
        return;	        
    ...    
    request.Clear();
}
```

System B
```csharp
public void OnUpdate(ref SystemState state)
{
    var requests = SystemAPI.GetSingleton<RequestSingleton>().Requests;
            
    new AddToListJob()
    {
        Requests = requests
    }.Schedule();
}
```

This code will throw a safety problem in `requests.Length == 0` that `AddToListJob` is writing to the singleton. But why?

System A has a RW dependency on `RequestSingleton` and we learned `BeforeOnUpdate` calls `.Complete`, so any write dependency
should be completed by the `.Complete` handle. Also, the job has long been finished when System A reads the singleton.
And why does the `.Complete` in System A `BeforeOnUpdate` not pick up the job when it has a clear dependency?

> When System A reads the singleton, the job handle for System B has not yet been completed
because System B `BeforeOnUpdate` hasn't been called yet. This makes the safety system think that a job is still running.

But calling `Dependency.Complete();` in System A fixes the problem!

> The complete dependency call in `BeforeOnUpdate` is NOT the same as calling `Dependency.Complete()` in `OnUpdate`!

We need to look at code from `SystemState.cs` to get a better understanding.

```csharp
public JobHandle Dependency
{
    get
    {
        if (NeedToGetDependencyFromSafetyManager)
        {
            var depMgr = m_DependencyManager;
            NeedToGetDependencyFromSafetyManager = false;
            m_JobHandle = depMgr->GetDependency(m_JobDependencyForReadingSystems.Ptr,
                m_JobDependencyForReadingSystems.Length, m_JobDependencyForWritingSystems.Ptr,
                m_JobDependencyForWritingSystems.Length, clearReadFencesAfterCombining:false);
        }

        return m_JobHandle;
    }
    set
    {
        NeedToGetDependencyFromSafetyManager = false;
        m_JobHandle = value;
    }
}
```
and
```csharp
internal void BeforeOnUpdate()
{
    BeforeUpdateVersioning();

    // We need to wait on all previous frame dependencies, otherwise it is possible that we create infinitely long dependency chains
    // without anyone ever waiting on it
    m_JobHandle.Complete();
    NeedToGetDependencyFromSafetyManager = true;
}
```

Notice `NeedToGetDependencyFromSafetyManager`. During `BeforeOnUpdate` m_JobHandle, which is the same as
`state.Dependency` in ISystem or `Dependency` in SystemBase, is called.

During that, `NeedToGetDependencyFromSafetyManager` is `false`, so it doesn't get an updated dependency as it's not
going through the while `if(NeedToGetDependencyFromSafetyManager)`
block. This means, all handles are completed BUT not handles that were registered after it.

This makes a big difference because calling `Dependency.Complete()` in `OnUpdate` does indeed get an updated
dependency (as `NeedToGetDependencyFromSafetyManager` is set to `true` at the end of `BeforeOnUpdate`) which would have
handles that came after it too.

So to fix this problem there are a few options:
- merge both systems into one, which would solve the problem that system A doesn't know about system B's job
- call `Dependency.Complete();` in SystemA
- or `EntityManager.CompleteDependencyBeforeRW<RequestSingleton>();` in SystemA 