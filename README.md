# DOTS-Manual

## Table of contents

- [Manual requirements](#manual-requirements)
- [Unity Jobs safety system](#unity-jobs-safety-system)
- [System dependencies](#system-dependencies)
- [Sync points](#sync-points)
- [API Gotchas](#api-gotchas)

### Manual requirements

This manual is about advanced parts of Unity DOTS usage and assumes you have already read the
following manuals:

* [Unity Job system manual](https://docs.unity3d.com/Manual/JobSystem.html)
* [Unity.Entities manual](https://docs.unity3d.com/Packages/com.unity.entities@1.2/manual/index.html)
* [Unity.Burst manual](https://docs.unity3d.com/Packages/com.unity.burst@1.8/manual/index.html)

## Unity Jobs safety system

### Purpose of safety system

Unity safety system is meant to prevent race conditions across jobs, in a way that can be fully
validated in editor time deterministically.

<img alt="image" height="256" src="https://github.com/bustedbunny/DOTS-Manual/assets/30902981/6110ba94-7caa-4eaa-945d-2eb517a3d3c6" width="512"/>

_Race condition example: thread A and thread B try to append the same variable._

### Native container

Jobs may only be scheduled only if they are fully `unmanaged`, meaning you can't have any managed objects like List or
[] arrays inside. In order to obtain data back from job you are supposed to use native
containers: `NativeArray`, `NativeList`, `NativeQueue` etc.

Only native containers implement safety for Unity Job system and any custom types that implement
the safety in the same way.

### Safety during scheduling

When job get scheduled it gets scanned recursively for all fields that are native containers and their safety
handles are being "reserved" under this job. So when another job gets scheduled it checks whether all it's reserved
handles are being used by other jobs in unsafe way:

* One job writes, other job reads to a same container
* Both jobs write to a same container

```
Only fields that have `Unity.Collections.ReadOnly` attribute are considered as read-only.
```

```
"Reserved" state gets released only when 'JobHandle.Complete()' also known as sync point will be called.
```

Whenever any unsafe behaviour is found Unity will throw an exception that would look like example below and job
will not be scheduled:

_InvalidOperationException: The previously scheduled job `JobType` writes to the `NativeContainerType`
`NativeContainerName`. You must call `JobHandle.Complete()` on the job `JobType`, before you can write
to the `NativeContainerType`safely._

### Avoiding race conditions

In order to avoid exceptions shown in previous section you need to use job dependencies. Each scheduled job accepts
`JobHandle` as input dependency and returns `JobHandle` as output dependency.

* If you pass `default(JobHandle)` as input that would make job run as soon as possible without any dependencies.
* If you pass `JobHandle` returned by `Schedule` method that would make job run only after input job is finished.

Using this behaviour you must ensure that no two jobs can ever cause a race condition.

_This behaviour is already implemented by Unity ECS. Read more in
[system dependencies section](#system-dependencies)._

### Safety during Native Container usage

Aside from scheduling safety, native containers also implement safety on every usage, by calling
`AtomicSafetyHandle.CheckReadAndThrow(m_Safety)` or `AtomicSafetyHandle.CheckWriteAndThrow(m_Safety)`.

These methods ensure that native container is actually allowed to perform read or write operation on the current
thread where they are being used.

Here examples where they would throw in intended way:

* You scheduled a job with a native array with `ReadWrite` access and right after `Schedule` method you are calling
  `array[0]` on same (main) thread. Since job is already scheduled, there is no guarantee that other thread is not
  writing to it.
* You scheduled a job with native array with `ReadOnly` access and then you are trying to write to it inside
  same job. There is no guarantee that there isn't another job that is currently also reading from same array.

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
Before any system updates, it has `BeforeUpdate` call, which gets `JobHandle` for each component type out of
this registry and combines them all into one `JobHandle`, which then gets assigned to `SystemState.Dependency`.

Then during `OnUpdate` user code will schedule jobs, using `SystemState.Dependency` as input dependency, which
makes all jobs scheduled by system depend on all those dependencies. At the same time all jobs scheduled by
system (at least in all normal cases) return `JobHandle` which then gets assigned to same `SystemState.Dependency`.

After system `OnUpdate` there is another `AfterUpdate` call, which assigns final `SystemState.Dependency` back
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

### Entity Command Buffer System dependency handling

In order to use Entity Command Buffer Systems you are supposed to call
`SystemAPI.GetSingleton<ECBSystem.Singleton>` which registers dependency on that component type.

ECB System will then just complete RW dependency on that type, which will ensure all jobs that write to
created `EntityCommandBuffer` will be complete, by the time ECB System is trying to read from it.

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
5. `EntityQuery.IsEmpty/CalculateEntityCount` - this causes the sync point, but **only** if there
   are enablable components/change or order filtering specified in the query constraints.

Notable API that cause sync points:

* `SystemAPI.GetSingletonBuffer` - causes a sync RO/RW type sync point, so any job that iterates this buffer or
  has a lookup for it will be synced.
* `SystemAPI.Query` - causes a sync point depending on whether you get `RefRO/RW` for each used component type

API that do not cause sync points:

* `SystemAPI.GetSingleton` and `SystemAPI.GetSingletonRW` - they won't cause a sync point, but they still will
  throw if there is a job, scheduled that uses those types (either via iteration or via lookup).
* `EntityQuery.IsEmptyIgnoreFilter/CalculateEntityCountWithoutFiltering` - this will calculate
  entity count but will exclude any filtering, that requires a sync point.

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

## API gotchas

Not every API usage as straightforward as it may seem initially.
Here's the list of API details that should be accounted for how these methods are used:

### `EntityManager.Add/RemoveComponent(EntityQuery)`

The problem with this specific overload comes from the difference of how this command works
compared to `Entity` overload.
`EntityQuery` overload instead of moving entity one by one between chunks, modifies chunks themselves
so they start to match the target archetype after adding/removing component. So that makes it
extremely efficient on large amounts of entities.

But at the same time it introduces a problem of "empty" chunks, that appears when this method being
frequently used on little amounts of entities.

For example: let's say we have 1 entity that matches the `EntityQuery` passed to
this method. So we modified chunk and it matches the target archetype.

Now later we again have entity of same archetype passed to same method and the same thing
happens: we modify source entity's chunk to make it match the target archetype.

And here's the problem: we have 2 chunks with just 1 entity inside in both (for reference, max
amount of entities in chunk is 128). If same behaviour will keep going - memory usage will be very
inefficient while performance of iteration is very poor.

So this detail should be considered very carefully depending on the context of logic.
Cases when this method is appropriate:

* One time operation of post-processing subscene

Cases when this method is not appropriate:

* Post-processing entities that get instantiated

### `EntityManager.Instantiate(Entity, NativeArray<Entity>)`

The problem with this method comes from the difference with normal `Entity` overload of `Instantiate`.
The normal method would instantiate whole `LinkedEntityGroup` of entities that are stored on
entity prefab. Meanwhile `NativeArray` overload would not.

So for example if you have your character entity baked with a bunch of child entities - they won't
be instantiated with this method.

Cases when this method is appropriate:

* When you know for sure, that archetype of entity does not contain `LinkedEntityGroup`

Cases when this method is not appropriate:

* Prefabs with unknown archetype (for example coming from baking)