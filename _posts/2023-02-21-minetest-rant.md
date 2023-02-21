---
layout: post
title:  "Why bows are bound to be janky in Minetest"
date:   2023-02-21 17:24:00 +0100
categories: minetest, gamedev, modding, jank, lua
---

So, you want(ed) to write a bow mod. It's time to confront you with the harsh reality of just how limited the Minetest engine is.

Disclaimer: Some concepts for the "ideal bow mod" detailed below are not feasible to be implemented and are mostly of theoretical nature.

## Visuals

You probably don't want your arrows to be a bunch of sprites, so your only option
is to use a heavyweight entity with a mesh visual.
Spamming arrows may drag FPS down (due to lack of batching);
each arrow will generate significant network traffic and increase the time
required to find any entities since all entities reside in the same global set to be
searched linearly for each and every query (see also poor collision performance and attempts to use spatial indexing).

You might want to add a particlespawner. Again the problem of no batching strikes.
Too many particles and your clients will start to croak.

Next point: Arrow rotation. Yaw is constant and can be set as the arrow is spawned (ideally inside `on_activate`); pitch however depends on the current velocity of the arrow.

Despite using an animation for this, older arrow mods
(read: [simple shooter](https://github.com/stujones11/shooter/blob/master/shooter_crossbow/init.lua))
would have arrows recalculate their pitch every rotation.

Newer arrow mods don't use an animation, but rather `set_rotation` which
provides Euler angle rotation interpolation barely good enough to handle this case well
(although you may temporarily get outdated rotations on clients).

Both approaches incur significant network traffic: All arrows have to send update packets every step.

The theoretically ideal solution within the engine constraints is to use an animation to rotate the arrow,
simply calculating the animation speed based on the initial velocity of the arrow.

### The Bow Item

Players expect a chargeable bow that can be fired. A bow item is straightforward to register; a decent loading & firing animation is however not as easy:

* Multiple items: Swapping the bow item to a "charged" bow; charging might use tool wear or other HUDs,
  perhaps even multiple charging "steps" are implemented using multiple "(partially) charged" items
  \- changing the texture always requires changing the item currently; after firing the bow is replaced with the original item
* 3D models: Modders might want a decent bow 3d model rather than just an (extruded) image.
  They can abuse a node for this, disabling placement & placement prediction.
  A firing animation is still not customizable; modders still have to swap items and accept MT's standard tool use anim inbetween.
* First-person attachments: Being a somewhat recent engine feature, this allows largely mitigating the problem:
  You can register a "bow" entity with an appropriate animated model, attach it to the player and properly animate it.
  It doesn't suffice to have it look well in first person though
  \- it must look decent to you *and other players* both in first and third person - requiring decent modeling skills.

### Synchronization

There is no proper synchronization. The entity spawning is offset by the time it takes the packet to arrive;
there is no timestamping or other elaborate synchronization mechanism. Networking must be kept almost instant.

## Sounds

Arrows require a "woosh" sound, bows the sound it makes as you release them.

The former can be implemented by attaching a sound to the arrow entity,
the latter by attaching the sound to the firing player.

This is probably the only part that isn't quite janky.

## Physics

### Acceleration

Just like every entity in the game arrows must be affected by gravity.
Unfortunately Minetest approximates gravity wrong such that it is highly dependant
on client/server step sizes. What the client sees at 60 FPS is thus unlikely to match
what the server sees at 10 steps per second; the error can be rather significant:
Arrows will fall faster the lower the steps per second.
This is particularly problematic if lag rises, even if only temporarily.

[A fix exists](https://github.com/minetest/minetest/pull/12353) but hasn't made its way into the engine yet.

A workaround might be to correct the position every step,
but this would suffer from being very janky on clients
(and again taking a huge toll on network traffic).

### Collisions

The arrow logic is quite simple: Whatever the parabola taken by the arrow tip first hits
is considered "hit" and the arrow is attached to it in the appropriate relative position.

Object collision shapes in Minetest are simply boxes, again greatly simplifying this but at the expense
of accurate visuals or simulations: The arrow might as well stick "in the air" the collisionbox provides.

A pseudo-ideal solution is to animate client meshes serverside (all *theoretically* possible in Lua),
then approximate the parabola through ray-triangle intersections with every animated triangle of the mesh,
which provide you with an exact point (and U-V-coordinates) of where the ray hit.
Attaching unfortunately only supports bones, so you have to be lucky with the model
and have all of the triangle vertices only be affected by a single bone to which you can attach the arrow.

In practice a demo of this (without animation support)
can be seen in [Epidermis](https://github.com/appgurueu/epidermis),
which allows accurately painting skins in Minetest.

The obvious massive downside of this is that it requires animating all meshes every step, which can get expensive
(but may be feasible using the new async Lua environment "to offload heavy computational tasks").
It also is quite some work to implement.

There are two straightforward ways to implement arrows currently:

* **DIY.** Have your arrows be nonphysical entities and approximate the parabola through a series of raycasts,
  usually performed every server step. Performance can trivially be seen to be quadratic in the number of arrows since raycasts
  are linear in the number of entities. Another downside is that raycasts currently operate only on selection-, not collisionboxes.
  A [feature request](https://github.com/minetest/minetest/issues/12673) for the latter exists. It is theoretically possible
  to make raycasts work on collisionboxes in Lua.
  In terms of clientside prediction the downside is that you will first see the arrow phase through the enemy,
  then have its position be reset to be stuck inside the entity.
* **Ready-made.** Have your arrows be *physical* entities *colliding with objects* and check the `moveresult` every step.
  Since object collision uses a linear search for each object, this can trivially be seen to be quadratic in the number of arrows as well.
  This differs from the first approach in that it properly deals with collisionboxes, but it fails to correctly determine the point *where* the arrow collided
  since the arrow will continue moving after the collision
  (the component of the velocity on the axis on which the collision occurred will be zeroed, but that's about it - movement will continue orthogonal to the enemy collisionbox),
  again leading to a poor clientside prediction (the arrow will continue moving parallel to the collisionbox face it collided into until it is reset to the point where it hit).
  To correctly determine the position where the arrow hit, you will in the end have to use raycasts anyways, making the "ready-made" option only differ
  in *how clientside prediction is off*: Either the arrow phases through its target or it continues moving parallel to the collisionbox face it collided with.

## Conclusion

Janky bow mods are possible in Minetest.
Server stepsize and round-trip time must be kept to a minimum to minimize jank.
The default server stepsize of ~10 steps per second is way too low.

Minetest could significantly cut down on the jank if it refrained from the current
server-only modding and allowed client modding to some extent (ideally server-sent client-side mods (SSCSM));
the current client-side mods (CSM) are too restricted to be of any use here.

A lot of the issues described here also apply to any other weapons implemented in Minetest,
making Minetest an unsuitable game engine for implementing FPS-style games.

In the end, it all *somehow* "just works" and is "good enough"
for players to not constantly complain, but it could definitely be much, much better.
This often goes hand in hand with modders limiting their mods significantly (e.g. throttling arrow speeds)
to not exacerbate the shortcomings of the Minetest engine, carefully designing their mods around the restrictions.
