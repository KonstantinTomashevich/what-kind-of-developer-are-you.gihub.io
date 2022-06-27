---
layout: post
title: 'Emergence: Interpolating transform'
date: 2022-06-27 20:30:00 GMT+3
categories: [Emergence, Development Log]
tags: [Emergence, GameDev, Math]
---

What if I say that every game object in Emergence has two transforms? What for? For the sake of making everything
look cool, of course! In this post I will describe transform interpolation, which is quite 
useful feature unless you're entirely sure that variable framerate is the best option for you.

### Framerate and world updates

There is a perfect [article about time steps by Glenn Fiedler](https://gafferongames.com/post/fix_your_timestep/).
But just sending you to the other blog wouldn't be polite, so I'll write my view on this topic too.

By far the simpliest approach to update game world is to execute it before every render and passing frame delta time
as time step. Despite being primitive, it is widely adopted and even used in Unreal Engine! But there is a one
not-so-simple problem with this approach: it makes your simulation non-deterministic, because time step is equal to
frame time delta and therefore is hardware-dependent. And this problem triggers a bunch of smaller problems:
physics simulation might be buggy and strange, multiplayer networking is difficult because of non-determinism,
subtle and hard to figure out gameplay bugs (or features! :) ) may arise.

The other option is fixed+normal updates approach: physics simulation, networking and "strong" gameplay logic are 
updated with fixed time steps using time accumulation, while render and visual logic are updated using variable 
framerate technique from previous paragraph. This allows to both have deterministic simulation of features that work
in fixed update, and responsive visual that works in normal update. It might sound like a miraculous solution,
but there is a reason why it is not as widely adopted as simple variable framerate: it is more difficult to
work with two separate update pipelines than with one. For example, it creates problem of transform interpolation:
transforms are updated in fixed pipeline, but we would like to see smooth movement instead of jittery transform
changes. And it is exactly the theme of this post!

