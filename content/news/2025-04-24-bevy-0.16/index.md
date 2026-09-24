+++
title = "Bevy 0.16"
date = 2025-04-24 
[extra]
image = "planet.jpg"
show_image = true
image_subtitle = "A planet from EmbersArc's in-development spaceflight simulation game, rendered with custom shaders in Bevy"
image_subtitle_link = "https://bsky.app/profile/embersarc.bsky.social"
+++

Thanks to **261** contributors, **1244** pull requests, community reviewers, and our [**generous donors**](/donate), we're happy to announce the **Bevy 0.16** release on [crates.io](https://crates.io/crates/bevy)!

For those who don't know, Bevy is a refreshingly simple data-driven game engine built in Rust. You can check out our [Quick Start Guide](/learn/quick-start) to try it today. It's free and open source forever! You can grab the full [source code](https://github.com/bevyengine/bevy) on GitHub. Check out [Bevy Assets](https://bevy.org/assets) for a collection of community-developed plugins, games, and learning resources.

To update an existing Bevy App or Plugin to **Bevy 0.16**, check out our [0.15 to 0.16 Migration Guide](/learn/migration-guides/0-15-to-0-16/).

Since our last release a few months ago we've added a _ton_ of new features, bug fixes, and quality of life tweaks, but here are some of the highlights:

- **GPU-Driven Rendering:** Bevy now does even more rendering work on the GPU (where possible), making Bevy dramatically faster on big, complex scenes.
- **Procedural Atmospheric Scattering:** Simulate realistic physically-based Earth-like sky at any time of day at a low performance cost.
- **Decals**: Dynamically layer textures onto rendered meshes.
- **Occlusion Culling**: Improve performance by not rendering objects that are obscured by other objects.
- **ECS Relationships:** One of the hottest ECS features is finally here: allowing you to easily and robustly model and work with entity-entity connections. Some caveats apply, but we're excited to get a simple and robust solution to users today.
- **Improved Spawn API:** Spawning entity hierarchies is now significantly easier!
- **Unified Error Handling:** Bevy now has first class error handling support, making it easy, flexible, and ergonomic, while also making debugging easier!
- **`no_std` Support:** `bevy` itself and a ton of our subcrates no longer rely on Rust's standard library, letting you use the same engine on everything from a modern gaming rig to a Gameboy Advance.
- **Faster Transform Propagation:** We've dramatically improved the performance of transform propagation for more objects at once, especially if they are static.
<!-- more -->

## GPU-Driven Rendering

{{ <heading_metadata authors={["@pcwalton", "@tychedelia", "@robswain"]} prs={[16427]} /> }}

Over the years, the trend in real-time rendering has increasingly been to move work from the CPU to the GPU. One of the latest developments in this area has been *GPU-driven rendering*, in which the GPU takes a representation of the scene and essentially works out what to draw on its own.

**Bevy 0.16** adds GPU-driven rendering support for most "standard" 3D mesh rendering, including skinned meshes. This dramatically reduces the amount of CPU time that the renderer needs for larger scenes. It's automatically enabled on platforms that support it; unless your application hooks into the rendering pipeline, upgrading to **Bevy 0.16** will automatically enable GPU-driven rendering for your meshes. This joins the support that **Bevy 0.14** and **0.15** added for GPU-driven rendering of [Virtual Geometry](/news/bevy-0-14/#virtual-geometry-experimental).

On Activision's "heavy" hotel section of the [Caldera scene](https://github.com/Activision/caldera) from Call of Duty Warzone, **Bevy 0.16** with GPU-driven rendering performs roughly 3x better than **Bevy 0.15**! (this includes *all* optimizations between these releases)

![Caldera scene rendered in Bevy](caldera.jpg)

On a mobile Nvidia 4090 with Vulkan / Linux, **Bevy 0.16** runs the scene at 10.16ms (~101 FPS), compared to 33.55ms (~30 FPS) on **Bevy 0.15**. Massive wins!

### Overview: CPU-Driven Rendering

To explain how GPU-driven rendering operates, it's easiest to first describe how CPU-driven rendering works:

1. The CPU determines the objects that are visible, via frustum culling and perhaps occlusion culling.
2. For each such object:
    * The CPU sends the object's transform to the GPU, possibly in addition to other data such as joint weights.
    * The CPU tells the GPU where the mesh data is.
    * The CPU writes the material data to the GPU.
    * The CPU tells the GPU where textures and other buffers needed to render the objects are (light data, etc.)
    * The CPU issues a drawcall.
    * The GPU renders the object.

### Overview: GPU-Driven Rendering

In contrast, GPU-driven rendering in Bevy works like this:

1. The CPU supplies a single buffer containing transform information for all objects to the GPU, so that shaders can process many objects at once.
2. If new objects have been spawned since the last frame, the CPU fills out tables specifying where the mesh data for the new objects are.
3. If materials have been modified since the last frame, the CPU uploads information about those materials to the GPU.
4. The CPU creates lists of objects to be rendered this frame. Each object is simply referenced by an integer ID, so these lists are small. The number of lists depends on the size and complexity of the scene, but there are rarely more than 15 such lists even for large scenes.
5. For each such list:
    * The CPU issues a *single* drawcall.
    * The GPU processes all objects in the list, determining which ones are truly visible.
    * The GPU renders each such visible object.

For large scenes that may have tens of thousands of objects, GPU-driven rendering frequently results in a reduction in CPU rendering overhead of 3× or more. It's also necessary for occlusion culling, because of the GPU transform step (5(b) above).

### GPU-Driven Rendering Techniques

Internally, GPU-driven rendering is less a single technique than a combination of several techniques. These include:

* *Multi-draw indirect* (MDI), a GPU API that allows multiple meshes to be drawn in a single drawcall, the details of which the GPU provides by filling out tables in GPU memory. In order to use MDI effectively, Bevy uses a new subsystem, the *mesh allocator*, which manages the details of packing meshes together in GPU memory.
  * *Multi-draw indirect count* (MDIC), an extension to multi-draw indirect that allows the GPU to determine the *number* of meshes to draw with minimal overhead.
* *Bindless resources*, which allow Bevy to supply the textures (and other resources) for many objects as a group, instead of having to bind textures one-by-one on the CPU. These resources are managed by a new subsystem known as the *material allocator*.
* *GPU transform and cull*, which allows Bevy to compute the position and visibility of every object from the camera's perspective on the GPU instead of on the CPU.
* The *retained render world*, which allows the CPU to avoid processing and uploading data that hasn't changed since the last frame.
* *Cached pipeline specialization*, which leverages Bevy's component-level change detection to more quickly determine when the rendering state for meshes is unchanged from the previous frame.

### GPU-Driven Platform Compatibility

At the moment, not all platforms offer full support for this feature. The following table summarizes the platform support for the various parts of GPU-driven rendering:

| OS      | Graphics API | GPU transform | Multi-draw & GPU cull | Bindless resources |
|---------|--------------|---------------|-----------------------|--------------------|
| Windows | Vulkan       | ✅            | ✅                   | ✅                |
| Windows | Direct3D 12  | ✅            | ❌                   |❌                 |
| Windows | OpenGL       | ✅            |❌                    |❌                 |
| Linux   | Vulkan       | ✅            | ✅                   | ✅                |
| Linux   | OpenGL       | ✅            |❌                    |❌                 |
| macOS   | Metal        | ✅            |❌                    |➖¹                |
| iOS     | Metal        | ✅            |❌                    |➖¹                |
| Android | Vulkan       | ➖²            |➖²                    |➖²               |
| Web     | WebGPU       | ✅            |❌                    |❌                 |
| Web     | WebGL 2       | ❌            |❌                    |❌                 |

¹ Bevy does support bindless resources on Metal, but the limits are currently significantly lower, potentially resulting in more drawcalls.

² Some Android drivers that are known to exhibit bugs in Bevy's workloads are denylisted and will cause Bevy to fall back to CPU-driven rendering.

In most cases, you don't need to do anything special in order for your application to support GPU-driven rendering. There are two main exceptions:

1. Materials with custom WGSL shaders will continue to use CPU-driven rendering by default. In order for your materials to use GPU-driven rendering, you'll want to use the new `#[bindless]` feature on `AsBindGroup`. See the `AsBindGroup` documentation and the `shader_material_bindless` example for more details. If you're using `ExtendedMaterial`, check out the new `extended_material_bindless` example.
2. Applications and plugins that hook into the renderer at a low level will need to be updated to support GPU-driven rendering. The newly-updated `custom_phase_item` and `specialized_mesh_pipeline` examples may prove useful as a guide to do this.

### What's Next for GPU-Driven Rendering?

Bevy's current GPU-driven rendering isn't the end of the story. There's a sizable amount of potential future work to be done:

* **Bevy 0.16** only supports GPU-driven rendering for the 3D pipeline, but the techniques are equally applicable to the 2D pipeline. Future versions of Bevy should support GPU-driven rendering for 2D mesh rendering, sprites, UI, and so on.
* Bevy currently draws objects with morph targets using CPU-driven rendering. This is something we plan to address in the future. Note that the presence of objects with morph targets doesn't prevent objects that don't have morph targets from being drawn with GPU-driven rendering.
* In the future, a portion of the GPU-driven rendering infrastructure could be ported to platforms that don't support the full set of features, offering some performance improvements on those platforms. For example, even on WebGL 2 the renderer could make use of the material allocator to pack data more efficiently.
* We're watching new API features, such as [Vulkan device generated commands] and [Direct3D 12 work graphs], with interest. These would allow future versions of Bevy to offload even more work to the GPU, such as sorting of transparent objects. While figuring out how to unify these disparate APIs in a single renderer will be a challenge, the future possibilities in this space are exciting.

If you're interested in any of these tasks, feel free to ask [in our Discord](https://discord.gg/bevy) or via [GitHub issues](https://github.com/bevyengine/bevy/issues).

[Vulkan device generated commands]: https://www.supergoodcode.com/device-generated-commands/

[Direct3D 12 work graphs]: https://devblogs.microsoft.com/directx/d3d12-work-graphs/

## Procedural Atmospheric Scattering

{{ <heading_metadata authors={["@ecoskey", "@mate-h"]} prs={[16314, 17555, 17672, 18582]} /> }}

<!-- Procedural atmospheric scattering -->
<!-- https://github.com/bevyengine/bevy/pull/16314 -->
**Bevy 0.16** introduces procedural atmospheric scattering, a customizable system for simulating sunsets, sunrises, and dynamic day/night cycles in real time:

<video controls loop aria-label="A showcase of simulated sunrises and sunsets made in Bevy"><source src="atmosphere-showcase.mp4" type="video/mp4"/></video>
Credit to `@aevyrie` for their amazing [atmosphere showcase]! It uses a fancy custom exposure curve to accentuate the near-dusk colors.

Enabling atmosphere rendering is simple, just add the new [`Atmosphere`] component to your camera!

```rs
commands.spawn((
    Camera3d::default(),
    Atmosphere::EARTH,
));

// the atmosphere will consider all directional lights in the scene as "suns"
commands.spawn(DirectionalLight::default());
```

When it is enabled, the primary Bevy skybox is overlaid with one that updates in real-time based on the directional lights in the scene, and the default distance fog is replaced with one that takes into account directional lights and the other atmosphere parameters. Distant objects will fade to blue on a clear day, and will be tinged orange and pink at sunset! Also, because the atmosphere is composited on *top* of the skybox, creating a nighttime starscape is easy ... just spawn the skybox and it'll naturally fade away as the sky grows brighter at dawn.

As with most PBR techniques, it's *correct*, but it can take some tweaking to look its best. All of the atmosphere parameters can also be customized: for example, a high desert sky might exhibit less Mie scattering due to the lack of humidity. The included example at `examples/3d/atmosphere.rs` includes some recommendations for lighting and camera settings.

There's some current limitations to be aware of: the atmosphere currently doesn't affect the [`EnvironmentMapLight`] or the direct lighting from directional lights on surfaces, so reflections might not be fully accurate. We're also working on integrating atmospheric scattering with volumetric fog (see the [What's Next?](#what-s-next) section for our ambitious plans!).

Because of a number of optimizations described in Sébastien Hillaire's [EGSR 2020 paper], our implementation is super fast, and should work great even on mobile devices and WebGPU. The secret sauce is that because the atmosphere is mostly symmetric, we can precalculate most of the ray marching inner loop ahead of time.

[atmosphere showcase]: https://github.com/aevyrie/bevy/tree/atmosphere_showcase
[EGSR 2020 paper]: https://sebh.github.io/publications/egsr2020.pdf
[`EnvironmentMapLight`]: https://docs.rs/bevy/0.16/bevy/pbr/environment_map/struct.EnvironmentMapLight.html
[`Atmosphere`]: https://docs.rs/bevy/0.16/bevy/pbr/struct.Atmosphere.html

## Decals

{{ <heading_metadata authors={["@naasblod", "@JMS55", "@pcwalton"]} prs={[16600, 17315]} /> }}

**Decals** are textures which can be dynamically layered on top of existing meshes, conforming to their geometry.
This has two benefits over simply changing a mesh's texture:

1. You can add them dynamically in response to player actions. Most famously, bullet holes in FPS games use decals for this.
2. You don't need to create an entirely new texture for every combination, which makes them more efficient and flexible when creating levels with details like graffiti or cracks in building facades.

Like many things in rendering, there are a huge number of ways to implement this feature, each with their own tradeoffs.
In **Bevy 0.16**, we've selected two complementary approaches: **forward decals** and **clustered decals**.

### Forward Decals

![decals](decals.jpg)

Our implementation of forward decals (or to be more precise, contact projective decals) was inspired by [Alexander Sannikovs talk on the rendering techniques of Path of Exile 2], and was upstreamed from the [`bevy_contact_projective_decals`] ecosystem crate.
Due to the nature of this technique, looking at the decal from very steep angles will cause distortion.
This can be mitigated by creating textures that are bigger than the effect, giving the decal more space to stretch.
To create a forward decal, spawn a [`ForwardDecal`] entity, which uses a [`ForwardDecalMaterial`] using the [`ForwardDecalMaterialExt`] material extension.

### Clustered Decals

![clustered decals](clustered-decals.jpg)

Clustered decals (or decal projectors) work by projecting images from a 1x1x1 cube onto surfaces found in the +Z direction.
They are clusterable objects, just like point lights and light probes, which means that decals are only evaluated for objects within the bounds of the projector.
To create a clustered decal, spawn a [`ClusteredDecal`] entity.

Ultimately, forward decals offer broader hardware and driver support, while clustered decals are higher quality and don't require the creation of bounding geometry, improving performance.
Currently clustered decals require bindless textures and thus don't support WebGL2, WebGPU, iOS and Mac targets. Forward decals _are_ available on these targets.

[Alexander Sannikovs talk on the rendering techniques of Path of Exile 2]: https://www.youtube.com/watch?v=TrHHTQqmAaM
[`bevy_contact_projective_decals`]: https://github.com/naasblod/bevy_contact_projective_decals
[`ForwardDecal`]: https://docs.rs/bevy/0.16/bevy/pbr/decal/struct.ForwardDecal.html
[`ForwardDecalMaterial`]: https://docs.rs/bevy/0.16/bevy/pbr/decal/type.ForwardDecalMaterial.html
[`ForwardDecalMaterialExt`]: https://docs.rs/bevy/0.16/bevy/pbr/decal/struct.ForwardDecalMaterialExt.html
[`ClusteredDecal`]: https://docs.rs/bevy/0.16/bevy/pbr/decal/clustered/struct.ClusteredDecal.html

## Experimental Occlusion Culling

{{ <heading_metadata authors={["@pcwalton", "@JMS55"]} prs={[12755, 17413, 17934, 17951]} /> }}

**Occlusion culling** is the idea that we don't need to draw something that's completely blocked by other opaque objects, from the perspective of the camera. For example: we don't need to draw a person hidden behind a wall, even if they're within the range used for frustum culling.

Bevy already has an optional [Depth Prepass](/news/bevy-0-10/#depth-and-normal-prepass), which renders a simple version of the scene and captures a 2D depth buffer. This can then be used to skip hidden objects in the more expensive main pass. However this doesn't skip the vertex shading overhead, and the depth checks in the fragment shader also add overhead.

In **Bevy 0.16**, we've added modern [two-phase occlusion culling](https://medium.com/@mil_kru/two-pass-occlusion-culling-4100edcad501) (in contrast to a traditional "potentially visible sets" design). This approach was already used by our [virtual geometry](/news/bevy-0-14/#virtual-geometry-experimental) rendering system, and works quite well with the GPU-driven rendering architecture that we've established during this cycle! For more details on our implementation, [check out this PR](https://github.com/bevyengine/bevy/pull/17413).

For now, this feature is marked as experimental, due to known precision issues that can mark meshes as occluded even when they're not.
In practice, we're not convinced that this is a serious concern, so please let us know how it goes!
To try out the new mesh occlusion culling, add the [`DepthPrepass`] and [`OcclusionCulling`] components to your camera.

An important note: occlusion culling won't be faster on all scenes. Small scenes, or those using simpler non-PBR rendering are particularly likely to be slower with occlusion culling turned on. Enabling occlusion culling incurs overhead ... the work that it skips must be more expensive than the cost of running the checks for it to be worth it!

Like always: you need to measure your performance to improve it.

If you're a rendering engineer who'd like to help us resolve these precision issues and stabilize this feature, we're looking to borrow from Hans-Kristian Arntzen's design in [Granite].
Chime in at [issue #14062] (and read our [Contributing Guide]) and we can help you get started.

[`DepthPrepass`]: https://docs.rs/bevy/0.16/bevy/core_pipeline/prepass/struct.DepthPrepass.html
[`OcclusionCulling`]: https://docs.rs/bevy/0.16/bevy/render/experimental/occlusion_culling/struct.OcclusionCulling.html
[issue #14062]: https://github.com/bevyengine/bevy/issues/14062
[Granite]: https://github.com/Themaister/Granite
[Contributing Guide]: https://bevy.org/learn/contribute/introduction/

## Anamorphic Bloom

{{ <heading_metadata authors={["@aevyrie"]} prs={[17096]} /> }}

Bloom is that soft glow bleeding out from bright areas of light: in the real world, this is caused by sensors on your camera (or eyes) becoming fully saturated.
In games, it's a powerful artistic tool for creating everything from cyberpunk neon signs to tastefully glowing windows to juicy geometrically-inspired arcade games.

Bevy has had bloom since version 0.9, but we're giving artists another simple lever to tweak: the ability to stretch, squash and otherwise distort the effect by setting the 2-dimensional `scale` parameter on the [`Bloom`] component on your camera.

![A realistic high-polygon model of a fancy black Porsche 911 demonstrating anamorphic bloom in its stretched-out tail light glow. Rendered in Bevy 0.16!](anamorphic-car-bloom.jpg)

When heavily skewed (usually horizontally), this effect is known as **anamorphic bloom**.
This effect is associated with a cinematic, futuristic vibe, and emulates the unusual geometry of certain film cameras as they compress a wider image onto narrower film.
But, regardless of why it occurs, it simply looks neat!

[`Bloom`]: https://docs.rs/bevy/0.16/bevy/core_pipeline/bloom/struct.Bloom.html

## ECS Relationships

{{ <heading_metadata authors={["@cart"]} prs={[17398]} /> }}

When building Bevy apps, it is often useful to "link" entities together. The most common case in Bevy is connecting parent and child entities together. In previous Bevy versions, a child would have a `Parent` component, which stored a reference to the parent entity, and the parent entity would have a `Children` component, which stored a list of all of its children entities. To ensure these connections remained valid, developers were not allowed to modify these components directly. Instead, all changes had to be done via specialized commands.

This worked, but it had some pretty glaring downsides:

1. Maintaining hierarchies was "separate" from the core ECS data model. This made it hard to improve our spawn APIs, and made interacting with hierarchies less natural.
2. The system was specialized and not reusable. Developers wanting to define their own relationship types had to reinvent the wheel.
3. To ensure data integrity, expensive scans were required to avoid duplicates.

In **Bevy 0.16** we have added initial support for **relationships**: a generalized and efficient component-driven system for linking entities together bidirectionally. This is what defining a new [`Relationship`] looks like:

```rust
/// This is a "relationship" component.
/// Add it to an entity that "likes" another entity.
#[derive(Component)]
#[relationship(relationship_target = LikedBy)]
struct Likes(pub Entity);

/// This is the "relationship target" component.
/// It will be automatically inserted and updated to contain
/// all entities that currently "like" this entity.
#[derive(Component, Deref)]
#[relationship_target(relationship = Likes)]
struct LikedBy(Vec<Entity>);

// Later in your app
let e1 = world.spawn_empty().id();
let e2 = world.spawn(Likes(e1)).id();
let e3 = world.spawn(Likes(e1)).id();

// e1 is liked by e2 and e3 
let liked_by = world.entity(e1).get::<LikedBy>().unwrap();
assert_eq!(&**liked_by, &[e2, e3]);
```

The [`Relationship`] component is the "source of truth", and the [`RelationshipTarget`] component is updated to reflect that source of truth. This means that adding/removing relationships should always be done via the [`Relationship`] component.

We use this "source of truth" model instead of allowing both components to "drive" for performance reasons. Allowing writes to both sides would require expensive scanning during inserts to ensure they are in sync and have no duplicates. The "relationships as the source of truth" approach allows us to make adding relationships constant-time (which is an improvement over the old Bevy parent/child approach!).

Relationships are built on top of Bevy's [Component Hooks](/news/bevy-0-14/#ecs-hooks-and-observers), which immediately and efficiently maintain the connection between the [`Relationship`] and the [`RelationshipTarget`] by plugging directly into the component add/remove/update lifecycle. In combination with the new [Immutable Components](#immutable-components) feature (relationship components are immutable), this ensures data integrity is maintained no matter what developers do!

Bevy's existing hierarchy system has been fully replaced by the new [`ChildOf`] [`Relationship`] and [`Children`] [`RelationshipTarget`]. Adding a child is now as simple as:

```rust
commands.spawn(ChildOf(some_parent));
```

Likewise reparenting an entity is as simple as:

```rust
commands.entity(some_entity).insert(ChildOf(new_parent));
```

We also took this chance to improve our spawn APIs more generally. Read the next section for details!

Note that this is just the first step for relationships. We have plans to expand their capabilities:

1. **Many-To-Many Relationships**: The current system is one-to-many (ex: The [`ChildOf`] [`Relationship`] points to "one" target entity and the [`Children`] [`RelationshipTarget`] can be targeted by "many" child entities). Some relationships could benefit from supporting many relationship targets.
2. **Fragmenting Relationships**: In the current system, relationship components "fragment" ECS archetypes based on their _type_, just like a normal component (Ex: `(Player, ChildOf(e1))`, and `(Player, ChildOf(e2))` exist in the same archetype). Fragmenting relationships would be an opt-in system that fragment archetypes based on their _value_ as well, which would result in entities with the same relationship targets being stored next to each other. This serves as an index, making querying by value faster, and making some access patterns more cache friendly.

[`Relationship`]: https://docs.rs/bevy/0.16/bevy/ecs/relationship/trait.Relationship.html
[`RelationshipTarget`]: https://docs.rs/bevy/0.16/bevy/prelude/trait.RelationshipTarget.html
[`ChildOf`]: https://docs.rs/bevy/0.16/bevy/prelude/struct.ChildOf.html
[`Children`]: https://docs.rs/bevy/0.16/bevy/ecs/hierarchy/struct.Children.html

## Improved Spawn API

{{ <heading_metadata authors={["@cart"]} prs={[17521]} /> }}

Spawning hierarchies in Bevy has historically been a bit cumbersome:

```rust
commands
    .spawn(Player)
    .with_children(|p| {
        p.spawn(RightHand).with_children(|p| {
            p.spawn(Glove);
            p.spawn(Sword);
        });
        p.spawn(LeftHand).with_children(|p| {
            p.spawn(Glove);
            p.spawn(Shield);
        });
    });
```

We have big plans to improve Bevy's spawning experience with our [Next Generation Scene / UI System](https://github.com/bevyengine/bevy/discussions/14437) (BSN). An important stepping stone on that path is making it possible to express hierarchies directly via data, rather than using builder methods. The addition of Relationships further increases the value of building such a system, as _all_ relationships can benefit from it.

In **Bevy 0.16** we have vastly improved the ergonomics of spawning hierarchies:

```rust
commands.spawn((
    Player,
    children![
        (RightHand, children![Glove, Sword]),
        (LeftHand, children![Glove, Shield]),
    ],
));
```

This builds on the existing Bundle API by adding support for "bundle effects", which are applied immediately after a Bundle is inserted. Notably, this enables developers to define functions that return a hierarchy like this:

```rust
fn player(name: &str) -> impl Bundle {
    (
        Player,
        Name::new(name),
        children![
            (RightHand, children![Glove, Sword]),
            (LeftHand, children![Glove, Shield]),
        ]
    )
}

// later in your app
commands.spawn(player("Bob"));
```

In most cases the `children!` macro should be preferred for ergonomics reasons. It expands to the following API:

```rust
commands.spawn((
    Player,
    Children::spawn((
        Spawn((
            RightHand,
            Children::spawn((Spawn(Glove), Spawn(Sword))),
        )),
        Spawn((
            LeftHand,
            Children::spawn((Spawn(Glove), Spawn(Shield))),
        )),
    )),
));
```

There are a number of spawn wrapper variants, which provide additional flexibility:

```rust
world.spawn((
    Name::new("Root"),
    Children::spawn((
        Spawn(Name::new("Child1")),   
        SpawnIter(["Child2", "Child3"].into_iter().map(Name::new)),
        SpawnWith(|parent: &mut ChildSpawner| {
            parent.spawn(Name::new("Child4"));
            parent.spawn(Name::new("Child5"));
        })
    )),
))
```

Notably, this API works for _all_ relationship types. For example, you could spawn a `Likes` / `LikedBy` relationship hierarchy (as defined in the relationships section above) like this:

```rust
world.spawn((
    Name::new("Monica"),
    LikedBy::spawn((
        Spawn(Name::new("Naomi")),
        Spawn(Name::new("Dwight")),
    ))
))
```

There is also a `related!` macro, which does the same thing as `children!`, but for any relationship type:

```rust
world.spawn((
    Name::new("Monica"),
    related!(LikedBy[
        Name::new("Naomi"),
        Name::new("Dwight"),
    ]),
))
```

This API also allows us to optimize hierarchy construction time by cutting down on re-allocations, as we can generally (with the exception of cases like `SpawnWith`) statically determine how many related entities an entity will have and preallocate space for them in the `RelationshipTarget` component (ex: `Children`).

## Unified ECS Error Handling

{{ <heading_metadata authors={["@NthTensor", "@JeanMertz", "@JaySpruce", "@cart", "@alice-i-cecile"]} prs={[16589, 17043, 17753, 18144, 18454]} /> }}

Bevy has historically left error handling as an exercise for the user. Many Bevy APIs return Results / errors, but Bevy ECS systems and commands did not themselves support returning Results / errors. Additionally, Bevy provides some "shorthand" panicking APIs that require less typing than unwrapping results. These things encouraged developers to write overly panick-ey behavior, even when that wasn't desired.

Ideally developers can _choose_ whether or not their systems should panic. For example, panic while you are developing to discover errors quickly (and force you to solve them), but when deploying to production just print errors to a console to avoid disrupting the user's experience.

**Bevy 0.16** introduces a new unified paradigm for error handling to help you ship crash-free games (and other applications!)
without sacrificing the loud-and-fast development that panics enable.

We've prioritized a simple and consistent design, with a few bells and whistles for easier debugging:

- Bevy systems, observers, and commands now support returning [`Result`], which is a type alias for `Result<(), BevyError>`
- [`Result`] expects the new [`BevyError`], which accepts any error type (much like [`anyhow`]). This makes it easy and ergonomic to use the [`?` operator] in systems / observers / commands to capture and return errors.
- [`BevyError`] automatically captures [high quality custom backtraces]. By default these are filtered down to just the essentials, cutting down on a lot of the noise from Rust and Bevy internals.
- Errors returned by systems, observers, and commands are now handled by the [`GLOBAL_ERROR_HANDLER`]. This defaults to panicking, but can be set to anything (log, panic, custom logic, etc). We generally encourage developing with the panicking default, then switching to logging errors when deploying to production.

We now encourage developers to bubble up errors whenever possible, rather than panicking immediately where the error occurs.

Instead of:

```rust
use bevy::prelude::*;

fn move_player(mut query: Query<&mut Transform, With<Player>>) {
    let mut player_transform = query.single_mut().unwrap();
    player_transform.translation.x += 1.0;
}
```

Try:

```rust
use bevy::prelude::*;

fn move_player(mut query: Query<&mut Transform, With<Player>>) -> Result {
    let mut player_transform = query.single_mut()?;
    player_transform.translation.x += 1.0;
    Ok(())
}
```

[`Result`]: https://docs.rs/bevy/0.16/bevy/ecs/error/type.Result.html
[`?` operator]: https://doc.rust-lang.org/rust-by-example/std/result/question_mark.html
[`anyhow`]: https://docs.rs/anyhow/latest/anyhow/
[high quality custom backtraces]: https://github.com/bevyengine/bevy/pull/18144
[`GLOBAL_ERROR_HANDLER`]: https://docs.rs/bevy/0.16/bevy/ecs/error/static.GLOBAL_ERROR_HANDLER.html
[`BevyError`]: https://docs.rs/bevy/0.16/bevy/ecs/error/struct.BevyError.html

## Bevy `no_std` Support

{{ <heading_metadata authors={["@bushrat011899"]} prs={[15281, 15463, 15464, 15465, 15528, 15810, 16256, 16633, 16758, 16874, 16995, 16998, 17027, 17028, 17030, 17031, 17086, 17470, 17490, 17491, 17505, 17507, 17955, 18061, 18333]} /> }}

<!-- Add `no_std` support to `bevy` -->
<!-- https://github.com/bevyengine/bevy/pull/17955 -->

Bevy now has support for `no_std` targets, allowing it to be used on a much wider range of platforms.

Early reports from users have shown Bevy working on bare-metal desktops, embedded devices, and even retro consoles such as the GameBoy Advance:

<video controls loop aria-label="A purple GameBoy Advance running a game made using Bevy"><source  src="bevy-gba.mp4" type="video/mp4"/></video>

Credit to [Chris Biscardi](https://www.youtube.com/@chrisbiscardi) for creating this awesome demo!

Bevy `no_std` support has been [discussed] going back over 4 years, but was initially dropped to avoid the added complexity managing `no_std` support can bring.
To be `no_std` compatible, your crate and _all_ of its dependencies must also be `no_std`.
Coordinating that kind of support across over a hundred dependencies was just not feasible, let alone losing access to Rust's standard library.

Since then, Rust's support for `no_std` has evolved dramatically with support for critical APIs such as [`Error`] coming in [Rust 1.81].
Starting with tracking issue [#15460] and a [`no_std` Working Group], Bevy's various crates were individually made `no_std` compatible where possible.
To aid this effort, [`bevy_platform`] was developed with the goal of providing opinionated alternatives to `std` items.

This effort reached a substantial milestone during the development of **Bevy 0.16**: support for `no_std` in our main `bevy` crate.
To use Bevy on a `no_std` platform, simply disable default features and use Bevy just like any other `no_std` dependency.

```toml
[dependencies]
bevy = { version = "0.16", default-features = false }
```

Note that not all of Bevy's features are compatible with `no_std` yet.
Rendering, audio, and assets are notable missing APIs that you will need to find an alternative for that's suitable for your platform.
But, Bevy's powerful [`Plugin`] system allows the community to build back support for that functionality for their particular platforms.

For those community members working on libraries for Bevy, we encourage you to try out `no_std` support if you can!
There's a new [`no_std` library] example which demonstrates how to make a crate that is compatible with `std` and `no_std` users, with detailed comments and advice.
During the release candidate period, quite a few libraries have successfully experimented with `no_std` support, such as [`bevy_rand`] and [`bevy_replicon`].

Determining what `no_std` targets support Bevy is still a work in progress. If you have an unusual platform you'd like to try getting Bevy working on, check out the [`#unusual-platforms`] channel on Bevy's Discord server for advice!

[`Error`]: https://doc.rust-lang.org/stable/core/error/trait.Error.html
[#15460]: https://github.com/bevyengine/bevy/issues/15460
[`no_std` Working Group]: https://discord.com/channels/691052431525675048/1303128171352293410
[`bevy_platform`]: https://crates.io/crates/bevy_platform/
[`#unusual-platforms`]: https://discord.com/channels/691052431525675048/1284885928837517432
[discussed]: https://github.com/bevyengine/bevy/discussions/705
[`Plugin`]: https://docs.rs/bevy/latest/bevy/app/trait.Plugin.html
[Rust 1.81]: https://releases.rs/docs/1.81.0/#stabilized-apis
[`no_std` library]: https://github.com/bevyengine/bevy/tree/main/examples/no_std/library
[`bevy_replicon`]: https://github.com/projectharmonia/bevy_replicon/tree/bevy-0.16-dev
[`bevy_rand`]: https://github.com/Bluefinger/bevy_rand
[`bevy_transform_interpolation`]: https://github.com/Jondolf/bevy_transform_interpolation

## Faster Transform Propagation

{{ <heading_metadata authors={["@aevyrie"]} prs={[17840, 18589]} /> }}

Transforms in Bevy (and other 3D software) come in two flavors:

1. [`GlobalTransform`]: represents the absolute world-space position, scale and rotation of an object.
2. [`Transform`]: represent the position, scale, and rotation of an object relative to its parent. This is also known as the "local" transform.

In order to compute the [`GlobalTransform`] of each object (which is what rendering and physics care about!),
we need to recursively combine the [`Transform`] of all of our objects down the parent-child hierarchy.
This process, known as transform propagation, can be pretty expensive, especially with many entities in a scene.

**Bevy 0.16** comes with *two* impressive performance optimizations:

1. **Improved parallelization strategies**: While we were already splitting the work across threads,
better work sharing, parallelization across trees and a leaf vs non-leaf split to optimize cache coherency made a huge difference.
2. **Saving work for trees where none of the objects have moved**: Level geometry and props are not typically moving around each frame, so this optimization applies to *many* cases! We're now propagating a "dirty bit" up the hierarchy towards ancestors; allowing transform propagation to ignore entire subtrees of the hierarchy if they encounter an entity without the dirty bit.

The results speak for themselves: taken together, our testing on the huge (127,515 objects) [Caldera Hotel] scene  from Call of Duty: Warzone shows that transform propagation took 1.1 ms in **Bevy 0.15** on an M4 Max Macbook, and 0.1 ms after these changes in **Bevy 0.16**.
Even fully dynamic scenes (like our [`many_foxes`] stress test) are substantially faster due to the improved parallelism. Nice!
This work matters even more for more typical hardware: on large scenes on mid or low-end hardware transform propagation could eat an entire 25% of the frame budget. Ouch!

While that's an impressive 11x performance improvement, the absolute magnitude of the time saved is the key metric.
With about 16 ms per frame at 60 FPS, that's 6% of your *entire* game's CPU budget saved. making huge open worlds or incredibly complex CAD assemblies more viable than ever before.

![A screenshot of a `tracy` histogram showing the effects of these changes on Caldera. 0.15 peaks at 1.1 ms, while 0.16 peaks at 0.1 ms.][caldera-transform-propagation-bench]

If you're interested in the gory technical details of these optimizations, take a look at [the code itself].
It's incredibly well-commented and great to learn from.

These optimizations were upstreamed from the [`big_space`] crate (by the author of that crate!)

[Caldera Hotel]: https://github.com/Activision/caldera
[the code itself]: https://github.com/bevyengine/bevy/blob/b0c446739888705d3e95b640e9d13e0f1f53f06d/crates/bevy_transform/src/systems.rs#L12
[caldera-transform-propagation-bench]: caldera-transform-propagation-bench.jpg
[`many_foxes`]: https://github.com/bevyengine/bevy/blob/main/examples/stress_tests/many_foxes.rs
[`big_space`]: https://github.com/aevyrie/big_space
[`GlobalTransform`]: https://docs.rs/bevy/0.16.0/bevy/transform/components/struct.GlobalTransform.html
[`Transform`]: https://docs.rs/bevy/0.16.0/bevy/transform/components/struct.Transform.html

## Specular Tints and Maps

{{ <heading_metadata authors={["@pcwalton"]} prs={[14069]} /> }}

<!-- Add support for specular tints and maps per the `KHR_materials_specular` glTF extension. -->
<!-- https://github.com/bevyengine/bevy/pull/14069 -->

If you have an eye for light (or training in visual arts), you'll notice that shiny curved surfaces get extra-bright spots of light.
That's a specular highlight!
In **Bevy 0.16**, we've implemented a standard physically-based rendering (PBR) feature of specular highlights: the ability to tint their color.

![A shiny floating sphere with an iridescent multicolored sheen. It reminds you of a Christmas tree ornament. You can see a reflection of a city scene in it.](specular-tint-sphere.jpg)

This can be done uniformly across the material, by simply setting the `specular_tint` field on the [`StandardMaterial`] for your object.

Like many other material properties (color, normals, emissiveness, roughness etc), this can be varied over the material via the use of a texture map,
which describes how this property varies via a 2-dimensional UV-space image.

Maps are relatively expensive: you need an entire 2D image, rather than a single float or color, and GPUs have limits on the number of textures you can use per material, unless they support bindless textures.
As a result, specular maps are off by default, and gated behind the `pbr_specular_textures` Cargo feature.

To support this work, we now support the KHR_materials_specular glTF extension, allowing artists to set these properties in 3D modelling tools like Blender, then import them into Bevy.

[`StandardMaterial`]: https://docs.rs/bevy/0.16/bevy/pbr/struct.StandardMaterial.html

## Experimental WESL Shaders

{{ <heading_metadata authors={["@tychedelia"]} prs={[17953]} /> }}

<!-- Add support for experimental WESL shader source -->
<!-- https://github.com/bevyengine/bevy/pull/17953 -->

Bevy continues to live life on the edge, the bleeding edge, of graphics technology. Today, Bevy supports [WESL](https://wesl-lang.dev/) shaders!

Most Bevy shaders are written in [WGSL](https://www.w3.org/TR/WGSL/), a modern shader language built for simplicity. But while WGSL is pretty simple as far as shader languages go, it also leaves a lot to be desired in terms of higher level features. Currently, Bevy has its own extended version of WGSL, which adds support for conditional compiling, importing between files, and other useful features.

WESL is a brand new shader language that extends WGSL (often in ways similar to Bevy's approach) and aims to bring common language conveniences to the GPU. WESL not only includes conditional compiling and importing between files, but it is also growing to support generics, package managers, and more.

It's important to note that WESL is still relatively early in development, and not all of its features are fully functional yet, nor are all its features supported in Bevy (yet). For that reason, WESL support is gated behind the cargo feature `shader_format_wesl`, which is disabled by default.

Despite the additional features, WESL is easy to layer on top of existing WGSL shaders. This is because it is a superset of WGSL (WGSL is valid WESL). That makes it easy to migrate existing WGSL to WESL, though it's worth mentioning that the Bevy's own "extended WGSL syntax" will need to be ported to its WESL counterparts. The WESL team (who helped write these notes!) is actively listening to feedback, and so is Bevy. If you do choose to use WESL in addition to or in replacement of WGSL, your thoughts, feature requests, and any pain points you encounter can be shared [here](https://github.com/wgsl-tooling-wg/wesl-rs).

If you're interested in trying WESL, check out the new [Material - WESL](https://bevy.org/examples/shaders/shader-material-wesl/) example. Before using this in a production environment, be sure to check out the original [PR](https://github.com/bevyengine/bevy/pull/17953) for a full list of caveats.

WESL is an exciting frontier for shader languages. You can track their progress and plans [here](https://wesl-lang.dev/).

## Virtual Geometry Improvements

{{ <heading_metadata authors={["@JMS55"]} prs={[17765, 16947, 16941]} /> }}

Virtual geometry (the `meshlet` cargo feature) is Bevy's Nanite-like rendering system, allowing much greater levels of geometry density than is otherwise possible, and freeing artists from manually creating LODs.

In **Bevy 0.16**, virtual geometry got some performance improvements thanks to new algorithms and GPU APIs.

Read more details on the [author's blog post](https://jms55.github.io/posts/2025-03-27-virtual-geometry-bevy-0-16).

Users are not required to regenerate their [`MeshletMesh`](https://docs.rs/bevy/0.16/bevy/pbr/experimental/meshlet/struct.MeshletMesh.html) assets, but doing so is recommended in order to take advantage of the improved clustering algorithm in **Bevy 0.16**.

![Screenshot of the meshlet example](meshlet_bunnies.jpg)

## Immutable Components

{{ <heading_metadata authors={["@bushrat011899"]} prs={[16372, 16668, 18532]} /> }}

<!-- Add Immutable `Component` Support -->
<!-- https://github.com/bevyengine/bevy/pull/16372 -->

**Bevy 0.16** adds the concept of **immutable components**: data that cannot be mutated once it is inserted (ex: you cannot query for `&mut MyComponent`). The only way to modify an immutable component is to insert a new instance on top.

A component can be made immutable by adding the `#[component(immutable)]` attribute:

```rust
#[derive(Component)]
#[component(immutable)]
pub struct MyComponent(pub u32);
```

By embracing this restriction, we can ensure that component lifecycle hooks and observers (add/remove/insert/replace) capture _every_ change that occurs, enabling them to uphold complex invariants.

To illustrate this, consider the following example, where we track the global sum of all `SumMe` components:

```rust
#[derive(Component)]
#[component(immutable)]
pub struct SumMe(pub u32);

// This will always hold the sum of all `SumMe` components on entities.
#[derive(Resource)]
struct TotalSum(u32);

// This observer will trigger when spawning or inserting a SumMe component
fn add_when_inserting(
    trigger: Trigger<OnInsert, SumMe>,
    query: Query<&SumMe>,
    mut total_sum: ResMut<TotalSum>,
) {
    if let Ok(sum_me) = query.get(trigger.target()) {
        total_sum.0 += sum_me.0;
    }
}

// This observer will trigger when despawning or removing a SumMe component
fn subtract_when_removing(
    trigger: Trigger<OnRemove, SumMe>,
    query: Query<&SumMe>,
    mut total_sum: ResMut<TotalSum>,
) {
    if let Ok(sum_me) = query.get(trigger.target()) {
        total_sum.0 -= sum_me.0;
    }
}

// Changing this to `&mut SumMe` would fail to compile!
fn modify_values(mut commands: Commands, query: Query<(Entity, &SumMe)>) {
    for (entity, sum_me) in query.iter() {
        // We can read the value, but not write to it.
        let current_value = sum_me.0;
        // This will overwrite: indirectly mutating the value
        // and triggering both observers: removal first, then insertion
        commands.entity(entity).insert(SumMe(current_value + 1));
    }
}
```

While immutable components are a niche tool, they're a great fit for rarely mutated (or small count) components where correctness is key.
For example, **Bevy 0.16** uses immutable components (and hooks!) as the foundation of our shiny new **relationships** to ensure that both sides of the relationship (e.g. [`ChildOf`] and [`Children`]) stay in sync.

We're keen to develop a first-class indexing solution using these new tools, and excited to hear about your ideas. Stay tuned; we've only scratched the surface here!

[`ChildOf`]: https://docs.rs/bevy/0.16/bevy/ecs/hierarchy/struct.ChildOf.html
[`Children`]: https://docs.rs/bevy/0.16/bevy/ecs/hierarchy/struct.Children.html

## Entity Cloning

{{ <heading_metadata authors={["@eugineerd", "@JaySpruce"]} prs={[16132, 16826]} /> }}

<!-- Entity cloning -->
<!-- https://github.com/bevyengine/bevy/pull/16132 -->
Bevy now has first-class support for cloning entities. While it was possible to do this before using reflection and `ReflectComponent` functionality, the common implementation was slow and required registering all cloneable components. With **Bevy 0.16**, entity cloning is supported natively and is as simple as adding `#[derive(Clone)]` to a component to make it cloneable.

```rust
#[derive(Component, Clone)]
#[require(MagicalIngredient)]
struct Potion;

#[derive(Component, Default, Clone)]
struct MagicalIngredient {
    amount: f32,
}

fn process_potions(
    input: Res<ButtonInput<KeyCode>>,
    mut commands: Commands,
    mut potions: Query<(Entity, &mut MagicalIngredient), With<Potion>>,
) {
    // Create a new potion
    if input.just_pressed(KeyCode::KeyS) {
        commands.spawn(
            (Name::new("Simple Potion"), Potion)
        );
    }
    // Add as much magic as we want
    else if input.just_pressed(KeyCode::KeyM) {
        for (_, mut ingredient) in potions.iter_mut() {
            ingredient.amount += 1.0
        }
    }
    // And then duplicate all the potions!
    else if input.just_pressed(KeyCode::KeyD) {
        for (potion, _) in potions.iter() {
            commands.entity(potion).clone_and_spawn();
        }
    }
}

```

`clone_and_spawn` spawns a new entity with all cloneable components, skipping those that can't be cloned. If your use case requires different behavior, there are more specialized methods:

- `clone_components` clones components from the source entity to a specified target entity instead of spawning a new one.
- `move_components` removes components from the source entity after cloning them to the target entity.
- `clone_and_spawn_with` and `clone_with` allow customization of the cloning behavior by providing access to `EntityClonerBuilder` before performing the clone.

`EntityClonerBuilder` can be used to configure how cloning is performed - for example, by filtering which components should be cloned, modifying how `required` components are cloned, or controlling whether entities linked by relationships should be cloned recursively.

An important note: components with generic type parameters will not be cloneable by default. For these cases, you should add `#[derive(Reflect)]` and `#[reflect(Component)]` to the component and register it for the entity cloning functionality to work properly.

```rust
#[derive(Component, Reflect, Clone)]
#[reflect(Component)]
struct GenericComponent<T> {
    // ...
}

fn main(){
    // ...
    app.register_type::<GenericComponent<i32>>;
    // ...
}
```

See documentation for `EntityCloner` if you're planing on implementing custom clone behaviors for components or need further explanation on how this works.

## Entity Disabling / Default Query Filters

{{ <heading_metadata authors={["@NiseVoid", "@alice-i-cecile"]} prs={[13120, 17514, 17768]} /> }}

**Bevy 0.16** adds the ability to disable entities by adding the [`Disabled`] component. This (by default) will hide the entity (and all of its components) from systems and queries that are looking for it.

This is implemented using the newly added **default query filters**. These do what they say on the tin: every query will act as if they have a `Without<Disabled>` filter, unless they explicitly mention [`Disabled`] (generally via a `With<Disabled>` or `Has<Disabled>` argument).

Because this is generic, developers can also define additional disabling components, which can be registered via [`App::register_disabling_component`]. Having multiple distinct disabling components can be useful if you want each form of disabling to have its own semantics / behaviors: you might use this feature for hiding networked entities, freezing entities in an off-screen chunk, creating a collection of prefab entities loaded and ready to spawn, or something else entirely.

Note that for maximum control and explicitness, only the entities that you directly add disabling components to are disabled. Their children or other related entities are not automatically disabled!

To disable an entity _and_ its children, consider the new [`Commands::insert_recursive`] and [`Commands::remove_recursive`].

[`Disabled`]: https://docs.rs/bevy/0.16/bevy/ecs/entity_disabling/struct.Disabled.html
[`Commands::insert_recursive`]: https://docs.rs/bevy/0.16/bevy/prelude/struct.EntityCommands.html#method.insert_recursive
[`Commands::remove_recursive`]: https://docs.rs/bevy/0.16/bevy/prelude/struct.EntityCommands.html#method.remove_recursive

## Better Location Tracking

{{ <heading_metadata authors={["@SpecificProtagonist", "@chescock"]} prs={[15607, 16047, 16778, 17075, 17602]} /> }}

<!-- Better source location tracking -->

Bevy's unified data model allows introspection and debugging tools to work for the entire engine:
For example, last release's `track_change_detection` feature flag lets you
[automatically track](/news/bevy-0-15/#change-detection-source-location-tracking) which which line of source code inserted or mutated any component or resource.

Now it also tracks which code
- triggered a hook or observer: `HookContext.caller`, `Trigger::caller()`
- sent an event: `EventId.caller`
- spawned or despawned an entity (until the entity index is reused): `EntityRef::spawned_by()`, `Entities::entity_get_spawned_or_despawned_by()`

And as a side effect, this leads to nicer error messages in some cases, such as this nicely improved despawn message:

`Entity 0v1 was despawned by src/main.rs:10:11`

Having outgrown its old name, the feature flag is now called `track_location`.

## Text Shadows

{{ <heading_metadata authors={["@ickshonpe"]} prs={[17559]} /> }}

Bevy UI now supports text shadows:

![image of button with text shadow](text_shadow.jpg)

Just add the [`TextShadow`] component to any [`Text`] entity. [`TextShadow`] can be used to configure the offset and the color of the shadow.

[`TextShadow`]: https://docs.rs/bevy/0.16/bevy/prelude/struct.TextShadow.html
[`Text`]: https://docs.rs/bevy/0.16/bevy/prelude/struct.Text.html

## Input Focus Improvements

{{ <heading_metadata authors={["@viridia", "@alice-i-cecile"]} prs={[15611, 16795]} /> }}

The concept of "input focus" is important for accessibility. Visually or motor-impaired users may have trouble using mice; with focus navigation they can control the UI using only the keyboard. This not only helps disabled users, but power users as well.

The same general idea can be applied to game controllers and other input devices. This is often seen in games where the number of controls on the screen exceeds the number of buttons on the gamepad, or for complex "inventory" pages with a grid of items.

The `bevy_a11y` crate has had a `Focus` resource for a long time, but it was situated in a way that made it hard to use. This has been replaced by an `InputFocus` resource which lives in a new crate, `bevy_input_focus`, and now includes a bunch of helper functions that make it easier to implement widgets that are focus-aware.

This new crate also supports "pluggable focus navigation strategies", of which there are currently two: there's a "tab navigation" strategy, which implements the traditional desktop sequential navigation using the TAB key, and a more "console-like" 2D navigation strategy that uses a hybrid of spatial searching and explicit navigation links.

## Retained `Gizmo`s

{{ <heading_metadata authors={["@tim-blackbird"]} prs={[15473]} /> }}

<!-- Retained `Gizmo`s -->
<!-- https://github.com/bevyengine/bevy/pull/15473 -->
In previous versions of Bevy, gizmos were always rendered in an "immediate mode" style: they were rendered for a single frame before disappearing. This is great for prototyping but also has a performance cost.

With retained gizmos, you can now spawn gizmos that persist, enabling higher performance! For a
static set of lines, we've measured a ~65-80x improvement in performance!

As an example, here's how to spawn a sphere that persists:

```rust
fn setup(
    mut commands: Commands,
    mut gizmo_assets: ResMut<Assets<GizmoAsset>>
) {
    let mut gizmo = GizmoAsset::default();

    // A sphere made out of one million lines!
    gizmo
        .sphere(default(), 1., CRIMSON)
        .resolution(1_000_000 / 3);

    commands.spawn(Gizmo {
        handle: gizmo_assets.add(gizmo),
        ..default()
    });
}
```

The immediate mode `Gizmos` API is still there if you want it though. It is still a great choice for easy debugging.

## Transparent Sprite Picking

{{ <heading_metadata authors={["@vandie"]} prs={[16388]} /> }}

<!-- Add optional transparency passthrough for sprite backend with bevy_picking -->
<!-- https://github.com/bevyengine/bevy/pull/16388 -->
In most cases when working on a game, you don't want clicks over the transparent part of sprites to count as a click on that sprite. This is especially true if you're working on a 2D game with a lot of overlapping sprites which have transparent regions.

In previous versions of bevy, sprite interactions were handled as simple bounding box checks. If your cursor was within the boundary of the sprite's box, it would block interactions with the sprites behind it, even if the area of the sprite is fully transparent.

In **Bevy 0.16**, when interacting with transparent sections of an entity sprite, those interactions will pass through to the entities beneath it. No changes required to your codebase!

By default passthrough will occur for any part of a sprite which has an alpha value of `0.1` or lower. If you wish to revert back to rect-checking for sprites or change the cutoff, you can do so by overwriting `SpritePickingSettings`.

```rust
// Change the alpha value cutoff to 0.2 instead of the default 0.1
app.insert_resource(SpritePickingSettings {
  picking_mode: SpritePickingMode::AlphaThreshold(0.2)
});

// Revert to Bounding Box picking mode
app.insert_resource(SpritePickingSettings {
  picking_mode: SpritePickingMode::BoundingBox
});
```

## Reflection: Function Overloading (Generic & Variadic Functions)

{{ <heading_metadata authors={["@MrGVSV"]} prs={[15074]} /> }}

<!-- bevy_reflect: Function Overloading (Generic & Variadic Functions) -->
<!-- https://github.com/bevyengine/bevy/pull/15074 -->

**Bevy 0.15** added support for reflecting functions to `bevy_reflect`, Bevy's type reflection crate.
This allows Rust functions to be called dynamically with a list of arguments generated at runtime—and safely!

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b
}

let reflect_add: DynamicFunction = add.into_function();

let args = ArgList::new()
  .push_owned(25_i32)
  .push_owned(75_i32);

let result = reflect_add.call(args).unwrap();

let sum = result.unwrap_owned().try_take::<i32>().unwrap();
assert_eq!(sum, 100);
```

However, due to limitations in Rust, it was not possible to make these dynamic functions generic.
This meant individual functions had to be created for all desired monomorphizations
and manually mapped to at runtime.

```rust
fn add<T: Add<Output=T>>(a: T, b: T) -> T {
    a + b
}

let reflect_add_i32 = add::<i32>.into_function();
let reflect_add_u32 = add::<u32>.into_function();
let reflect_add_f32 = add::<f32>.into_function();
// ...
```

While the original Rust limitations still exist, Bevy 0.16 improves the developer experience by adding support for function overloading.
The term "function overloading" might be familiar to developers from other programming languages,
but essentially it means that one function can have multiple argument signatures.

This allows us to simplify the previous example:

```rust
let reflect_add = add::<i32>.into_function()
  .with_overload(add::<u32>)
  .with_overload(add::<f32>);
```

The first `add::<i32>` acts as the base case, with each overload acting on top of it.
When the function is called, the corresponding overload is selected based on the types of the provided arguments.

And by extension of the fact that function overloading allows for multiple argument signatures,
this also means that we can define functions that take a variable number of arguments,
commonly known as "variadic functions."

This allows for some interesting use cases:

```rust
#[derive(Reflect)]
struct Player {
    name: Option<String>,
    health: u32,
}

// Creates a `Player` with one of the following based on the provided arguments:  
// - No name and 100 health  
// - A name and 100 health  
// - No name and custom health  
// - A name and custom health
let create_player = (|| Player {
    name: None,
    health: 100,
  })
  .into_function()
  .with_overload(|name: String| Player {
    name: Some(name),
    health: 100,
  })
  .with_overload(|health: u32| Player {
    name: None,
    health
  })
  .with_overload(|name: String, health: u32| Player {
    name: Some(name),
    health,
  });
```

## Curve Derivatives

{{ <heading_metadata authors={["@mweatherley"]} prs={[16503]} /> }}

`bevy_math` has collected a sizable collection of curves and methods for working with curves, which are useful for everything from animations to color gradients to gameplay logic.

One of the most natural and important things you might want to do with a curve is to inspect its **derivative**:
the rate at which it's changing.
You might even be after its **second derivative**: the rate at which the rate of change is changing.

In **Bevy 0.16** you can now easily calculate these things!

```rust
let points = [
    vec2(-1.0, -20.0),
    vec2(3.0, 2.0),
    vec2(5.0, 3.0),
    vec2(9.0, 8.0),
];

// A cubic spline curve that goes through `points`.
let curve = CubicCardinalSpline::new(0.3, points).to_curve().unwrap();

// Calling `with_derivative` causes derivative output to be included in the output of the curve API.
let curve_with_derivative = curve.with_derivative();

// A `Curve<f32>` that outputs the speed of the original.
let speed_curve = curve_with_derivative.map(|x| x.derivative.norm());
```

We've implemented the required traits for most of our native curve types: splines, lines, and all manner of compound curves.
Curves which accept arbitrary functions are not covered (build your own specialized curve types),
as Rust does not have a first-class notion of a differentiable function!

## `AssetChanged` Query Filter

{{ <heading_metadata authors={["@tychedelia"]} prs={[16810]} /> }}

In Bevy, "assets" are data we want to hold a single copy of, even when used by many entities: things like sounds, images and 3D models.
Assets like `Image` are stored in the [`Assets<Image>`] resource. Entities then have components like [`Sprite`], which hold a [`Handle<Image>`] inside of it, identifying which asset to use for that entity.

While this works great for avoiding storing ten thousands ogre meshes in memory when you're making a tower defense,
this relatively indirect pattern makes it hard to figure out when an asset that you're relying on has changed.
That's because the [`Handle<T>`] could change, pointing to a new asset, or the underlying asset in [`Assets<T>`] could change,
modifying the underlying data.

While querying for `Changed<Sprite>` will catch changes to the handle, it *won't* catch changes to the underlying asset,
resulting in frustrating bugs that are hard to detect, as they only occur when things change in an unusual way.

To solve this, we've added an [`AssetChanged`] query filter, which works for any type (like [`Sprite`])
which implements the new [`AsAssetId`] trait.
Something like `Query<&mut Aabb, With<AssetChanged<Mesh3d>>>` now Just Works™, allowing you to recompute data whenever the underlying asset is changed for any reason.

[`Assets<T>`]: https://docs.rs/bevy/0.16/bevy/asset/struct.Assets.html
[`Sprite`]: https://docs.rs/bevy/0.16/bevy/prelude/struct.Sprite.html
[`Handle<T>`]: https://docs.rs/bevy/0.16/bevy/asset/enum.Handle.html
[`AssetChanged`]: https://docs.rs/bevy/0.16/bevy/asset/prelude/struct.AssetChanged.html
[`AsAssetId`]: https://docs.rs/bevy/0.16/bevy/asset/trait.AsAssetId.html

## Asset Processing no longer auto-generates .meta files

{{ <heading_metadata authors={["@andriyDev"]} prs={[17216]} /> }}

<!-- Stop automatically generating meta files for assets while using asset processing. -->
<!-- https://github.com/bevyengine/bevy/pull/17216 -->

In **Bevy 0.12**, we [introduced Assets V2](/news/bevy-0-12/#bevy-asset-v2). This was a complete rewrite of our asset system, which added features like "asset preprocessing" (the ability to process assets into more efficient forms at development time, prior to deploying your game). This however
created "meta files" for every asset in your project - meaning when you started using asset
preprocessing *at all*, your entire `assets` folder would be filled with these meta files
automatically (even for assets that don't need any preprocessing).

To alleviate this pain, enabling asset preprocessing no longer automatically writes meta files! This
makes it easier to enable asset preprocessing and adopt it gradually.

In addition, we've added `AssetServer::write_default_loader_meta_file_for_path` and
`AssetProcessor::write_default_meta_file_for_path` to allow users to explicitly generate the default
meta files for assets when necessary.

Consider enabling asset processing with:

```rust
app.add_plugins(DefaultPlugins.set(
    AssetPlugin {
        mode: AssetMode::Processed,
        ..default()
    }
));
```

Enabling the `bevy/asset_processor` feature will then process files automatically for you. See
[the asset processing example](https://github.com/bevyengine/bevy/blob/main/examples/asset/processing/asset_processing.rs)
for more details!

## Mesh Tags

{{ <heading_metadata authors={["@tychedelia"]} prs={[17648]} /> }}

Bevy has powerful support for [automatic instancing] for any entities that share the same mesh and material. However, sometimes it can still be useful to reference data that is not the same across all instances of a material. Previously, this required either writing significant amount of custom rendering code or giving up the performance benefits of automatic instancing by creating more materials.

The new `MeshTag` component allows adding a custom `u32` tag to mesh-material entities that can be referenced in the vertex shader for a material. In combination with storage textures or the [`ShaderStorageBuffer` asset] added in **Bevy 0.15**, this provides a flexible new mechanism to access external data on a per-instance basis or otherwise tag your mesh instances.

Spawn a mesh tag with a mesh-material entity: 
```rust
commands.spawn((
    // Clone a mesh and material handle to enable automatic instancing 
    Mesh3d(mesh_handle.clone()),
    MeshMaterial3d(material_handle.clone()),
    // The mesh tag can be any `u32` that is meaningful to your application, like
    // a particular variant of an enum or an index into some external data 
    MeshTag(1234), 
));
```

Lookup the tag in a vertex shader:
```wgsl
#import bevy_pbr::mesh_functions

@vertex
fn vertex(vertex: Vertex) -> VertexOutput {
    // Lookup the tag for the given mesh
    let tag = mesh_functions::get_tag(vertex.instance_index);

    // Index into a storage buffer, read a storage texture texel, etc...
}

```

[automatic instancing]: https://bevy.org/examples/shaders/automatic-instancing/
[`ShaderStorageBuffer` asset]: https://bevy.org/news/bevy-0-15/#shader-storage-buffer-asset

## GPU Timestamps in Traces

{{ <heading_metadata authors={["@JMS55"]} prs={[18490]} /> }}

If you wish to optimize something, you must first measure and understand it.
When looking at the performance of applications, [tracy] is our tool of choice.
It gives us a clear understanding of how long work takes, when it happens relative to other work each frame,
and how various threads are used.
Read our [profiling docs] to get started!

But until now, it's had a critical limitation: work done on the GPU wasn't shown,
forcing devs to pull up dedicated GPU-focused tools (like [NSight] or [RenderDoc]) and struggle to piece together an intuition for how it all fits together.

In 0.16, we've connected the [rendering diagnostics added in Bevy 0.14] to [tracy], creating a cohesive picture of
all of the work that's being done in a Bevy application in a single convenient place.

That said, we've only instrumented a few of our passes so far.
While we will improve this in the future, you will need to add spans to your own custom rendering code,
and specialized GPU diagnostic tools will always be more powerful: capturing all GPU-related work done,
and providing more detailed information.

Special thanks to [@wumpf] for trailblazing this work in the excellent [wgpu-profiler] tool, and demonstrating how to wire [wgpu] and [tracy] together.

[tracy]: https://github.com/wolfpld/tracy
[profiling docs]: https://github.com/bevyengine/bevy/blob/main/docs/profiling.md
[NSight]: https://developer.nvidia.com/nsight-systems
[RenderDoc]: https://renderdoc.org/
[@wumpf]: https://github.com/Wumpf
[wgpu-profiler]: https://github.com/Wumpf/wgpu-profiler
[rendering diagnostics added in Bevy 0.14]: https://bevy.org/news/bevy-0-14/#tools-for-profiling-gpu-performance

## Trait Tags on docs.rs

{{ <heading_metadata authors={["@SpecificProtagonist", "@Jondolf"]} prs={[17758]} /> }}

<!-- Trait tags on docs.rs -->
<!-- https://github.com/bevyengine/bevy/pull/17758 -->

Bevy provides several core traits that define how a type is used. For example, to attach data to an entity it must implement `Component`. When reading the Rust API docs, determining whether a type is a `Component` (or some other core Bevy trait) requires scrolling through all of the docs until you find the relevant trait. But in **Bevy 0.16**, on docs.rs Bevy now displays labels indicating which core Bevy traits a type implements:

![Rustdoc showing a "Component" label below "Camera" type](trait-tags.jpg)

This happens for the traits
`Plugin` / `PluginGroup`,
`Component`,
`Resource`,
`Asset`,
`Event`,
`ScheduleLabel`,
`SystemSet`,
`SystemParam`,
`Relationship` and
`RelationshipTarget`.

This is done for now through [javascript](https://github.com/bevyengine/bevy/tree/release-0.16.0/docs-rs). This should be useful for other Rust frameworks than Bevy, and we'll work with the rustdoc team on how to make this built in and more generic. Get in touch if you're interested so we can start a specification!

## What's Next?

The features above may be great, but what else does Bevy have in flight?
Peering deep into the mists of time (predictions are _extra_ hard when your team is almost all volunteers!), we can see some exciting work taking shape:

- **A revamped observers API:** Observers are incredibly popular, but come with some weird quirks. We're looking to smooth those out, and make them the easiest way to write one-off logic for UI.
- **Resources-as-entities:** Sure would be nice if hooks, observers, relations and more worked with resources. Rather than duplicating all of the code, we'd like to [make them components on singleton entities](https://github.com/bevyengine/bevy/pull/17485) under the hood.
- **A .bsn file format and bsn! macro:** With the foundations laid (required components, improved spawning and relations!), it's time to build out the terse and robust Bevy-native scene format (and matching macro) described in [bevy#14437](https://github.com/bevyengine/bevy/discussions/14437).
- **Light textures:** Also known as "light cookies", these are great for everything from dappled sunlight to shadow boxing.
- **NVIDIA Deep Learning Super Sampling:** DLSS is a neural-net powered approach to temporal anti-aliasing and upscaling for NVIDIA RTX GPUs. We're working on integrating DLSS into Bevy to provide a cheaper and higher quality anti-aliasing solution than Bevy's current TAA (on supported platforms).
- **Unified volumetrics system:** God rays, fogs, cascading shadow maps, and atmospheric scattering: there's a huge number of rendering features that fundamentally care about the optical properties of volumes of open air (or water!). We're hoping to unify and extend these features for easier to use, more beautiful physically-based rendering.
- **Ray-tracing foundations:** Hardware-accelerated ray-tracing is all the rage, and with `wgpu`'s help we're ready to start making the first steps, walking towards a world of dynamic ray-traced global illumination.
- **More game-focused examples:** New users continue to flock to Bevy, and need up-to-date learning materials. Our API-focused approach to examples isn't enough: we need to start demonstrating how to use Bevy to do common game dev tasks like making an inventory, saving user preferences or placing structures on a map.

{{ <support_bevy /> }}

{{ <contributors version="0.16" /> }}

For those interested in a complete changelog, you can see the entire log (and linked pull requests) via the [relevant commit history](https://github.com/bevyengine/bevy/compare/v0.15.0...v0.16.0-rc.5).
