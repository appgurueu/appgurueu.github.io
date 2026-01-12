---
layout: post
title:  "Why some things are bound to be janky in Luanti"
date:   2023-02-21
tags:
- luanti
- Luanti
- gamedev
- modding
- jank
- lua
aliases:
- "../2023/02/21/minetest-rant.html"
---

So, you want(ed) to write a bow mod. It's time to confront you with the harsh reality of just how limited the Luanti engine is.

Disclaimer: Some concepts for the "ideal bow mod" detailed below are not feasible to be implemented and are mostly of theoretical nature.

## Visuals

You probably don't want your arrows to be a bunch of sprites, so your only option
is to use a heavyweight entity with a mesh visual.
Spamming arrows may drag FPS down (due to lack of batching - one drawcall is issued for each entity);
additionally, each arrow will generate significant network traffic and increase the time
required to find any entities since all entities reside in the same global set to be
searched linearly for each and every query (see also poor collision performance and attempts to use spatial indexing).

You might want to add a particlespawner. Again the problem of no batching strikes.
Too many particles and your clients will start to croak.

Next point: Arrow rotation. Yaw is constant and can be set as the arrow is spawned (ideally inside `on_activate`);
pitch however depends on the current velocity of the arrow.

Despite using an animation (as a hack to implement pitch, which used to not be supported) for this,
older arrow mods (read: [simple shooter](https://github.com/stujones11/shooter/blob/master/shooter_crossbow/init.lua))
would have arrows recalculate their pitch, setting the appropriate animation frame every server step.

Newer arrow mods don't use an animation, but rather `set_rotation` which
provides rotation interpolation (using Euler angles rather than Quaterion spherical linear interpolation however)
barely good enough to handle this case well
(although you may temporarily get outdated rotations on clients).

Both approaches incur significant network traffic: All arrows have to send update packets every step.

The theoretically ideal solution within the engine constraints would be to use an animation to rotate the arrow,
simply calculating the animation speed (and animation frame to set) based on the initial velocity and direction of the arrow.

### The Bow Item

Players expect a "chargeable" bow that can be "fired".
A bow item is straightforward to register;
a decent "loading" & "firing" however animation is not as easy:

* Multiple items: Swapping the bow item to a "charged" bow; charging might use tool wear or other HUDs,
  perhaps even multiple charging "steps" are implemented using multiple "(partially) charged" items
  \- currently, changing the texture always requires changing the item
  (there is an [issue](https://github.com/Luanti/Luanti/issues/5686) to change this, but PRs went nowhere so far);
  after firing the bow is replaced with the original item.
* 3D models: Modders might want a decent bow model rather than just an (extruded) item image.
  They can abuse a node for this, disabling placement (serverside) & placement prediction (clientside).
  A "firing" / "digging" animation is still not customizable;
  modders still have to swap items and accept Luanti's standard "tool use" / "punch" animation inbetween.
* First-person attachments: Being a somewhat recent engine feature, this allows largely mitigating the problem:
  It allows registering a "bow" entity with an appropriate animated model,
  attaching it to the player and properly animating it.
  It doesn't suffice to have it look well in first person though
  \- it must look decent to you *and other players* both in first and third person
  \- requiring decent modeling skills.

All of these approaches will, for lack of [server-sent client-side mod support](https://github.com/Luanti/Luanti/projects/1#card-87374775)
(and for limitations of client-side mod capabilities), be rather janky:
From the perspective of the client, every action will require one round-trip time to take effect.

### Synchronization

In Luanti, there is no proper synchronization.
The entity spawning is offset by the time it takes the packet to arrive;
there is no timestamping or other elaborate synchronization mechanism
(however, entity positions will eventually get resent).
Networking should be kept at a very low latency for this not to be problematic.

## Sounds

Arrows should make a "woosh" sound, bows are supposed to make a "zoing" sound as you release the "string".

The former can be implemented by attaching a sound to the arrow entity,
the latter by attaching the sound to the firing player.

This is probably the only part that isn't quite janky.

## Physics

### Acceleration

~~Just like every entity in the game arrows must be affected by gravity.
Unfortunately Luanti approximates gravity wrong such that it is highly dependent
on client/server step sizes. What the client sees at 60 FPS is thus unlikely to match
what the server sees at 10 steps per second; the error can be rather significant:
Arrows will fall faster the lower the steps per second.
This is particularly problematic if lag rises, even if only temporarily.~~

~~A workaround might be to correct the position every step,
but this would suffer from being very janky on clients
(and again taking a huge toll on network traffic).~~

[The fix has since made its way into the engine; the workaround is now obsolete.](https://github.com/Luanti/Luanti/pull/12353).

### Collisions

#### In theory

The arrow logic is quite simple: Whatever the parabola taken by the arrow tip first hits
is considered "hit" and the arrow is attached to it in the appropriate relative position.

Object collision shapes in Luanti are simply boxes, greatly simplifying the code at the expense
of accurate visuals or simulations: The arrow might as well stick "in the air" the collision box provides
(since collision boxes are usually too wide; if a collision box were too narrow, arrows may be stuck too deep inside entities).

A pseudo-ideal mod-only solution would *theoretically* be to animate client meshes serverside (fully possible in pure Lua),
then approximate the parabola through ray-triangle intersections with every animated triangle of the mesh,
which provide you with an exact point (and texture coordinates) of where the ray first hit the mesh.
Attaching unfortunately only supports bones, so you have to be lucky with the model:
All triangle vertices may only be affected by a single bone to which you could then attach the arrow
in the appropriate relative position.
This is the case e.g. for the default Luanti Game player model (Jordach's "Sam").

In practice a demo of accurately determining texture coordinates from in-world ray-triangle intersections
(without animation support however)
can be seen in [Epidermis](https://github.com/appgurueu/epidermis),
which allows accurately painting skins in Luanti.

The obvious massive downside of this is that it would require animating all meshes every step,
which can get expensive (but may be feasible using the new async Lua environment "to offload heavy computational tasks").
It also is quite some work to implement.

#### In practice

There are two straightforward ways to implement arrows *in practice*:

* **DIY.** Have your arrows be nonphysical entities and approximate the parabola through a series of raycasts,
  usually performed every server step. The time complexity can trivially be seen to be quadratic in the number of arrows since
  the time complexity of raycasts is linear in the number of entities.
  Another downside is that raycasts currently operate only on selection- rather than collision boxes.
  A [feature request](https://github.com/Luanti/Luanti/issues/12673) for the latter exists.
  It is theoretically possible to make raycasts work on collision boxes in pure Lua.
  In terms of clientside prediction the downside is that clients will first see the arrow "phase through" the entity it hit,
  then have its position be reset to where the arrow hit the entity.
* **Ready-made.** Let your arrows be *physical* entities *colliding with objects* and check the `moveresult` every step.
  Since object collision uses a linear search for each object, this also has a time complexity quadratic in the number of arrows.
  This differs from the first approach in that it properly deals with collision boxes, but it fails to correctly determine the point *where* the arrow collided
  since the arrow will continue moving after the collision
  (only the component of the velocity on the axis on which the collision occurred will be zeroed; the other two components are left untouched),
  again leading to a poor clientside prediction (the arrow will continue moving parallel to the collision box face it bumped into until it is reset to the point where it hit).
  To correctly determine the position where the arrow hit, you will ultimately have to use raycasts anyways, making the "ready-made" option only differ
  in *how clientside prediction is off*: Either the arrow phases through its target or it continues moving parallel to the collisionbox face it collided with.

## Conclusion

Janky bow mods are possible in Luanti.
Server step size and round-trip time must be kept to a minimum to minimize jank.
The default server stepsize of ~10 steps per second is way too low for fast arrows.

Luanti could significantly cut down on the jank if it refrained from the current
server-only modding and allowed client modding to some extent (ideally server-sent client-side mods);
the current client-side mods are too restricted to be of any use here.

A lot of the issues described here also apply to any other weapons implemented in Luanti,
making Luanti an unsuitable game engine for implementing first-person-shooter-style games.

In the end, it all *somehow* "just works" and is "good enough"
for players to not constantly complain, but it could definitely be much, much better given adequate engine support.
The current state of Luanti leads to modders limiting their mods significantly (e.g. throttling arrow speeds)
to not exacerbate the shortcomings of the Luanti engine, carefully designing their mods around the restrictions.