For those who have read [Glenn's article](https://gafferongames.com/post/fix_your_timestep/), I would like to highlight 
that [Celerity](https://github.com/KonstantinTomashevich/Emergence/tree/72ff166/Library/Public/Celerity)
treats time accumulation in a bit different way. In Glenn's version normal update "produces" time and fixed update
"consumes" time, but in my library I've decided to take inversed approach: fixed update "produces" time and normal
update "consumes" it. The reason for this is that real time is always way ahead of render time if article version is 
used:

```
----F----F--R-F--T-|----|----
F -- executed fixed updates.
R -- render time.
T -- real time.
| -- planned fixed update.
```

After inversion fixed updates are ahead of real time, but render time is equal to real time:

```
----F----F----F--T-F----|----
```

Nevertheless, from perspective of input both variants have 1 frame latency, therefore difference is not so big and you're 
free to use any variant.

### No interpolation vs Interpolation 

Let's imagine that we directly use transforms from fixed update to render our objects. At 60 FPS everything probably 
looks superb. But then designers introduce slowdown feature, that allows player to run game at x0.25 speed for
several seconds. Suddenly, during slowdown everything starts looking jittery:

![NoInterpolation.gif](/assets/img/EmergenceInterpolatingTransform/NoInterpolation.gif)

But we don't want player to think that game lags during slowdown! We want something smoother, like this:

![Interpolation.gif](/assets/img/EmergenceInterpolatingTransform/Interpolation.gif)

To achieve smoothier picture we need to adapt to the situation and use different transforms for render.
And that is exactly how visual transform interpolation works: we adjust render transforms and everything 
looks like miracle!

### Interpolation use cases

Some rare slowdown feature is not the only case where you need interpolation. There are many more other cases, 
for example:

- Sometimes users might like to watch the game at much higher framerate that we could possibly achieve by doing
  1 normal update per 1 fixed update. I don't really see the difference between 60 and 120 FPS, but some people do.
  For these people we need to render several frames per fixed update and interpolate transforms.

- Sometimes fixed update might drastically slow down game performance, for example when doing difficult dynamics
  calculations. It might even result in death spiral: it is when we are doing more and more fixed updates
  each frame because frame time is bigger than fixed step time. To solve this issue we need to temporarily
  increase fixed time step, for example from 60 FPS to 15 FPS (yeah, technically we're increasing it, 
  because time step value increases), and to keep visual effects clean while doing so we need to interpolate.

- Multiplayer games usually have low replication frequency to optimize traffic usage. For example, Apex Legends
  sends replication data at 20 samples per second. Of course, it would have looked laggy without interpolation!

### How to implement interpolation?

Now it's finally time to tell how I implemented interpolation in 
[Celerity](https://github.com/KonstantinTomashevich/Emergence/tree/72ff166/Library/Public/Celerity)!

Let's start from the concept of interpolated movement: we need to enable interpolation only when object actually moves,
otherwise we will not only spend CPU time on useless work, but could also see incorrect results: for example, we can 
interpolate teleportation or interpolate spawn by moving object from world zero to spawn transform. To achieve that
we need to use special flag inside `Transform3dComponent` that I called `visualTransformSyncNeeded`. This flag
is true only when object is in interpolated movement mode and always false otherwise. For spawns and teleports there is
special a `_skipInterpolation` parameter in transform setter, that allows to move objects without enabling
`visualTransformSyncNeeded`. That's how logical transform setter code looks like:

```c++
void Transform3dComponent::SetLogicalLocalTransform (const Math::Transform3d &_transform,
                                                     bool _skipInterpolation) noexcept
{
    // ... Irrelevant setter logic ...

    if (_skipInterpolation)
    {
        // We do not enable interpolation, therefore we need to set visual transform directly.
        SetVisualLocalTransform (logicalLocalTransform);

        // If object was in interpoolated movement mode -- interrupt it.
        visualTransformSyncNeeded = false;
    }
    else
    {
        // If we're not in interpolated movement mode, we need to reset sync timer 
        // to correctly process pauses between two different interpolated movements. 
        // Otherwise, interpolation result would be incorrect.
        // More about that timer below.
        if (!visualTransformSyncNeeded)
        {
            visualTransformLastSyncTimeNs = 0u;
            visualTransformSyncNeeded = true;
        }
    }
}
```

To apply interpolation we need to know 2 time stamps:

1. Last fixed timestamp when logical transform was changed.

2. Last normal time stamp during which visual transform had desired value.

Getting first one is pretty easy: we need to save it as variable inside transform and update it during interpolated
movement whenever we detect that transform was changed, like this:

```c++
// Celerity uses revision-driven transform change detection instead of dirty flags, 
// because it allows to avoid going through full hierarchy when transform is setted.
if (transform->lastObservedLogicalTransformRevision != transform->logicalLocalTransformRevision)
{
    transform->lastObservedLogicalTransformRevision = transform->logicalLocalTransformRevision;

    // We've detected transform change and now can save required timestamp.
    transform->logicalTransformLastObservationTimeNs = time->fixedTimeNs;
}
```

Getting second one is more difficult at first glance, but actually it is easy too. After each interpolation operation
visual transform has desired value, therefore during interpolated movement second timestamp is always equal to the
normal time of previous frame. But what about first frame of interpolated movement? Because movement has just started,
object is already in desired transform, therefore time stamp is equal to frame normal time. Code looks like
this:

```c++
// Interpolated movement detected. Remember setter code above? 
// Yep, that's why we set it to zero there.
if (transform->visualTransformLastSyncTimeNs == 0u)
{
    transform->visualTransformLastSyncTimeNs = time->normalTimeNs;
}

// ... Actual interpolation execution ...

transform->visualTransformLastSyncTimeNs = time->normalTimeNs;
```

This two time stamps are our interpolation borders and current normal time is point at which we need to calculate 
transform. Now we have enough information to do transform linear interpolation:

```c++
// Time elapsed since last disered visual transform.
const uint64_t elapsed = time->normalTimeNs - 
                         transform->visualTransformLastSyncTimeNs;

// Known interpolation interval duration.
const uint64_t duration = transform->logicalTransformLastObservationTimeNs - 
                          transform->visualTransformLastSyncTimeNs;

const float progress = Math::Clamp (
    static_cast<float> (elapsed) / static_cast<float> (duration), 0.0f, 1.0f);

// Note, that we are only working with local transforms here: 
// we don't need to take global ones into account because
// we're processing every transform that needs interpolation,
// so if parent transform was changed, it would be processed separately.
const Math::Transform3d &source = transform->GetVisualLocalTransform ();
const Math::Transform3d &target = transform->GetLogicalLocalTransform ();

transform->SetVisualLocalTransform ({Math::Lerp (source.translation, target.translation, progress),
                                     Math::SLerp (source.rotation, target.rotation, progress),
                                     Math::Lerp (source.scale, target.scale, progress)});
```

Simple and straighforward, isn't it? There is only one last detail left: we need to exit from 
interpolated movement mode after arriving at target destination. It is easier than it sounds:

```c++
transform->visualTransformSyncNeeded =
    transform->visualTransformLastSyncTimeNs < transform->logicalTransformLastObservationTimeNs;
```

And that's all! Transform interpolation might sound scary, but it is quite simple and minimalistic once you get it
done right. You could check resulting code files here:

- [Transform3dComponent header file.](https://github.com/KonstantinTomashevich/Emergence/blob/72ff166/Library/Public/Celerity/Extension/Transform/Celerity/Transform/Transform3dComponent.hpp)
- [Transform3dComponent object file.](https://github.com/KonstantinTomashevich/Emergence/blob/72ff166/Library/Public/Celerity/Extension/Transform/Celerity/Transform/Transform3dComponent.cpp)
- [Transform3dVisualSynchronizer task implementation.](https://github.com/KonstantinTomashevich/Emergence/blob/72ff166/Library/Public/Celerity/Extension/Transform/Celerity/Transform/Transform3dVisualSync.cpp)

Hope you've enjoyed reading! :)
