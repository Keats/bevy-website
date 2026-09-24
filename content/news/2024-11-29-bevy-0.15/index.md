+++
title = "Bevy 0.15"
date = 2024-11-29
[extra]
image = "cover.png"
show_image = true
image_subtitle = "A snake statue in volumetric fog illuminated by volumetric lighting"
image_subtitle_link = "https://sketchfab.com/3d-models/snake-statue-794b77a3e4654a669cf259d20dc89ec7"
+++

Thanks to **294** contributors, **1217** pull requests, community reviewers, and our [**generous donors**](/donate), we're happy to announce the **Bevy 0.15** release on [crates.io](https://crates.io/crates/bevy)!

For those who don't know, Bevy is a refreshingly simple data-driven game engine built in Rust. You can check out our [Quick Start Guide](/learn/quick-start) to try it today. It's free and open source forever! You can grab the full [source code](https://github.com/bevyengine/bevy) on GitHub. Check out [Bevy Assets](https://bevy.org/assets) for a collection of community-developed plugins, games, and learning resources.

To update an existing Bevy App or Plugin to **Bevy 0.15**, check out our [0.14 to 0.15 Migration Guide](/learn/migration-guides/0-14-to-0-15/).

Since our last release a few months ago we've added a _ton_ of new features, bug fixes, and quality of life tweaks, but here are some of the highlights:

- **Required Components**: A rethink of how spawning entities works that significantly improves the Bevy user experience
- **Entity Picking / Selection**: A modular system for selecting entities across contexts
- **Animation Improvements**: generalized entity animation, animation masks, additive blending, and animation events
- **Curves**: a new `Curve` trait, cyclic splines, common easing functions, color gradient curves
- **Reflection Improvements**: Function reflection, unique reflect, remote type reflection
- **Bevy Remote Protocol (BRP)**: A new protocol that allows external clients (such as editors) to interact with running Bevy games
- **Visibility Bitmask Ambient Occlusion (VBAO)**: An improved GTAO algorithm that improves ambient occlusion quality
- **Chromatic Aberration**: A new post processing effect that simulates lenses that fail to focus light to a single point
- **Volumetric Fog Improvements**: "Fog volumes" that define where volumetric fog is rendered (and what form it takes), along with Point Lights and Spotlight compatibility
- **Order Independent Transparency**: A new opt-in transparency algorithm that improves the stability / quality of transparent objects as their distance from the camera changes
- **Improved Text Rendering**: We've switched to Cosmic Text for our text rendering, which significantly improves our ability to render text, especially for non-Latin-based languages that require font shaping and bidirectional text
- **Gamepads as Entities**: Gamepads are now represented as entities, making them much easier to interact with
- **UI Box Shadows**: Bevy UI nodes can now render configurable box shadows

Bevy 0.15 was prepared using our new **release candidate** process to help ensure that you can upgrade right away with peace of mind. We worked closely with both plugin authors and ordinary users to catch critical bugs, polish new features, and refine the migration guide. For each release candidate, we prepared fixes, [shipped a new release candidate on crates.io](https://crates.io/crates/bevy/versions?sort=date), let core ecosystem crates update, and listened closely for show-stopping problems. A huge thanks to [everyone who helped out](https://discord.com/channels/691052431525675048/1295069829740499015)! These efforts are a vital step towards making Bevy something that teams large and small can trust to work reliably.

<!-- more -->

## Required Components

{{ <heading_metadata authors={["@cart", "@Jondolf"]} prs={[14791, 15458, 15269]} /> }}

![Graph image of sprite component requiring transform component](required_component.svg)

First: buckle up because **Required Components** is one of the most profound improvements to the Bevy API surface since Bevy was first released.

Since Bevy's creation, `Bundle` has been our abstraction for spawning an entity of a given "type". A `Bundle` is just a Rust type, where each field is a `Component`:

```rust
#[derive(Bundle)]
struct PlayerBundle {
    player: Player,
    team: Team,
    sprite: Sprite,
    transform: Transform,
    global_transform: GlobalTransform,
    visibility: Visibility,
    inherited_visibility: InheritedVisibility,
    view_visibility: ViewVisibility,
}
```

Then whenever a new player needs to be spawned, developers would initialize and insert a `PlayerBundle` on an entity:

```rust
commands.spawn(PlayerBundle {
    player: Player { 
        name: "hello".into(),
        ..default()
    },
    team: Team::Blue,
    ..default()
});
```

This inserts all of the components in `PlayerBundle`, including the ones not explicitly being set. The `Bundle` concept is functional (it has gotten us this far), but it is also far from ideal:

1. It is an entirely new set of APIs that developers need to learn. Someone that wants to spawn a `Player` entity needs to know that `PlayerBundle` exists.
2. Bundle APIs don't exist at runtime after insertion ... they are an additional spawn-only concept that developers need to think about. You don't write `PlayerBundle` behaviors. You write `Player` behaviors.
3. The `Player` component _needs_ the components in `PlayerBundle` to function as a `Player`. Spawning `Player` on its own is possible, and it likely (depending on the implementation) wouldn't function as intended.
4. Bundles are always "flat" (by convention). The person defining the `Player` component needs to define _all of the component dependencies_. `Sprite` needs `Transform` and `Visibility`, `Transform` needs `GlobalTransform`, `Visibility` needs `InheritedVisibility` and `ViewVisibility`. This lack of "dependency inheritance" makes defining bundles much harder and error prone than it needs to be. It requires consumers of APIs to be intimately aware of what amounts to implementation details. And when these details change, _the developer of the `Bundle` needs to be aware and update the `Bundle` accordingly_. Nested bundles are supported, but they are a _pain_ for users to work with and we have disallowed them in upstream Bevy bundles for a while now.
5. `PlayerBundle` is effectively defined by the needs of the `Player` component, but when spawning it is possible to _never mention the `Player` symbol_. Ex: `commands.spawn(PlayerBundle::default())`. This is odd given that `Player` is the "driving concept".
6. Bundles introduce significant "stutter" to the API. Notice the `player: Player` and `team: Team` in the example above.
7. Bundles introduce additional (arguably excessive) nesting and `..default()` usage.

Every one of these points has a sizable impact on what it feels like to use Bevy on a day-to-day basis. In **Bevy 0.15** we've landed **Required Components**, which solves these problems by fundamentally rethinking how this all works.

**Required Components** are the first step in our [Next Generation Scene / UI](https://github.com/bevyengine/bevy/discussions/14437) effort, which aims to make Bevy a best-in-class app / scene / UI development framework. **Required Components** stand on their own as a direct improvement to Bevy developers' lives, but they also help set the stage for making Bevy's upcoming next generation scene system (and the upcoming Bevy Editor) something truly special.

### What are they?

**Required Components** enable developers to define which components a given component needs:

```rust
#[derive(Component, Default)]
#[require(Team, Sprite)]
struct Player {
    name: String,
}
```

When the `Player` component is inserted, its **Required Components** _and the components required by those components_ are automatically inserted as well!

```rust
commands.spawn(Player::default());
```

The code above automatically inserts `Team` and `Sprite`. `Sprite` requires `Transform` and `Visibility`, so those are automatically inserted as well. Likewise `Transform` requires `GlobalTransform` and `Visibility` requires `InheritedVisibility` and `ViewVisibility`.

This code produces the same result as the `PlayerBundle` code in the previous section:

```rust
commands.spawn((
    Player {
        name: "hello".into(),
        ..default()
    },
    Team::Blue,
))
```

Much better right? The `Player` type is easier and less error prone to define, and spawning it takes less typing and is easier to read.

### Efficiency

We've implemented **Required Components** in a way that makes them effectively "free":

1. Required Components are only initialized and inserted if the caller did not insert them manually. No redundancy!
2. Required Components are inserted alongside the normal components, meaning (for you ECS nerds out there) there are no additional archetype changes or table moves. From this perspective, the Required Components version of the `Player` example is identical to the `PlayerBundle` approach, which manually defines all of the components up front.
3. Required Components are cached on the archetype graph, meaning computing what required components are necessary for a given type of insert only happens once.

### Component Initialization

By default, **Required Components** will use the `Default` impl for the component (and fail to compile if one does not exist):

```rust
#[derive(Component)]
#[require(Team)] // Team::Red is the default value
struct Player {
    name: String,
}

#[derive(Component, Default)]
enum Team {
    #[default]
    Red,
    Blue,
}
```

This can be overridden by passing in a function that returns the component:

```rust
#[derive(Component)]
#[require(Team(blue_team))]
struct Player {
    name: String,
}

fn blue_team() -> Team {
    Team::Blue
}
```

To save space, you can also pass a closure to the require directly:

```rust
#[derive(Component)]
#[require(Team(|| Team::Blue))]
struct Player {
    name: String,
}
```

### Isn't this a bit like inheritance?

**Required Components** _can_ be considered a form of inheritance. But it is notably _not_ traditional object-oriented inheritance. Instead it is "inheritance by composition". A `Button` widget can (and should) require `Node` to make it a "UI node". In a way, a `Button` "is a" `Node` like it would be in traditional inheritance. But unlike traditional inheritance:

1. It is expressed as a "has a" relationship, not an "is a" relationship.
2. `Button` and `Node` are still two entirely separate types (with their own data), which you query for separately in the ECS.
3. A `Button` can require more components in _addition_ to `Node`. You aren't limited to "straight line" standard object-oriented inheritance. Composition is still the dominating pattern.
4. You don't _need_ to require components to add them. You can still tack on whatever additional components you want during spawn to add behaviors in the normal "composition style".

### What is happening to Bundles?

The `Bundle` trait will continue to exist, and it is still the fundamental building block for insert APIs (tuples of components still implement `Bundle`). Developers are still free to define their own custom bundles using the `Bundle` derive. Bundles play nicely with **Required Components**, so you can use them with each other.

That being said, as of Bevy **0.15** we have deprecated all built-in bundles like `SpriteBundle`, `NodeBundle`, `PbrBundle`, etc. in favor of **Required Components**. In general, **Required Components** are now the preferred / idiomatic approach. We encourage Bevy plugin and app developers to port their bundles over to **Required Components**.

### Porting Bevy to Required Components

As mentioned above, _all_ built-in Bevy bundles have been deprecated in favor of **Required Components**. We've also made API changes to take advantage of this new paradigm. This does mean breakage in a few places, but the changes are so nice that we think people won't complain too much :)

In general, we are moving in the direction specified by our [Next Generation Scene / UI](https://github.com/bevyengine/bevy/discussions/14437) document. Some general design guidelines:

1. When spawning an entity, generally there should be a "driving concept" component.  When implementing a new entity type / behavior, give it a concept name ... that is the name of your "driving component" (ex: the "player" concept is a `Player` component). That component should require any additional components necessary to perform its functionality.
2. People should think directly in terms of components and their fields when spawning. Prefer using component fields directly on the "concept component" as the "public API" for the feature.
3. Prefer simple APIs / don't over-componentize. By default, if you need to attach new properties to a concept, just add them as fields to that concept's component. Only break out new components / concepts when you have a good reason, and that reason is motivated by user experience or performance (and weight user experience highly). If a given "concept" (ex: a `Sprite`) is broken up into 10 components, that is _very_ hard for users to reason about and work with.
4. Instead of using Asset handles directly as components, define new components that hold the necessary handles. Raw asset handles as components were problematic for a variety of reasons (a big one is that you can't define context-specific **Required Components** for them), so we have removed the `Component` implementation for `Handle<T>` to encourage (well ... force) people to adopt this pattern.

#### UI

Bevy UI has benefitted tremendously from **Required Components**. UI nodes require a variety of components to function, and now all of those requirements are consolidated on `Node`. Defining a new UI node type is now as simple as adding `#[require(Node)]` to your component.

```rust
#[derive(Component)]
#[require(Node)]
struct MyNode;

commands.spawn(MyNode);
```

The `Style` component fields have been moved into `Node`. `Style` was never a comprehensive "style sheet", but rather just a collection of properties shared by all UI nodes. A "true" ECS style system would style properties _across_ components (`Node`, `Button`, etc), and we [do have plans to build a true style system](https://github.com/bevyengine/bevy/discussions/14437). All "computed" node properties (such as the size of the node after it has been laid out) have been moved to the `ComputedNode` component.

This change has made spawning UI nodes in Bevy _much_ cleaner and clearer:

```rust
commands.spawn(Node {
    width: Val::Px(100.),
    ..default()
});
```

Compare that to what it was before!

```rust
commands.spawn(NodeBundle {
    style: Style {
        width: Val::Px(100.),
        ..default()
    },
    ..default()
})
```

UI components like `Button`, `ImageNode` (previously `UiImage`), and `Text` now require `Node`. Notably, `Text` has been reworked to be easier to use and more component driven (we'll cover this more in the next section):

```rust
commands.spawn(Text::new("Hello there!"));
```

`MaterialNode<M: UiMaterial>` is now a proper component for "UI material shaders", and it also requires `Node`:

```rust
commands.spawn(MaterialNode(my_material));
```

#### Text

Bevy's Text API has been reworked to be simpler and more component driven. There are still two primary text components: `Text` (the UI text component) and `Text2d` (the world-space 2D text component).

The first thing that has changed is that these primary components are literally _just_ a `String` newtype:

```rust
commands.spawn(Text("hello".to_string()))
commands.spawn(Text::new("hello"))
commands.spawn(Text2d("hello".to_string()))
commands.spawn(Text2d::new("hello"))
```

Spawn one of these components, and you have text! Both of these components now require the following components:

* `TextFont`: configures the font / size
* `TextColor`: configures the color
* `TextLayout`: configures how the text is laid out.

`Text`, which is the UI component, also requires `Node` because it _is_ a node. Similarly, `Text2d` requires a `Transform` because it is positioned in world space.

Both `Text` and `Text2d` are a standalone "block" of text. These top level text components also contribute a single "span" of text, which is added to the "block". If you need "rich text" with multiple colors / fonts / sizes, you can add `TextSpan` entities as children of either `Text` or `Text2d`. `TextSpans` use the same `TextFont` / `TextLayout` components to configure text. Each `TextSpan` will contribute its span to its parent text:

```rust
// The `Text` UI node will render "hello world!", where "hello" is red and "world!" is blue
commands.spawn(Text::default())
    .with_child((
        TextSpan::new("hello"),
        TextColor::from(RED),
    ))
    .with_child((
        TextSpan::new(" world!"),
        TextColor::from(BLUE),
    ));
```

This produces the exact same output, but uses the "default" span on the top-level `Text` component:

```rust
commands.spawn((
    Text::new("hello"),
    TextColor::from(RED),
))
.with_child((
    TextSpan::new(" world!"),
    TextColor::from(BLUE),
));
```

This "entity driven" approach to text spans replaces the "internal span array" approach used in previous Bevy versions. This yields significant benefits. First, it lets you use the normal Bevy ECS tools, such as marker components and queries, to mark a text span and access it directly. This is much easier (and more resilient) than using indices in an array, which are hard to guess and unstable as the array contents change:

```rust
#[derive(Component)]
struct NameText;

commands.spawn(Text::new("Name: "))
    .with_child((
        TextSpan::new("Unknown"),
        NameText, 
    ));

fn set_name(mut names: Query<&mut TextSpan, With<NameText>>) {
    names.single_mut().0 = "George".to_string();
}
```

Text spans as entities play nicer with Bevy Scenes (including the upcoming [Next Generation Scene / UI](https://github.com/bevyengine/bevy/discussions/14437) system), and allow it to integrate nicely with existing tools like entity inspectors, animation systems, timers, etc.

#### Sprites

Sprites are largely unchanged. In addition to the **Required Components** port (`Sprite` now requires `Transform` and `Visibility`), we've also done some component consolidation. The `TextureAtlas` component is now an optional `Sprite::texture_atlas` field. Likewise the `ImageScaleMode` component is now a `Sprite::image_mode` field. Spawning sprites is now super simple!

```rust
commands.spawn(Sprite {
    image: assets.load("player.png"),
    ..default()
});
```

#### Transforms

`Transform` now requires `GlobalTransform`. If you want your entity to have "hierarchical transforms", require `Transform` (and it will add `GlobalTransform`). If you just want your entity to have a "flat" global transform, require `GlobalTransform`.

Most Bevy components that are intended to exist in world space now require `Transform`.

#### Visibility

The `Visibility` component now requires `InheritedVisibility` and `ViewVisibility`, meaning that you can now just require `Visibility` if you want your entity to be visible. Bevy's built-in "visible" components, such as `Sprite`, require `Visibility`.

#### Cameras

The `Camera2d` and `Camera3d` components now each require `Camera`. `Camera` requires the various camera components (`Frustum`, `Transform`, etc.). This means that you can spawn a 2D or 3D camera like this:

```rust
commands.spawn(Camera2d::default());
commands.spawn(Camera3d::default());
```

`Camera2d` and `Camera3d` also require the components that set the relevant default render graphs and enable the default render features relevant to the 2D and 3D contexts (respectively).

You can of course explicitly set the values of the other components:

```rust
commands.spawn((
    Camera3d::default(),
    Camera {
        hdr: true,
        ..default()
    },
    Transform {
        translation: Vec3::new(1.0, 2.0, 3.0),
        ..default()
    },
));
```

Bevy has a number of components that enable "camera render features": `MotionBlur`, `TemporalAntiAliasing`, `ScreenSpaceAmbientOcclusion`, and `ScreenSpaceReflections`. Some of these camera features depend on _other_ camera feature components to function. These dependencies are now expressed and enforced using **Required Components**. For example, `MotionBlur` now requires `DepthPrepass` and `MotionVectorPrepass`. This makes enabling camera features much easier!

```rust
commands.spawn((
    Camera3d::default(),
    MotionBlur,
))
```

#### Meshes

The old mesh approach relied on adding `Handle<Mesh>` and `Handle<M: Material>` components directly (via `PbrBundle` and `MaterialMeshBundle`), neither of which were compatible with required components.

In **Bevy 0.15** you use `Mesh3d` and `MeshMaterial3d<M: Material>` to render a mesh in 3D:

```rust
commands.spawn((
    Mesh3d(mesh),
    MeshMaterial3d(material),
));
```

`Mesh3d` requires `Transform` and `Visibility`.

There are also 2D equivalents:

```rust
commands.spawn((
    Mesh2d(mesh),
    MeshMaterial2d(material),
));
```

#### Meshlets

Bevy's "virtual geometry" implementation (similar to Nanite), has also been ported. It uses the same pattern as `Mesh3d` and `Mesh2d`:

```rust
commands.spawn((
    MeshletMesh3d(mesh),
    MeshMaterial3d(material),
));
```

#### Lights

The light port involved no major changes to the component structure. All of the spatial light types (`PointLight`, `DirectionalLight`, `SpotLight`) now require `Transform` and `Visibility`, and each light component requires the relevant light-specific configuration components (ex: `PointLight` requires `CubemapFrusta` and `CubemapVisibleEntities`).

Spawning a light of a given type is now as simple as:

```rust
commands.spawn(PointLight {
    intensity: 1000.0,
    ..default()
});
```

The `LightProbe` component now also requires `Transform` and `Visibility`.

#### Volumetric Fog

The `FogVolume` component now requires `Transform` and `Visibility`, meaning you can now add volumetric fog like this:

```rust
commands.spawn(FogVolume {
    density_factor: 0.2,
    ..default()
});
```

#### Scenes

Scenes previously used raw `Handle<Scene>` components, spawned via `SceneBundle`. **Bevy 0.15** introduces the `SceneRoot` component, which wraps the scene handle and requires `Transform` and `Visibility`:

```rust
commands.spawn(SceneRoot(some_scene));
```

Likewise, there is now `DynamicSceneRoot`, which is exactly like `SceneRoot`, but it wraps `Handle<DynamicScene>` instead of `Handle<Scene>`.

#### Audio

Audio also used a raw `Handle<AudioSource>` spawned via an  `AudioBundle`. We've added a new `AudioPlayer` component, which will trigger audio playback when spawned:

```rust
command.spawn(AudioPlayer(assets.load("music.mp3")));
```

`AudioPlayer` requires the `PlaybackSettings` component.

Non-standard audio from arbitrary `Decodable` trait impls can use the `AudioSourcePlayer` component, which also requires `PlaybackSettings`.  

### IDE Integration

**Required Components** play nicely with Rust Analyzer. You can "go to definition" / press `F12` on required components to navigate to where they are defined in code.

### Runtime Required Components

In some cases, developers without direct control over a component might want to add _additional_ **Required Components** on top of the ones provided directly on the type via `#[require(Thing)]`. This is supported!

```rust
// Make `Bird` require `Wings` with a `Default` constructor.
app.register_required_components::<Bird, Wings>();

// Make `Wings` require `FlapSpeed` with a custom constructor.
app.register_required_components_with::<Wings, FlapSpeed>(|| FlapSpeed::from_duration(1.0 / 80.0));
```

Note that only _adding_ Required Components is allowed. Removing Required Components from a type you do not own is explicitly not supported, as that could invalidate assumptions made upstream.

In general, this is intended to be used in very specific, targeted contexts, such as a physics plugin adding additional metadata to a core type it does not control. Adding a new component requirement could change the performance characteristics of the app or break it in unexpected ways. When in doubt, don't do it!

## Chromatic Aberration

{{ <heading_metadata authors={["@pcwalton"]} prs={[13695]} /> }}

<!-- Start a built-in postprocessing stack, and implement chromatic aberration in it. -->
<!-- https://github.com/bevyengine/bevy/pull/13695 -->

We've added [chromatic aberration](https://en.wikipedia.org/wiki/Chromatic_aberration), which is a common postprocessing effect that simulates lenses that fail to focus all colors of light to a single point. It's often used for impact effects and/or horror games. Our implementation uses the technique from Inside (Gjøl & Svendsen 2016), which allows the developer to customize the particular color pattern to achieve different effects.

![chromatic aberration](chromatic_aberration.png)

To use it, add the [`ChromaticAberration`](https://docs.rs/bevy/0.15/bevy/core_pipeline/post_process/struct.ChromaticAberration.html) component to your camera:

```rust
commands.spawn((Camera3d::default(), ChromaticAberration));
```

## Visibility Bitmask Ambient Occlusion (VBAO)

{{ <heading_metadata authors={["@dragostis"]} prs={[13454]} /> }}

**Bevy 0.15** introduces a new Screen Space Ambient Occlusion (SSAO) algorithm: [Visibility Bitmask Ambient Occlusion](https://arxiv.org/abs/2301.11376) (VBAO). VBAO builds on GTAO by adding a bitmask that allows multiple "sectors" of a considered half circle to be occluded, instead of just one angle. This improves the accuracy of the technique and is particularly important on thin geometry (like the chair legs in the scene below):

{{ <compare_slider left_title="GTAO"
    left_image="gtao.jpg"
    right_title="VBAO"
    right_image="vbao.jpg" /> }}

VBAO produces a significant enough quality improvement that we have replaced our old GTAO algorithm entirely. Just add the existing [`ScreenSpaceAmbientOcclusion`] component to your camera to enable it.

[`ScreenSpaceAmbientOcclusion`]: https://docs.rs/bevy/0.15/bevy/pbr/struct.ScreenSpaceAmbientOcclusion.html

## Volumetric Fog Support for Point Lights and Spotlights

{{ <heading_metadata authors={["@Soulghost"]} prs={[15361]} /> }}

Volumetric fog was [introduced in Bevy 0.14](https://bevy.org/news/bevy-0-14/#volumetric-fog-and-volumetric-lighting-light-shafts-god-rays). Initially, only directional lights could interact with it. In Bevy 0.15, point lights and spot lights work with it too:

![volumetric fog](volumetric_fog.jpg)

To add volumetric fog to your scene, add [VolumetricFog](https://docs.rs/bevy/0.15/bevy/pbr/struct.VolumetricFog.html) to the camera, and add [VolumetricLight](https://docs.rs/bevy/0.15/bevy/pbr/struct.VolumetricLight.html) to directional light, point light, or spot light that you wish to be volumetric.

```rust
// Add VolumetricFog to the camera.
commands
    .spawn((
        Camera3d::default(),
        Camera {
            hdr: true,
            ..default()
        },
    ))
    .insert(VolumetricFog {
        // This value is explicitly set to 0 since we have no environment map light.
        ambient_intensity: 0.0,
        ..default()
    });

// Add VolumetricLight to point light.
commands.spawn((
    PointLight {
        shadows_enabled: true,
        range: 150.0,
        color: RED.into(),
        intensity: 1000.0,
        ..default()
    },
    VolumetricLight,
    Transform::from_xyz(-0.4, 1.9, 1.0),
));

// Add VolumetricLight to spot light.
commands.spawn((
    SpotLight {
        intensity: 5000.0, // lumens
        color: Color::WHITE,
        shadows_enabled: true,
        inner_angle: 0.76,
        outer_angle: 0.94,
        ..default()
    },
    VolumetricLight,
    Transform::from_xyz(-1.8, 3.9, -2.7).looking_at(Vec3::ZERO, Vec3::Y),
));
```

## Fog Volumes

{{ <heading_metadata authors={["@pcwalton", "@jirisvd"]} prs={[14099, 14868]} /> }}

<!-- Allow volumetric fog to be localized to specific, optionally voxelized, regions. -->
<!-- https://github.com/bevyengine/bevy/pull/14099 -->

**Bevy 0.15** adds the concept of "fog volumes". These are entities with the [`FogVolume`] component, which defines a bounding box for fog, which can be scaled and positioned to define where the fog will be rendered.

A camera with the [`VolumetricFog`] component will render any [`FogVolume`] entities in its view. Fog volumes can also define a density texture, which is a 3D texture of voxels that specify the density of the fog at each point:

![fog volume](fog_volume.png)

[`FogVolume`] has a `density_texture_offset`, which allows the 3D texture to be "scrolled". This allows effects such as clouds "passing through" the volume:

<video controls><source src="scrolling_fog.mp4" type="video/mp4"/></video>

[`FogVolume`]: https://docs.rs/bevy/0.15.0/bevy/pbr/struct.FogVolume.html
[`VolumetricFog`]: https://docs.rs/bevy/0.15.0/bevy/pbr/struct.VolumetricFog.html

## Order Independent Transparency

{{ <heading_metadata authors={["@IceSentry"]} prs={[14876]} /> }}

<!-- Add Order Independent Transparency -->
<!-- https://github.com/bevyengine/bevy/pull/14876 -->

<video controls autoplay loop muted><source src="oit-demo.mp4" type="video/mp4"/></video>

Before this feature, Bevy only used alpha blending to render transparent meshes. We now have the option to use Order Independent Transparency when rendering transparent meshes. Instead of only sorting the mesh, this will sort every pixel that contributes to a transparent triangle. This is useful if you have a lot of transparent layers in your scene.

The implementation is currently pretty simple and uses a lot of GPU memory, but it should always render perfectly accurate transparency as long as you have configured enough layers.

This feature is still a work in progress and we will keep working on improving it.

This feature was contributed to Bevy by Foresight Spatial Labs. It is based on an internal implementation they use in their applications.

## User-Friendly CPU Drawing

{{ <heading_metadata authors={["@inodentry"]} prs={[10392]} /> }}

There are many situations where you might want to just set the color of pixels
from CPU code.  Procedural assets, certain art styles, or simply because
it is easier. No need to bother with shaders and materials, when you just want
to change a few specific pixels!

In previous versions of Bevy, this was difficult and tedious. Bevy gives you
access to the raw data bytes of an [`Image`], but you had to compute the byte
offset corresponding to your desired pixel coordinate, make sure to encode your
bytes with respect to the [`TextureFormat`], etc. Very low level!

In Bevy 0.15, there are now user-friendly APIs for reading and writing the
colors of pixels in an [`Image`]. The tricky low-level details are dealt with
for you! You can even use `bevy_color`'s fancy color space APIs!

```rust
fn my_system(mut images: ResMut<Assets<Image>>, mut commands: Commands) {
    // Create a new image.
    let mut image = Image::new_fill(
        // 64x64 size
        Extent3d {
            width: 64,
            height: 64,
            depth_or_array_layers: 1,
        },
        TextureDimension::D2,
        &Srgba::WHITE.to_u8_array(),
        TextureFormat::Rgba8UnormSrgb,
        RenderAssetUsages::all(),
    );

    // This is new:

    // Make the pixel at x: 23, y: 32 magenta
    image.set_color_at(23, 32, Color::srgb(1.0, 0.0, 1.0))
        .expect("Error writing color");

    // Set the pixel at 10,10 to a color specified using the Oklch color space:
    image.set_color_at(10, 10, Color::oklch(0.3, 0.2, 0.5))
        .expect("Error writing color");

    // read the bytes of the pixel we just wrote:
    let bytes = image.pixel_bytes(UVec3::new(10, 10, 0)).unwrap();

    // read the (approximate) color back (as sRGB):
    let color = image.get_color_at(10, 10);

    // We could add our new image to Bevy's assets
    // and spawn a sprite to display it:
    commands.spawn(Sprite {
        image: images.add(image),
        ..default()
    });
}
```

Note: The [`Color`]-based methods are lossy. They have to convert to/from the
[`Image`]'s [`TextureFormat`]. If you read back the color you wrote, it will be slightly different.

[`Image`]: https://docs.rs/bevy/0.15/bevy/prelude/struct.Image.html
[`TextureFormat`]: https://docs.rs/bevy/0.15/bevy/render/render_resource/enum.TextureFormat.html
[`Color`]: https://docs.rs/bevy/0.15/bevy/color/enum.Color.html

## Entity Picking / Selection

{{ <heading_metadata authors={["@aevyrie", "@NthTensor", "@TotalKrill", "@jnhyatt", "@Jondolf"]} prs={[13677, 14686, 14695, 14757, 15800]} /> }}

<!-- Add mesh picking backend and `MeshRayCast` system parameter -->
<!-- https://github.com/bevyengine/bevy/pull/15800 -->

![A collection of geometric shapes, with a pointer showing a point on a hovered mesh. The indicator is perpendicular to the surface.](mesh_picking.png)

Being able to click on objects to select them is a vital and seemingly simple task in any game.
Since 2020, doing this in Bevy has largely meant pulling in `@aevyrie`'s beloved ecosystem crate, [`bevy_mod_picking`] and its simple raycasting companion [`bevy_mod_raycast`].

Over the years, this crate has been refined and battle-tested, both by [Foresight Spatial Labs] (a CAD-creating, Bevy-using company where Aevyrie works) and
the broader open source community of game developers that have used it for everything from first-person-shooters to point-and-click adventures.
Bevy is thrilled to have had the chance to work with the team behind [`bevy_mod_picking`] and have adopted the project wholesale into Bevy itself.
Integrating a large project is a ton of work, and we're incredibly grateful to the contributors who have made `bevy_picking` a stable, first-class feature of the engine.

The new `bevy_picking` crate follows the existing modular architecture closely:

1. Inputs are gathered from mouse, touch and pen devices. Each pointing device (humans are equipped with 10 by default) gets a screen-space [`PointerLocation`].
2. Each modular [backend] performs the domain-specific work (like raycasting) of figuring out how these pointer locations map to [`PointerHits`] on objects that they're watching.
3. The hit information from each backend is combined and sorted to produce a coherent [`HoverMap`], which lists which entities each pointer is hovering over.
4. High level events (both ordinary events and observers!) are emitted for each hovered entity, capturing complex behavior such as clicking, dragging or releasing various objects.

In Bevy 0.15, we're shipping three first-party picking backends for UI, sprites, and meshes. Each of these comes with its own caveats for now:

- UI: both the legacy [`Interaction`] and new [`PickingInteraction`] components exist [for now](https://github.com/bevyengine/bevy/issues/15550), with subtle behavioral differences.
- Sprites: picking always uses the full rectangle, and [alpha transparency is not taken into account](https://github.com/bevyengine/bevy/issues/14929).
- Mesh: this is a naive raycast against the full mesh. If you run into performance problems here, you should use simplified meshes and an acceleration data structure like a [BVH](https://en.wikipedia.org/wiki/Bounding_volume_hierarchy) to speed this up. As a result, this functionality is currently disabled by default. It can be enabled by adding the [`MeshPickingPlugin`].

We expect both [`bevy_rapier`] and [`avian`] (the two most popular ecosystem physics crates for Bevy) to add their own accelerated collider picking backends to work with the newly upstreamed API. Unless you're debugging, building an editor or really care about the exact triangles of raw meshes, you should use one of those crates for efficient mesh picking.

### Usage

There are two good ways to get started with the API:

First, you might want to quickly update the state of your objects (be they UI or game objects) based on what is being done to them, typically highlighting them or changing their color. For that, simply query for changes to the [`PickingInteraction`] component, which will change based on the current picking state.

Second, you might want to respond dynamically to various pointer-powered events. For that, we recommend using observers. Here, we're spawning a simple text node and responding to pointer events:

```rust
// UI text that prints a message when clicked:
commands
    .spawn(Text::new("Click Me!"))
    .observe(on_click_print_hello);

// A cube that spins when dragged:
commands
    .spawn((
        Mesh3d(meshes.add(Cuboid::default())),
        MeshMaterial3d(materials.add(Color::WHITE)),
    ))
    .observe(on_drag_spin);
}

fn on_click_print_hello(click: Trigger<Pointer<Click>>) {
    println!("{} was clicked!", click.entity());
}

fn on_drag_spin(drag: Trigger<Pointer<Drag>>, mut transforms: Query<&mut Transform>) {
    let mut transform = transforms.get_mut(drag.entity()).unwrap();
    transform.rotate_y(drag.delta.x * 0.02);
}
```

If you want to override how an entity interacts with picking, add the [`PickingBehavior`] component to them and configure it to meet your needs.

[`bevy_mod_picking`]: https://crates.io/crates/bevy_mod_picking/
[`bevy_mod_raycast`]: https://crates.io/crates/bevy_mod_raycast/
[Foresight Spatial Labs]: https://www.fslabs.ca/
[`PointerLocation`]: https://docs.rs/bevy/0.15.0/bevy/picking/backend/prelude/struct.PointerLocation.html
[backend]: https://docs.rs/bevy/0.15.0/bevy/picking/backend/index.html
[`PointerHits`]: https://docs.rs/bevy/0.15.0/bevy/picking/backend/struct.PointerHits.html
[`HoverMap`]: https://docs.rs/bevy/0.15.0/bevy/picking/focus/struct.HoverMap.html
[`Interaction`]: https://docs.rs/bevy/0.15.0/bevy/prelude/enum.Interaction.html
[`PickingInteraction`]: https://docs.rs/bevy/0.15.0/bevy/picking/focus/enum.PickingInteraction.html
[`bevy_rapier`]: https://crates.io/crates/bevy_rapier3d
[`avian`]: https://crates.io/crates/avian3d
[`PickingBehavior`]: https://docs.rs/bevy/0.15.0/bevy/picking/struct.PickingBehavior.html
[`MeshPickingPlugin`]: https://docs.rs/bevy/0.15.0/bevy/picking/mesh_picking/struct.MeshPickingPlugin.html

## Bubbling Observers

{{ <heading_metadata authors={["@NthTensor"]} prs={[13991, 15385]} /> }}

<!-- Minimal Bubbling Observers -->
<!-- https://github.com/bevyengine/bevy/pull/13991 -->

Virtually every pointer interaction (like mouse click) is rare (humans are slow!), and often requires a complex response.

This pattern is particularly useful in UI, where unhandled interactions are often intended for the pane that *contains* the entity that's on top, but is also valuable for in-game interactions: clicking on a unit's sword should select the unit!

To support this, we've extended the [`Event`] trait to include an associated `Traversal` type and an associated `AUTO_PROPAGATE` constant.
This behavior is opt-in: when you derive the `Event` type, these are set to `()` and `false` respectively.

For the [`Pointer<E>`] event type, we've chosen to implement this as:

```rust
impl <E> Event for Pointer<E>{
    type Traversal = &Parent;
    const AUTO_PROPAGATE: bool = true;
}
```

This means that, unless you call [`Trigger::propagate(false)`], pointer events will be bubbled up the hierarchy (accessing the `Entity` stored in the [`Parent`] component) until it reaches the entity root.

Any type that implements the [`Traversal`] trait can be used as the associated type and can access arbitrary read-only query data from the world.
While using the standard entity hierarchy is a sensible choice for *many* applications, bubbling can be used for arbitrary event propagation using your own [proto-relations].
Let us know what you cook up: user feedback is indispensable for building a better Bevy!

[`Event`]: https://docs.rs/bevy/0.15.0/bevy/ecs/event/trait.Event.html
[`Pointer<E>`]: https://docs.rs/bevy/0.15.0/bevy/picking/events/struct.Pointer.html
[`Trigger::propagate(false)`]: https://docs.rs/bevy/0.15.0/bevy/ecs/prelude/struct.Trigger.html#method.propagate
[`Parent`]: https://docs.rs/bevy/0.15.0/bevy/hierarchy/struct.Parent.html
[`Traversal`]: https://docs.rs/bevy/0.15.0/bevy/ecs/traversal/trait.Traversal.html
[proto-relations]: https://github.com/bevyengine/bevy/issues/3742

## Virtual Geometry Improvements

{{ <heading_metadata authors={["@JMS55"]} prs={[14193, 14623, 15023, 15084, 15643, 15846, 15886, 15955, 16049, 16111]} /> }}

Virtual geometry (the `meshlet` feature) got a ton of improvements in Bevy 0.15. It's still not production ready, and will remain as an experimental module, but performance has been greatly improved upon since the last release.

For all the interesting details, read the [author's blog post](https://jms55.github.io/posts/2024-11-14-virtual-geometry-bevy-0-15).

For existing users of this feature:
* Your GPU must now support `WgpuFeatures::SHADER_INT64_ATOMIC_MIN_MAX` to use this feature. As forewarned in the previous release, older GPUs may no longer be compatible.
* You must regenerate your MeshletMesh assets. MeshletMesh assets generated in Bevy 0.14 are not compatible with Bevy 0.15.
* Make sure you read both the migration guide and the updated rustdoc for full details on how to upgrade your project.

## Animation Masks

{{ <heading_metadata authors={["@pcwalton"]} prs={[15013]} /> }}

<!-- Implement animation masks, allowing fine control of the targets that animations affect. -->
<!-- https://github.com/bevyengine/bevy/pull/15013 -->

Animations now support masking out animation targets (joints).
This is implemented at the level of animation blend graphs (`AnimationGraph`)
and can be used to play different animations on separate parts of the
same model without interfering with one another. For example, you can play separate animations on a character's upper and lower body.

In this video, the fox's head and legs are playing two separate animations, thanks to animation masks:
<video controls><source src="masked-animation.mp4" type="video/mp4"/></video>

## Generalized Animation

{{ <heading_metadata authors={["@pcwalton", "@mweatherley"]} prs={[15282, 15434]} /> }}

<!-- Allow animation clips to animate arbitrary properties. -->
<!-- https://github.com/bevyengine/bevy/pull/15282 -->
[`AnimationClip`] can now be used to animate component fields with arbitrary curves.

```rust
animation_clip.add_curve_to_target(
    animation_target_id,
    AnimatableCurve::new(
        animated_field!(TextFont::font_size),
        // Oscillate the font size during the length of the animation.
        FunctionCurve::new(
            Interval::UNIT, 
            |t| 25.0 * f32::sin(TAU * t) + 50.0
        )
    )
);
```

<video controls><source src="animated-font-size.mp4" type="video/mp4"/></video>

This works for any named field and uses the new `Curve` API, which supports arbitrary curve types.
Animating `Transform` fields will likely be the most common use case:

```rust
animation_clip.add_curve_to_target(
    animation_target_id,
    AnimatableCurve::new(
        animated_field!(Transform::translation),
        // Construct a `Curve<Vec3>`` using a built-in easing curve constructor.
        EasingCurve::new(
            vec3(-10., 2., 0.),
            vec3(6., 2., 0.),
            EaseFunction::CubicInOut,
        )
    )
);
```

Bevy's internal animation handling for things like GLTF animations uses the same API!

If you need more complicated logic than "animate a specific component field", you can implement [`AnimatableProperty`], which can be used in
[`AnimatableCurve`] in place of [`animated_field!`].

[`AnimationClip`]: https://docs.rs/bevy/0.15/bevy/animation/struct.AnimationClip.html
[`AnimatableProperty`]: https://docs.rs/bevy/0.15/bevy/animation/animation_curves/trait.AnimatableProperty.html
[`AnimatableCurve`]: https://docs.rs/bevy/0.15/bevy/animation/animation_curves/struct.AnimatableCurve.html
[`animated_field!`]: https://docs.rs/bevy/0.15/bevy/animation/macro.animated_field.html

## Animation Graph: Additive Blending

{{ <heading_metadata authors={["@pcwalton"]} prs={[15631]} /> }}

<!-- Implement additive blending for animation graphs. -->
<!-- https://github.com/bevyengine/bevy/pull/15631 -->

Bevy's animation graphs (`AnimationGraph`), which are used to combine
simultaneously playing animations, now support *additive blending*.

Additive blending is a technique which allows separately authored animations
to be applied on top of an arbitrary base animation. For instance, an animation
in which a character swings a weapon may be applied additively on top of a 
walking or running animation.

Within an animation graph itself, this is accomplished by using `Add` nodes.
The situation above might be described with an animation graph that looks
something like this (weights omitted):

```
┌─────┐                              
│Walk ┼─┐                            
└─────┘ │ ┌─────┐                    
        ┼─┼Blend┼─┐                  
┌─────┐ │ └─────┘ │ ┌─────┐   ┌─────┐
│Run  ┼─┘         ┼─┤Add  ┼───┼Root │
└─────┘   ┌─────┐ │ └─────┘   └─────┘
          │Swing┼─┘                  
          └─────┘                    
```

The `Add` node functions by taking its first input (here, a blend of the 'Walk'
and 'Run' clips) as-is and then applying the subsequent inputs additively on
top of it. In code, the graph might be constructed as follows:

```rust
let mut animation_graph = AnimationGraph::new();

// Attach an `Add` node to the root.
let add_node = animation_graph.add_additive_blend(1.0, animation_graph.root);

// Add the `Blend` node and the additive clip as children; the `Blend` result
// will be used as the base because it is listed first.
let blend_node = animation_graph.add_blend(1.0, add_node);
animation_graph.add_clip(swing_clip_handle, 1.0, add_node);

// Finally, blend the 'Walk' and 'Run' clips to use as a base.
animation_graph.add_clip(walk_clip_handle, 0.5, blend_node);
animation_graph.add_clip(run_clip_handle, 0.5, blend_node);
```

## Animation Events

{{ <heading_metadata authors={["@atornity", "@cart"]} prs={[15538]} /> }}

<!-- Add Support for Triggering Events via `AnimationEvent`s -->
<!-- https://github.com/bevyengine/bevy/pull/15538 -->

Triggering gameplay events at specific points in an animation is a common pattern for synchronizing the visual, audible, and mechanical parts of your game. In **Bevy 0.15** we've added "animation event" support to [`AnimationClip`], which means that you can trigger a specific [`Event`] at a given point in time during [`AnimationClip`] playback:

```rust
#[derive(Event, Clone)]
struct PlaySound {
    sound: Handle<AudioSource>,
}

// This will trigger the PlaySound event at the 1.5 second mark in `animation_clip`
animation_clip.add_event(1.5, PlaySound {
    sound: assets.load("sound.mp3"),
});

app.add_observer(|trigger: Trigger<PlaySound>, mut commands: Commands| {
    let sound = trigger.event().sound.clone();
    commands.spawn(AudioPlayer::new(sound));
});
```

You can also trigger events for specific animation targets (such as bones):

```rust
animation_clip.add_event_to_target(AnimationTargetId::from_iter(["LeftLeg", "LeftFoot"], 0.5, TouchingGround);
```

This enables things like "triggering a dust effect each time a foot touches the ground in an animation":

<video controls><source src="animated_fox.mp4" type="video/mp4"/></video>

[`AnimationClip`]: https://docs.rs/bevy/0.15/bevy/animation/struct.AnimationClip.html
[`Event`]: https://docs.rs/bevy/0.15/bevy/ecs/event/trait.Event.html

## Bevy Remote Protocol (BRP)

{{ <heading_metadata authors={["@mweatherley"]} prs={[14880]} /> }}

The Bevy Remote Protocol allows the ECS of a running
Bevy application to be interacted with remotely. This can be used, for example,
to inspect and edit entities and their components at runtime. We anticipate 
that this will be used to create things like inspectors which monitor the
content of the ECS from a separate process. We're planning on using BRP in the
upcoming Bevy Editor to communicate with remote Bevy apps.

Currently, you can use BRP to:

- Get the serialized values of a set of components from an entity
- Perform a query for all entities matching a set of components and retrieve the matching values
- Create a new entity with a given set of component values
- For a given entity, insert or remove a set of components
- Despawn an entity
- Reparent one or more entities
- List the components registered in the ECS or present on an entity

Here is the minimal app setup required to use BRP over HTTP:

```rust
fn main() {
    App::new()
        .add_plugins((
            DefaultPlugins,
            // The "core" plugin, which handles remote requests provided by transports
            RemotePlugin::default(),
            // Provides remote request transport via HTTP
            RemoteHttpPlugin::default(),
        ))
        .run();
}
```

Here is a sample request:
```json
{
    "method": "bevy/get",
    "id": 0,
    "params": {
        "entity": 4294967298,
        "components": [
            "bevy_transform::components::transform::Transform"
        ]
    }
}
```

And here is a sample response:
```json
{
    "jsonrpc": "2.0",
    "id": 0,
    "result": {
        "bevy_transform::components::transform::Transform": {
            "rotation": { "x": 0.0, "y": 0.0, "z": 0.0, "w": 1.0 },
            "scale": { "x": 1.0, "y": 1.0, "z": 1.0 },
            "translation": { "x": 0.0, "y": 0.5, "z": 0.0 }
        }
    }
}
```

## Gamepads as Entities

{{ <heading_metadata authors={["@s-puig", "@Shatur"]} prs={[12770, 16222, 16233]} /> }}

<!-- Implement gamepads as entities -->
<!-- https://github.com/bevyengine/bevy/pull/12770 -->

Gamepads are now represented as entities, which makes them easier to work with!
The [`Gamepad`] component provides button and axis state, as well as metadata such as the vendor and product ID.
The [`GamepadSettings`] component provides configurable settings for a given [`Gamepad`], such as deadzones and sensitivity. The name of the gamepad is now stored in Bevy's standard [`Name`] component.

In Bevy 0.14, you might write:

```rust
fn gamepad_system(
   gamepads: Res<Gamepads>,
   button_inputs: Res<ButtonInput<GamepadButton>>,
   button_axes: Res<Axis<GamepadButton>>,
   axes: Res<Axis<GamepadAxis>>,
) {
    for gamepad in &gamepads {
        if button_inputs.just_pressed(
            GamepadButton::new(gamepad, GamepadButtonType::South)
        ) {
            info!("just pressed South");
        }

        let right_trigger = button_axes
           .get(GamepadButton::new(
               gamepad,
               GamepadButtonType::RightTrigger2,
           ))
           .unwrap();
        if right_trigger.abs() > 0.01 {
            info!("RightTrigger2 value is {}", right_trigger);
        }

        let left_stick_x = axes
           .get(GamepadAxis::new(gamepad, GamepadAxisType::LeftStickX))
           .unwrap();
        if left_stick_x.abs() > 0.01 {
            info!("LeftStickX value is {}", left_stick_x);
        }
    }
}
```

In 0.15, we can write this much more simply as:

```rust
fn gamepad_system(gamepads: Query<&Gamepad>) {
    for gamepad in &gamepads {
        if gamepad.just_pressed(GamepadButton::South) {
            println!("just pressed South");
        }

        let right_trigger = gamepad.get(GamepadButton::RightTrigger2).unwrap();
        if right_trigger.abs() > 0.01 {
            info!("RightTrigger2 value is {}", right_trigger);
        }

        let left_stick_x = gamepad.get(GamepadAxis::LeftStickX).unwrap();
        if left_stick_x.abs() > 0.01 {
            info!("LeftStickX value is {}", left_stick_x);
        }
    }
}
```

Much better!

[`Gamepad`]: https://docs.rs/bevy/0.15/bevy/input/gamepad/struct.Gamepad.html
[`GamepadSettings`]: https://docs.rs/bevy/0.15/bevy/input/gamepad/struct.GamepadSettings.html
[`Name`]: https://docs.rs/bevy/0.15/bevy/core/struct.Name.html

## Box Shadows

{{ <heading_metadata authors={["@ickshonpe"]} prs={[15204]} /> }}

<!-- box shadow -->
<!-- https://github.com/bevyengine/bevy/pull/15204 -->

![A demonstration of Bevy's box shadows. There are 12 shapes on a light blue background, and the border radius, aspect ratio and softness of the shadows is varied for each of them. The shadows cause the buttons to appear to hover above the page.](box_shadow.png)

Bevy UI now supports box shadows! Box shadows can be used to achieve particular artistic effects, such as creating a sense of depth to cue to users that an element is interactable.

By adding the new [`BoxShadow`] component to any [`Node`] entity, you can draw a pretty shadow behind your widgets.

```rust
#[derive(Component)]
pub struct BoxShadow {
    /// The shadow's color
    pub color: Color,
    /// Horizontal offset
    pub x_offset: Val,
    /// Vertical offset
    pub y_offset: Val,
    /// How much the shadow should spread outward.
    ///
    /// Negative values will make the shadow shrink inwards.
    /// Percentage values are based on the width of the UI node.
    pub spread_radius: Val,
    /// Blurriness of the shadow
    pub blur_radius: Val,
}
```

We have plans for future improvements: enable using shadows to create an inset / sunken look, and adding shadow support for images and text. If those possibilities excite you, get involved! We love helping [new contributors] land the features they care about.

[`BoxShadow`]: https://docs.rs/bevy/0.15/bevy/prelude/struct.BoxShadow.html
[`Node`]: https://docs.rs/bevy/0.15/bevy/prelude/struct.Node.html
[new contributors]: https://bevy.org/learn/contribute/introduction/

## Cosmic Text

{{ <heading_metadata authors={["@tigregalis", "@TotalKrill"]} prs={[10193]} /> }}

<!-- Cosmic text -->
<!-- https://github.com/bevyengine/bevy/pull/10193 -->

Historically, Bevy has used the `ab_glyph` library to render text. This handled simple latin text rendering reasonably well. But Bevy aims to be a generic app framework usable with any language, and there were a number of areas where `ab_glyph` wasn't meeting our needs.

The Rust text space has evolved significantly since we selected `ab_glyph`. Fortunately there are a number of good options now. We chose [`cosmic-text`](https://github.com/pop-os/cosmic-text) because of its robust feature set and its use in production applications (Iced, Cosmic Desktop, Zed, Lapce, etc). Notably, `cosmic-text` gives us support for:

* **Font Shaping**: The ability to take a string of character codes and perform layout and transformation rules. This can involve moving, modifying, and combining characters (such as ligatures). This is _extremely_ important for non-Latin-based languages.
* **System Font Loading**: The ability to scan for the available fonts installed on a system and load them.
* **Bidirectional Text**: Not all languages go from left to right! Cosmic Text gives us support for bidirectional text.
* **Text Editing**: Cosmic Text has its own internal text editing model, which we can take advantage of.

In **Bevy 0.15** we ported our text rendering to `cosmic-text`. This was largely an internal change (unlike the other "high level" text API changes this release, such as the port to Required Components).

That being said, you will definitely notice our improved ability to render text! Here is Bevy rendering Arabic text, right-to-left, using the Noto Sans Arabic font:

![arabic text](arabic_text.png)

Note that we haven't yet wired up `cosmic-text`'s "system font loading" features, but we're working on it!

## UI Scrolling

{{ <heading_metadata authors={["@Piefayth", "@nicoburns"]} prs={[15291]} /> }}

Bevy 0.15 introduces scrolling support for UI containers.

A UI `Node` with the `overflow` property set to `Overflow::scroll()` will offset its contents by the positive `offset_x` and `offset_y` values of the `ScrollPosition` component on the node.

Scrolling is done by modifying `ScrollPosition` directly; there is currently no built-in scroll input handler. A new [`scroll`](https://github.com/bevyengine/bevy/tree/v0.15.0/examples/ui/scroll.rs) example demonstrates handling simple mouse wheel scrolling. Axes of a node without `OverflowAxis::Scroll` will ignore changes to `ScrollPosition`.

## Retained Rendering World

{{ <heading_metadata authors={["@re0312", "@trashtalk217", "@kristoff3r", "@tychedelia"]} prs={[14449, 15320, 15582, 15756]} /> }}

<!-- Retained Rendering -->
<!-- https://github.com/bevyengine/bevy/pull/14449 Original PR -->
<!-- https://github.com/bevyengine/bevy/pull/15320 Adopted PR -->
<!-- https://github.com/bevyengine/bevy/pull/15582 Synchronized Removed Components -->
<!-- https://github.com/bevyengine/bevy/pull/15756 Type Safe Retained Render World -->
For awhile now, Bevy has had a ["parallel pipelined renderer"](/news/bevy-0-6/#pipelined-rendering-extract-prepare-queue-render). To enable this, we added a Render World, in addition to the Main World (a `World` holds ECS data like Entities, Components, and Resources). The Main World is the source of truth for app logic. While the Render World is rendering the current frame, the Main World can be simulating the next frame. There is a brief "extract step", where we synchronize the two and copy relevant data from the Main World to the Render World.

In previous versions of Bevy, we employed an "immediate mode" approach to Main World -> Render World synchronization: we fully cleared the Render World entities every frame. This accomplished a couple of things:

1. It allowed us to ensure entity IDs "lined up", allowing us to reuse entities in both places.
2. It prevented us from needing to solve the "desync problem". By clearing every frame and re-syncing, we ensure the two Worlds are always perfectly in sync.

There was also precedent for the "immediate mode" pipelined rendering approach: Bungie's Destiny renderer uses it to great effect!

However we learned pretty quickly that clearing every frame had major downsides:

1. The clearing process itself had overhead.
2. "Table" ECS storage could be expensive to rebuild every frame relative to alternatives, due to "archetype moves". As a result, we employed many workarounds such as moving storage outside of the ECS.
3. Full resyncs every frame meant re-doing work that didn't need redoing. ECS gives us a nice global view of how our data is changing. We should take advantage of that!

In **Bevy 0.15** we switched to a "retained Render World". We no longer clear each frame. We no longer rely on a shared entity ID space. Instead:

1. Each world has its own entities
2. For entities that are related, we store that relationship as components (ex: Render World entities have a `MainEntity` component and Main World entities have a `RenderEntity` component). If a Main World entity with `SyncToRenderWorld` is spawned, we spawn an equivalent in the Render World. If a Main World entity is despawned, we despawn the relevant entity in the Render World.

Ensuring synchronization is perfect is _not_ an easy problem. Plugging all of the holes took a lot of time this cycle and we will likely continue to evolve our synchronization strategy in the future. But we think "retained" is fundamentally better for Bevy, and we're excited to have this foundation laid!

## Curves

{{ <heading_metadata authors={["@mweatherley"]} prs={[14630, 14976, 15675, 16637]} /> }}

<!-- A Curve trait for general interoperation — Part I -->
<!-- https://github.com/bevyengine/bevy/pull/14630 -->

The new [`Curve<T>`] trait provides a shared interface for curves, describing how values of type `T` change as we vary a `f32` parameter `t` over some domain.

What's changing, and the domain that it's changing *over* are both incredibly flexible.
You might choose to set the generic parameter `T` to anything from position, to damage, to colors (like we did to create a powerful abstraction for [color gradients](https://docs.rs/bevy/0.15.0/bevy/color/struct.ColorCurve.html)).

The progress parameter `t` often represents time (like in animation), but it can also represent things like
a fraction/percentage of progress between a starting and ending value or
distance (such as curves that are mapped into 2D or 3D space),

### Constructing Curves

Each curve may be defined in a variety of ways. For example, a curve may be:

* defined by a function
* interpolated from samples
* constructed using splines
* produced by an easing function

Take a look at the constructors on the [`Curve<T>`] trait for more details.

### Modifying curves

Procedurally modifying curves is a powerful tool for both creating curves with the desired behavior and dynamically altering them.

Bevy 0.15 provides a number of flexible adaptors for taking an existing curve and
modifying its output and/or parametrization.

For example:

```rust
let timed_angles = [
  (0.0, 0.0),
  (1.0, -FRAC_PI_2),
  (2.0, 0.0),
  (3.0, FRAC_PI_2),
  (4.0, 0.0)
];

// A curve interpolating our list of (time, angle)-pairs. At each time, it
// produces the angle, so it is a `Curve<f32>` parametrized over `[0, 4]`.
let angle_curve = UnevenSampleAutoCurve::new(timed_angles).unwrap();

// Interpret these angles as angles of rotation for a `Curve<Rot2>`.
let rotation_curve = angle_curve.map(Rot2::radians);

// Change the parameterizing interval so that the whole loop happens in
// only 1 second instead of 4.
let fast_rotation_curve = rotation_curve.reparametrize_linear(Interval::UNIT).unwrap();
```

A number of other adaptors are available. For instance:

* a curve may be reversed, repeated, or ping-ponged
* two curves may be chained together to form a longer curve
* two curves may be zipped together to form a curve valued in tuples

### Sampling from curves

Sampling is the process of asking "what is the value of this curve at some particular value of `t`".
To do so, just call [`Curve::sample`]!

Much like how vector graphics can be rasterized into pixels, curves can be rasterized into regular, discretized intervals.
By resampling into an approximation derived from sample interpolation
on the original curve, we can make curves of diverse origin
uniform at the level of data.

While this may seem exotic, this technique is critical for serializing curves or
approximating properties via numerical methods.

```rust
// A curve defined by a function, which may be challenging to store as data.
let exponential_curve = FunctionCurve::new(
  interval(0.0, 10.0).unwrap(), 
  |t| f32::exp(2.0 * t)
);

// A curve approximating the original by resampling on 100 segments.
// Internally, this just holds the samples and the parameter interval.
let raster_curve = exponential_curve.resample_auto(100).unwrap();
```

[`Curve<T>`]: https://docs.rs/bevy/0.15.0/bevy/math/trait.Curve.html
[`Curve::sample`]: https://docs.rs/bevy/0.15.0/bevy/math/trait.Curve.html#method.sample

## Common Easing Functions

{{ <heading_metadata authors={["@RobWalt", "@mockersf", "@mweatherley"]} prs={[14788, 15675, 15711]} /> }}

<!-- Add most common interpolations -->
<!-- https://github.com/bevyengine/bevy/pull/15675 -->

"Easing functions" can be used to easily construct curves that interpolate between two values.
There are many "common" easing functions that each have a different "character" to them. These
are often used in "tweening" scenarios to give life to the interpolation.

<video controls><source src="ease-functions.mp4" type="video/mp4"/></video>

**Bevy 0.15** adds a new `Ease` trait, which defines how to interpolate a value of a given type. The `Ease` types include:

* vector types (`f32`, `Vec2`, `Vec3`, ...);
* direction types (`Dir2`, `Dir3`, `Dir3A`);
* rotation types (`Rot2`, `Quat`).

We've also added an `EaseFunction` enum, which defines many common easing functions. The new `EasingCurve` type uses these as inputs to define a final `Curve` from the given easing parameters.

For example, we can use an easing function to interpolate between two rotations:

```rust
// Ease between no rotation and a rotation of angle PI/2 about the y-axis.
let rotation_curve = EasingCurve::new(
    Quat::IDENTITY,
    Quat::from_rotation_y(FRAC_PI_2),
    EaseFunction::ElasticInOut,
)
.reparametrize_linear(interval(0.0, 4.0).unwrap())
.unwrap();
```

<video controls><source src="eased-rotation.mp4" type="video/mp4"/></video>

## Cyclic Splines

{{ <heading_metadata authors={["@mweatherley"]} prs={[14106]} /> }}

<!-- Cyclic splines -->
<!-- https://github.com/bevyengine/bevy/pull/14106 -->

Most cubic spline constructions now support creating a closed loop instead of just
a path, if desired. This can be convenient for constructing things like periodic
paths for NPCs or other game entities.

The only difference is that `to_curve_cyclic` must be called in place of `to_curve`.
The supported spline constructions are:
- Hermite splines (`CubicHermite`),
- Cardinal splines (`CubicCardinalSpline`),
- B-splines (`CubicBSpline`),
- Linear splines (`LinearSpline`).

![A closed loop constructed using a cubic cardinal spline][cyclic-spline]

[cyclic-spline]: cyclic-spline.png

## `PartialReflect`

{{ <heading_metadata authors={["@soqb", "@nicopap"]} prs={[7207]} /> }}

Bevy boasts a powerful [reflection] system that allows you to introspect and build types at runtime.
It works by passing around data as [`Reflect`] trait objects like `Box<dyn Reflect>`.
This has the effect of erasing the compile-time type information,
allowing data to be stored and moved around without having to know the exact type
behind the trait object.

Because of this type erasure, `bevy_reflect` can also get away with some interesting tricks.
For instance, there are many cases where a type needs to be built up field-by-field,
such as during deserialization.
This works fine when you know the type at compile-time, but it becomes very challenging to do
at runtime.
To solve this, `bevy_reflect` has a concept of _dynamic_ types.

Dynamic types exist as a way to dynamically construct and store reflected data in a way
that appears like a concrete type.
Behind the scenes, `bevy_reflect` uses these types to build up a representation of the target type.
And it can do so since we hide the actual type behind the `dyn Reflect` trait object.

Unfortunately, this comes with a very common issue: it becomes very easy to accidentally
believe a `dyn Reflect` is a concrete type when it's actually a dynamic type representing
that concrete type.

To address this problem, Bevy 0.15 has reworked the `Reflect` trait based on the [Unique Reflect RFC].
This splits it into two separate traits: `Reflect` and [`PartialReflect`].

`PartialReflect` is much like the `Reflect` trait of previous versions.
It allows access to fundamental reflection capabilities and allows for type-erasure behind
a `dyn PartialReflect` trait object.
It allows for both concrete types and dynamic types to be used interchangeably.

`Reflect`, on the other hand, has become a much stricter trait.
It's a subset of `PartialReflect` that guarantees the underlying type beneath the trait object
is exactly the concrete type it says it is.

This split allows reflection-based APIs and user code to be more explicit about the dynamic-ness of
the trait objects they're working with. It moves the knowledge of whether a type is dynamic or not
to compile-time, preventing many common pitfalls of working with dynamic types,
including knowing when they need to be handled separately.

[reflection]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/
[`Reflect`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/trait.Reflect.html
[`PartialReflect`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/trait.PartialReflect.html
[Unique Reflect RFC]: https://github.com/bevyengine/rfcs/pull/56

## Reflect Remote Types

{{ <heading_metadata authors={["@MrGVSV"]} prs={[6042]} /> }}

The [`bevy_reflect`] crate relies on types implementing [`Reflect`] in order to make them reflectable.
Fields of structs and enums that don't implement `Reflect` must be specifically ignored with `#[reflect(ignore)]`.
And due to Rust's [orphan rule], this is often the case for types not owned by the current crate.

Following [`serde`'s example](https://serde.rs/remote-derive.html), Bevy 0.15 introduces a way to reflect remote types
using a new [`#[reflect_remote(...)]`][reflect_remote] attribute macro.
This allows users to define a model for reflection to base its behavior on,
while still operating with the actual type.

```rust
// Pretend this type is defined in a crate called `external_crate`
#[derive(Default)]
struct Name {
    pub value: String,
}

// We can define our model, including other derives and reflection attributes
#[reflect_remote(external_crate::Name)]
#[derive(Default)]
#[reflect(Default)]
struct NameWrapper {
    pub value: String,
}

// Now we can use `Name` as a field in a reflected type without having to ignore it
#[derive(Reflect)]
struct Player {
    #[reflect(remote = NameWrapper)]
    name: external_crate::Name,
}
```

Under the hood, this works by transforming our model into a transparent wrapper around the actual type:

```rust
#[repr(transparent)]
struct NameWrapper(pub external_crate::Name);
```

The macro then uses the model to generate all the reflection trait implementations,
driven by a new [`ReflectRemote`] trait for swapping between our wrapper and the remote type.
Compile-time assertions are also generated to help ensure the model and the actual type stay in sync.

While this feature has many aspects complete, including generic support, enum support, and nesting,
there are still some limitations we hope to address in future releases,
including support for reflecting a remote type with private fields.

[`bevy_reflect`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/
[`Reflect`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/trait.Reflect.html
[orphan rule]: https://doc.rust-lang.org/book/ch10-02-traits.html#implementing-a-trait-on-a-type
[reflect_remote]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/attr.reflect_remote.html
[`ReflectRemote`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/trait.ReflectRemote.html

## The `Reflectable` Trait

{{ <heading_metadata authors={["@MrGVSV"]} prs={[5772]} /> }}

[`bevy_reflect`] is powered by many different traits working together
to provide the full reflection API. These include traits like [`Reflect`], but also other traits
like [`TypePath`], [`Typed`], and [`GetTypeRegistration`].

This can make adding the right bounds on generic parameters a bit confusing,
and it's easy to forget to include one of these traits.

To make this simpler, 0.15 introduces the [`Reflectable`] trait. All the traits listed above are
supertraits of `Reflectable`, allowing it to be used in place of all of them where necessary.

[`bevy_reflect`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/
[`Reflect`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/trait.Reflect.html
[`TypePath`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/trait.TypePath.html
[`Typed`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/trait.Typed.html
[`GetTypeRegistration`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/trait.GetTypeRegistration.html
[`Reflectable`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/trait.Reflectable.html

## Function Reflection

{{ <heading_metadata authors={["@MrGVSV", "@nixpulvis", "@hooded-shrimp"]} prs={[13152, 14098, 14141, 14174, 14201, 14641, 14647, 14666, 14704, 14813, 15086, 15145, 15147, 15148, 15205, 15484]} /> }}

Rust's options for working with functions in a dynamic context are limited.
We're forced to either coerce the function to a function pointer (e.g. `fn(i32, i32) -> i32`)
or turn it into a trait object (e.g. `Box<dyn Fn(i32, i32) -> i32>`).

In both cases, only functions with the same signature (both inputs and outputs) can be stored as an object of the same type.
For truly dynamic contexts, such as working with scripting languages or fetching functions by name,
this can be a significant limitation.

Bevy's [`bevy_reflect`] crate already removes the need for compile-time knowledge of types through
reflection.
In Bevy 0.15, functions can be reflected as well!

This feature is opt-in and requires the `reflect_functions` feature to be enabled on `bevy`
(or the `functions` feature on `bevy_reflect` if using that crate directly).

It works by converting regular functions which arguments and return type derive [`Reflect`]
into a [`DynamicFunction`] type using a new [`IntoFunction`] trait.

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b
}

let function = add.into_function();
```

With a `DynamicFunction`, we can then generate our list of arguments into an [`ArgList`]
and call the function:

```rust
let args = ArgList::new()
    .push_owned(25_i32)
    .push_owned(75_i32);

let result = function.call(args);
```

Calling a function returns a [`FunctionResult`] which contains our [`Return`] data 
or a [`FunctionError`] if something went wrong.

```rust
match result {
    Ok(Return::Owned(value)) => {
        let value = value.try_take::<i32>().unwrap();
        println!("Got: {}", value);
    }
    Err(err) => println!("Error: {:?}", err),
    _ => unreachable!("our function always returns an owned value"),
}
```

#### Closure Reflection

This feature doesn't just work for regular functions—it works on closures too!

For closures that capture their environment immutably, we can continue using `DynamicFunction`
and `IntoFunction`. For closures that capture their environment mutably, there's
[`DynamicFunctionMut`] and [`IntoFunctionMut`].

```rust
let mut total = 0;

let increment = || total += 1;

let mut function = increment.into_function_mut();

function.call(ArgList::new()).unwrap();
function.call(ArgList::new()).unwrap();
function.call(ArgList::new()).unwrap();

// Drop the function to release the mutable borrow of `total`.
// Alternatively, our last call could have used `call_once` instead.
drop(function);

assert_eq!(total, 3);
```

#### `FunctionInfo`

Reflected functions hold onto their type metadata via [`FunctionInfo`] which is automatically
generated by the [`TypedFunction`] trait. This allows them to return information about the
function including its name, arguments, and return type.

```rust
let info = String::len.get_function_info();

assert_eq!(info.name().unwrap(), "alloc::string::String::len");
assert_eq!(info.arg_count(), 1);
assert!(info.args()[0].is::<&String>());
assert!(info.return_info().is::<usize>());
```

One thing to note is that closures, anonymous functions, and function pointers
are not automatically given names. For these cases, names can be provided manually.

The same is true for all arguments including `self` arguments: names are not automatically
generated and must be supplied manually if desired.

Using `FunctionInfo`, a `DynamicFunction` will print out its signature when debug-printed.

```rust
dbg!(String::len.into_function());
// Outputs:
// DynamicFunction(fn alloc::string::String::len(_: &alloc::string::String) -> usize)
```

#### Manual Construction

For cases where `IntoFunction` won't work, such as for functions with too many arguments
or for functions with more complex lifetimes, `DynamicFunction` can also be constructed manually.

```rust
// Note: This function would work with `IntoFunction`,
// but for demonstration purposes, we'll construct it manually.
let add_to = DynamicFunction::new(
    |mut args| {
        let a = args.take::<i32>()?;
        let b = args.take_mut::<i32>()?;

        *b += a;

        Ok(Return::unit())
    },
    FunctionInfo::named("add_to")
        .with_arg::<i32>("a")
        .with_arg::<&mut i32>("b")
        .with_return::<()>(),
);
```

#### The Function Registry

To make it easier to work with reflected functions, a dedicated [`FunctionRegistry`] has been added.
This works similarly to the [`TypeRegistry`] where functions can be registered and retrieved by name.

```rust
let mut registry = FunctionRegistry::default();
registry
    // Named functions can be registered directly
    .register(add)?
    // Unnamed functions (e.g. closures) must be registered with a name
    .register_with_name("add_3", |a: i32, b: i32, c: i32| a + b + c)?;

let add = registry.get("my_crate::math::add").unwrap();
let add_3 = registry.get("add_3").unwrap();
```

For better integration with the rest of Bevy, a new [`AppFunctionRegistry`] resource has been added
along with registration methods on [`App`].

#### The `Function` Trait

A new reflection trait—appropriately called [`Function`]—has been added to correspond to functions.

Due to limitations in Rust, we're unable to implement this trait for all functions,
but it does make it possible to pass around a `DynamicFunction` as a [`PartialReflect`] trait object.

```rust
#[derive(Reflect)]
#[reflect(from_reflect = false)]
struct EventHandler {
    callback: DynamicFunction<'static>,
}

let event_handler: Box<dyn Struct> = Box::new(EventHandler {
    callback: (|| println!("Event fired!")).into_function(),
});

let field = event_handler.field("callback").unwrap();

if let ReflectRef::Function(callback) = field.reflect_ref() {
    callback.reflect_call(ArgList::new()).unwrap();
}
```

#### Limitations

While this feature is quite powerful already, there are still a number of limitations.

Firstly, `IntoFunction`/`IntoFunctionMut` only work for functions with up to 16 arguments,
and only support returning borrowed data where the lifetime is tied to the first argument
(normally `self` in methods).

Secondly, the `Function` trait can't be implemented for all functions due to how the function
reflection traits are defined.

Thirdly, all arguments and return types must have derived `Reflect`.
This can be confusing for certain types such as `&str` since only `&'static str` implements
`Reflect` and its borrowed version would be `&&'static str`.

Lastly, while generic functions are supported, they must first be manually monomorphized.
This means that if you have a generic function like `fn foo<T>()`, you have to create the
`DynamicFunction` like `foo::<i32>.into_function()`.

Most of these limitations are due to Rust itself.
The [lack of variadics] and [issues with coherence] are among the two biggest difficulties
to work around.
Despite this, we will be looking into ways of improving the ergonomics and capabilities 
of this feature in future releases.

We already have a [PR](https://github.com/bevyengine/bevy/pull/15074) up to add support for overloaded functions: functions with a variable
number of arguments and argument types.

[`bevy_reflect`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/
[`Reflect`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/trait.Reflect.html
[`DynamicFunction`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/func/struct.DynamicFunction.html
[`IntoFunction`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/func/trait.IntoFunction.html
[`ArgList`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/func/args/struct.ArgList.html
[`FunctionResult`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/func/type.FunctionResult.html
[`Return`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/func/enum.Return.html
[`FunctionError`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/func/enum.FunctionError.html
[`DynamicFunctionMut`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/func/struct.DynamicFunctionMut.html
[`IntoFunctionMut`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/func/trait.IntoFunctionMut.html
[`FunctionInfo`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/func/struct.FunctionInfo.html
[`TypedFunction`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/func/trait.TypedFunction.html
[`FunctionRegistry`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/func/struct.FunctionRegistry.html
[`TypeRegistry`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/struct.TypeRegistry.html
[`AppFunctionRegistry`]: https://docs.rs/bevy_reflect/0.15/bevy_ecs/reflect/struct.AppTypeRegistry.html
[`App`]: https://docs.rs/bevy_reflect/0.15/bevy_app/struct.App.html
[`Function`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/func/trait.Function.html
[`PartialReflect`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/trait.PartialReflect.html
[lack of variadics]: https://poignardazur.github.io/2024/05/25/report-on-rustnl-variadics/
[issues with coherence]: https://doc.rust-lang.org/rustc/lints/listing/warn-by-default.html#coherence-leak-check

## `TypeInfo` Improvements

{{ <heading_metadata authors={["@MrGVSV"]} prs={[15475, 13321, 13320, 15235]} /> }}

With [`bevy_reflect`], compile-time type information can be retrieved from a reflected type as [`TypeInfo`].

Bevy 0.15 adds many improvements and convenience methods for working with `TypeInfo`.

#### Generic Parameter Info

The first addition is the ability to get information about a type's generic parameters.
This not includes the parameter's type, but also its name and—if it's a const parameter—its default value.

```rust
#[derive(Reflect)]
struct MyStruct<T>(T);

let generics = MyStruct::<f32>::type_info().generics();

let t = generics.get(0).unwrap();
assert_eq!(t.name(), "T");
assert!(t.ty().is::<f32>());
assert!(!t.is_const());
```

#### Nested `TypeInfo`

Pretty much every type in Rust is made up of other types.
Structs, maps, lists—they all contain other types.

In previous versions of Bevy, `TypeInfo` granted you limited access to type information of
these nested types. It mostly just provided the type's [`TypeId`] and [`TypePath`].

However, in Bevy 0.15, you can now directly access the `TypeInfo` of these nested types.

```rust
#[derive(Reflect)]
struct Row {
  id: usize
}

let struct_info: StructInfo = Row::type_info().as_struct();

let field: NamedField = struct_info.field("id").unwrap();

// `NamedField` now exposes a way to fetch the `TypeInfo` of the field's type
let field_info: TypeInfo = field.type_info().unwrap();
assert!(field_info.is::<usize>());
```

#### `TypeInfo` Convenience Casts

In most cases, `TypeInfo` needs to first be pattern matched to the correct variant in order to gain full access
to the type's compile-time information.
This can be mildly annoying when you already know the variant ahead of time.
This often occurs when writing tests, but also shows up when trying to get the type's [`ReflectRef`] data
along with its `TypeInfo`. It tends to looks something like:

```rust
// We have to pattern match on `ReflectRef`...
let ReflectRef::List(list) = reflected_value.reflect_ref() else {
    panic!("expected a list");
};

// ...and still need to pattern match on `TypeInfo`
let TypeInfo::List(list_info) = reflected_value.get_represented_type_info().unwrap() else {
    panic!("expected a list info");
};
```

In such cases, the variant is already verified via the `ReflectRef` but the
`TypeInfo` must still be pattern matched regardless.

In Bevy 0.15, convenience methods have been added to `TypeInfo`, `ReflectRef`, [`ReflectMut`], and [`ReflectOwned`]
to conveniently cast to the expected variant or return an error upon failure.

```rust
// We can simply verify the kind of our reflected value once...
let ReflectRef::List(list) = reflected_value.reflect_ref() else {
    panic!("expected a list");
};

// ...and just assert the `TypeInfo`
let list_info = reflected_value.get_represented_type_info().unwrap().as_list().unwrap();
```

If the `.as_list()` cast fails in the snippet above, it will return an error detailing what [kind]
we expected (i.e. `List`) and what we actually got (e.g. `Array`, `Struct`, etc.).

And this works in the opposite direction as well:

```rust
let TypeInfo::List(list_info) = reflected_value.get_represented_type_info().unwrap() else {
    panic!("expected a list info");
};

let list = reflected_value.reflect_ref().as_list().unwrap();
```

[`bevy_reflect`]: https://docs.rs/bevy_reflect/0.15/
[`TypeInfo`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/enum.TypeInfo.html
[`ReflectRef`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/enum.ReflectRef.html
[`ReflectMut`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/enum.ReflectMut.html
[`ReflectOwned`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/enum.ReflectOwned.html
[`TypeId`]: https://doc.rust-lang.org/std/any/struct.TypeId.html
[`TypePath`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/trait.TypePath.html
[kind]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/enum.ReflectKind.html

## The `Type` Type

{{ <heading_metadata authors={["@MrGVSV"]} prs={[14838]} /> }}

Rust's [`TypeId`] is a unique identifier for a type, making it a perfect candidate for use as a
key in mappings and for checking whether two types are the same at runtime.
And since it's essentially just two `u64` values, it's extremely cheap to copy, compare, and hash.

One of the downsides to using `TypeId`, though, is that it doesn't contain any other information
about the type, including its name. This can make debugging somewhat frustrating as you can't
easily tell which type a `TypeId` corresponds to.

Since [`bevy_reflect`] makes heavy use of `TypeId`, 0.15 introduces a new type to help alleviate
the debugging issue while still maintaining the benefits of `TypeId`: [`Type`].

[`Type`] is a simple wrapper around `TypeId` that also stores the [`TypePathTable`].
Like `TypeId` it's `Copy`, `Eq`, and `Hash`, delegating to the underlying `TypeId` for the latter
two. But unlike `TypeId`, its `Debug` implementation will print the [type path] of the type it represents.
This debuggability comes at the cost of an extra 32 bytes, but may often be well worth it,
especially if that data would have been stored elsewhere anyway.

It can be constructed from any type that implements [`TypePath`]:

```rust
let ty = Type::of::<String>();

let mut map = HashMap::<Type, i32>::new();
map.insert(ty, 25);

let debug = format!("{:?}", map);
assert_eq!(debug, "{alloc::string::String: 25}");
```

[`TypeId`]: https://doc.rust-lang.org/std/any/struct.TypeId.html
[`bevy_reflect`]: https://docs.rs/bevy_reflect/0.15/
[`Type`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/struct.Type.html
[`TypePathTable`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/struct.TypePathTable.html
[type path]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/trait.TypePath.html#tymethod.type_path
[`TypePath`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/trait.TypePath.html

## Reflection support for Sets

{{ <heading_metadata authors={["@RobWalt"]} prs={[13014]} /> }}

<!-- Dedicated `Reflect` implementation for `Set`-like things -->
<!-- https://github.com/bevyengine/bevy/pull/13014 -->

Inside of `bevy_reflect`, every reflected Rust object ends up being mapped to one of a handful of [`ReflectKind`] variants.

Before Bevy 0.15, sets (like [`HashSet`]) were treated as opaque "values": there was no way to view or modify their contents via reflection.
With these changes, we can now properly represent sets of all kinds, which is particularly handy for runtime debugging tools like [`bevy-inspector-egui`]!

[`ReflectKind`]: https://docs.rs/bevy/0.15.0/bevy/reflect/enum.ReflectKind.html
[`HashSet`]: https://doc.rust-lang.org/stable/std/collections/struct.HashSet.html
[`bevy-inspector-egui`]: https://github.com/jakobhellermann/bevy-inspector-egui

## Change Detection Source Location Tracking

{{ <heading_metadata authors={["@aevyrie"]} prs={[14034]} /> }}

<!-- Track source location in change detection -->
<!-- https://github.com/bevyengine/bevy/pull/14034 -->

Keeping track of when and where values are changed can be tricky in any complex program, and Bevy applications are no exception.
Thankfully, our unified ECS-backed data model makes it easy for us to add debugging tools that work right out of the box, with no user configuration required.

When you turn on the `track_change_detection` feature flag, Bevy will record the exact line of code that mutated your component or resource side-by-side with the value.
While this is obviously too expensive for ordinary use, it's a godsend for debugging tricky issues, as the value can be logged or read directly via the debugger of your choice.

As shown in the [`change_detection` example],
simply turn on the feature and call `my_component.changed_by()` on any [`Ref`], [`Mut`], [`Res`] or [`ResMut`] smart pointer to get a helpful string pointing you straight to the last line of code that mutated your data!

[`change_detection` example]: https://github.com/bevyengine/bevy/blob/main/examples/ecs/change_detection.rs
[`Ref`]: https://docs.rs/bevy/0.15.0/bevy/ecs/change_detection/struct.Ref.html
[`Mut`]: https://docs.rs/bevy/0.15.0/bevy/ecs/change_detection/struct.Mut.html
[`Res`]: https://docs.rs/bevy/0.15.0/bevy/ecs/change_detection/struct.Res.html
[`ResMut`]: https://docs.rs/bevy/0.15.0/bevy/ecs/change_detection/struct.ResMut.html

## Optimized Iteration of Mixed Sparse Set and Table Components

{{ <heading_metadata authors={["@re0312"]} prs={[14049, 14673]} /> }}

<!-- Opportunistically use dense iteration for  archetypal iteration -->
<!-- https://github.com/bevyengine/bevy/pull/14049 -->

In Bevy, components can be [stored] using one of two different mechanisms,
according to the [`StorageType`] set when implementing the [`Component`] trait.

Table storage is the traditional archetypal ECS storage, where component data is densely packed into tables of raw data with other entities who share the same set of components.
By contrast, sparse set storage keeps the component information out of the table, separating entities by archetype (the set of components they have) without fragmenting otherwise shared tables.

As a result of the map-like storage strategy used by sparse set components, they have faster insertion and removal speed, at the cost of slower random-access iteration.
This is a reasonable tradeoff, but historically, one that Bevy developers were unlikely to use.

That's because a long-standing bug caused iteration to use the slower, fallback sparse-style iteration if even one of the components in the query or its filters were sparse sets, regardless of whether or not this was necessary.
The fix has resulted in query iteration speeds that are between 1.8 and 3.5 times faster (when using parallel iteration) for these scenarios!

Iterating over the data in sparse set components is still relatively slow,
but they should finally be a good default choice for any repeatedly inserted or dataless components.

[stored]: https://docs.rs/bevy/0.15/bevy/ecs/component/trait.Component.html#associatedconstant.STORAGE_TYPE
[`StorageType`]: https://docs.rs/bevy/0.15/bevy/ecs/component/enum.StorageType.html
[`Component`]: https://docs.rs/bevy/0.15/bevy/ecs/component/trait.Component.html#associatedconstant.STORAGE_TYPE

## Expose winit's `MonitorHandle`

{{ <heading_metadata authors={["@tychedelia"]} prs={[13669]} /> }}

The new `Monitor` component simplifies the process of working with multi-monitor setups by providing easy access to monitor properties such as resolution, refresh rate, position, and scaling factor. This feature is especially useful for developers who need to spawn windows on specific displays, gather monitor details, or adjust their application based on available hardware. This is especially useful for creative setups like multi-projector installations or LED video walls, where precise control over display environments is critical.

`Monitor` can be queried for and used for things like spawning or resizing Windows:

```rust
fn spawn_windows(
    mut commands: Commands,
    monitors: Query<(Entity, &Monitor)>,
) {
    for (entity, monitor) in monitors_added.iter() {
        commands.spawn(Window {
            mode: WindowMode::Fullscreen(MonitorSelection::Entity(entity)),
            position: WindowPosition::Centered(MonitorSelection::Entity(entity)),
            ..default()
        });
    }
}
```

## Custom Cursors

{{ <heading_metadata authors={["@eero-lehtinen"]} prs={[14284]} /> }}

<!-- Add custom cursors -->
<!-- https://github.com/bevyengine/bevy/pull/14284 -->

Previously Bevy's native window cursors supported only a fixed set of built-in OS cursors. Bevy now also supports arbitrary images as "custom cursors". Custom cursors still use native facilities of the OS, which allows them to stay perfectly responsive even when the frame rate of the application drops.

Insert the [`CursorIcon`] component with a [`CustomCursor`] to set a [`Window`] entity's cursor:

```rust
commands
    .entity(window)
    .insert(CursorIcon::Custom(CustomCursor::Image {
        handle: asset_server.load("cursor_icon.png"),
        hotspot: (5, 5),
    }));
```

[`CursorIcon`]: https://docs.rs/bevy/0.15/bevy/winit/cursor/enum.CursorIcon.html
[`CustomCursor`]: https://docs.rs/bevy/0.15/bevy/winit/cursor/enum.CustomCursor.html
[`Window`]: https://docs.rs/bevy/0.15/bevy/prelude/struct.Window.html

## Uniform Mesh Sampling

{{ <heading_metadata authors={["@mweatherley"]} prs={[14071]} /> }}

<!-- Uniform mesh sampling -->
<!-- https://github.com/bevyengine/bevy/pull/14071 -->

The surfaces of meshes can now be randomly sampled. This can be used for things like placing scenery or particle effects.

This consists of:

1. The `Mesh::triangles` method, which allows the extraction of a `Mesh`'s list of 
  triangles (`Triangle3d`).
2. The `UniformMeshSampler` type, which allows the creation of a [`Distribution`] that
  uniformly samples points in space (`Vec3`) from a collection of triangles.

The functionality comes from putting these together:

```rust
let mut rng = StdRng::seed_from_u64(8765309);

// Get an iterator over triangles in the mesh. This can fail if the mesh has
// the wrong format or if its vertex/index data is malformed.
let triangles = my_mesh.triangles().unwrap();

// Construct the distribution. This can fail in some cases - most notably if 
// the mesh surface has zero area.
let distribution = UniformMeshSampler::try_new(triangles).unwrap();

// Get 1000 points uniformly sampled from the surface of the mesh.
let samples: Vec<Vec3> = distribution.sample_iter(&mut rng).take(1000).collect();
```

[`Distribution`]: https://docs.rs/rand/0.8.5/rand/distributions/trait.Distribution.html

## `EventMutator`

{{ <heading_metadata authors={["@BobG1983"]} prs={[13818]} /> }}

<!-- Created an EventMutator for when you want to mutate an event before reading -->
<!-- https://github.com/bevyengine/bevy/pull/13818 -->

<!-- TODO -->

When working with complex event-driven logic, you may find that you want to conditionally modify events without changing their type or re-emitting them.
While this has always been possible, it was quite onerous:

```rust
// We need to manually track which events this system has read
// using a system-local `EventCursor`, previously called `ManualEventReader`.
fn mutate_events(mut events: ResMut<Events<MyEvent>>, mut local_cursor: Local<EventCursor<MyEvent>>){    
    for event in local_cursor.read_mut(&mut *events){
        event.some_mutation();
    }
}
```

Now, you can simply use the new [`EventMutator`] system param, which keeps track of this bookkeeping for you.

```rust
fn mutate_events(mut event_mutator: EventMutator<MyEvent>>){    
    for event in event_mutator.read(){
        event.some_mutation();
    }
}
```

[`EventMutator`]: https://docs.rs/bevy/0.15/bevy/ecs/event/struct.EventMutator.html

## Isometry Types

{{ <heading_metadata authors={["@mweatherley", "@Jondolf"]} prs={[14269]} /> }}

<!-- Basic isometry types -->
<!-- https://github.com/bevyengine/bevy/pull/14269 -->

Vectors and quaternions are commonly used in 3D to describe relative and absolute positions and orientations of objects.
However, when performing more complicated transformations, such as going from a global frame of reference to an object's local space and back,
or composing multiple translations and rotations together, they can get rather unwieldy and difficult to reason about.

The new [`Isometry2d`] and [`Isometry3d`] types introduced in **Bevy 0.15** are a simple yet powerful tool for efficiently describing
these kinds of transformations. An isometry represents a rotation followed by a translation, similar to a [`Transform`] with a scale of 1.

```rust
// Create an isometry from a translation and rotation.
let iso1 = Isometry3d::new(Vec3::new(2.0, 1.0, 3.0), Quat::from_rotation_z(FRAC_PI_2));

// Transform a point using the isometry.
let point = Vec3::new(4.0, 4.0, 4.0);
let result = iso1.transform_point(point); // or iso1 * point
assert_relative_eq!(result, Vec3::new(-2.0, 5.0, 7.0));

// Create another isometry.
let iso2 = Isometry3d::from_rotation(Quat::from_rotation_z(FRAC_PI_2));

// Compute the relative translation and rotation.
let relative_iso = iso1.inverse_mul(iso2); // or iso1.inverse() * iso2
```

Isometries are most useful in mathematical contexts where scaling is not desired, such as when describing relative positions of objects
for intersection tests and other geometric queries. However, they are now also used in some APIs, including gizmo methods:

```rust
// Specify rectangle position and orientation with an isometry.
gizmos.rect_2d(Isometry2d::new(translation, Rot2::degrees(45.0)), Vec2::splat(250.0), CYAN);

// Many methods take an `impl Into<Isometry3d>`, so it is enough to only provide
// translation or rotation if a full isometry isn't needed.
gizmos.sphere(translation, 1.0, PURPLE);
```

[`Transform`] and [`GlobalTransform`] can also be converted to an [`Isometry3d`] using the [`to_isometry`] method,
providing a convenient way to use these APIs when you already have access to entity transforms.

Note that unlike [`Transform`], these isometry types are *not* components. They are purely convenience types for math.

[`Isometry2d`]: https://docs.rs/bevy/0.15/bevy/math/struct.Isometry2d.html
[`Isometry3d`]: https://docs.rs/bevy/0.15/bevy/math/struct.Isometry3d.html
[`Transform`]: https://docs.rs/bevy/0.15/bevy/transform/components/struct.Transform.html
[`GlobalTransform`]: https://docs.rs/bevy/0.15/bevy/transform/components/struct.GlobalTransform.html
[`to_isometry`]: https://docs.rs/bevy/0.15/bevy/transform/components/struct.Transform.html#method.to_isometry

## Lifecycle Hook & Observer Trigger for Replaced Values

{{ <heading_metadata authors={["@BigWingBeat"]} prs={[14212]} /> }}

<!-- Component Lifecycle Hook & Observer Trigger for replaced values -->
<!-- https://github.com/bevyengine/bevy/pull/14212 -->

Bevy 0.14 introduced [Component Lifecycle Hooks and Observers](https://bevy.org/news/bevy-0-14/#ecs-hooks-and-observers),
and included several built-in observer triggers for each way that components could be added to or removed from entities: `OnAdd`, `OnInsert` and `OnRemove`.
However, there was a hole in this API. While `OnRemove` is a counterpart to `OnAdd`, `OnInsert` had no such counterpart,
meaning certain operations had no corresponding lifecycle hook or observer trigger:

```rust
use bevy::{
    ecs::component::{ComponentHooks, StorageType},
    prelude::{Commands, Component, Deref, DerefMut, Entity, Query, Resource},
    utils::HashMap,
};

#[derive(Hash, PartialEq, Eq, Clone, Copy)]
struct SomeId(u32);

#[derive(Resource, Deref, DerefMut)]
struct EntityLookupById(HashMap<SomeId, Entity>);

impl Component for SomeId {
    const STORAGE_TYPE: StorageType = StorageType::Table;

    fn register_component_hooks(hooks: &mut ComponentHooks) {
        hooks
            .on_insert(|mut world, entity, _| {
                let this = *world.entity(entity).get::<Self>().unwrap();
                world
                    .resource_mut::<EntityLookupById>()
                    .insert(this, entity);
            })
            .on_remove(|mut world, entity, _| {
                let this = *world.entity(entity).get::<Self>().unwrap();
                world.resource_mut::<EntityLookupById>().remove(&this);
            });
    }
}

fn some_system(mut commands: Commands, query: Query<(Entity, &SomeId)>) {
    let mut iter = query.iter();

    let Some((a_entity, _)) = iter.next() else {
        return;
    };

    let Some((_, &b_id)) = iter.next() else {
        return;
    };

    commands.entity(a_entity).insert(b_id);
}
```

In this example, the system inserts a new component value onto an entity that already has one,
which overwrites the previous component value.
This causes the `on_insert` lifecycle hook to run for the new value,
but the `on_remove` hook doesn't run for the previous value.
As a result, the hashmap entry for the previous ID value is still present, even though it has been replaced.

Bevy 0.15 introduces a new component lifecycle hook and observer trigger for this scenario: `on_replace`/`OnReplace`.
This hook runs just before the `on_remove` hook in all cases,
and additionally runs in the aforementioned scenario where a component value is entirely replaced.
The hook runs just before the replacement occurs,
letting you access the soon-to-be-dropped value to perform bookkeeping and cleanup.

The above example would be fixed by simply replacing the `on_remove` hook with the new `on_replace` hook:

```diff
21                     .resource_mut::<EntityLookupById>()                                          
22                     .insert(this, entity);       
23             })                                   
-24             .on_remove(|mut world, entity, _| {  
+24             .on_replace(|mut world, entity, _| {
25                 let this = *world.entity(entity).get::<Self>().unwrap();                         
26                 world.resource_mut::<EntityLookupById>().remove(&this);                          
27             });                                  
```


Note that it *does not* run if a component value is merely *mutated* - in those cases you want to use change detection instead.

## Pack multiple vertex and index arrays together into growable buffers

{{ <heading_metadata authors={["@pcwalton"]} prs={[14257, 15566, 15569]} /> }}

<!-- Pack multiple vertex and index arrays together into growable buffers. -->
<!-- https://github.com/bevyengine/bevy/pull/14257 -->

<!-- TODO -->

**Bevy 0.15** changes the way meshes are stored on the GPU to greatly improve CPU performance. Instead of using separate vertex and index buffers for every mesh as is done in Bevy 0.14, now they are coalesced respectively into 'slabs' of configurable size. This cuts down on how frequently we need to change bind groups, winning us up to 2x speedups!

The `MeshAllocatorSettings` resource allows tuning slab sizes, growth rate, and cut-offs to best fit your application's needs. The defaults should already be a significant win for most scenes.

WebGL2 does not support packing vertex buffers together, so only index buffers get combined on this platform.

Some measurements on the [Bistro] scene:

Overall frame time improves from 8.74 ms to 5.53 ms (1.58x speedup)
Render system time improves from 6.57 ms to 3.54 ms (1.86x speedup)
Opaque pass time improves from 4.64 ms to 2.33 ms (1.99x speedup)

[Bistro]: https://github.com/DGriffin91/bevy_bistro_scene
[`MeshAllocatorSettings`]: https://docs.rs/bevy/0.15/bevy/render/mesh/allocator/struct.MeshAllocatorSettings.html

## Rewrite screenshots

{{ <heading_metadata authors={["@tychedelia"]} prs={[14833]} /> }}

Screenshots can now be taken with a new observer based API that allows targeting any `RenderTarget` that can be used with a `Camera`, not just windows. 

```rust
// Capture the primary window
commands
    .spawn(Screenshot::primary_window())
    .observe(save_to_disk(path));

// Or a `Handle<Image>`
commands
    .spawn(Screenshot::image(render_target))
    .observe(save_to_disk(path));
```

The observer triggers with a `ScreenshotCaptured` event containing an Image that can be used for saving to disk, post-processing, or generating thumbnails. This flexible approach makes it easier to capture content from any part of your rendering pipeline, whether it’s a window, an off-screen render target, or a texture in a custom render pass.

## `SystemParamBuilder`

{{ <heading_metadata authors={["@chescock"]} prs={[14050, 14821, 14818, 15189, 14817]} /> }}

<!-- SystemParamBuilder - Support dynamic system parameters -->
<!-- https://github.com/bevyengine/bevy/pull/14817 -->

Bevy 0.14 introduced [the `SystemBuilder` type](https://bevy.org/news/bevy-0-14/#systembuilder) to allow systems to be created with dynamic queries.
In Bevy 0.15, this has been extended to many more types of system parameters!

The `SystemBuilder` type has been replaced with a `SystemParamBuilder<P>` trait to make it easier to compose builders.
Aggregates of parameters, including [tuples, `ParamSet`](https://github.com/bevyengine/bevy/pull/14050), [`Vec<T>`](https://github.com/bevyengine/bevy/pull/14821), and [custom parameters using `#[derive(SystemParam)]`](https://github.com/bevyengine/bevy/pull/14818), can now be used in dynamic systems.
For example, a `ParamSet<Vec<Query<FilteredEntityMut>>>` can be used to pass a variable number of dynamic queries that may conflict.

New [`FilteredResources` and `FilteredResourcesMut`](https://github.com/bevyengine/bevy/pull/15189) types can access a set of resources configured at runtime, similar to how the existing `FilteredEntityRef` and `FilteredEntityMut` access a set of components on one entity.

Finally, a new [`DynSystemParam`](https://github.com/bevyengine/bevy/pull/14817) type allows systems to use parameters of dynamic type and then downcast them.
This is especially useful for implementing part of a system with trait objects, where each trait implementation can use a different system parameter type.

Taken together, these can be used to build a system that runs a script defined at runtime, where the script needs a variable number of query and resource parameters.
Or, they can be used to build systems out of parts assembled at runtime!

```rust
fn buildable_system(
    query_a: Query<&A>,
    query_b: Query<&B>,
    queries_with_locals: Vec<(Query<FilteredEntityMut>, Local<usize>)>,
    mut dynamic_params: ParamSet<Vec<DynSystemParam>>,
    resources: FilteredResourcesMut,
) {
    // Parameters in a `ParamSet<Vec>` are accessed by index.
    let mut dyn_param_0: DynSystemParam = dynamic_params.get_mut(0);
    // Parameters in a `DynSystemParam` are accessed by downcasting to the original type.
    let param: Local<&str> = dyn_param_0.downcast_mut::<Local<&str>>().unwrap();
    // `FilteredResources` and `FilteredResourcesMut` have methods to get resources by type or by ID.
    let res: Ref<R> = resources.get::<R>().unwrap();
}

let param_builder = (
    // Parameters that don't need configuration can be built using `ParamBuilder` or its factory methods.
    ParamBuilder,
    ParamBuilder::query(),
    // A `Vec` of parameters can be built using a `Vec` of builders.
    vec![
        // A tuple of parameters can be built using a tuple of builders.
        (
            // Queries are built with a callback that supplies a `QueryBuilder` to configure the query.
            QueryParamBuilder::new(|builder| { builder.data::<&A>(); }),
            // Locals are built by passing the initial value for the local.
            LocalBuilder(123),
        ),
    ],
    // A `ParamSet` can be built for either a tuple or a `Vec`.
    ParamSetBuilder(vec![
        // A `DynSystemParam` is built using a builder for any type, and can be downcast to that type.
        DynParamBuilder::new(LocalBuilder("hello")),
        DynParamBuilder::new(ParamBuilder::resource::<R>()),
        // The type may be any system parameter, even a tuple or a `Vec`!
        DynParamBuilder::new((ParamBuilder::query::<&A>(), ParamBuilder::query::<&B>())),
    ]),
    // `FilteredResources` and `FilteredResourcesMut` are built with a callback
    // that supplies a builder to configure the resource access.
    FilteredResourcesMutParamBuilder::new(|builder| { builder.add_read::<R>(); }),
);

let system = param_builder
    .build_state(&mut world)
    .build_system(buildable_system);

// The built system is just like any other system, and can be added to a schedule.
schedule.add_systems(system);
```

## State Scoped Events

{{ <heading_metadata authors={["@UkoeHB"]} prs={[15085]} /> }}

<!-- Add state scoped events -->
<!-- https://github.com/bevyengine/bevy/pull/15085 -->

State scoped events will be automatically cleared when exiting a state (similar to [StateScoped entities](https://bevy.org/news/bevy-0-14/#state-scoped-entities)). This is useful when you want to guarantee clean state transitions.

Normally, you would configure your event via:
```rust
fn setup(app: &mut App) {
    app.add_event::<MyGameEvent>();
}
```

If you want the events to be cleared when you exit a specific state, change this to:
```rust
fn setup(app: &mut App) {
    app.add_state_scoped_event::<MyGameEvent>(GameState::Play);
}
```

## `EntityRefExcept` and `EntityMutExcept`

{{ <heading_metadata authors={["@pcwalton"]} prs={[15207]} /> }}

<!-- Add `EntityRefExcept` and `EntityMutExcept` world queries, in preparation for generalized animation. -->
<!-- https://github.com/bevyengine/bevy/pull/15207 -->

<!-- TODO -->

[`EntityMut`] and [`EntityRef`] are powerful tools for interacting with all components of a given entity at once in arbitrary ways.
These types implement `QueryData`, so you can add them to any `Query` you'd like!

However, because they can access *any* component information, Rust's prohibition against mutable aliasing prevent you from simultaneously accessing other component information,
even if you pinky promise not to read any data that's being written to.

```rust
// This system is forbidden!
// 
// Inside the body of the function, we could choose to mutate the `AnimationPlayer` itself
// while reading its value!
fn animate_anything(query: Query<(&AnimationPlayer, EntityMut)> ){}
```

To let you work around this limitation, we've introduced a matching pair of tools: [`EntityMutExcept`] and [`EntityRefExcept`],
which work just like the [`EntityMut`] and [`EntityRef`] but *don't* provide access to a bundle of components that you declare off-limits.

```rust
/// Look mom, no mutable aliasing!
fn animate_anything(query: Query<(&AnimationPlayer, EntityMutExcept<AnimationPlayer>)> ){}
```

[`EntityMut`]: https://docs.rs/bevy/0.15/bevy/ecs/prelude/struct.EntityMut.html
[`EntityRef`]:https://docs.rs/bevy/0.15/bevy/ecs/prelude/struct.EntityRef.html
[`EntityMutExcept`]: https://docs.rs/bevy/0.15/bevy/ecs/world/struct.EntityMutExcept.html
[`EntityRefExcept`]: https://docs.rs/bevy/0.15/bevy/ecs/world/struct.EntityRefExcept.html

## Cached One-shot Systems

{{ <heading_metadata authors={["@benfrankel"]} prs={[14920]} /> }}

**Bevy 0.15** introduces a convenient new "cached" API for running one-shot systems:

```rust
// Old, uncached API:
let foo_id = commands.register_system(foo);
commands.run_system(foo_id);

// New, cached API:
commands.run_system_cached(foo);
```

This allows you to call `register_system_cached` without needing to worry about producing duplicate systems.

```rust
// Uncached API:
let id1 = world.register_system(quux);
let id2 = world.register_system(quux);
assert!(id1 != id2);

// Cached API:
let id1 = world.register_system_cached(quux);
let id2 = world.register_system_cached(quux);
assert!(id1 == id2);
```

### Comparison to `run_system_once`

`run_system_once` sets up a system, runs it once, and tears it down. This means system parameters like `Local` and `EventReader` that rely on persistent state between runs will be lost. Any system parameters like `Query` that rely on cached computations to improve performance will have to rebuild their cache each time, which can be costly. As a consequence, `run_system_once` is only recommended for diagnostic use (e.g. unit tests), and `run_system` or `run_system_cached` should be preferred for "real" code.

### Limitations

With the cached API, different systems cannot be cached under the same `CachedSystemId<S>`. There can be no more than one distinct system of type `S`. This is true when `size_of::<S>() == 0`, which is almost always true in practice. To enforce correctness, the new API will give you a compile-time error if you try to use a non-zero-sized function (like a function pointer or a capturing closure).

## Fallible System Parameters

{{ <heading_metadata authors={["@MiniaczQ"]} prs={[15276, 15476, 15488]} /> }}

<!-- Reduce runtime panics through `SystemParam` validation -->
<!-- https://github.com/bevyengine/bevy/pull/15276 -->

<!-- `QuerySingle` family of system params -->
<!-- https://github.com/bevyengine/bevy/pull/15476 -->

<!-- `Populated` (query) system param -->
<!-- https://github.com/bevyengine/bevy/pull/15488 -->


In Bevy 0.14 and prior, the following code would panic:
```rust
#[derive(Resource)]
struct MyResource;

fn my_system(my_resource: Res<MyResource>) {}

fn main() {
    let mut app = App::new();
    app.add_plugins(DefaultPlugins)
    app.add_systems(my_system);
    // Panic here: `my_system` cannot fetch `MyResource`, because it was never added.
    app.run();
}
```
but in Bevy 0.15, `my_system` simply won't be executed and a warning will be logged.

This works for all system-based features:

- Systems and Observers will be skipped.
- Run Conditions will be skipped and return `false`.

Compound systems, like `system_a.pipe(system_b)`, are currently skipped if any required data is missing.

Pre-existing parameters which now benefit from this feature are: `Res` and `ResMut` as well as their siblings `NonSend` and `NonSendMut`.
Parameters that build on top of other parameters: tuples, `DynSystemParam` and `ParamSet` are considered present if and only if all of their system parameters are present.

Additionally, few new system params were introduced to simplify existing code:
- [`Single<D, F>`] - Works like `Query<D, F>::single`, fails if query contains 0 or more than 1 match,
- `Option<Single<D, F>>` - Works like `Query<D, F>::single`, fails if query contains more than 1 match,
- [`Populated<D, F>`] - Works like a `Query<D, F>`, fails if query contains no matches.

### Warnings

Fallible system params come with a primitive warning mechanic.
Currently, systems can behave in one of two ways:
- (default) warn exactly once,
- never warn.

The default can be changed as following:
```rust
// For systems
app.add_systems(my_system.never_param_warn());
// For observers
app.add_observer(my_observer.never_param_warn());
// For run conditions
app.add_systems(my_system.run_if(my_condition.never_param_warn()));
```

Let us know what other warning strategies you'd like!

[`Single<D, F>`]: https://docs.rs/bevy/0.15/bevy/ecs/prelude/struct.Single.html
[`Populated<D, F>`]: https://docs.rs/bevy/0.15/bevy/ecs/prelude/struct.Populated.html

## Passing Data Into Systems By Reference

{{ <heading_metadata authors={["@ItsDoot"]} prs={[15184]} /> }}

<!-- Support systems that take references as input -->
<!-- https://github.com/bevyengine/bevy/pull/15184 -->

System piping is a powerful (if relatively niche) tool to pass data directly from one system to another.
While this is useful for [error handling],
it's a general purpose tool for composing fragments of logic by gluing together matching inputs and outputs.

This machinery has since been repurposed for use with [one-shot systems],
allowing you to call [`World::run_system_with_input`] to evaluate systems with whatever input you supply,
and get the return value back out.
Great for writing tests!

However, this set of tools has always had a frustrating and confusing limitation: any data passed into a system must have a static lifetime.
This seems absurd; the data is passed directly from one owner to the next and the systems are run as if they were a single unit.

With the liberal application of some type magic pixie dust, this limitation has been lifted!

```rust
let mut world = World::new();

let mut value = 2;

// This always worked:
fn square(In(input): In<usize>) -> usize {
    input * input
}
value = world.run_system_with_input(value, square);

// Now possible:
fn square_ref(InRef(input): InRef<usize>) -> usize {
    *input * *input
}
value = world.run_system_with_input(&value, square_ref);

// Mutably:
fn square_mut(InMut(input): InMut<usize>) {
    *input *= *input;
}
world.run_system_with_input(&mut value, square_mut);
```

We're excited to see what you do with this newfound power.

[error handling]: https://github.com/bevyengine/bevy/blob/main/examples/ecs/system_piping.rs
[one-shot systems]: https://bevy.org/news/bevy-0-12/#one-shot-systems
[`World::run_system_with_input`]: https://docs.rs/bevy/0.15.0/bevy/ecs/prelude/struct.World.html#method.run_system_with_input

## List Components in QueryEntityError::QueryDoesNotMatch

{{ <heading_metadata authors={["@SpecificProtagonist"]} prs={[15435]} /> }}

When accessing an entity through a query fails due to mismatched components, the error now includes the names of the components the entity has:

```
QueryDoesNotMatch(0v1 with components Sprite, Transform, GlobalTransform, Visibility, InheritedVisibility, ViewVisibility, SyncToRenderWorld)
```

## `no_std` Progress

{{ <heading_metadata authors={["@bushrat011899"]} prs={[15281]} /> }}

<!-- Add `core` and `alloc` over `std` Lints -->
<!-- https://github.com/bevyengine/bevy/pull/15281 -->

Bevy relies heavily on Rust's [standard library](https://doc.rust-lang.org/std/), making it
challenging to use on embedded, niche platforms, and even certain consoles. But what if that
_wasn't_ the case?

We've undertaken a new initiative to challenge the reliance on the standard
library, with the eventual goal of providing a [`no_std`](https://docs.rust-embedded.org/book/intro/no-std.html)
compatible subset of Bevy which could be used on a much wider range of platforms.

The first very simple step is to enable a new set of lints:

* [`std_instead_of_core`](https://rust-lang.github.io/rust-clippy/master/index.html#std_instead_of_core)
* [`std_instead_of_alloc`](https://rust-lang.github.io/rust-clippy/master/index.html#std_instead_of_alloc)
* [`alloc_instead_of_core`](https://rust-lang.github.io/rust-clippy/master/index.html#alloc_instead_of_core)

For those unfamiliar with `no_std` Rust, the standard library, `std`, gets a lot of its functionality
from two smaller crates, [`core`](https://doc.rust-lang.org/core/) and [`alloc`](https://doc.rust-lang.org/alloc/).
The `core` crate is available on every Rust target with very few exceptions, providing the
fundamental infrastructure that the Rust language relies on, such as iterators, `Result`, and many more.
Complementing that the `alloc` crate provides access to allocation-related functionality, such as
`Vec`, `Box`, and `String`.

Rust's support for platforms follows a [three tiered policy](https://doc.rust-lang.org/rustc/platform-support.html),
where tier 1 is guaranteed to work and will always provide the `std` crate, and tiers 2 and 3 _may_
have the `std` crate, but often do not.
The reason for this is some platforms simply don't support the features the `std` crate requires,
such as a filesystem, networking, or threading.

But why should Bevy care about these platforms? When a new platform is added to Rust,
it is often lacking tier 1 support.
Even modern consoles such as the Nintendo Switch, PlayStation 5, or Xbox Series don't have tier 1
support due to non-disclosure agreements and platform specifics.
Adding `no_std` support to Bevy will make it easier for commercial teams developing for these
platforms to get started and stay up to date.

Beyond the commercially-relevant modern consoles, there is a vibrant community of embedded and retro
enthusiasts developing for platforms that may never support the standard library.
Crates such as [`agb`](https://crates.io/crates/agb) and [`psx`](https://crates.io/crates/psx) provide
support for developing games on the GameBoy Advance and PlayStation One respectively.
With `no_std` support in Bevy, users may be able to leverage the wider Rust ecosystem to run their
software on these platforms.

We're still a while away from true `no_std` support in Bevy, but the first few changes have already
been accepted, with many more lined up for the next release in 0.16.

If this work sounds interesting, check out the
[`no_std` tracking issue](https://github.com/bevyengine/bevy/issues/15460) on GitHub, where you can
find a list of pull requests, and even prototypes of Bevy running in `no_std` environments.

## `GltfMaterialName` Component

{{ <heading_metadata authors={["@Soulghost"]} prs={[13912]} /> }}

The glTF 3D model file format allows a single mesh to be associated with multiple materials. For example, a teapot may consist of a single mesh, yet each part may have a different material. When a single mesh is assigned multiple materials, it is divided into several primitive nodes, with each primitive assigned a unique material.

```json
{
  "meshes": [
    {
      "name": "Cube",
      "primitives": [
        {
          "attributes": { "POSITION": 0, "NORMAL": 1, "TEXCOORD_0": 2 },
          "indices": 3,
          "material": 0
        },
        {
          "attributes": { "POSITION": 4, "NORMAL": 5, "TEXCOORD_0": 6 },
          "indices": 7,
          "material": 1
        },
        {
          "attributes": { "POSITION": 8, "NORMAL": 9, "TEXCOORD_0": 10 },
          "indices": 11,
          "material": 2
        },
        {
          "attributes": { "POSITION": 12, "NORMAL": 13, "TEXCOORD_0": 14 },
          "indices": 15,
          "material": 3
        }
      ]
    }
  ]
}
```

In Bevy 0.14 and before, these primitives are named using the format "Mesh.Index", which complicates querying. A new component [GltfMaterialName](https://docs.rs/bevy/0.15/bevy/gltf/struct.GltfMaterialName.html) is now added to each primitive node that has a material, letting you quickly look up the primitive by using this component with the material name.

```rust
fn find_top_material_and_mesh(
    mut materials: ResMut<Assets<StandardMaterial>>,
    mut meshes: ResMut<Assets<Mesh>>,
    time: Res<Time>,
    mesh_materials: Query<(
        &MeshMaterial3d<StandardMaterial>,
        &Mesh3d,
        &GltfMaterialName,
    )>,
) {
    for (mat_handle, mesh_handle, name) in &mesh_materials {
        // locate the material and associated submesh by name
        if name.0 == "Top" {
            if let Some(material) = materials.get_mut(mat_handle) {
                // ...
            }

            if let Some(mesh) = meshes.get_mut(mesh_handle) {
                // ...
            }
        }
    }
}
```

## GPU Readback

{{ <heading_metadata authors={["@tychedelia"]} prs={[15419]} /> }}

The new `Readback` component simplifies the tricky process of getting data back from the GPU to the CPU using an observer-based API.

```rust
commands.spawn(Readback::buffer(buffer.clone())).observe(
    |trigger: Trigger<ReadbackComplete>| {
        let data = trigger.event().to_shader_type();
        // ...
    },
);
```

Normally, manually retrieving data from the GPU involves a lot of boilerplate and careful management of GPU resources. You have to deal with synchronization, ensure the GPU has finished processing, and handle copying data between memory spaces—which isn’t straightforward!

The new `Readback` component streamlines this process. When spawned into the main world, `Readback` will queue a `Handle<Image>` or `Handle<ShaderStorageBuffer>` to be asynchronously read and copied back from the GPU to CPU in a future frame where it will trigger a `ReadbackComplete` event containing the raw bytes of the resource.

This is especially useful for debugging, saving GPU-generated data, or performing CPU-side computations with results from the GPU. It’s perfect for scenarios where you need to analyze simulation data, capture rendered frames, or process large datasets on the GPU and retrieve the results for further use on the CPU.

## Android: Configurable `GameActivity` and `NativeActivity`

{{ <heading_metadata authors={["@Litttlefish"]} prs={[12095]} /> }}

<!-- Add features to switch `NativeActivity` and `GameActivity` usage -->
<!-- https://github.com/bevyengine/bevy/pull/12095 -->
Bevy now uses `GameActivity` as the default `Activity` for Android projects, replacing
`NativeActivity`. `NativeActivity` is still available, but has been placed behind a feature flag.

This change updates Bevy to a more modern Android stack, and includes an SDK minimum version bump to
[PlayStore's current version
requirement](https://developer.android.com/distribute/best-practices/develop/target-sdk). We've also
switched to a [`cargo-ndk`](https://docs.rs/crate/cargo-ndk/3.5.4) based build, which gives us more control by default. Gradle projects for both `GameActivity` and
`NativeActivity` are provided.

`GameActivity` brings with it improvements to game interaction (`SurfaceView` rendering, improved
touch and input handling), more frequent updates, and access to other parts of the
[JetPack](https://developer.android.com/jetpack) ecosystem. It is better placed to integrate with
Rust code without excessive JNI wrangling. You can read more about `GameActivity`
[here](https://developer.android.com/games/agdk/game-activity).

## Reflection Serialization Improvements

{{ <heading_metadata authors={["@MrGVSV", "@aecsocket"]} prs={[8611, 15482, 15548, 13888]} /> }}

#### Serialization with registry context

[`bevy_reflect`] provides a way to easily serialize and deserialize nearly any type that
implement [`Reflect`]. It does so by relying purely on the reflection APIs and the [`TypeRegistry`],
without having to know the type at compile-time.

However, sometimes serialization/deserialization for a type requires more explicit control.
In such cases, a custom `Serialize`/`Deserialize` implementation can be provided by registering
the [`ReflectSerialize`]/[`ReflectDeserialize`] type data for the type in the `TypeRegistry`.

This approach generally works well enough for most cases.
However, sometimes you want to handle the case for your type alone and continue using reflection
for the rest of the fields.
For example, you might want to serialize your type as a map that includes a few extra entries,
but you still want to use the reflection serializer for each value.

Unfortunately, not only does this not nest well within serializers,
but it also means you need to manually capture a reference to the `TypeRegistry` 
so that you can pass it down to the nested reflection serializers.
What this basically means is that you can't use custom logic along with reflection-based serialization.

Thankfully, Bevy 0.15 introduces the [`SerializeWithRegistry`] and [`DeserializeWithRegistry`] traits,
which work much like `Serialize` and `Deserialize` but with an additional `TypeRegistry` parameter.
This allows you to perform your custom logic while still being able to continue using reflection
for the rest.

```rust
impl SerializeWithRegistry for MyType {
    fn serialize<S>(&self, serializer: S, registry: &TypeRegistry) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        let state = serializer.serialize_map(None)?;

        // ...custom logic...

        state.serialize_entry(
            "data",
            // Continue using reflection-based serialization
            &ReflectSerializer::new(
                self.data,
                registry,
            ),
        )?;

        state.end()
    }
}
```

With your custom serialization and deserialization logic in place, you can then register the 
[`ReflectSerializeWithRegistry`] and [`ReflectDeserializeWithRegistry`] type data for your type
to have the reflection serializer/deserializer make use of your custom logic for all instances
of your type.

#### Reflect de/serializer processors

Alongside `SerializeWithRegistry` and `DeserializeWithRegistry`, a new tool has been added for users
who use the reflect machinery for de/serialization. When using the `ReflectSerializer` or
`ReflectDeserializer`, you can now use `with_processor` and pass in a *de/serializer processor*.
This processor allows you to override the de/serialization logic for specific values and specific
types, while also capturing any context you might need inside the processor itself.

The motivating example for this is being able to deserialize `Handle<T>`s properly inside an asset
loader when reflect-deserializing. Let's imagine that we have an asset that looks like this:

```rust
#[derive(Debug, Clone, Reflect)]
struct AnimationGraph {
    nodes: Vec<Box<dyn AnimationNode>>,
}

trait AnimationNode: Send + Sync + Reflect { /* .. */ }

#[derive(Debug, Clone, Reflect)]
struct ClipNode {
    clip: Handle<AnimationClip>
}

impl AnimationNode for ClipNode { /* .. */ }

#[derive(Debug, Clone, Reflect)]
struct AdjustSpeedNode {
    speed_multiplier: f32,
}

impl AnimationNode for AdjustSpeedNode { /* .. */ }
```

```ron
(
    animation_graph: (
        nodes: [
            {
                "my_app::animation::node::ClipNode": (
                    clip: "animations/run.anim.ron",
                )
            },
            {
                "my_app::animation::node::AdjustSpeedNode": (
                    speed_multiplier: 1.5,
                )
            }
        ]
    )
)
```

When we write an `AssetLoader` for this `AnimationGraph`, we have access to a `&mut LoadContext`
which we can use to start new asset load operations, and get a `Handle` to that asset. We can also
use the existing `ReflectDeserializer` to deserialize `Box<dyn AnimationNode>`s. However, when the
deserializer encounters a `Handle<AnimationClip>`, this will be deserialized as `Handle::default`
and no asset load will be kicked off, making the handle useless.

With a [`ReflectDeserializerProcessor`], we can pass in a processor which captures the
`&mut LoadContext` and, if it encounters a `Handle<T>`, it will kick off an asset load for `T`,
and assigns the result of that load to the field it's deserializing.

```rust
struct HandleProcessor<'a> {
    load_context: &'a mut LoadContext,
}

impl ReflectDeserializerProcessor for HandleProcessor<'_> {
    fn try_deserialize<'de, D>(
        &mut self,
        registration: &TypeRegistration,
        _registry: &TypeRegistry,
        deserializer: D,
    ) -> Result<Result<Box<dyn PartialReflect>, D>, D::Error>
    where
        D: Deserializer<'de>,
    {
        let Some(reflect_handle) = registration.data::<ReflectHandle>() else {
            // we don't want to deserialize this - give the deserializer back
            // and do default deserialization logic
            return Ok(Err(deserializer));
        };

        let asset_type_id = reflect_handle.asset_type_id();
        let asset_path = deserializer.deserialize_str(AssetPathVisitor)?;

        let handle: Handle<LoadedUntypedAsset> = self.load_context
            .loader()
            .with_dynamic_type(asset_type_id)
            .load(asset_path);
        Ok(Box::new(handle))
    }
}
```

Combined with [`ReflectSerializerProcessor`], this can be used to round-trip `Handle`s to/from string
asset paths.

The processors take priority over all other serde logic, including `De/SerializeWithRegistry`, so it
can be used to override any reflect serialization logic.

#### Contextual Serialization Errors

Sometimes when working with the reflection serializer and deserializer, 
it can be difficult to track down the source of an error. 
Since we can't tell whether a type can be serialized or not until runtime, 
an un-serializable type might slip into a type that is supposed to be serializable.

In Bevy 0.15, a new default [`debug`] feature has been added to the `bevy_reflect` crate,
which allows the serializers and deserializers to retain contextual information in order to
provide the type's "stack" when an error occurs.

These messages can be used to more easily track down the source of the error:

```
type `bevy_utils::Instant` did not register the `ReflectSerialize` type data. For certain types, this may need to be registered manually using `register_type_data` (stack: `bevy_time::time::Time<bevy_time::real::Real>` -> `bevy_time::real::Real` -> `bevy_utils::Instant`)
```

[`bevy_reflect`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/
[`Reflect`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/trait.Reflect.html
[`TypeRegistry`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/struct.TypeRegistry.html
[`ReflectSerialize`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/trait.ReflectSerialize.html
[`ReflectDeserialize`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/trait.ReflectDeserialize.html
[`SerializeWithRegistry`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/serde/trait.SerializeWithRegistry.html
[`DeserializeWithRegistry`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/serde/trait.DeserializeWithRegistry.html
[`ReflectSerializeWithRegistry`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/serde/trait.ReflectSerializeWithRegistry.html
[`ReflectDeserializeWithRegistry`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/serde/trait.ReflectDeserializeWithRegistry.html
[`ReflectDeserializerProcessor`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/serde/trait.ReflectDeserializerProcessor.html
[`ReflectSerializerProcessor`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/serde/trait.ReflectSerializerProcessor.html
[`debug`]: https://docs.rs/bevy_reflect/0.15/bevy_reflect/index.html#debug

## Simplified Multi-Entity Access

{{ <heading_metadata authors={["@ItsDoot"]} prs={[15614]} /> }}

<!-- Allow `World::entity` family of functions to take multiple entities and get multiple references back -->
<!-- https://github.com/bevyengine/bevy/pull/15614 -->

When using some of the more advanced features of Bevy's ECS, like hooks or exclusive systems, it's common to want to fetch entities straight out of a `World`:

```rust
#[derive(Component)]
#[component(on_add = on_foo_added)]
struct Foo;

fn on_foo_added(world: DeferredWorld, entity: Entity, _: ComponentId) {
    let has_foo = world.entity(entity);
    println!("{:?} has a Foo", has_foo.id());
}
```

In previous versions of Bevy, you could grab multiple entities from a `World` using a variety of different functions:

- `World::many_entities<N>(&self, [Entity; N]) -> [EntityRef; N]`
- `World::many_entities_mut<N>(&mut self, [Entity; N]) -> [EntityMut; N]`
- `World::get_many_entities<N>(&self, [Entity; N]) -> Result<[EntityRef; N], Entity>`
- `World::get_many_entities_dynamic(&self, &[Entity]) -> Result<Vec<EntityRef>, Entity>`
- `World::get_many_entities_mut<N>(&mut self, [Entity; N]) -> Result<[EntityMut; N], QueryEntityError>`
- `World::get_many_entities_dynamic_mut(&self, &[Entity]) -> Result<Vec<EntityMut>, QueryEntityError>`
- `World::get_many_entities_from_set_mut(&mut self, &EntityHashSet) -> Result<Vec<EntityMut>, QueryEntityError>`

As you can see, that's a lot of functions with very long names! But the gist of them is that we want to support the ability to
give a bunch of entity IDs, and receive a bunch of entity references. Surely there's a better way!

In `0.15`, all of those functions have been deprecated and now all you need is
the panicking `World::entity`/`World::entity_mut` or the non-panicking `World::get_entity`/`World::get_entity_mut`:

```rust
let e1: Entity = world.spawn_empty().id();
let e2: Entity = world.spawn_empty().id();

// Note: use World::get_entity or World::get_entity_mut instead to receive a Result

// You can still pass a single ID as normal:
let eref = world.entity(e1);  
let emut = world.entity_mut(e1);

// But you can also pass in an array of IDs (any amount N supported!):
let [eref1, eref2]: [EntityRef; 2] = world.entity([e1, e2]);
let [emut1, emut2]: [EntityMut; 2] = world.entity_mut([e1, e2]);

// Or a slice of IDs:
let ids = vec![e1, e2];
let eref_vec: Vec<EntityRef> = world.entity(&ids);
let emut_vec: Vec<EntityMut> = world.entity_mut(&ids);

// Or even a set of IDs:
let ids = EntityHashSet::from_iter([e1, e2]);
let eref_map: EntityHashMap<EntityRef> = world.entity(&ids);
let emut_map: EntityHashMap<EntityMut> = world.entity_mut(&ids);
```

It might *feel* like magic, but it's all standard Rust code! The `Entity` id
parameter that the `World::entity` family of functions accept was changed to
instead accept anything that implements a newly introduced trait:
[`WorldEntityFetch`][WorldEntityFetch]. Check out the trait and
[`World::entity`][World::entity] to learn more about how it was accomplished.

[WorldEntityFetch]: https://docs.rs/bevy/0.15/bevy/ecs/world/trait.WorldEntityFetch.html
[World::entity]: https://docs.rs/bevy/0.15/bevy/ecs/prelude/struct.World.html#method.entity

## Hierarchy Traversal Tools

{{ <heading_metadata authors={["@alice-i-cecile"]} prs={[15627]} /> }}

<!-- Add more tools for traversing hierarchies -->
<!-- https://github.com/bevyengine/bevy/pull/15627 -->

We've spruced up the [`HierarchyQueryExt`] [extension trait],
making it easier to traverse entity hierarchies defined by the [`Parent`] and [`Children`] components.

The full set of methods is now:

- `parent` (new)
- `children` (new)
- `root_ancestor` (new)
- `iter_leaves` (new)
- `iter_siblings` (new)
- `iter_descendants`
- `iter_descendants_depth_first` (new)
- `iter_ancestors`

All of these operations were previously possible, but we hope that this API makes working with hierarchies more pleasant, especially for UI and animation.

[`HierarchyQueryExt`]: https://docs.rs/bevy/0.15/bevy/hierarchy/trait.HierarchyQueryExt.html
[`Parent`]: https://docs.rs/bevy/0.15/bevy/hierarchy/struct.Parent.html
[`Children`]: https://docs.rs/bevy/0.15/bevy/hierarchy/struct.Children.html
[extension trait]: https://rust-lang.github.io/rfcs/0445-extension-trait-conventions.html

## Shader Storage Buffer Asset

{{ <heading_metadata authors={["@tychedelia"]} prs={[14663]} /> }}

A new asset `ShaderStorageBuffer` has been added to simplify working with storage buffers in custom materials and compute shaders. Storage buffers are large, GPU-accessible memory buffers designed for storing data that can be read from or written to by shaders. Unlike smaller, more restricted uniform buffers, storage buffers allow you to store large amounts of data, making them perfect for general purpose tasks where large datasets need to be processed. Examples include managing complex data in physics simulations (like particle systems), holding the transformation data for thousands of objects in a scene, or storing procedural geometry information for dynamic terrain generation. Storage buffers are particularly useful when different stages of the rendering pipeline (such as compute shaders and rendering passes) need to share and update large amounts of data efficiently.

```rust
#[derive(Asset, TypePath, AsBindGroup, Debug, Clone)]
struct CustomMaterial {
    #[storage(0, read_only)]
    colors: Handle<ShaderStorageBuffer>,
}

fn setup(
    mut buffers: ResMut<Assets<ShaderStorageBuffer>>,
    mut materials: ResMut<Assets<CustomMaterial>>,
) {
    // Example data for the storage buffer
    let color_data: Vec<[f32; 4]> = vec![
        [1.0, 0.0, 0.0, 1.0],
        [0.0, 1.0, 0.0, 1.0],
        [0.0, 0.0, 1.0, 1.0],
        [1.0, 1.0, 0.0, 1.0],
        [0.0, 1.0, 1.0, 1.0],
    ];

    let colors = buffers.add(ShaderStorageBuffer::from(color_data));

    // Create the custom material with the storage buffer
    let custom_material = CustomMaterial { colors };

    materials.add(custom_material);
}
```

By declaring `Handle<ShaderStorageBuffer>` on the material using `AsBindGroup`, this buffer can now be accessed in the shader:

```wgsl
@group(2) @binding(0) var<storage, read> colors: array<vec4<f32>, 5>;
```

## Accumulated Mouse Inputs

{{ <heading_metadata authors={["@Aztro-dev", "@alice-i-cecile"]} prs={[14044]} /> }}

<!-- Accumulated mouse inputs -->
<!-- https://github.com/bevyengine/bevy/pull/14044 -->

"How much has the player moved their mouse this frame" is a natural question for games when the player is trying to aim or scroll a map.
Unfortunately, the operating system, and thus [`winit`], only provides us with a stream of events, in the form of individual [`MouseMotion`] events.

To get the summarized information (and the equivalent [`MouseScroll`]) information that most game systems care about, you had to sum them yourself.

```rust
pub fn accumulate_mouse_motion_system(
    mut mouse_motion_event: EventReader<MouseMotion>,
    mut accumulated_mouse_motion: ResMut<AccumulatedMouseMotion>,
) {
    let mut delta = Vec2::ZERO;
    for event in mouse_motion_event.read() {
        delta += event.delta;
    }
    accumulated_mouse_motion.delta = delta;
}
```

Bevy now does this for you, exposed in the new [`AccumulatedMouseMotion`] and [`AccumulatedMouseScroll`] resources.

[`winit`]: https://docs.rs/winit/latest/winit/
[`MouseMotion`]: https://docs.rs/bevy/0.15.0/bevy/input/mouse/struct.MouseMotion.html
[`MouseScroll`]: https://docs.rs/bevy/0.15.0/bevy/input/mouse/struct.MouseScroll.html
[`AccumulatedMouseMotion`]: https://docs.rs/bevy/0.15.0/bevy/input/mouse/struct.AccumulatedMouseMotion.html
[`AccumulatedMouseScroll`]: https://docs.rs/bevy/0.15.0/bevy/input/mouse/struct.AccumulatedMouseScroll.html

## Stable Interpolation and Smooth Following

{{ <heading_metadata authors={["@mweatherley"]} prs={[13741]} /> }}

When animating cameras or programming unit AI (not that kind of AI!), moving something continuously towards a target is an essential basic operation.
Simply [lerping] to the target seems easy enough, but as [Freya Holmer explains],
making sure that this interpolation is timestep independent is both vital and surprisingly tricky.

We've done the math for you; you just need to use the [`StableInterpolate`] trait's `interpolate_stable` and `smooth_nudge` methods
and tune the `decay_rate` parameter to really optimize your _game feel_.
Fear not: it even works on quaternions!
Stable, smooth camera controllers have never been easier.

[lerping]: https://en.wikipedia.org/wiki/Linear_interpolation
[Freya Holmer explains]: https://www.youtube.com/watch?v=LSNQuFEDOyQ
[`StableInterpolate`]: https://docs.rs/bevy/0.15/bevy/math/trait.StableInterpolate.html

## What's Next?

The features above may be great, but what else does Bevy have in flight?
Peering deep into the mists of time (predictions are _extra_ hard when your team is almost all volunteers!), we can see some exciting work taking shape:

- **Bevy Scene Notation:** Required components mark the first step on Cart's [master plan](https://github.com/bevyengine/bevy/discussions/14437) for BSN. Over the next few months, he's going to be heads-down developing a Bevy-specific file format (complete with matching macro and IDE support), the `Construct` trait (to easily include asset data in scenes), patches (to layer modifications to scenes) and experimenting with approaches to reactivity for UI.
- **Better font support:** While `cosmic_text` is a huge leap forward for text shaping and rendering, our approach to handling fonts and type-faces is still quite crude. Bidirectional text, working with system fonts, a convenient Markdown-style "bold this section of the text" API, font fallback and more are planned.
- **Picking-Powered UI Interaction:** `bevy_picking` introduces a much more powerful and expressive way to handle pointer interactions, but we're [not leveraging its full power](https://github.com/bevyengine/bevy/issues/15550) within `bevy_ui` itself. While picking events are great, a single source of truth for "what's the user doing with this button" is vital for responsive widget styling.
- **`bevy_lint`:** Try as we might, it _is_ possible to misuse Bevy's API! As part of a broader [`bevy_cli`](https://github.com/theBevyFlock/bevy_cli) project, the Bevy community has developed a Bevy-specific linter to catch common mistakes or hazards and are looking for early adopters to try it out!
- **Focus abstraction:** Keeping track of which UI element is focused is vital to allow users of screen readers, gamepads and keyboards to comfortably navigate the UI. We're planning to build on our success with `bevy_picking` and develop a complementary [focus-tracking solution](https://github.com/bevyengine/bevy/issues/15378), along with a few simple backends to opt-in to keyboard or gamepad-based UI navigation.
- **Immutable components:** Component hooks and observers are really powerful for responding to changes and upholding invariants, but they're easily bypassed by simply mutating the component. The mad science crew has been [experimenting with](https://github.com/bevyengine/bevy/issues/16208) a way to opt-out of direct mutation, opening the door to more robust hierarchies, complex observer-powered reactions and a first-party component indexing solution.
- **Actually Retained Rendering:** While the render world is _technically_ retained in Bevy 0.15, most of our existing code still spawns and despawns entities every frame to reduce the risk of introducing bugs during the migration. We're looking forward to gradually changing this and profiling the performance impact!
- **`no_std` Bevy:** To better support weird platforms (like the [Playdate](https://play.date/)!) and make life easier for devs experimenting with Bevy on modern consoles, we've been [working towards](https://github.com/bevyengine/bevy/issues/15460) ensuring that (much of) Bevy can compile and run without Rust's standard library.

{{ <support_bevy /> }}
{{ <contributors version="0.15" /> }}
{{ <changelog version="0.15" /> }}
