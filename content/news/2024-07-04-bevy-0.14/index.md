+++
title = "Bevy 0.14"
date = 2024-07-04
[extra]
image = "cover.jpg"
show_image = true
image_subtitle = "A forested scene illustrating Bevy's new volumetric fog, depth of field, and screen-space reflections"
image_subtitle_link = "https://github.com/IceSentry/bevy_forest_scene"
+++

Thanks to **256** contributors, **993** pull requests, community reviewers, and our [**generous donors**](/donate), we're happy to announce the **Bevy 0.14** release on [crates.io](https://crates.io/crates/bevy)!

For those who don't know, Bevy is a refreshingly simple data-driven game engine built in Rust. You can check out our [Quick Start Guide](/learn/quick-start) to try it today. It's free and open source forever! You can grab the full [source code](https://github.com/bevyengine/bevy) on GitHub. Check out [Bevy Assets](https://bevy.org/assets) for a collection of community-developed plugins, games, and learning resources.

To update an existing Bevy App or Plugin to **Bevy 0.14**, check out our [0.13 to 0.14 Migration Guide](/learn/migration-guides/0-13-to-0-14/).

Since our last release a few months ago we've added a _ton_ of new features, bug fixes, and quality of life tweaks, but here are some of the highlights:

- **Virtual Geometry**: Preprocess meshes into "meshlets", enabling efficient rendering of huge amounts of geometry
- **Sharp Screen Space Reflections**: Approximate real time raymarched screen space reflections
- **Depth of Field**: Cause objects at specific depths to go "out of focus", mimicking the behavior of physical lenses
- **Per-Object Motion Blur**: Blur objects moving fast relative to the camera
- **Volumetric Fog / Lighting**: Simulates fog in 3d space, enabling lights to produce beautiful "god rays"
- **Filmic Color Grading**: Fine tune tonemapping in your game with a complete set of filmic color grading tools
- **PBR Anisotropy**: Improve rendering of surfaces whose roughness varies along the tangent and bitangent directions of a mesh, such as brushed metal and hair
- **Auto Exposure**: Configure cameras to dynamically adjust their exposure based on what they are looking at
- **PCF for Point Lights**: Smooth out point light shadows, improving their quality
- **Animation blending:** Our new low-level animation graph adds support for animation blending, and sets the stage for first- and third-party graphical, asset-driven animation tools.
- **ECS Hooks and Observers:** Automatically (and immediately) respond to arbitrary events, such as component addition and removal
- **Better colors:** type-safe colors make it clear which color space you're operating in, and offer an awesome array of useful methods.
- **Computed states and substates:** Modeling complex app state is a breeze with these type-safe extensions to our `States` abstraction.
- **Rounded corners:** Rounding off one of `bevy_ui`'s roughest edges, you can now procedurally set the corner radius on your UI elements.

For the first time, Bevy 0.14 was prepared using a **release candidate** process to help ensure that you can upgrade right away with peace of mind.
We've worked closely with both plugin authors and ordinary users to catch critical bugs, round the rough corners off our new features, and refine the migration guide.
As we prepared fixes, we've [shipped new release candidates on crates.io](https://crates.io/crates/bevy/versions?sort=date), letting core ecosystem crates update and listening closely for show-stopping problems.
Thank you so much to [everyone who helped out](https://discord.com/channels/691052431525675048/1239930965267054623): these efforts are a vital step towards making Bevy something that teams large and small can trust to work reliably.

<!-- more -->

## Virtual Geometry (Experimental)

{{ <heading_metadata authors={["@JMS55", "@atlv24", "@zeux", "@ricky26"]} prs={["10164"]} /> }}

After several months of hard work, we're super excited to bring you the experimental release of a new virtual geometry feature!

This new rendering feature works much like Unreal Engine 5's Nanite renderer. You can take a very high-poly mesh, preprocess it to generate a [`MeshletMesh`] during build time, and then at runtime render huge amounts of geometry - much more than Bevy's standard renderer can support. No explicit LODs are needed - it's all automatic, and near seamless.

This feature is still a WIP, and comes with several constraints compared to Bevy's standard renderer, so be sure to read the docs and report any bugs you encounter. We still have a lot left to do, so look forward to more performance improvements (and associated breaking changes) in future releases!

Note that this feature does not use GPU "mesh shaders", so older GPUs are compatible for now. However, they are not recommended, and are likely to become unsupported in the near future.

In addition to the below user guide, checkout:

* [The Bevy example for this feature](https://github.com/bevyengine/bevy/blob/release-0.14.0/examples/3d/meshlet.rs)
* [The technical deep dive article by the main author of this feature](https://jms55.github.io/posts/2024-06-09-virtual-geometry-bevy-0-14)

<video controls loop><source src="many_bunnies.mp4" type="video/mp4"/></video>

Users wanting to use virtual geometry should compile with the `meshlet` cargo feature at runtime, and `meshlet_processor` cargo feature at build time for preprocessing meshes into the special meshlet-specific format ([`MeshletMesh`]) the meshlet renderer uses.

Enabling the meshlet feature unlocks a new module: [`bevy::pbr::experimental::meshlet`].

First step, add [`MeshletPlugin`] to your app:

```rust
app.add_plugins(MeshletPlugin);
```

Next, preprocess your [`Mesh`] into a [`MeshletMesh`]. Currently, this needs to be done manually via `MeshletMesh::from_mesh()`(again, you need the `meshlet_processor` feature enabled). This step is fairly slow, and should be done once ahead of time, and then saved to an asset file. Note that there are limitations on the types of meshes and materials supported, make sure to read the docs.

Automatic GLTF/scene conversions via Bevy's asset preprocessing system is planned, but unfortunately did not make the cut in time for this release. For now, you'll have to come up with your own asset conversion and management system. If you come up with a good system, let us know!

Now, spawn your entities. In the same vein as `MaterialMeshBundle`, there's a `MaterialMeshletMeshBundle`, which uses a [`MeshletMesh`] instead of the typical [`Mesh`].

```rust
commands.spawn(MaterialMeshletMeshBundle {
    meshlet_mesh: meshlet_mesh_handle.clone(),
    material: material_handle.clone(),
    transform,
    ..default()
});
```

Lastly, a note on materials. Meshlet entities use the same [`Material`] trait as regular mesh entities, however, the standard material methods are not used. Instead there are 3 new methods: `meshlet_mesh_fragment_shader`, `meshlet_mesh_prepass_fragment_shader`, and `meshlet_mesh_deferred_fragment_shader`. All 3 methods of forward, forward with prepasses, and deferred rendering are supported.

Notice however that there is no access to vertex shaders. Meshlet rendering uses a hardcoded vertex shader that cannot be changed.

The actual fragment shader code for meshlet materials are mostly the same as fragment shaders for regular mesh entities. The key difference is that instead of this:

```rust
@fragment
fn fragment(vertex_output: VertexOutput) -> @location(0) vec4<f32> {
    // ...
}
```

You should use this:

```rust
#import bevy_pbr::meshlet_visibility_buffer_resolve::resolve_vertex_output

@fragment
fn fragment(@builtin(position) frag_coord: vec4<f32>) -> @location(0) vec4<f32> {
    let vertex_output = resolve_vertex_output(frag_coord);
    // ...
}
```

[`MeshletMesh`]: https://docs.rs/bevy/0.14/bevy/pbr/experimental/meshlet/struct.MeshletMesh.html
[`Mesh`]: https://docs.rs/bevy/0.14/bevy/prelude/struct.Mesh.html
[`bevy::pbr::experimental::meshlet`]: https://docs.rs/bevy/0.14/bevy/pbr/experimental/meshlet/index.html
[`Material`]: https://docs.rs/bevy/0.14/bevy/pbr/trait.Material.html
[`MeshletPlugin`]: https://docs.rs/bevy/0.14/bevy/pbr/experimental/meshlet/struct.MeshletPlugin.html

## Sharp Screen-Space Reflections

{{ <heading_metadata authors={["@pcwalton"]} prs={["13418"]} /> }}

<!-- Implement opt-in sharp screen-space reflections for the deferred renderer, with improved raymarching code. -->
<!-- https://github.com/bevyengine/bevy/pull/13418 -->

{{ <compare_slider left_title="No SSR"
    left_image="no_ssr.jpg"
    right_title="SSR"
    right_image="ssr.jpg" /> }}

[Screen-space reflections](https://lettier.github.io/3d-game-shaders-for-beginners/screen-space-reflection.html) (SSR) approximate real-time reflections by raymarching through the depth buffer and copying samples from the final rendered frame.
Our initial implementation is relatively minimal, to provide a flexible base to build on, but is based on the production-quality [raymarching code by Tomasz Stachowiak](https://gist.github.com/h3r2tic/9c8356bdaefbe80b1a22ae0aaee192db), one of the creators of the indie darling Bevy game [Tiny Glade](https://store.steampowered.com/app/2198150/Tiny_Glade/).
As a result, there are a few caveats to bear in mind:

1. Currently, this feature is built on top of the deferred renderer and is currently only supported in that mode. Forward screen-space reflections are possible albeit uncommon (though e.g. Doom Eternal uses them); however, they require tracing from the previous frame, which would add complexity. This patch leaves the door open to implementing SSR in the forward rendering path but doesn't itself have such an implementation.
2. Screen-space reflections aren't supported in WebGL 2, because they require sampling from the depth buffer, which `naga` can't do because of a bug (`sampler2DShadow` is incorrectly generated instead of `sampler2D`; this is the same reason why depth of field is disabled on that platform).
3. No temporal filtering or blurring is performed at all. For this reason, SSR currently only operates on very low-roughness / smooth surfaces.
4. We don't perform acceleration via the hierarchical Z-buffer and reflections are traced at full resolution. As a result, you may notice performance issues depending on your scene and hardware.

To add screen-space reflections to a camera, insert the [`ScreenSpaceReflectionsSettings`] component.
In addition to [`ScreenSpaceReflectionsSettings`], [`DepthPrepass`], and [`DeferredPrepass`] must also be present for the reflections to show up.
Conveniently, the [`ScreenSpaceReflectionsBundle`] bundles these all up for you!
While the [`ScreenSpaceReflectionsSettings`] comes with sensible defaults, it also contains several settings that artists can tweak.

[`ScreenSpaceReflectionsBundle`]: https://docs.rs/bevy/0.14/bevy/pbr/struct.ScreenSpaceReflectionsBundle.html
[`ScreenSpaceReflectionsSettings`]:https://docs.rs/bevy/0.14/bevy/pbr/struct.ScreenSpaceReflectionsSettings.html
[`DepthPrepass`]: https://docs.rs/bevy/0.14/bevy/core_pipeline/prepass/struct.DepthPrepass.html
[`DeferredPrepass`]: https://docs.rs/bevy/0.14/bevy/core_pipeline/prepass/struct.DeferredPrepass.html

## Volumetric Fog and Volumetric Lighting (light shafts / god rays)

{{ <heading_metadata authors={["@pcwalton"]} prs={["13057"]} /> }}

Not all fog is created equal.
Bevy's existing implementation covers [distance fog](https://en.wikipedia.org/wiki/Distance_fog), which is fast, simple, and not particularly realistic.

In Bevy 0.14, this is supplemented with volumetric fog, based on [volumetric lighting](https://en.wikipedia.org/wiki/Volumetric_lighting), which simulates fog using actual 3D space, rather than simply distance from the camera.
As you might expect, this is both prettier and more computationally expensive!

In particular, this allows for the creation of stunningly beautiful "god rays" (more properly, crepuscular rays) shining through the fog.

{{ <compare_slider left_title="Without Volumetric Fog"
    left_image="without_volumetric_fog.jpg"
    right_title="With Volumetric Fog"
    right_image="with_volumetric_fog.jpg" /> }}

Bevy's algorithm, which is implemented as a postprocessing effect, is a combination of the techniques described in [Scratchapixel](https://www.scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/intro-volume-rendering.html) and [Alexandre Pestana's blog post](https://www.alexandre-pestana.com/volumetric-lights/). It uses raymarching ([ported to WGSL by h3r2tic](https://gist.github.com/h3r2tic/9c8356bdaefbe80b1a22ae0aaee192db)) in screen space, transformed into shadow map space for sampling and combined with physically-based modeling of absorption and scattering. Bevy employs the widely-used Henyey-Greenstein phase function to model asymmetry; this essentially allows light shafts to fade into and out of existence as the user views them.

To add volumetric fog to a scene, add [`VolumetricFogSettings`] to the camera, and add [`VolumetricLight`] to directional lights that you wish to be volumetric. [`VolumetricFogSettings`] has numerous settings that allow you to define the accuracy of the simulation, as well as the look of the fog. Currently, only interaction with directional lights that have shadow maps is supported. Note that the overhead of the effect scales directly with the number of directional lights in use, so apply [`VolumetricLight`] sparingly for the best results.

Try it hands on with our [`volumetric_fog` example](https://github.com/bevyengine/bevy/tree/v0.14.0/examples/3d/volumetric_fog.rs).

[`VolumetricFogSettings`]: https://docs.rs/bevy/0.14/bevy/pbr/struct.VolumetricFogSettings.html
[`VolumetricLight`]: https://docs.rs/bevy/0.14/bevy/pbr/struct.VolumetricLight.html

## Per-Object Motion Blur

{{ <heading_metadata authors={["@aevyrie", "@torsteingrindvik"]} prs={["9924"]} /> }}

We've added a post-processing effect that blurs fast-moving objects in the direction of motion.
Our implementation uses motion vectors, which means it works with Bevy's built in PBR materials, skinned meshes, or anything else that writes motion vectors and depth.
The effect is used to convey high speed motion, which can otherwise look like flickering or teleporting when the image is perfectly sharp.

Blur scales with the motion of objects relative to the camera.
If the camera is tracking a fast moving object, like a vehicle, the vehicle will remain sharp, while stationary objects will be blurred.
Conversely, if the camera is pointing at a stationary object, and a fast moving vehicle moves through the frame, only the fast moving object will be blurred.

The implementation is configured with [camera shutter angle](https://en.wikipedia.org/wiki/Rotary_disc_shutter), which corresponds to how long the virtual shutter is open during a frame.
In practice, this means the effect scales with framerate, so users running at high refresh rates aren't subjected to over-blurring.

<video controls loop><source src="motion_blur_cars.mp4" type="video/mp4"/></video>

You can enable motion blur by adding [`MotionBlurBundle`](https://docs.rs/bevy/0.14/bevy/core_pipeline/motion_blur/struct.MotionBlurBundle.html) to your camera entity, as shown in our [`motion blur` example](https://github.com/bevyengine/bevy/tree/v0.14.0/examples/3d/motion_blur.rs).

## Filmic Color Grading

{{ <heading_metadata authors={["@pcwalton"]} prs={["13121"]} /> }}

<!-- Implement filmic color grading. -->
<!-- https://github.com/bevyengine/bevy/pull/13121 -->

Artists want to get exactly the right look for their game, and color plays a huge role.

To support this, Bevy's [existing tonemapping tools](https://bevy.org/news/bevy-0-10/#more-tonemapping-choices) have been extended to include a complete set of filmic color grading tools. In addition to a [base tonemap](https://docs.rs/bevy/0.14/bevy/core_pipeline/tonemapping/enum.Tonemapping.html), you can now configure:

- White point adjustment. This is inspired by Unity's implementation of the feature, but simplified and optimized. Temperature and tint control the adjustments to the x and y chromaticity values of CIE 1931. Following Unity, the adjustments are made relative to the D65 standard illuminant in the LMS color space.
- Hue rotation: converts the RGB value to HSV, alters the hue, and converts back.
- Color correction: allows the gamma, gain, and lift values to be adjusted according to the standard ASC CDL combined function. This can be done separately for shadows, midtones and highlights To avoid abrupt color changes, a small crossfade is used between the different sections of the image.

We've followed [Blender's](https://www.blender.org/) implementation as closely as possible to ensure that what you see in your modelling software matches what you see in the game.

![A very orange image of a test scene, with controls for exposure, temperature, tint and hue. Saturation, contrast, gamma, gain, and lift can all be configured for the highlights, midtones, and shadows separately.](filmic_color_grading.jpg)

We've provided a new, [`color_grading`](https://github.com/bevyengine/bevy/tree/v0.14.0/examples/3d/color_grading.rs) example, with a shiny GUI to change all the color grading settings.
Perfect for copy-pasting into your own game's dev tools and playing with the settings!
Note that these settings can all be changed at runtime: giving artists control over the exact mood of the scene, or shift it dynamically based on weather or time of day.

## Auto Exposure

{{ <heading_metadata authors={["@Kurble", "@alice-i-cecile"]} prs={["12792"]} /> }}

Since **Bevy 0.13**, you can [configure the EV100 of a camera](/news/bevy-0-13/#camera-exposure), which allows you to adjust the exposure of the camera in a physically based way. This also allows you to dynamically change the exposure values for various effects. However, this is a manual process and requires you to adjust the exposure values yourself.

**Bevy 0.14** introduces **Auto Exposure**, which automatically adjusts the exposure of your camera based on the brightness of the scene. This can be useful when you want to create the feeling of a very high dynamic range, since your eyes also adjust to large changes in brightness. Note that this is not a replacement for hand-tuning the exposure values, rather an additional tool that you can use to create dramatic effects when brightness changes rapidly. Check out this video recorded from the [example](https://github.com/bevyengine/bevy/tree/v0.14.0/examples/3d/auto_exposure.rs) to see it in action!

<video controls><source src="auto_exposure.mp4" type="video/mp4"/></video>

Bevy's Auto Exposure is implemented by making a **histogram** of the scene's brightness in a post processing step. The exposure is then adjusted based on the average of the histogram. Because the histogram is calculated using a compute shader, Auto Exposure is **not available on WebGL**. It's also not enabled by default, so you need to add the [`AutoExposurePlugin`](https://docs.rs/bevy/0.14/bevy/core_pipeline/auto_exposure/struct.AutoExposurePlugin.html) to your app.

Auto Exposure is controlled by the [`AutoExposureSettings`](https://docs.rs/bevy/0.14/bevy/core_pipeline/auto_exposure/struct.AutoExposureSettings.html) component, which you can add to your camera entity. You can configure a few things:

* A relative *range* of F-stops that the exposure can change by.
* The *speed* at which the exposure changes.
* An optional **metering mask**, which allows you to, for example, give more weight to the center of the image.
* An optional histogram *filter*, which allows you to ignore very bright or very dark pixels.

## Fast Depth of Field

{{ <heading_metadata authors={["@pcwalton", "@alice-i-cecile", "@Kurble"]} prs={["13009"]} /> }}

In rendering, **depth of field** is an effect that mimics the [limitations of physical lenses](https://en.wikipedia.org/wiki/Depth_of_field).
By virtue of the way light works, lenses (like that of the human eye or a film camera) can only focus on objects that are within a specific range (depth) from them, causing all others to be blurry and out of focus.

Bevy now ships with this effect, implemented as a post-processing shader.
There are two options available: a fast Gaussian blur or a more physically accurate hexagonal bokeh technique.
The bokeh blur is generally more aesthetically pleasing than the Gaussian blur, as it simulates the effect of a camera more accurately. The shape of the bokeh circles are determined by the number of blades of the aperture. In our case, we use a hexagon, which is usually considered specific to lower-quality cameras.

{{ <compare_slider left_title="No Depth of Field"
    left_image="no_dof.jpg"
    right_title="Bokeh Depth of Field"
    right_image="bokeh_dof.jpg" /> }}

The blur amount is generally specified by the [f-number](https://en.wikipedia.org/wiki/F-number), which we use to compute the [focal length](https://en.wikipedia.org/wiki/Focal_length) from the film size and [field-of-view](https://en.wikipedia.org/wiki/Field_of_view). By default, we simulate standard cinematic cameras with an f/1 f-number and a film size corresponding to the classic Super 35 film format. The developer can customize these values as desired.

To see how this new API, please check out the dedicated [`depth_of_field` example](https://github.com/bevyengine/bevy/tree/v0.14.0/examples/3d/depth_of_field.rs).

## PBR Anisotropy

{{ <heading_metadata authors={["@pcwalton"]} prs={["13450"]} /> }}

<!-- Implement PBR anisotropy per `KHR_materials_anisotropy`. -->
<!-- https://github.com/bevyengine/bevy/pull/13450 -->

[Anisotropic materials](https://en.wikipedia.org/wiki/Anisotropy) change based on the axis of motion, such as how wood behaves very differently when working with versus against the grain.
But in the context of physically-based rendering, **anisotropy** refers specifically to a feature that allows roughness to vary along the tangent and bitangent directions of a mesh.
In effect, this causes the specular light to stretch out into lines instead of a round lobe. This is useful for modeling brushed metal, hair, and similar surfaces.
Support for anisotropy is a common feature in major game and graphics engines; Unity, Unreal, Godot, three.js, and Blender all support it to varying degrees.

{{ <compare_slider left_title="Without Anisotropy"
    left_image="without_anisotropy.jpg"
    right_title="With Anisotropy"
    right_image="with_anisotropy.jpg" /> }}

Two new parameters have been added to [`StandardMaterial`](https://docs.rs/bevy/0.14/bevy/pbr/struct.StandardMaterial.html): `anisotropy_strength` and `anisotropy_rotation`.
Anisotropy strength, which ranges from 0 to 1, represents how much the roughness differs between the tangent and the bitangent of the mesh.
In effect, it controls how stretched the specular highlight is. Anisotropy rotation allows the roughness direction to differ from the tangent of the model.

In addition to these two fixed parameters, an anisotropy texture can be supplied, in the linear texture format specified by `KHR_materials_anisotropy`.

Like always, give it a spin at the corresponding [`anisotropy` example](https://github.com/bevyengine/bevy/tree/v0.14.0/examples/3d/anisotropy.rs).

## Percentage-Closer Filtering (PCF) for Point Lights

{{ <heading_metadata authors={["@pcwalton"]} prs={["12910"]} /> }}

Percentage-closer filtering is a standard anti-aliasing technique used to get softer, less jagged shadows.
To do so, we sample from the shadow map near the pixel of interest using a Gaussian kernel, averaging the results to reduce sudden transitions as we move in / out of the shadow.

As a result, Bevy's point lights now  look softer and more natural, without any changes to end user code. As before, you can configure the exact strategy used to anti-alias your shadows by setting the [`ShadowFilteringMethod`](https://docs.rs/bevy/0.14/bevy/pbr/enum.ShadowFilteringMethod.html) component on your 3D cameras.

{{ <compare_slider left_title="Without PCF filtering"
    left_image="before_pcf.jpg"
    right_title="With PCF filtering"
    right_image="after_pcf.jpg" /> }}

Full support for percentage-closer shadows is [in the works](https://github.com/bevyengine/bevy/pull/13497): testing and reviews for this are, like always, extremely welcome.

## Subpixel Morphological Antialiasing (SMAA)

{{ <heading_metadata authors={["@pcwalton", "@alice-i-cecile"]} prs={["13423"]} /> }}

<!-- Implement subpixel morphological antialiasing, or SMAA. -->
<!-- https://github.com/bevyengine/bevy/pull/13423 -->

Jagged edges are the bane of game developers' existence: a wide variety of anti-aliasing techniques have been invented and are still in use to fix them without degrading image quality.
In addition to [MSAA](https://en.wikipedia.org/wiki/Multisample_anti-aliasing), [FXAA](https://en.wikipedia.org/wiki/Fast_approximate_anti-aliasing), and [TAA](https://en.wikipedia.org/wiki/Temporal_anti-aliasing), Bevy now implements [SMAA](https://en.wikipedia.org/wiki/Morphological_antialiasing): subpixel morphological antialiasing.

SMAA is a 2011 antialiasing technique that detects borders in the image, then averages nearby border pixels, eliminating the dreaded jaggies.
Despite its age, it's been a continual staple of games for over a decade. Four quality presets are available: low, medium, high, and ultra. Due to advancements in consumer hardware, Bevy's default is high.

You can see how it compares to no anti-aliasing in the pair of images below:

{{ <compare_slider left_title="No AA"
    left_image="no_aa.jpg"
    right_title="SMAA"
    right_image="smaa.jpg" /> }}

The best way to get a sense for the tradeoffs of the various anti-aliasing methods is to experiment with a test scene using the [`anti_aliasing` example](https://github.com/bevyengine/bevy/tree/v0.14.0/examples/3d/anti_aliasing.rs) or by simply trying it out in your own game.

## Visibility Ranges (hierarchical levels of detail / HLODs)

{{ <heading_metadata authors={["@pcwalton", "@cart"]} prs={["12916"]} /> }}

<!-- Implement visibility ranges, also known as hierarchical levels of detail (HLODs). -->
<!-- https://github.com/bevyengine/bevy/pull/12916 -->

When looking at objects far away, it's hard to make out the details!
This obvious fact is just as true in rendering as it is in real life.
As a result, using complex, high-fidelity models for distant objects is a waste: we can replace their meshes with simplified equivalents.

By automatically varying the **level-of-detail** (LOD) of our models in this way, we can render much larger scenes (or the same open world with a higher draw distance), swapping out meshes on the fly based on their proximity to the player.
Bevy now supports one of the most foundational tools for this: **visibility ranges** (sometimes called hierarchical levels of detail, as it allows users to replace multiple meshes with a single object).

By setting the [`VisibilityRange`] component on your mesh entities, developers can automatically control the range from the camera at which their meshes will appear and disappear, automatically fading between the two options using dithering.
Hiding meshes happens early in the rendering pipeline, so this feature can be efficiently used for level of detail optimization.
As a bonus, this feature is properly evaluated per-view, so different views can show different levels of detail.

Note that this feature differs from proper mesh LODs (where the geometry itself is simplified automatically), which will come later.
While mesh LODs are useful for optimization and don't require any additional setup, they're less flexible than visibility ranges.
Games often want to use objects other than meshes to replace distant models, such as octahedral or [billboard](https://github.com/bevyengine/bevy/issues/3688) imposters: implementing visibility ranges first gives users the flexibility to start implementing these solutions today.

You can see how this feature is used in the [`visibility_range` example](https://github.com/bevyengine/bevy/tree/v0.14.0/examples/3d/visibility_range.rs).

[`VisibilityRange`]: https://docs.rs/bevy/0.14/bevy/render/view/struct.VisibilityRange.html

## ECS Hooks and Observers

{{ <heading_metadata authors={["@james-j-obrien", "@cart"]} prs={["10839"]} /> }}

<!-- Hooks: https://github.com/bevyengine/bevy/pull/10756 -->
<!-- Observers: https://github.com/bevyengine/bevy/pull/10839 -->

As much as we love churning through homogeneous blocks of data in a tight loop here at Bevy, not every task is a perfect fit for the straightforward ECS model.
Responding to changes and/or processing events are vital tasks in any application, and games are no exception.

Bevy already has a number of distinct tools to handle this:

- **Buffered [`Event`]s**: Multiple-producer, multiple-consumer queues. Flexible and efficient, but requires regular polling as part of a schedule. Events are dropped after two frames.
- **Change detection via [`Added`] and [`Changed`]**: Enable writing queries that can respond to added or changed components. These queries linearly scan the change state of components that match the query to see if they have been added or changed.
- **[`RemovedComponents`]**: A special form of event that is triggered when a component is removed from an entity, or an entity with that component is despawned.

All of these (and systems themselves!) use a ["pull"-style mechanism]: events are sent regardless of whether or not anyone is listening, and listeners must periodically poll to ask if anything has changed.
This is a useful pattern, and one we intend to keep around!
By polling, we can process events in batch, getting more context and improving data locality (which makes the CPU go *brr*).

But it comes with some limitations:

- There is an unavoidable delay between an event being triggered and the response being processed
- Polling introduces a small (but non-zero) overhead every frame

This delay is the critical problem:

- Data (like indexes or hierarchies) can exist, even momentarily, in an invalid state
- We can't process arbitrary chain of events of recursive logic within a single cycle

To overcome these limitations, **Bevy 0.14** introduces **Component Lifecycle Hooks** and **Observers**: two complementary "push"-style mechanisms inspired by the ever-wonderful [flecs] ECS.

#### Component Lifecycle Hooks

[Component Hooks](https://docs.rs/bevy/0.14/bevy/ecs/component/struct.ComponentHooks.html) are functions (capable of interacting with the ECS World) registered for a specific component type (as part of the [`Component`] trait impl), which are run automatically in response to "component lifecycle events", such as when that component is added, overwritten, or removed.

For a given component type, only one hook can be registered for a given lifecycle event, and it cannot be overwritten.

Hooks exist to enforce invariants tied to that component (ex: maintaining indices or hierarchy correctness).
Hooks cannot be removed and always take priority over observers: they run before any on-add / on-insert observers, but after any on-remove observers.
As a result, they can be thought of as something closer to constructors & destructors, and are more suitable for maintaining critical safety or correctness invariants.
Hooks are also somewhat faster than observers, as their reduced flexibility means that fewer lookups are involved.

Let's examine a simple example where we care about maintaining invariants: one entity (with a `Target` component) targeting another entity (with a `Targetable` component).

```rust
#[derive(Component)]
struct Target(Option<Entity>);

#[derive(Component)]
struct Targetable {
    targeted_by: Vec<Entity>
};
```

We want to automatically clear the `Target` when the target entity is despawned: how do we do this?

If we were to use the pull-based approach (`RemovedComponents` in this case), there could be a delay between the entity being despawned and the `Target` component being updated. We can remove that delay with hooks!

Let's see what this looks like with a hook on `Targetable`:

```rust
// Rather than a derive, let's configure the hooks with a custom
// implementation of Component
impl Component for Targetable {
    const STORAGE_TYPE: StorageType = StorageType::Table;

    fn register_component_hooks(hooks: &mut ComponentHooks) {
        // Whenever this component is removed, or an entity with
        // this component is despawned...
        hooks.on_remove(|mut world, targeted_entity, _component_id|{
            // Grab the data that's about to be removed
            let targetable = world.get::<Targetable>(targeted_entity).unwrap();
            for targeting_entity in targetable.targeted_by {
                // Track down the entity that's targeting us
                let mut targeting = world.get_mut::<Target>(targeting_entity).unwrap();
                // And clear its target, cleaning up any dangling references
                targeting.0 = None;
            }
        })
    }
}
```

#### Observers

Observers are on-demand systems that listen to "triggered" events. These events can be triggered for specific entities *or* they can be triggered "globally" (no entity target).

In contrast to hooks, observers are a flexible tool intended for higher level application logic. They can watch for when user-defined events are triggered.

```rust
#[derive(Event)]
struct Message {
    text: String
}

world.observe(|trigger: Trigger<Message>| {
    println!("{}", trigger.event().message.text);
});
```

Observers are run *immediately* when an event they are watching for is triggered:

```rust
// All registered `Message` observers are immediately run here
world.trigger(Message { text: "Hello".to_string() });
```

If an event is triggered via a [`Command`], the observers will run when the [`Command`] is flushed:

```rust
fn send_message(mut commands: Commands) {
    // This will trigger all `Message` observers when this system's commands are flushed
    commands.trigger(Message { text: "Hello".to_string() } );
}
```

Events can also be triggered with an entity target:

```rust
#[derive(Event)]
struct Resize { size: usize }

commands.trigger_targets(Resize { size: 10 }, some_entity);
```

You can trigger an event for more than one entity at the same time:

```rust
commands.trigger_targets(Resize { size: 10 }, [e1, e2]);
```

A "global" observer will be executed when *any* target is triggered:

```rust
fn main() {
    App::new()
        .observe(on_resize)
        .run()
}

fn on_resize(trigger: Trigger<Resize>, query: Query<&mut Size>) {
    let size = query.get_mut(trigger.entity()).unwrap();
    size.value = trigger.event().size;
} 
```

Notice that observers can use system parameters like [`Query`], just like a normal system.

You can also add observers that only run for *specific* entities:

```rust
commands
    .spawn(Widget)
    .observe(|trigger: Trigger<Resize>| {
        println!("This specific widget entity was resized!");
    });
```

Observers are actually just an entity with the [`Observer`](https://docs.rs/bevy/0.14/ecs/observer/struct.Observer.html) component. All of the `observe()` methods used above are just shorthand for spawning a new observer entity. This is what a "global" observer entity looks like:

```rust
commands.spawn(Observer::new(|trigger: Trigger<Message>| {}));
```

Likewise, an observer watching a specific entity looks like this

```rust
commands.spawn(
    Observer::new(|trigger: Trigger<Resize>| {})
        .with_entity(some_entity)
);
```

This API makes it easy to manage and clean up observers. It also enables advanced use cases, such as sharing observers across multiple targets!

Now that we know a bit about observers, lets examine the API through a simple gameplay-flavored example:

<details>
<summary>Click to expand...</summary>

```rust
use bevy::prelude::*;

#[derive(Event)]
struct DealDamage {
    damage: u8,
}

#[derive(Event)]
struct LoseLife {
    life_lost: u8,
}

#[derive(Event)]
struct PlayerDeath;

#[derive(Component)]
struct Player;

#[derive(Component)]
struct Life(u8);

#[derive(Component)]
struct Defense(u8);

#[derive(Component, Deref, DerefMut)]
struct Damage(u8);

#[derive(Component)]
struct Monster;

fn main() {
    App::new()
        .add_systems(Startup, spawn_player)
        .add_systems(Update, attack_player)
        .observe(on_player_death);
}

fn spawn_player(mut commands: Commands) {
    commands
        .spawn((Player, Life(10), Defense(2)))
        .observe(on_damage_taken)
        .observe(on_losing_life);
}

fn attack_player(
    mut commands: Commands,
    monster_query: Query<&Damage, With<Monster>>,
    player_query: Query<Entity, With<Player>>,
) {
    let player_entity = player_query.single();

    for damage in &monster_query {
        commands.trigger_targets(DealDamage { damage: damage.0 }, player_entity);
    }
}

fn on_damage_taken(
    trigger: Trigger<DealDamage>,
    mut commands: Commands,
    query: Query<&Defense>,
) {
    let defense = query.get(trigger.entity()).unwrap();
    let damage = trigger.event().damage;
    let life_lost = damage.saturating_sub(defense.0);
    // Observers can be chained into each other by sending more triggers using commands.
    // This is what makes observers so powerful ... this chain of events is evaluated
    // as a single transaction when the first event is triggered.
    commands.trigger_targets(LoseLife { life_lost }, trigger.entity());
}

fn on_losing_life(
    trigger: Trigger<LoseLife>,
    mut commands: Commands,
    mut life_query: Query<&mut Life>,
    player_query: Query<Entity, With<Player>>,
) {
    let mut life = life_query.get_mut(trigger.entity()).unwrap();
    let life_lost = trigger.event().life_lost;
    life.0 = life.0.saturating_sub(life_lost);

    if life.0 == 0 && player_query.contains(trigger.entity()) {
        commands.trigger(PlayerDeath);
    }
}

fn on_player_death(_trigger: Trigger<PlayerDeath>, mut app_exit: EventWriter<AppExit>) {
    println!("You died. Game over!");
    app_exit.send_default();
}
```

</details>

In the future, we intend to use hooks and observers to [replace `RemovedComponents`], [make our hierarchy management more robust], create a first-party replacement for [`bevy_eventlistener`] as part of our UI work, and [build out relations].
These are powerful, general-purpose tools: we can't wait to see the mad science the community cooks up with them!

When you're ready to get started, check out the [`component hooks`] and [`observers`] examples for more API details.

[`Event`]: https://docs.rs/bevy/0.14/bevy/ecs/event/trait.Event.html
[`Added`]: https://docs.rs/bevy/0.14/bevy/ecs/prelude/struct.Added.html
[`Changed`]: https://docs.rs/bevy/0.14/bevy/ecs/prelude/struct.Changed.html
[`RemovedComponents`]: https://docs.rs/bevy/latest/bevy/ecs/prelude/struct.RemovedComponents.html
["pull"-style mechanism]: https://dev.to/anubhavitis/push-vs-pull-api-architecture-1djo
[flecs]: https://www.flecs.dev/flecs/
[replace `RemovedComponents`]: https://github.com/bevyengine/bevy/issues/13928
[make our hierarchy management more robust]: https://github.com/bevyengine/bevy/issues/12235
[`bevy_eventlistener`]: https://github.com/aevyrie/bevy_eventlistener
[build out relations]: https://github.com/bevyengine/rfcs/pull/79
[`component hooks`]: https://github.com/bevyengine/bevy/tree/v0.14.0/examples/ecs/component_hooks.rs
[`observers`]: https://github.com/bevyengine/bevy/tree/v0.14.0/examples/ecs/observers.rs
[`Component`]: https://docs.rs/bevy/0.14/bevy/ecs/component/trait.Component.html
[`Command`]: https://docs.rs/bevy/0.14/bevy/ecs/world/trait.Command.html
[`Query`]: https://docs.rs/bevy/0.14/bevy/ecs/prelude/struct.Query.html

## glTF KHR_texture_transform Support

{{ <heading_metadata authors={["@janhohenheim", "@yrns", "@Kanabenki"]} prs={["11904"]} /> }}

The GLTF extension `KHR_texture_transform` is used to transform a texture before applying it. By reading this extension, Bevy can now support a variety of new workflows.
The one we want to highlight here is the ability to easily repeat textures a set number of times. This is useful for creating textures that are meant to be tiled across a surface. We will show how to do this using Blender, but the same principles apply to any 3D modeling software.

Let's look at an example scene that we've prepared in Blender, exported as a GLTF file and loaded into Bevy. We will first use the most basic shader node setup available in Blender:

![Basic shader node setup](basic_nodes.jpg)

The result is the following scene in Bevy:

![Scene with stretched textures](bevy_no_rep.jpg)

Oh no! Everything is stretched! This is because we have set up our UVs in a way that maps the texture exactly once onto the mesh. There are a few ways to deal with this, but the most convenient is to add shader nodes that scale the texture so that it repeats:

![Repeating shader node setup](rep_nodes.jpg)

The data of the `Mapping` node is the one exported to `KHR_texture_transform`. Look at the part in red. These scaling factors determine how often the texture should be repeated in the material. Tweaking this value for all textures results in a much nicer render:

![Scene with repeated textures](bevy_rep.jpg)

## UI Node Border Radius

{{ <heading_metadata authors={["@chompaa", "@pablo-lua", "@alice-i-cecile", "@bushrat011899"]} prs={["12500"]} /> }}

Border radius for UI nodes has been a long-requested feature for Bevy. Now it's supported!

To apply border radius to a UI node, there is a new component [`BorderRadius`](https://docs.rs/bevy/0.14/bevy/prelude/struct.BorderRadius.html). [`NodeBundle`](https://docs.rs/bevy/0.14/bevy/prelude/struct.NodeBundle.html) and [`ButtonBundle`](https://docs.rs/bevy/0.14/bevy/prelude/struct.ButtonBundle.html) now have a field for this component called `border_radius`:

```rs
commands.spawn(NodeBundle {
    style: Style {
        width: Val::Px(50.0),
        height: Val::Px(50.0),
        // We need a border to round a border, after all!
        border: UiRect::all(Val::Px(5.0)),
        ..default()
    },
    border_color: BorderColor(Color::BLACK),
    // Apply the radius to all corners. 
    // Optionally, you could use `BorderRadius::all`.
    border_radius: BorderRadius {
        top_left: Val::Px(50.0),
        top_right: Val::Px(50.0),
        bottom_right: Val::Px(50.0),
        bottom_left: Val::Px(50.0),
    },
    ..default()
});
```

There's a [new example](https://github.com/bevyengine/bevy/tree/v0.14.0/examples/ui/rounded_borders.rs) showcasing this new API, a screenshot of which can be seen below:

![`rounded_borders` example](./rounded_borders.jpg)

## Animation Blending with the `AnimationGraph`

{{ <heading_metadata authors={["@pcwalton", "@rparrett", "@james7132"]} prs={["11989"]} /> }}

Through the eyes of a beginner, handling animation seems simple enough.
Define a series of keyframes which transform the various bits of your model to match those poses.
We slap some interpolation on there to smoothly move between them, and the user tells you when to start and stop the animation. Easy!

But modern animation pipelines (especially in 3D!) are substantially more complex:
animators expect to be able to smoothly blend and programmatically alter different animations dynamically in response to gameplay.
In order to capture this richness, the industry has developed the notion of an **animation graph**, which is used to couple the underlying [state machine] of a game object to the animations that should be playing, and the transitions that should occur between each of the various states.

A player character may be walking, running, slashing a sword, defending with a sword...
to create a polished effect, animators need to be able to change between these animations smoothly, change the speed of the walk cycle to match the movement speed along the ground and even perform multiple animations at once!

In Bevy 0.14, we've implemented the [Animation Composition RFC], providing a low-level API that brings code- and asset-driven animation blending to Bevy.

```rust
#[derive(Resource)]
struct ExampleAnimationGraph(Handle<AnimationGraph>);

fn programmatic_animation_graph(
    mut commands: Commands,
    asset_server: ResMut<AssetServer>,
    animation_graphs: ResMut<Assets<AnimationGraph>>,
) {
    // Create the nodes.
    let mut animation_graph = AnimationGraph::new();
    let blend_node = animation_graph.add_blend(0.5, animation_graph.root);
    animation_graph.add_clip(
        asset_server.load(GltfAssetLabel::Animation(0).from_asset("models/animated/Fox.glb")),
        1.0,
        animation_graph.root,
    );
    animation_graph.add_clip(
        asset_server.load(GltfAssetLabel::Animation(1).from_asset("models/animated/Fox.glb")),
        1.0,
        blend_node,
    );
    animation_graph.add_clip(
        asset_server.load(GltfAssetLabel::Animation(2).from_asset("models/animated/Fox.glb")),
        1.0,
        blend_node,
    );

    // Add the graph to our collection of assets.
    let handle = animation_graphs.add(animation_graph);

    // Hold onto the handle
    commands.insert_resource(ExampleAnimationGraph(handle));
}
```

While it can be used to great effect today, most animators will ultimately prefer editing these graphs with a GUI. We plan to build a GUI on top of this API as part of the fabled Bevy Editor. Today, there are also third party solutions like [`bevy_animation_graph`].

To learn more and see what the asset-driven approach looks like, take a look at the new [`animation_graph` example].

[state machine]: https://en.wikipedia.org/wiki/Finite-state_machine
[Animation Composition RFC]: https://github.com/bevyengine/rfcs/blob/main/rfcs/51-animation-composition.md
[`bevy_animation_graph`]: https://crates.io/crates/bevy_animation_graph
[`animation_graph` example]: https://github.com/bevyengine/bevy/tree/v0.14.0/examples/animation/animation_graph.rs

## Improved Color API

{{ <heading_metadata authors={["@viridia", "@mockersf"]} prs={["12013"]} /> }}

Colors are a huge part of building a good game: UI, effects, shaders and more all need fully-featured, correct and convenient color tools.

Bevy now supports a broad selection of color spaces, each with their own type (e.g. [`LinearRgba`], [`Hsla`], [`Oklaba`]),
and offers a wide range of fully documented operations on and conversions between them.

The new API is more error-resistant, more idiomatic and allows us to save work by storing the [`LinearRgba`] type in our rendering internals.
This solid foundation has allowed us to implement a wide range of useful operations, clustered into traits like [`Hue`] or [`Alpha`],
allowing you to operate over any color space with the required property.
Critically, color mixing / blending is now supported: perfect for procedurally generating color palettes and working with animations.

```rust
use bevy_color::prelude::*;

// Each color space now corresponds to a specific type
let red = Srgba::rgb(1., 0., 0.);

// All non-standard color space conversions are done through the shortest path between
// the source and target color spaces to avoid a quadratic explosion of generated code.
// This conversion...
let red = Oklcha::from(red);
// ...is implemented using
let red = Oklcha::from(Oklaba::from(LinearRgba::from(red)));

// We've added the `tailwind` palette colors: perfect for quick-but-pretty prototyping!
// And the existing CSS palette is now actually consistent with the industry standard :p
let blue = tailwind::BLUE_500;

// The color space that you're mixing your colors in has a huge impact!
// Consider using the scientifically-motivated `Oklcha` or `Oklaba` for a perceptually uniform effect.
let purple = red.mix(blue, 0.5);
```

Most of the user-facing APIs still accept a colorspace-agnostic [`Color`] (which now wraps our color-space types),
while rendering internals use the physically-based [`LinearRgba`] type.
For an overview of the different color spaces, and what they're each good for, please check out our [color space usage](https://docs.rs/bevy/0.14/bevy/color/index.html#color-space-usage) documentation.

`bevy_color` offers a solid, type-safe foundation, but it's just getting started.
If you'd like another color space or there are more things you'd like to do to your colors, please open an issue or PR and we'd be happy to help!

Also note that `bevy_color` is intended to operate effectively as a stand-alone crate: feel free to take a dependency on it for your non-Bevy projects as well.

[`LinearRgba`]: https://docs.rs/bevy/0.14/bevy/color/struct.LinearRgba.html
[`Hsla`]: https://docs.rs/bevy/0.14/bevy/color/struct.Hsla.html
[`Oklaba`]: https://docs.rs/bevy/0.14/bevy/color/struct.Oklaba.html
[`Hue`]: https://docs.rs/bevy/0.14/bevy/color/trait.Hue.html
[`Alpha`]: https://docs.rs/bevy/0.14/bevy/color/trait.Alpha.html
[`Color`]: https://docs.rs/bevy/0.14/bevy/color/enum.Color.html

## Extruded Shapes

{{ <heading_metadata authors={["@lynn-lumen"]} prs={["13270"]} /> }}

**Bevy 0.14** introduces an entirely new group of primitives: extrusions!

An extrusion is a 2D primitive (the base shape) that is *extruded* into a third dimension by some depth. The resulting shape is a prism (or in the special case of the circle, a cylinder).

```rust
// Create an ellipse with width 2 and height 1.
let my_ellipse = Ellipse::from_size(2.0, 1.0);

// Create an extrusion of this ellipse with a depth of 1.
let my_extrusion = Extrusion::new(my_ellipse, 1.);
```

All extrusions are extruded along the Z-axis. This guarantees that an extrusion of depth 0 and the corresponding base shape are identical, just as one would expect.

#### Measuring and Sampling

Since all extrusions with base shapes that implement [`Measured2d`] implement [`Measured3d`], you can easily get the surface area or volume of an extrusion.
If you have an extrusion of a custom 2D primitive, you can simply implement [`Measured2d`] for your primitive and [`Measured3d`] will be implemented automatically for the extrusion.

Likewise, you can sample the boundary and interior of any extrusion if the base shape of the extrusion implements [`ShapeSample<Output = Vec2>`](https://docs.rs/bevy/0.14/bevy/math/trait.ShapeSample.html) and [`Measured2d`].

```rust
// Create a 2D capsule with radius 1 and length 2, extruded to a depth of 3
let extrusion = Extrusion::new(Capsule2d::new(1.0, 2.0), 3.0);

// Get the volume of the extrusion
let volume = extrusion.volume();

// Get the surface area of the extrusion
let surface_area = extrusion.area();


// Create a random number generator
let mut rng = StdRng::seed_from_u64(4);

// Sample a random point inside the extrusion
let interior_sample = extrusion.sample_interior(&mut rng);

// Sample a random point on the surface of the extrusion
let boundary_sample = extrusion.sample_boundary(&mut rng);
```

#### Bounding

You can also get bounding spheres and Axis Aligned Bounding Boxes (AABBs) for extrusions. If you have a custom 2D primitive that implements [`Bounded2d`], you can simply implement [`BoundedExtrusion`]) for your primitive. The default implementation will give optimal results but may be slower than a solution fitted to your primitive.

#### Meshing

Extrusions do not exist in the world of maths only though. They can also be meshed and displayed on the screen!

And again, adding meshing support for your own primitives is made easy by Bevy! You simply need to implement meshing for your 2D primitive and then implement [`Extrudable`] for your 2D primitive's [`MeshBuilder`].

When implementing [`Extrudable`], you have to provide information about whether segments of the perimeter of the base shape are to be shaded smooth or flat, and what vertices belong to each of these perimeter segments.

![a 2D heart primitive and its extrusion](heart_extrusion.jpg)

The [`Extrudable`] trait allows you to easily implement meshing for extrusions of custom primitives. Of course, you could also implement meshing manually for your extrusion.

If you want to see a full implementation of this, you can check out the [custom primitives example](https://github.com/bevyengine/bevy/tree/v0.14.0/examples/math/custom_primitives.rs).

[`Measured2d`]: https://docs.rs/bevy/0.14/bevy/math/prelude/trait.Measured2d.html
[`Measured3d`]: https://docs.rs/bevy/0.14/bevy/math/prelude/trait.Measured3d.html
[`Extrudable`]: https://docs.rs/bevy/0.14/bevy/render/mesh/trait.Extrudable.html
[`Bounded2d`]: https://docs.rs/bevy/0.14/bevy/math/bounding/trait.Bounded2d.html
[`BoundedExtrusion`]: https://docs.rs/bevy/0.14/bevy/math/bounding/trait.BoundedExtrusion.html
[`MeshBuilder`]: https://docs.rs/bevy/0.14/bevy/prelude/trait.MeshBuilder.html

## More Gizmos

{{ <heading_metadata authors={["@mweatherley", "@Kanabenki", "@MrGVSV", "@solis-lumine-vorago", "@alice-i-cecile"]} prs={["12211"]} /> }}

Gizmos in Bevy allow developers to easily draw arbitrary shapes to help debugging or authoring content, but also to visualize specific properties of your scene, such has the AABB of your meshes.

In 0.14, several new gizmos have been added to [`bevy::gizmos`]:

#### Rounded box gizmos

Rounded boxes and cubes are great for visualizing regions and colliders.

If you set the `corner_radius` or `edge_radius` to a positive value, the corners will be rounded outwards. However, if you provide a negative value, the corners will flip and curve inwards.

![rounded gizmos cuboids](gizmos_rounded_cuboid.jpg)
![rounded gizmos rectangles](gizmos_rounded_rect.jpg)

#### Grid Gizmos

New grid gizmo types were added with [`Gizmos::grid_2d`] and [`Gizmos::grid`] to draw a plane grid in either 2D or 3D, alongside [`Gizmos::grid_3d`] to draw a 3D grid.

Each grid type can be skewed, scaled and subdivided along its axis, and you can separately control which outer edges to draw.

![Grid gizmos screenshot](grid_gizmos.jpg)

#### Coordinate Axes Gizmo

The new [`Gizmos::axes`] add a simple way to show the position, orientation and scale of any object from its [`Transform`] plus a base size.
The size of each axis arrow is proportional to the corresponding axis scale in the provided [`Transform`].

![Axes gizmo screenshot](axes_gizmo.jpg)

#### Light Gizmos

The new [`ShowLightGizmo`] component implements a retained gizmo to visualize lights for [`SpotLight`], [`PointLight`] and [`DirectionalLight`].
Most light properties are visually represented by the gizmos, and the gizmo color can be set to match the light instance or use a variety of other behaviors.

Similar to other retained gizmos, [`ShowLightGizmo`] can be configured per-instance or globally with [`LightGizmoConfigGroup`].

![Light gizmos screenshot](light_gizmos.jpg)

[`bevy::gizmos`]: https://docs.rs/bevy/0.14.0/bevy/gizmos/index.html
[`Gizmos::grid_2d`]: https://docs.rs/bevy/0.14.0/bevy/gizmos/prelude/struct.Gizmos.html#method.grid_2d
[`Gizmos::grid`]: https://docs.rs/bevy/0.14.0/bevy/gizmos/prelude/struct.Gizmos.html#method.grid
[`Gizmos::grid_3d`]: https://docs.rs/bevy/0.14.0/bevy/gizmos/prelude/struct.Gizmos.html#method.grid_3d
[`Gizmos::axes`]: https://docs.rs/bevy/0.14.0/bevy/gizmos/prelude/struct.Gizmos.html#method.axes
[`Transform`]: https://docs.rs/bevy/0.14.0/bevy/prelude/struct.Transform.html
[`ShowLightGizmo`]: https://docs.rs/bevy/0.14.0/bevy/gizmos/light/struct.ShowLightGizmo.html
[`SpotLight`]: https://docs.rs/bevy/0.14.0/bevy/pbr/struct.SpotLight.html
[`PointLight`]: https://docs.rs/bevy/0.14.0/bevy/pbr/struct.PointLight.html
[`DirectionalLight`]: https://docs.rs/bevy/0.14.0/bevy/pbr/struct.DirectionalLight.html
[`LightGizmoConfigGroup`]: https://docs.rs/bevy/0.14.0/bevy/gizmos/light/struct.LightGizmoConfigGroup.html

## Gizmo Line Styles and Joints

{{ <heading_metadata authors={["@lynn-lumen"]} prs={["12394"]} /> }}

Previous versions of Bevy supported drawing line gizmos:

```rust
fn draw_gizmos(mut gizmos: Gizmos) {
    gizmos.line_2d(Vec2::ZERO, Vec2::splat(-80.), RED);
}
```

However the only way to customize gizmos was to change their color, which may be limiting for some use cases. Additionally, the meeting points of two lines in a line strip, their *joints*, had little gaps.

As of Bevy 0.14, you can change the style of the lines and their joints for each gizmo config group:

```rust
fn draw_gizmos(mut gizmos: Gizmos) {
    gizmos.line_2d(Vec2::ZERO, Vec2::splat(-80.), RED);
}

fn setup(mut config_store: ResMut<GizmoConfigStore>) {
    // Get the config for you gizmo config group
    let (config, _) = config_store.config_mut::<DefaultGizmoConfigGroup>();
    // Set the line style and joints for this config group
    config.line_style = GizmoLineStyle::Dotted;
    config.line_joints = GizmoLineJoint::Bevel;
}
```

The new line styles can be used in both 2D and 3D and respect the `line_perspective` option of their config groups.

Available line styles are:

- `GizmoLineStyle::Dotted`: draws a dotted line with each dot being a square
- `GizmoLineStyle::Solid`: draws a solid line - this is the default behavior and the only one available before Bevy 0.14

![new gizmos line styles](gizmos_line_styles.jpg)

Similarly, the new line joints offer a variety of options:

- `GizmoLineJoint::Miter`, which extends both lines until they meet at a common miter point,
- `GizmoLineJoint::Round(resolution)`, which will approximate an arc filling the gap between the two lines. The `resolution` determines the amount of triangles used to approximate the geometry of the arc.
- `GizmoLineJoint::Bevel`, which connects the ends of the two joining lines with a straight segment, and
- `GizmoLineJoint::None`, which uses no joints and leaves small gaps - this is the default behavior and the only one available before Bevy 0.14.

![new gizmos line joints](gizmos_line_joints.jpg)

You can check out the [2D gizmos example](https://github.com/bevyengine/bevy/tree/v0.14.0/examples/gizmos/2d_gizmos.rs), which demonstrates the use of line styles and joints!

## UI Node Outline Gizmos

{{ <heading_metadata authors={["@pablo-lua", "@nicopap", "@alice-i-cecile"]} prs={["11237"]} /> }}

When working with UI on the web, being able to quickly debug the size of all your boxes is wildly useful.
We now have a native [layout tool](https://docs.rs/bevy/0.14/bevy/dev_tools/ui_debug_overlay/struct.DebugUiPlugin.html) which adds gizmos outlines to all [Nodes](https://docs.rs/bevy/0.14/bevy/ui/struct.Node.html)

An example of what the tool looks like after enabled

![Ui example with the overlay tool enabled](bevy_ui_outlines.jpg)

```rust
use bevy::prelude::*;

// You first have to add the DebugUiPlugin to your app
let mut app = App::new()
    .add_plugins(bevy::dev_tools::ui_debug_overlay::DebugUiPlugin);

// In order to enable the tool at runtime, you can add a system to toggle it
fn toggle_overlay(
    input: Res<ButtonInput<KeyCode>>,
    mut options: ResMut<bevy::dev_tools::ui_debug_overlay::UiDebugOptions>,
) {
    info_once!("The debug outlines are enabled, press Space to turn them on/off");
    if input.just_pressed(KeyCode::Space) {
        // The toggle method will enable the debug_overlay if disabled and disable if enabled
        options.toggle();
    }

}

// And add the system to the app
app.add_systems(Update, toggle_overlay);
```

## Contextually Clearing Gizmos

{{ <heading_metadata authors={["@Aceeri"]} prs={["10973"]} /> }}

Gizmos are drawn via an immediate mode API. This means that every **update** you draw all gizmos you want to display, and only those will be shown. Previously, update referred to "once every time the `Main` schedule runs". This matches the frame rate, so it usually works great! But when you try to draw gizmos during `FixedMain`, they will flicker or be rendered multiple times. In Bevy 0.14, this now just works!

This can be extended for use with custom schedules. Instead of a single storage, there now are multiple [storages](https://docs.rs/bevy/0.14/bevy/gizmos/gizmos/struct.GizmoStorage.html) differentiated by a context type parameter. You can also set a type parameter on the [`Gizmos`](https://docs.rs/bevy/0.14/bevy/gizmos/gizmos/struct.Gizmos.html) system param to choose what storage to write to. You choose when storages you add get drawn or cleared: Any gizmos in the default storage (the `()` context) during the `Last` schedule will be shown.

## Query Joins

{{ <heading_metadata authors={["@hymm"]} prs={["11535"]} /> }}

ECS Queries can now be combined, returning the data for entities that are contained in both queries.

```rust
fn helper_function(a: &mut Query<&A>, b: &mut Query<&B>){    
    let a_and_b: QueryLens<(Entity, &A, &B)> = a.join(b);
    assert!(a_and_b.iter().len() <= a.len());
    assert!(a_and_b.iter().len() <= b.len());
}
```

In most cases, you should continue to simply add more parameters to your original query. `Query<&A, &B>` will generally be clearer than joining them later.
But when a complex system or helper function backs you into a corner, query joins are there if you need them.

If you're familiar with database terminology, this is an ["inner join"](https://www.w3schools.com/sql/sql_join.asp).
Other types of query joins are being considered. Maybe you could take a crack at the [follow-up issue](https://github.com/bevyengine/bevy/issues/13633)?

## Computed States & Sub-States

{{ <heading_metadata authors={["@lee-orr", "@marcelchampagne", "@MiniaczQ", "@alice-i-cecile"]} prs={["11426"]} /> }}

Bevy's [`States`] are a simple but powerful abstraction for managing the control flow of your app.

But as users' games (and non-game applications!) grew in complexity, their limitations became more apparent.
What happens if we want to capture the notion of "in a menu", but then have different states corresponding to which submenu should be open?
What if we want to ask questions like "is the game paused", but that question only makes sense while we're within a game?

Finding a good abstraction for this required [several](https://github.com/bevyengine/bevy/pull/9957) [attempts](https://github.com/bevyengine/bevy/pull/10088) and a great deal of both experimentation and discussion.

While your existing [`States`] code will work exactly as before, there are now two additional tools you can reach for if you're looking for more expressiveness: **computed states** and **sub states**.

Let's begin with a simple state declaration:

```rust
#[derive(States, Clone, PartialEq, Eq, Hash, Debug, Default)]
enum GameState {
    #[default]
    Menu,
    InGame {
        paused: bool
    },
}
```

The addition of `pause` field means that simply checking for `GameState::InGame` doesn't work ... the states are different depending on its value and we may want to distinguish between game systems that run when the game is paused or not!

#### Computed States

While we can simply do `OnEnter(GameState::InGame{paused: true})`,
we need to be able to reason about "while we're in the game, paused or not".
To this end, we define the `InGame` computed state:

```rust
#[derive(Clone, PartialEq, Eq, Hash, Debug)]
struct InGame;

impl ComputedStates for InGame {
    // Computed states can be calculated from one or many source states.
    type SourceStates = GameState;

    // Now, we define the rule that determines the value of our computed state.
    fn compute(sources: GameState) -> Option<InGame> {
        match sources {
            // We can use pattern matching to express the
            //"I don't care whether or not the game is paused" logic!
            GameState::InGame {..} => Some(InGame),
            _ => None,
        }
    }
}
```

#### Sub-States

In contrast, sub-states should be used when you want to keep manual
control over the value through `NextState`, but still bind their
existence to some parent state.

```rust
#[derive(SubStates, Clone, PartialEq, Eq, Hash, Debug, Default)]
// This macro means that `GamePhase` will only exist when we're in the `InGame` computed state.
// The intermediate computed state is helpful for clarity here, but isn't required:
// you can manually `impl SubStates` for more control, multiple parent states and non-default initial value!
#[source(InGame = InGame)]
enum GamePhase {
    #[default]
    Setup,
    Battle,
    Conclusion
}
```

#### Initialization

Initializing our states is easy: just call the appropriate method on `App`
and all of the required machinery will be set up for you.

```rust
App::new()
   .init_state::<GameState>()
   .add_computed_state::<InGame>()
   .add_sub_state::<GamePhase>()
```

Just like any other state, computed states and substates work with all of the tools you're used to:
the `State` and `NextState` resources, `OnEnter`, `OnExit` and `OnTransition` schedules and the `in_state` run condition.
Make sure to visit [both](https://github.com/bevyengine/bevy/tree/v0.14.0/examples/state/computed_states.rs) [examples](https://github.com/bevyengine/bevy/tree/v0.14.0/examples/state/sub_states.rs) for more information!

The only exception is that, for correctness, computed states *cannot* be mutated through `NextState`.
Instead, they are strictly derived from their parent states; added, removed and updated automatically during state transitions based on the provided `compute` method.

All of Bevy's state tools are now found in a dedicated `bevy_state` crate, which can be controlled via a feature flag.
Yearning for the days of state stacks? Wish that there was a method for re-entering states?
All of the state machinery relies *only* on public ECS tools: resources, schedules, and run conditions, making it easy to build on top of.
We know that state machines are very much a matter of taste; so if our design isn't to your taste consider taking advantage of Bevy's modularity and writing your own abstraction or using one supplied by the community!

[`States`]: https://docs.rs/bevy/0.14/bevy/prelude/trait.States.html

## State Scoped Entities

{{ <heading_metadata authors={["@MiniaczQ", "@alice-i-cecile", "@mockersf"]} prs={["13649"]} /> }}

State scoped entities is a pattern that naturally emerged in community projects. **Bevy 0.14** has embraced it!

```rust
#[derive(Clone, Copy, PartialEq, Eq, Hash, Debug, Default, States)]
enum GameState {
 #[default]
  Menu,
  InGame,
}

fn spawn_player(mut commands: Commands) {
    commands.spawn((
        // We mark this entity with the `StateScoped` component.
        // When the provided state is exited, the entity will be
        // deleted recursively with all children.
        StateScoped(GameState::InGame)
        SpriteBundle { ... }
    ))
}

App::new()
    .init_state::<GameState>()
    // We need to install the appropriate machinery for the cleanup code
    // to run, once for each state type.
    .enable_state_scoped_entities::<GameState>()
    .add_systems(OnEnter(GameState::InGame), spawn_player);
```

By binding entity lifetime to a state during setup, we can dramatically reduce the amount of cleanup code we have to write!

## State Identity Transitions

{{ <heading_metadata authors={["@MiniaczQ", "@alice-i-cecile"]} prs={["13579"]} /> }}

Users have sometimes asked for us to trigger exit and entry steps when moving from a state to itself.
While this has its uses (refreshing is the core idea), it can be surprising and unwanted in other cases.
We've found a compromise that lets users hook into this type of transition if it's something they need.

`StateEventTransition` events will now include transitions from a state to itself,
which will also propagate to all dependent `ComputedStates` and `SubStates`.

Because it is a niche feature, `OnExit` and `OnEnter` schedules will ignore the new identity transitions by default,
but you can visit the new [`custom_transitions`](https://github.com/bevyengine/bevy/tree/v0.14.0/examples/state/custom_transitions.rs) example to see how you can bypass or change that behavior!

## GPU Frustum Culling

{{ <heading_metadata authors={["@pcwalton"]} prs={["12889"]} /> }}

<!-- GPU Frustum Culling: https://github.com/bevyengine/bevy/pull/12889-->

Bevy's rendering stack is often CPU-bound: by shifting more work onto the GPU, we can better balance the load and render more shiny things faster.
Frustum culling is an optimization technique that automatically hides objects that are outside of a camera's view (its frustum).
In Bevy 0.14, users can choose to have this work performed on the GPU, depending on the performance characteristics of their project.

Two new components are available to control frustum culling: `GpuCulling` and `NoCpuCulling`. Attach the appropriate combination of these
components to a camera, and you're set.

```rust
commands.spawn((
    Camera3dBundle::default(),
    // Enable GPU frustum culling (does not automatically disable CPU frustum culling).
    GpuCulling,
    // Disable CPU frustum culling.
    NoCpuCulling
));
```

## World Command Queue

{{ <heading_metadata authors={["@james7132", "@james-j-obrien"]} prs={["11823"]} /> }}

Working with [`Commands`] when you have exclusive world access has always been a pain.
Create a [`CommandQueue`], generate a [`Commands`] out of that, send your commands and then apply it?
Not exactly the most intuitive solution.

Now, you can access the [`World`]'s own command queue:

```rust
let mut world = World::new();
let mut commands = world.commands();
commands.spawn(TestComponent);
world.flush_commands();
```

While this isn't the most performant approach (just apply the mutations directly to the world and skip the indirection),
this API can be great for quickly prototyping with or easily testing your custom commands. It is also used internally to power component lifecycle hooks and observers.

As a bonus, one-shot systems now apply their commands (and other deferred system params) immediately when run!
We already have exclusive world access: why introduce delays and subtle bugs?

[`World`]: https://docs.rs/bevy/0.14/bevy/ecs/prelude/struct.World.html
[`Commands`]: https://docs.rs/bevy/0.14/bevy/ecs/prelude/struct.Commands.html
[`CommandQueue`]: https://docs.rs/bevy/0.14/bevy/ecs/world/struct.CommandQueue.html

## Reduced Multi-Threaded Execution Overhead

{{ <heading_metadata authors={["@chescock", "@james7132"]} prs={["11906"]} /> }}

The largest source of overhead in Bevy's multithreaded system executor is from
[thread context switching](https://en.wikipedia.org/wiki/Context_switch), i.e. starting and stopping threads.
Each time a thread is woken up it can take up to 30us if the cache for the thread is cold.
Minimizing these switches is an important optimization for the executor. In this cycle we landed
two changes that show improvements for this:

#### Run the multi-threaded executor at the end of each system task

The system executor is responsible for checking that the dependencies for a system have run already
and evaluating the run criteria and then running a task for that system.
The old version of the multithreaded executor ran as a separate task that was woken up after each task
completed. This would sometimes cause a new thread to be woken up for the executor to process the system completing.

By changing it so the system task tries to run the multithreaded executor after each system completes, we ensure that the multithreaded executor always runs on a thread that is already awake. This prevents one source of context switches. In practice this reduces the number of context switches per a `Schedule` run by 1-3 times, for an improvement of around 30us per schedule. When an app has many schedules, this can add up!

#### Combined Event update system

There used to be one
instance of the "event update system" for each event type. With just the `DefaultPlugins`, that results in 20+ instances of the system.

Each instance ran very quick, so the overhead of spawning the system tasks and waking up threads to run all these systems dominated the time it took for the `First` schedule to run. So combining all these into one system avoids this overhead and makes the `First` schedule run much faster. In testing this made running the schedule go from 140us to 25us. Again, not a *huge* win, but we're all about saving every microsecond we can!

## Decouple `BackgroundColor` from `UiImage`

{{ <heading_metadata authors={["@benfrankel"]} prs={["11165"]} /> }}

UI images can now be given solid background colors:

![UI image with background color](ui_image_background_color.jpg)

The [`BackgroundColor`] component now works for UI images instead of applying a color tint on the image itself. You can still apply a color tint by setting `UiImage::color`. For example:

```rust
commands.spawn((
    ImageBundle {
        image: UiImage {
            handle: assets.load("logo.png"),
            color: DARK_RED.into(),
            ..default()
        },
        ..default()
    },
    BackgroundColor(ANTIQUE_WHITE.into()),
    Outline::new(Val::Px(8.0), Val::ZERO, CRIMSON.into()),
));
```

[`BackgroundColor`]: https://docs.rs/bevy/0.14/bevy/prelude/struct.BackgroundColor.html

## Combined WinitEvent

{{ <heading_metadata authors={["@UkoeHB"]} prs={["12100"]} /> }}

When handling inputs, the exact ordering of the events received is often very significant, even when the events are not the same type!
Consider a simple drag-and-drop operation. When, exactly, did the user release the mouse button relative to the many tiny movements that they performed?
Getting these details right goes a long way to a responsive, precise user experience.

We now expose the blanket [`WinitEvent`](https://docs.rs/bevy/0.14/bevy/winit/enum.WinitEvent.html) event stream, in addition to the existing separated event streams, which can be read and matched on directly whenever these problems arise.

## Recursive Reflect Registration

{{ <heading_metadata authors={["@MrGVSV", "@soqb", "@cart", "@james7132"]} prs={["5781"]} /> }}

Bevy uses [reflection](https://docs.rs/bevy_reflect/latest/bevy_reflect/) in order to dynamically process data for things like serialization and deserialization.
A Bevy app has a `TypeRegistry` to keep track of which types exist.
Users can register their custom types when initializing the app or plugin.

```rust
#[derive(Reflect)]
struct Data<T> {
  value: T,
}

#[derive(Reflect)]
struct Blob {
  contents: Vec<u8>,
}

app
  .register_type::<Data<Blob>>()
  .register_type::<Blob>()
  .register_type::<Vec<u8>>()
```

In the code above, `Data<Blob>` depends on `Blob` which depends on `Vec<u8>`,
which means that all three types need to be manually registered—
even if we only care about `Data<Blob>`.

This is both tedious and error-prone, especially when these type dependencies are only
used in the context of other types (i.e. they aren't used as standalone types).

In 0.14, any type that derives `Reflect` will automatically register all of its type dependencies.
So when we register `Data<Blob>`, `Blob` will be registered as well (which will register `Vec<u8>`),
thus simplifying our registration down to a single line:

```rust
app.register_type::<Data<Blob>>()
```

Note that removing the registration for `Data<Blob>` now also means that `Blob` and `Vec<u8>` may
not be registered either, unless they were registered some other way.
If those types are needed as standalone types, they should be registered separately.

## `Rot2` Type for 2D Rotations

{{ <heading_metadata authors={["@Jondolf", "@IQuick143", "@tguichaoua"]} prs={["11658"]} /> }}

Ever wanted to work with rotations in 2D and get frustrated with having to choose between quaternions and a raw `f32`?
Us too!

We've added a convenient [`Rot2`] type for you, with plenty of helper methods.
Feel free to replace that helper type you wrote, and submit little PRs for any useful functionality we're missing.

[`Rot2`] is a great complement to the [`Dir2`] type (formerly `Direction2d`).
The former represents an angle, while the latter is a unit vector.
These types are similar but not interchangeable, and the choice of representation depends heavily on the task at hand. You can rotate a direction using `direction = rotation * Dir2::X`. To recover the rotation, use `Dir2::X::rotation_to(direction)` or in this case the helper `Dir2::rotation_from_x(direction)`.

While these types aren't used widely within the engine yet, we *are* aware of your pain
and are evaluating [proposals](https://github.com/bevyengine/rfcs/pull/82) for how we can make working with transforms in 2D more straightforward and pleasant.

[`Rot2`]: https://docs.rs/bevy/0.14/bevy/math/struct.Rot2.html
[`Dir2`]: https://docs.rs/bevy/0.14/bevy/math/struct.Dir2.html

## Alignment API for Transforms

{{ <heading_metadata authors={["@mweatherley"]} prs={["12187"]} /> }}

**Bevy 0.14** adds a new [`Transform::align`] function, which is a more general form of [`Transform::look_to`], which allows you to specify any local axis you want to use for the main and secondary axes.

This allows you to do things like point the front of a spaceship at a planet you're heading toward while keeping the right wing pointed in the direction of another ship. or point the top of a ship in the direction of the tractor beam pulling it in, while the front rotates to match the bigger ship's direction.

Lets consider a ship where we're going to use the front of the ship and the right wing as local axes:

![before calling Transform::align](align-before-move.jpg)

```rust
// point the local negative-z axis in the global Y axis direction
// point the local x-axis in the global Z axis direction
transform.align(Vec3::NEG_Z, Vec3::Y, Vec3::X, Vec3::Z)
```

`align` will move it to match the desired positions as closely as possible:

![after calling Transform::align](align-after-move.jpg)

 Note that not all rotations can be constructed and the [documentation](https://docs.rs/bevy/0.14/bevy/transform/components/struct.Transform.html#method.align) explains what happens in such scenarios.

[`Transform::look_to`]: https://docs.rs/bevy/0.14/bevy/transform/components/struct.Transform.html#method.look_to
[`Transform::align`]: https://docs.rs/bevy/0.14/bevy/transform/components/struct.Transform.html#method.align

## Random Sampling of Shapes and Directions

{{ <heading_metadata authors={["@13ros27", "@mweatherley", "@lynn-lumen"]} prs={["12484"]} /> }}

In the context of game development, it's often helpful to have access to random values, whether that's in the interest of driving behavior for NPCs, creating effects, or just trying to create variety. To help support this, a few random sampling features have been added to `bevy_math`, gated behind the `rand` feature. These are primarily geometric in nature, and they come in a couple of flavors.

First, one can sample random points from the boundaries and interiors of a variety of mathematical primitives:
![Image of several primitives side-by-side with points randomly sampled from their interiors][sampling-primitives]

In code, this can be performed in a couple different ways, using either the `sample_interior`/`sample_boundary` or `interior_dist`/`boundary_dist` APIs:

```rust
use bevy::math::prelude::*;
use rand::{Rng, SeedableRng};
use rand_chacha::ChaCha8Rng;

let sphere = Sphere::new(1.5);

// Instantiate an Rng:
let rng = &mut ChaCha8Rng::seed_from_u64(7355608);

// Using these, sample a random point from the interior of this sphere:
let interior_pt: Vec3 = sphere.sample_interior(rng);
// or from the boundary:
let boundary_pt: Vec3 = sphere.sample_boundary(rng);

// Or, if we want a lot of points, we can use a Distribution instead...
// to sample 100000 random points from the interior:
let interior_pts: Vec<Vec3> = sphere.interior_dist().sample_iter(rng).take(100000).collect();
// or 100000 random points from the boundary:
let boundary_pts: Vec<Vec3> = sphere.boundary_dist().sample_iter(rng).take(100000).collect();
```

Note that these methods explicitly require an [`Rng`](https://docs.rs/rand/0.8.5/rand/trait.Rng.html) object, giving you control over the randomization strategy and seed.

The currently supported shapes are as follows:

2D: `Circle`, `Rectangle`, `Triangle2d`, `Annulus`, `Capsule2d`.

3D: `Sphere`, `Cuboid`, `Triangle3d`, `Tetrahedron`, `Cylinder`, `Capsule3d`, and extrusions of sampleable 2D shapes (`Extrusion`).

---

Similarly, the direction types (`Dir2`, `Dir3`, `Dir3A`) and quaternions (`Quat`) can now be constructed randomly using `from_rng`:

```rust
use bevy::math::prelude::*;
use rand::{random, Rng, SeedableRng};
use rand_chacha::ChaCha8Rng;

// Instantiate an Rng:
let rng = &mut ChaCha8Rng::seed_from_u64(7355608);

// Get a random direction:
let direction = Dir3::from_rng(rng);

// Similar, but requires left-hand type annotations or inference:
let another_direction: Dir3 = rng.gen();

// Using `random` to grab a value using implicit thread-local rng:
let yet_another_direction: Dir3 = random();
```

[sampling-primitives]: sampling_primitives.jpg

## Tools for Profiling GPU Performance

{{ <heading_metadata authors={["@LeshaInc"]} prs={["9135"]} /> }}

While [Tracy](https://github.com/bevyengine/bevy/blob/main/docs/profiling.md) already lets us measure CPU time per system, our GPU diagnostics are much weaker.
In Bevy 0.14 we've added support for two classes of rendering-focused statistics via the [`RenderDiagnosticsPlugin`](https://docs.rs/bevy/0.14/bevy/render/diagnostic/struct.RenderDiagnosticsPlugin.html):

1. **Timestamp queries:** how long did specific bits of work take on the GPU?
2. **Pipeline statistics:** information about the quantity of work sent to the GPU.

While it may sound like timestamp queries are the ultimate diagnostic tool, they come with several caveats.
Firstly, they vary quite heavily from frame-to-frame as GPUs dynamically ramp up and down clock speed due to workload (idle gaps in GPU work, e.g., a bunch of consecutive barriers, or the tail end of a large dispatch) or the physical temperature of the GPU.
To get an accurate measurement, you need to look at summary statistics: mean, median, 75th percentile and so on.

Secondly, while timestamp queries will tell you how long something takes, but it will not tell you why things are slow.
For finding bottlenecks, you want to use a GPU profiler from your GPU vendor (Nvidia's NSight, AMD's RGP, Intel's GPA or Apple's XCode).
These tools will give you much more detailed stats about cache hit rate, warp occupancy, and so on.
On the other hand they lock your GPU's clock to base speeds for stable results, so they won't give you a good indicator of real world performance.

[`RenderDiagnosticsPlugin`] tracks the following pipeline statistics, recorded in Bevy's [`DiagnosticsStore`](https://docs.rs/bevy/0.14/bevy/diagnostic/struct.DiagnosticsStore.html): Elapsed CPU time, Elapsed GPU time, [Vertex shader](https://www.khronos.org/opengl/wiki/Vertex_Shader) invocations, [Fragment shader](https://www.khronos.org/opengl/wiki/Fragment_Shader) invocations, [Compute shader](https://www.khronos.org/opengl/wiki/Compute_Shader) invocations, [Clipper invocations](http://gpa.helpmax.net/en/intel-graphics-performance-analyzers-help/metrics-descriptions/extended-metrics-description/rasterizer-metrics/clipper-invocations/), and [Clipper primitives](http://gpa.helpmax.net/en/intel-graphics-performance-analyzers-help/metrics-descriptions/extended-metrics-description/rasterizer-metrics/post-clip-primitives/).

You can also track individual render/compute passes, groups of passes (e.g. all shadow passes), and individual commands inside passes (like draw calls).
 To do so, instrument them using methods from the [`RecordDiagnostics`](https://docs.rs/bevy/0.14/bevy/render/diagnostic/trait.RecordDiagnostics.html) trait.

[`RenderDiagnosticsPlugin`]: https://docs.rs/bevy/0.14/bevy/render/diagnostic/struct.RenderDiagnosticsPlugin.html

## New Geometric Primitives

{{ <heading_metadata authors={["@vitorfhc", "@Chubercik", "@andristarr", "@spectria-limina", "@salvadorcarvalhinho", "@aristaeus", "@mweatherley"]} prs={["12508"]} /> }}

Geometric shapes find a variety of applications in game development, ranging from rendering simple items to the screen for display / debugging to use in
colliders, physics, raycasting, and more.

For this, geometric shape primitives were [introduced in Bevy 0.13](https://bevy.org/news/bevy-0-13/#primitive-shapes), and work on this area has continued with Bevy 0.14, which brings the addition of 
[`Triangle3d`] and [`Tetrahedron`] 3D primitives, along with [`Rhombus`], [`Annulus`], [`Arc2d`], [`CircularSegment`], and [`CircularSector`] 2D 
primitives. As usual, these each have methods for querying geometric information like perimeter, area, and volume, and they all support meshing (where 
applicable) as well as gizmo display. 

[`Triangle3d`]: https://docs.rs/bevy/0.14.0/bevy/math/primitives/struct.Triangle3d.html
[`Tetrahedron`]: https://docs.rs/bevy/0.14.0/bevy/math/primitives/struct.Tetrahedron.html
[`Rhombus`]: https://docs.rs/bevy/0.14.0/bevy/math/primitives/struct.Rhombus.html
[`Annulus`]: https://docs.rs/bevy/0.14.0/bevy/math/primitives/struct.Annulus.html
[`Arc2d`]: https://docs.rs/bevy/0.14.0/bevy/math/primitives/struct.Arc2d.html
[`CircularSegment`]: https://docs.rs/bevy/0.14.0/bevy/math/primitives/struct.CircularSegment.html
[`CircularSector`]: https://docs.rs/bevy/0.14.0/bevy/math/primitives/struct.CircularSector.html

## Improve `Point` and rename it to `VectorSpace`

{{ <heading_metadata authors={["@mweatherley", "@bushrat011899", "@JohnTheCoolingFan", "@NthTensor", "@IQuick143", "@alice-i-cecile"]} prs={["12747"]} /> }}

Linear algebra is used everywhere in games, and we want to make sure it's easy to get right. That's why we've added a new `VectorSpace` trait, as part of our work to make `bevy_math` more general, expressive, and mathematically sound. Anything that implements `VectorSpace` behaves like a vector. More formally, the trait requires that implementations satisfy the vector space axioms for vector addition and scalar multiplication. We've also added a `NormedVectorSpace` trait, which includes an api for distance and magnitude.

These traits underpin the new curve and shape sampling apis. `VectorSpace` is implemented for `f32`, the `glam` vector types, and several of the new color-space types. It completely replaces `bevy_math::Point`.

The splines module in Bevy has been lacking some features for a long time. Splines are extremely useful in game development, so improving them would improve everything that uses them.

The biggest addition is NURBS support! It is a variant of a B-Spline with much more parameters that can be tweaked to create specific curve shapes. We also added a `LinearSpline`, which can be used to put straight line segments in a curve. `CubicCurve` now acts as a sequence of curve segments to which you can add new pieces, so you can mix various spline types together to form a single path.

## 2D Mesh Wireframes

{{ <heading_metadata authors={["@msvbg", "@IceSentry"]} prs={["12135"]} /> }}

Wireframe materials are used to render the individual edges and faces of a mesh. They are often used as a debugging tool to visualize geometry, but can also be used for various stylized effects. Bevy supports displaying 3D meshes as wireframes, but lacked the ability to do this for 2D meshes until now.

To render your 2D mesh as a wireframe, add `Wireframe2dPlugin` to your app and a `Wireframe2d` component to your sprite. The color of the wireframe can be configured per-object by adding the `Wireframe2dColor` component, or globally by inserting a `Wireframe2dConfig` resource.

For an example of how to use the feature, have a look at the new [wireframe_2d example](https://github.com/bevyengine/bevy/blob/b17292f9d11cf3d3fb4a2fb3e3324fb80afd8c88/examples/2d/wireframe_2d.rs):

![A screenshot demonstrating the new 2D wireframe material](./12135_Support_wireframes_for_2D_meshes.jpg)

## Custom Reflect Field Attributes

{{ <heading_metadata authors={["@MrGVSV"]} prs={["11659"]} /> }}

One of the features of Bevy's reflection system is the ability to attach arbitrary "type data" to a type.
This is most often used to allow trait methods to be called dynamically.
However, some users saw it as an opportunity to do other awesome things.

The amazing [bevy-inspector-egui](https://github.com/jakobhellermann/bevy-inspector-egui) used type data to great effect
in order to allow users to configure their inspector UI per field:

```rust
use bevy_inspector_egui::prelude::*;
use bevy_reflect::Reflect;

#[derive(Reflect, Default, InspectorOptions)]
#[reflect(InspectorOptions)]
struct Slider {
    #[inspector(min = 0.0, max = 1.0)]
    value: f32,
}
```

Taking inspiration from this, Bevy 0.14 adds proper support for custom attributes when deriving `Reflect`,
so users and third-party crates should no longer need to create custom type data specifically for this purpose.
These attributes can be attached to structs, enums, fields, and variants using the `#[reflect(@...)]` syntax,
where the `...` can be any expression that resolves to a type implementing `Reflect`.

For example, we can use Rust's built-in `RangeInclusive` type to specify our own range for a field:

```rust
use std::ops::RangeInclusive;
use bevy_reflect::Reflect;

#[derive(Reflect, Default)]
struct Slider {
    #[reflect(@RangeInclusive<f32>::new(0.0, 1.0))]
    // Since this accepts any expression,
    // we could have also used Rust's shorthand syntax:
    // #[reflect(@0.0..=1.0_f32)]
    value: f32,
}
```

Attributes can then be accessed dynamically using [`TypeInfo`](https://docs.rs/bevy/latest/bevy/reflect/enum.TypeInfo.html):

```rust
let TypeInfo::Struct(type_info) = Slider::type_info() else {
    panic!("expected struct");
};

let field = type_info.field("value").unwrap();

let range = field.get_attribute::<RangeInclusive<f32>>().unwrap();
assert_eq!(*range, 0.0..=1.0);
```

This feature opens up a lot of possibilities for things built on top of Bevy's reflection system.
And by making it agnostic to any particular usage, it allows for a wide range of use cases,
including aiding editor work down the road.

In fact, this feature has already been put to use by [`bevy_reactor`](https://github.com/viridia/bevy_reactor/blob/main/examples/complex/reflect_demo.rs)
to power their custom inspector UI:

```rust
#[derive(Resource, Debug, Reflect, Clone, Default)]
pub struct TestStruct {
    pub selected: bool,

    #[reflect(@ValueRange::<f32>(0.0..1.0))]
    pub scale: f32,

    pub color: Srgba,
    pub position: Vec3,
    pub unlit: Option<bool>,

    #[reflect(@ValueRange::<f32>(0.0..10.0))]
    pub roughness: Option<f32>,

    #[reflect(@Precision(2))]
    pub metallicity: Option<f32>,

    #[reflect(@ValueRange::<f32>(0.0..1000.0))]
    pub factors: Vec<f32>,
}
```

![A custom UI inspector built using the code above in bevy_reactor](./custom_attributes_demo.jpg)

## Query Iteration Sorting

{{ <heading_metadata authors={["@Victoronz"]} prs={["13417"]} /> }}

Bevy does not make any guarantees about the order of items. So if we wish to work with our query items in a certain order, we need to sort them!
We might want to display the scores of the players in order, or ensure a consistent iteration order for the sake of networking stability.
In 0.13 a sort could look like this:

```rust
#[derive(Component, Copy, Clone, Deref)]
pub struct Attack(pub usize)

fn handle_enemies(enemies: Query<(&Health, &Attack, &Defense)>) {
    // An allocation!
    let mut enemies: Vec<_> = enemies.iter().collect();
    enemies.sort_by_key(|(_, atk, ..)| *atk)
    for enemy in enemies {
        work_with(enemy)
    }
}
```

This can get especially unwieldy and repetitive when sorting within multiple systems.
Even if we always want the same sort, different [`Query`] types make it unreasonably difficult to abstract away as a user!
To solve this, we implemented new sort methods on the [`QueryIter`] type, turning the example into:

```rust
// To be used as a sort key, `Attack` now implements Ord.
#[derive(Component, Copy, Clone, Deref, PartialEq, Eq, PartialOrd, Ord)]
pub struct Attack(pub usize)

fn handle_enemies(enemies: Query<(&Health, &Attack, &Defense)>) {
    // Still an allocation, but undercover.
    for enemy in enemies.iter().sort::<&Attack>() {
        work_with(enemy)
    }
}
```

To sort our query with the `Attack` component, we specify it as the generic parameter to [`sort`].
To sort by more than one [`Component`], we can do so, independent of [`Component`] order in the original [`Query`] type: `enemies.iter().sort::<(&Defense, &Attack)>()`

The generic parameter can be thought of as being a [lens] or "subset" of the original query, on which the underlying sort is actually performed. The result is then internally used to return a new sorted query iterator over the original query items.
With the default [`sort`], the lens has to be fully [`Ord`], like with [`slice::sort`].
If this is not enough, we also have the counterparts to the remaining 6 sort methods from [`slice`]!

The generic lens argument works the same way as with [`Query::transmute_lens`]. We do not use filters, they are inherited from the original query.
The [`transmute_lens`] infrastructure has some nice additional features, which allows for this:

```rust
fn handle_enemies(enemies: Query<(&Health, &Attack, &Defense, &Rarity)>) {
    for enemy in enemies.iter().sort_unstable::<Entity>() {
        work_with(enemy)
    }
}
```

Because we can add [`Entity`] to any lens, we can sort by it without including it in the original query!

These sort methods work with both [`Query::iter`] and [`Query::iter_mut`]! The rest of the of the iterator methods on [`Query`] do not currently support sorting.
The sorts return [`QuerySortedIter`], itself an iterator, enabling the use of further iterator adapters on it.

Keep in mind that the lensing does add some overhead, so these query iterator sorts do not perform the same as a manual sort on average. However, this *strongly* depends on workload, so best test it yourself if relevant!

[`Query`]: https://docs.rs/bevy/0.14/bevy/ecs/prelude/struct.Query.html
[`QueryIter`]: https://docs.rs/bevy/0.14/bevy/ecs/query/struct.QueryIter.html
[`sort`]: https://docs.rs/bevy/0.14/bevy/ecs/query/struct.QueryIter.html?search=Component#method.sort
[`Component`]: https://docs.rs/bevy/0.14/bevy/ecs/component/trait.Component.html
[lens]: https://docs.rs/bevy/0.14/bevy/ecs/prelude/struct.Query.html#method.transmute_lens
[`Ord`]: https://doc.rust-lang.org/stable/std/cmp/trait.Ord.html
[`slice::sort`]: https://doc.rust-lang.org/nightly/std/primitive.slice.html#method.sort
[`slice`]: https://doc.rust-lang.org/nightly/std/primitive.slice.html
[`Query::transmute_lens`]: https://docs.rs/bevy/0.14/bevy/ecs/prelude/struct.Query.html#method.transmute_lens
[`transmute_lens`]: https://docs.rs/bevy/0.14/bevy/ecs/prelude/struct.Query.html#method.transmute_lens
[`Entity`]: https://docs.rs/bevy/0.14/bevy/ecs/prelude/struct.Entity.html
[`Query::iter`]: https://docs.rs/bevy/0.14/bevy/ecs/prelude/struct.Query.html#method.iter
[`Query::iter_mut`]: https://docs.rs/bevy/0.14/bevy/ecs/prelude/struct.Query.html#method.iter_mut
[`QuerySortedIter`]: https://docs.rs/bevy/0.14/bevy/ecs/query/struct.QuerySortedIter.html

## SystemBuilder

{{ <heading_metadata authors={["@james-j-obrien"]} prs={["13123"]} /> }}

Bevy users *love* systems, so we made a builder for their systems so they can build systems from within systems.
At runtime, using dynamically-defined component and resource types!

While you can use [`SystemBuilder`](https://docs.rs/bevy/0.14/bevy/ecs/prelude/struct.SystemBuilder.html) as an ergonomic alternative to the [`SystemState`](https://docs.rs/bevy/0.14/bevy/ecs/system/struct.SystemState.html) API for splitting the [`World`](https://docs.rs/bevy/0.14/bevy/ecs/prelude/struct.World.html) into disjoint borrows, its true values lies in its dynamic usage.

You can choose to create a different system based on runtime branches or, more intriguingly, the queries and so on can use runtime-defined component IDs.
This is another vital step towards creating an ergonomic and safe API to work with [dynamic queries](https://bevy.org/news/bevy-0-13/#dynamic-queries),
laying the groundwork for the devs who want to integrate scripting languages or bake in sophisticated modding support for their game.

```rust
// Start by creating builder from the world
let system = SystemBuilder::<()>::new(&mut world)
    // Various helper methods exist to add `SystemParam`.
    .resource::<R>()
    .query::<&A>()
    // Alternatively use `.param::<T>()` for any other `SystemParam` types.
    .param::<MyParam>()
    // Finish it all up with a call `.build`
    .build(my_system);
// The parameters the builder is initialized with will appear first in the arguments.
let system = SystemBuilder::<(Res<R>, Query<&A>)>::new(&mut world)
    .param::<MyParam>()
    .build(my_system);
// Parameters like `Query` that implement `BuildableSystemParam` can use
// `.builder::<T>()` to build in place.
let system = SystemBuilder::<()>::new(&mut world)
    .resource::<R>()
    // This turns our query into a `Query<&A, With<B>>`
    .builder::<Query<&A>>(|builder| { builder.with::<B>(); })
    .param::<MyParam>()
    .build(my_system);
world.run_system_once(system);
```

## Throttle Render Assets

{{ <heading_metadata authors={["@robtfm", "@IceSentry", "@mockersf"]} prs={["12622"]} /> }}

<!-- Throttle: https://github.com/bevyengine/bevy/pull/12622 -->

Using a lot of assets? Uploading lots of bytes to the GPU in a short time might cause stutters due to the render world waiting for uploads to finish.

Often it's a more delightful experience if an application runs smoothly than if it stutters, and a few frames worth of delay before seeing an asset appear
is often not even perceptible.

This experience is now possible:

```rust
fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .insert_resource(RenderAssetBytesPerFrame::new(1_000_000_000)) // Tune to your situation by experimenting!
        .run();
}
```

That's it!
The number provided should be chosen by figuring out a nice trade-off between no stuttering and an acceptable delay.

This feature relies on assets knowing how many bytes they will occupy when sent to the GPU.
Currently this is known by images and meshes, with more assets types expected to be able to report this in the future.

## StandardMaterial UV Channel Selection

{{ <heading_metadata authors={["@geckoxx"]} prs={["13200"]} /> }}

Previously, StandardMaterial always defaulted to using ATTRIBUTE_UV_0 for each texture except lightmap, which isn't flexible enough for a lot of gltf files. In **Bevy 0.14**, a new UvChannel enum was added allowing you to select the channel to use for each texture in StandardMaterial.

Here's a before and after showing the support of ATTRIBUTE_UV_1 across textures:

![UV Channel Selection](uv_channel_selection.jpg)

## Remove limit on RenderLayers

{{ <heading_metadata authors={["@tychedelia", "@robtfm", "@UkoeHB"]} prs={["13317"]} /> }}

<!-- #12502 Remove limit on RenderLayers. -->
<!-- https://github.com/bevyengine/bevy/pull/13317 -->

Render layers are used to quickly toggle the visibility of sets of objects, and control which objects can be seen by which cameras.
This can be useful for things like debug views, gear preview screens, toggle-able diegetic UI and so on.

Before Bevy 0.14 the membership was defined by a bitmask which had limited slots available.
Now, there is no longer any practical limit to how many layers you can define, which is particularly helpful for creative coding applications like [nannou](https://nannou.cc/)!
We've made sure to keep the common case fast, but now use a growable mask that will allocate space for additional layers as needed. Remember, there's still a cost to check visibility per layer, but this allows for more dynamic uses where layers can be created on demand without worrying about going over a limit.

## `on_unimplemented` Diagnostics

{{ <heading_metadata authors={["@bushrat011899", "@alice-i-cecile", "@Themayu"]} prs={["13347"]} /> }}

Bevy takes full advantage of the powerful type system Rust provides, but with that power can often come confusion when even minor mistakes are made.

```rust
use bevy::prelude::*;

struct MyResource;

fn main() {
    App::new()
        .insert_resource(MyResource)
        .run();
}
```

Running the above will produce a compiler error, let's see why...

<details>
<summary>Click to expand...</summary>

```txt
error[E0277]: the trait bound `MyResource: Resource` is not satisfied
   --> example.rs:6:32
    |
6   |     App::new().insert_resource(MyResource).run();
    |                --------------- ^^^^^^^^^^ the trait `Resource` is not implemented for `MyResource`
    |                |
    |                required by a bound introduced by this call
    |
    = help: the following other types implement trait `Resource`:
              AccessibilityRequested
              ManageAccessibilityUpdates
              bevy::a11y::Focus
              DiagnosticsStore
              FrameCount
              bevy::prelude::Axis<T>
              WinitActionHandlers
              ButtonInput<T>
            and 127 others
note: required by a bound in `bevy::prelude::App::insert_resource`
   --> /bevy/crates/bevy_app/src/app.rs:537:31
    |
537 |     pub fn insert_resource<R: Resource>(&mut self, resource: R) -> &mut Self {
    |                               ^^^^^^^^ required by this bound in `App::insert_resource`
```

</details>

The compiler suggests we use a different type that implements `Resource`, or that we implement the trait on `MyResource`. The former doesn't help us at all, and the latter fails to mention the available derive macro.

With the release of Rust 1.78, Bevy can now provide more direct messages for certain types of errors during compilation using [diagnostic attributes](https://blog.rust-lang.org/2024/05/02/Rust-1.78.0.html#diagnostic-attributes).

```txt
error[E0277]: `MyResource` is not a `Resource`
   --> example.rs:6:32
    |
6   |     App::new().insert_resource(MyResource).run();
    |                --------------- ^^^^^^^^^^ invalid `Resource`
    |                |
    |                required by a bound introduced by this call
    |
    = help: the trait `Resource` is not implemented for `MyResource`
    = note: consider annotating `MyResource` with `#[derive(Resource)]`
    = help: the following other types implement trait `Resource`:
              AccessibilityRequested
...
```

Now, the error message has a more approachable entry point, and a new `note` section pointing to the derive macro for resources. If Bevy's suggestions _aren't_ the solution to your problem, the rest of the compiler error is still included just in case.

These diagnostics have been implemented for various traits across Bevy, and we hope to improve this experience as new features are added to Rust. For example, we'd really like to improve the experience of working with tuples of `Component`'s, but we're not quite there yet. You can read more about this change in the [pull request](https://github.com/bevyengine/bevy/pull/13347) and associated [issue](https://github.com/bevyengine/bevy/issues/12377).

## Motion Vectors and TAA for Animated Meshes

{{ <heading_metadata authors={["@pcwalton"]} prs={["13572"]} /> }}

<!-- Implement motion vectors and TAA for skinned meshes and meshes with morph targets. -->
<!-- https://github.com/bevyengine/bevy/pull/13572 -->

Back in **Bevy 0.11** we added [Temporal Anti Aliasing (TAA)](/news/bevy-0-11/#temporal-anti-aliasing), which uses Motion Vectors to determine how fast an object is moving. However, in **Bevy 0.11** we only added Motion Vector support for "static" meshes, meaning TAA did not work for animated meshes using skeletal animation or morph targets.

In **Bevy 0.14**, we implemented [Per-Object Motion Blur](#per-object-motion-blur), which _also_ uses Motion Vectors and therefore would have that same limitation.

Fortunately in **Bevy 0.14** we implemented Motion Vectors for skinned meshes and meshes with morph targets, closing this gap and enabling TAA, Per-Object Motion Blur, and future Motion Vector features to work with animated meshes.

## Improved Matrix Naming

{{ <heading_metadata authors={["@ricky26"]} prs={["13489"]} /> }}

<!-- Normalize matrix naming -->
<!-- https://github.com/bevyengine/bevy/pull/13489 -->

Game engines generally provide a set of matrices to perform space transformations in the game world. Commonly, the following spaces are used:

- **Normalized Device Coordinates**: used by the graphics API directly
- **Clip Space**: coordinates after projection but before perspective divide
- **View Space**: coordinates in the camera's view
- **World Space**: global coordinates (this is the one we most often talk about!)
- **Model Space**: (or local space) coordinates relative to an entity

A common example is the 'model view projection matrix', which is the transformation from model space to NDC space (peculiarly in this shorthand,
the view matrix is often a transformation from world _to view_ space, but the model matrix is a transformation _from model_ (or local) space to world space).
Usually, matrices are referred to as part of that shorthand, so for example, the projection matrix transforms from view coordinates to NDC coordinates.

In a couple of places, Bevy had a view matrix, which was the transformation from view to world space (rather than from world to view space as above).
Additionally, even when used consistently, the single-word shorthands are ambiguous and can cause confusion. We felt that a clearer convention was needed.

From now on, matrices in Bevy are named `y_from_x`, for example `world_from_local`, which would denote the transformation from local to world-space coordinates.
One tidy benefit of this is that the inverse matrices are named `x_from_y`, and when multiplying between spaces, it's easy to see that it's correct.

For example, instead of writing:

```rust
let model_view_projection = projection * view * model;
```

You would now write:

```rust
let clip_from_local = clip_from_view * view_from_world * world_from_local;
```

## Typed glTF Labels

{{ <heading_metadata authors={["@mockersf", "@rparrett", "@alice-i-cecile"]} prs={["13586"]} /> }}

<!-- glTF labels: add enum to avoid misspelling and keep up-to-date list documented -->
<!-- https://github.com/bevyengine/bevy/pull/13586 -->

If you've been using [`glTF`] files for your scenes or looked at an example that does you've might have seen the _labels_ at the end of the asset path:

```rust
let model_pine = asset_server.load("models/trees/pine.gltf#Scene0");
let model_hen = asset_server.load("models/animals/hen.gltf#Scene0");
let animation_hen = asset_server.load("models/animals/hen.gltf#Aniamtion1"); // Oh no!
```

Notice the `#Scene0` syntax at the end. The glTF format is able to contain many things in a single file, including several scenes, animations, lights, and more.

These labels are a way of telling Bevy which part of the file we're loading.

However this is prone to user-error, and it looks like an error snuck in! The hen animation got the label `Aniamtion1` instead of `Animation1`.

No more! The above can now be re-written like so:

```rust
let hen = "models/animals/hen.gltf"; // Can re-use this more easily too
let model_pine = asset_server.load(GltfAssetLabel::Scene(0).from_asset("models/trees/pine.gltf"));
let model_hen = asset_server.load(GltfAssetLabel::Scene(0).from_asset(hen));
let animation_hen = asset_server.load(GltfAssetLabel::Animation(0).from_asset(hen)); // No typo!
```

Check out [`glTF label docs`] to know which parts you can query for.

[`glTF`]: https://www.khronos.org/gltf/
[`glTF label docs`]: https://docs.rs/bevy/0.14/bevy/gltf/enum.GltfAssetLabel.html

## winit v0.30

{{ <heading_metadata authors={["@pietrosophya", "@mockersf"]} prs={["13366"]} /> }}

[Winit v0.30] changed its API to support a trait based architecture instead of a plain event-based one. Bevy 0.14 now implements that new architecture, making the event loop handling easier to follow.

[Winit v0.30]: https://docs.rs/winit/0.30.0/winit/changelog/v0_30/index.html

It's now possible to define a custom `winit` user event, that can be used to trigger App updates,
and that can be read inside systems to trigger specific behaviors. This is particularly useful to
send events from outside the `winit` event loop and manage them inside Bevy systems
(see the [`window/custom_user_event.rs`] example).

[`window/custom_user_event.rs`]: https://github.com/bevyengine/bevy/blob/release-0.14.0/examples/window/custom_user_event.rs

The `UpdateMode` enum now accepts only two values: `Continuous` and `Reactive`. The latter exposes 3 new properties to enable reactivity to device, user, or window events. The previous `UpdateMode::Reactive` is now equivalent to `UpdateMode::reactive()`, while `UpdateMode::ReactiveLowPower` maps to `UpdateMode::reactive_low_power()`.

* `Idle`: the loop has not started yet
* `Running` (previously called `Started`): the loop is running
* `WillSuspend`: the loop is going to be suspended
* `Suspended`: the loop is suspended
* `WillResume`: the loop is going to be resumed

Note: the `Resumed` state has been removed since the resumed app is just `Running`.

## Scene, Mesh, and Material glTF Extras

{{ <heading_metadata authors={["@kaosat-dev"]} prs={["13453"]} /> }}

The glTF 3D model file format allows passing additional user defined metadata in the *extras* properties, and
in addition to the glTF extras at the primitive/node level , Bevy now has specific GltfExtras for:

- scenes: **SceneGltfExtras** injected at the scene level if any
- meshes: **MeshGltfExtras**, injected at the mesh level if any
- materials: **MaterialGltfExtras**, injected at the mesh level if any: ie if a mesh has a material that has gltf extras, the component will be injected there.

You can now easily query for these specific extras

```rust
fn check_for_gltf_extras(
    gltf_extras_per_entity: Query<(
        Entity,
        Option<&Name>,
        Option<&GltfSceneExtras>,
        Option<&GltfExtras>,
        Option<&GltfMeshExtras>,
        Option<&GltfMaterialExtras>,
    )>,
) {
    // use the extras' data 
    for (id, name, scene_extras, extras, mesh_extras, material_extras) in
        gltf_extras_per_entity.iter()
    {

    }
}

```

This makes passing information from programs such as Blender to Bevy via gltf files more spec compliant, and more practical!

## Resource Entity Mapping in Scenes

{{ <heading_metadata authors={["@brandon-reinhart"]} prs={["13650"]} /> }}

Bevy's `DynamicScene` is a collection of resources and entities that can be serialized to create collections like prefabs or savegame data. When a DynamicScene is deserialized and written into a World - such as when a saved game is loaded - the dynamic entity identifiers inside the scene must be mapped to their newly spawned counterparts.

Previously, this mapping was only available to Entity identifiers stored on Components. In Bevy 0.14, Resources can reflect `MapEntitiesResource` and implement the `MapEntities` trait to get access to the `EntityMapper`.

```rust
    // This resource reflects MapEntitiesResource and implements the MapEntities trait.
    #[derive(Resource, Reflect, Debug)]
    #[reflect(Resource, MapEntitiesResource)]
    struct TestResource {
        entity_a: Entity,
        entity_b: Entity,
    }

    // A simple and common use is a straight mapping of the old entity to the new.
    impl MapEntities for TestResource {
        fn map_entities<M: EntityMapper>(&mut self, entity_mapper: &mut M) {
            self.entity_a = entity_mapper.map_entity(self.entity_a);
            self.entity_b = entity_mapper.map_entity(self.entity_b);
        }
    }
```

## CompassQuadrant and CompassOctant

{{ <heading_metadata authors={["@BobG1983", "@alice-i-cecile"]} prs={["13653"]} /> }}

<!-- Added CompassQuadrant and CompassOctant as per #13647 -->
<!-- https://github.com/bevyengine/bevy/pull/13653 -->

There are many instances in game development where its important to know the compass facing for a given direction. This is particularly true in 2D games that use four or eight direction sprites, or want to map analog input into discrete movement directions.

In order to make this easier the enums `CompassQuadrant` (for a four-way division) and `CompassOctant` (for an eight-way division) have been added with implementations to and `From<Dir2>` for ease of use.

## Support `AsyncSeek` When Loading Assets

{{ <heading_metadata authors={["@BeastLe9enD"]} prs={["12547"]} /> }}

<!-- Add AsyncSeek trait to Reader to be able to seek inside asset loaders -->
<!-- https://github.com/bevyengine/bevy/pull/12547 -->

Assets can be huge, and you don't always need all of the data contained in a single file.

Bevy allows you to add your [own asset loaders].
Starting in Bevy 0.14,  you can now seek to an offset of your choice, reading partway through the file.

Perhaps you have the `.celestial` file format which encodes the universe, but you want to only look at lil' asteroids which always appear at some offset:

```rust
#[derive(Default)]
struct UniverseLoader;

#[derive(Asset, TypePath, Debug)]
struct JustALilAsteroid([u8; 128]); // Each lil' asteroid uses this much data

impl AssetLoader for UniverseLoader {
    type Asset = JustALilAsteroid;
    type Settings = ();
    type Error = std::io::Error;
    async fn load<'a>(
        &'a self,
        reader: &'a mut Reader<'_>,
        _settings: &'a (),
        _load_context: &'a mut LoadContext<'_>,
    ) -> Result<JustALilAsteroid, Self::Error> {
        // The universe is big, and our lil' asteroids don't appear until this offset
        // in the celestial file format!
        let offset_of_lil_asteroids = 5_000_000_000_000;

        // Skip vast parts of the universe with the new async seek trait!
        reader
            .seek(SeekFrom::Start(offset_of_lil_asteroids))
            .await?;

        let mut asteroid_buf = [0; 128];
        reader.read_exact(&mut asteroid_buf).await?;

        Ok(JustALilAsteroid(asteroid_buf))
    }

    fn extensions(&self) -> &[&str] {
        &["celestial"]
    }
}
```

This works because Bevy's [`reader`] type passed into the asset loader's `load` function now implements [`AsyncSeek`].

Real world use cases might for example be:

- You have packed several assets in an archive and you wish to skip to an asset within and read that
- You are dealing with big datasets such as map data and you know where to extract some locations of interest

[own asset loaders]: https://github.com/bevyengine/bevy/blob/release-0.14.0/examples/asset/processing/asset_processing.rs
[`reader`]: https://docs.rs/bevy/0.14/bevy/asset/io/type.Reader.html
[`AsyncSeek`]: https://docs.rs/futures-io/latest/futures_io/trait.AsyncSeek.html

## LoadState::Failed Now Has Error Info

{{ <heading_metadata authors={["@bugsweeper"]} prs={["12709"]} /> }}

Rust prides itself on its error handling, and Bevy has been steadily catching up.
Previously, when checking if an asset was loaded using [`AssetServer::get_load_state`](https://docs.rs/bevy/0.14/bevy/asset/struct.AssetServer.html#method.get_load_state),
all you'd get back was a data-less [`LoadState::Failed`](https://docs.rs/bevy/0.14/bevy/asset/enum.LoadState.html) if something went wrong.
Not very useful for debugging!

Now, a full [`AssetLoadError`](https://docs.rs/bevy/0.14/bevy/asset/enum.AssetLoadError.html) is included, with 14 different variants telling you exactly what went wrong.
Great for troubleshooting, and it opens the door to proper error handling in more complex apps.

## `AppExit` Errors

{{ <heading_metadata authors={["@Brezak", "@alice-i-cecile"]} prs={["13022"]} /> }}

When running an app, there might be many reasons to trigger an exit. Maybe the user has pressed the quit button, or the render thread has encountered an error and died. You might want to distinguish between these two situations and return an appropriate [exit code](https://doc.rust-lang.org/std/process/struct.ExitCode.html#impl-From%3Cu8%3E-for-ExitCode) from your application.

In **Bevy 0.14**, you can. The `AppExit` event is now an enum with two variants: `Success` and `Error`. The error variant also holds a non-zero code, which you're allowed to use however you wish. Since `AppExit` events now contain useful information, app runners and `App::run` now return the event that resulted in the application exiting.

For plugin developers, `App` has gained a new method, `App::should_exit`, which will check if any `AppExit` events were sent in the last two updates. To make sure `AppExit::Success` events won't drown out useful error information, this method will return any `AppExit::Error` events, even if they were sent after an `AppExit::Success`.

Finally, `AppExit` also implements the [`Termination`](https://doc.rust-lang.org/stable/std/process/trait.Termination.html) trait, so it can be returned from main.

```rust
use bevy::prelude::*;

fn exit_with_a_error_code(mut events: EventWriter<AppExit>) {
    events.send(AppExit::from_code(42));
}

fn main() -> AppExit {
    App::new()
        .add_plugins(MinimalPlugins)
        .add_systems(Update, exit_with_a_error_code)
        .run() // There's no semicolon here, `run()` returns `AppExit`.
}
```

![App returning a 42 exit code](exit_with_a_42.jpg)

## Make dynamic_linking a no-op on WASM targets

{{ <heading_metadata authors={["@james7132"]} prs={["12672"]} /> }}

WASM does not support dynamic libraries that can be linked to during runtime. Before, Bevy would fail to compile if you enabled the `dynamic_linking` feature.

```bash
$ cargo build --target wasm32-unknown-unknown --features bevy/dynamic_linking
error: cannot produce dylib for `bevy_dylib v0.13.2` as the target `wasm32-unknown-unknown` does not support these crate types
```

Now, Bevy will fallback to static linking for all WASM targets. If you enable `dynamic_linking` for development, you no longer need to disable it for WASM.

## Deprecate Dynamic Plugins

{{ <heading_metadata authors={["@BD103"]} prs={["13080"]} /> }}

`bevy_dynamic_plugin` was a tool added in Bevy's original 0.1 release: intended to serve as a tool for dynamically loading / linking Rust code for use with things like modding.
Unfortunately, this feature didn't see much community uptake, and as a result had a vanishingly small number of contributions to refine and document it over the years.

Combined with a challenging, intrinsically unsafe API that was producing [worrying failures](https://github.com/bevyengine/bevy/issues/13073) for users, we've decided to deprecate `bevy_dynamic_plugin` and will be removing it completely in Bevy 0.15.
If you were a happy user of this, simply copy the rather-small crate into your own project and proceed as before.

We still think that both modding and hot-reloading code for faster development times are valuable use cases that Bevy *should* help support one day.
Our hope is that by removing this as a first-party crate, we can spur on third-party experiments and avoid wasting users' time as they investigate a complex potential solution before concluding that it doesn't yet meet their needs.

## Bevy Working Groups

{{ <heading_metadata authors={["@alice-i-cecile"]} prs={["13162"]} /> }}

Bevy has a ton of incredibly talented contributors, so keeping track of what's going on and making informed decisions can be a real challenge.
We're experimenting with [working groups]: ad hoc groups tackling harder issues by creating a design document, getting sign-off from the experts, and then implementing it.
If you'd like to help make complex, high-impact changes to Bevy: join or form a working group!

[working groups]: https://github.com/bevyengine/bevy/blob/main/CONTRIBUTING.md#join-a-working-group

## What's Next?

The features above may be great, but what else does Bevy have in flight?
Peering deep into the mists of time (predictions are _extra_ hard when your team is almost all volunteers!), we can see some exciting work taking shape:

- **Better Scenes:** Scenes are one of Bevy's core building blocks: designed to be a powerful tool for authoring levels and creating reusable game objects, whether they're a radio button widget or a monster. We're working on a new scene system with a new syntax that will make defining scenes in assets _and_ in code more powerful and more pleasant. You can check out the(now slightly old) [project kickoff discussion](https://github.com/bevyengine/bevy/discussions/9538) for more information. We're also very close to putting out a design document outlining our plans and the current state of the implementation.
- **ECS Relations:** Relations (a first-class feature for linking entities together) is wildly desired but remarkably complex, driving features and refactors to our ECS internals. The [working group](https://discord.com/channels/691052431525675048/1237010014355456115) has been patiently laying out what we need to do and why in this [RFC](https://github.com/bevyengine/rfcs/pull/79).
- **Better Audio:** Bevy's built-in audio solution has never really hit the right notes. The [Better Audio working group](https://discord.com/channels/691052431525675048/1236113088793677888) is plotting a path forward.
- **Contributing Book:** Our documentation on how to contribute is scattered to the four corners of our repositories. By gathering this together, the [Contributing Book working group](https://discord.com/channels/691052431525675048/1236112637662724127) hopes to make it easier to discover and maintain.
- **Curve Abstraction:** Curves come up all of the time in game dev, and the mathmagicians that make up the [Curve Crew](https://discord.com/channels/691052431525675048/1236110755212820581) are [designing a trait](https://github.com/bevyengine/rfcs/pull/80) to unify and power them.
- **Better Text:** our existing text solution isn't up to the demands of modern UI. The "Lorem Ipsum" working group is [looking into](https://discord.com/channels/691052431525675048/1248074018612051978) replacing it with a better solution.
- **A Unified View on Dev Tools:** In 0.14, we've added a stub `bevy_dev_tools` crate: a place for tools and overlays that speed up game development such as performance monitors, fly cameras, or in-game commands to spawn game objects. We're working on adding more tools, and creating a [dev tool abstraction](https://github.com/bevyengine/rfcs/pull/77). This will give us a unified way to enable/disable, customize and group this grab bag of tools into toolboxes to create something like Quake console or VSCode Command Palette with tools from around the ecosystem.
- **Bevy Remote Protocol:** Communicating with actively running Bevy games is an incredibly powerful tool for building editors, debuggers and other tools. [We're developing](https://github.com/bevyengine/bevy/pull/13563) a reflection-powered protocol to create a solution that's ready to power a whole ecosystem.
- **A Modular, Maintainable Render Graph:** Bevy's existing rendering architecture is already quite good at providing reusable renderer features like `RenderPhases`, batching, and draw commands. However, the render graph interface itself is one remaining pain points. Since it's distributed across many files the control flow is hard to understand, and its heavy use of ECS resources for passing around rendering data actively works against modularity. While the exact design hasn't been finalized (and feedback is very welcome!), we've been actively working to [redesign the render graph](https://github.com/bevyengine/bevy/pull/13397) in order to build up to a larger refactor of the renderer towards modularity and ease of use.

{{ <support_bevy /> }}
{{ <contributors version="0.14" /> }}
{{ <changelog version="0.14" /> }}
