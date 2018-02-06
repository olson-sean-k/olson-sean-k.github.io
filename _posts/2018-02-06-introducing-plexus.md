---
layout: post
title: Introducing Plexus
published: false
---

Nearly two years ago, I decided to learn Rust. I started with a project called
[Bismuth](https://github.com/olson-sean-k/bismuth), which took inspiration from
the [Cube engine](https://en.wikipedia.org/wiki/cube_(video_game)) and its
legacy: representing a dynamic game world as a deformable
[oct-tree](https://en.wikipedia.org/wiki/octree).

It didn't take long for some modules to become candidates for independent
crates. One such module generated vertex data for primitive meshes like cubes.
As I spent more and more time on it, I spun that code into its own crate called
[Plexus](https://github.com/olson-sean-k/plexus). Plexus now provides
primitives, graphs, and buffers that can be used to generate and manipulate 2D
and 3D mesh data. Here's an introduction to its main features and goals. I plan
to discuss some of this in more detail in future posts.

## Generating Mesh Data

One of the first problems I encountered in Bismuth was generating vertex data
for simple meshes. Each non-empty partition of the oct-tree is represented by
deformations to a unit cube. For each partition, these deformations are applied
to the positions of a unit cube mesh. This led to generators in Plexus.

### Iterator Expressions

Initially inspired by [genmesh](https://github.com/gfx-rs/genmesh), I started
with primitive generators that use iterator expressions to manipulate topology
and geometry. These expressions can be arbitrarily complex and are typically
collected into graphs or buffers. Here's a simple example:

```rust
use plexus::prelude::*;
use plexus::primitive::sphere::UvSphere;
use plexus::primitive::HashIndexer;

let (indeces, vertices) = UvSphere::new(16, 16)
    .polygons_with_position()
    .triangulate()
    .flat_index_vertices(HashIndexer::default());
```

The above example generates raw buffers from a UV-sphere. These are commonly
known as _index_ and _vertex buffers_ and are typically used for [indexed
drawing](https://learnopengl.com/getting-started/hello-triangle). The
`polygons_with_position` generator function produces an iterator over polygons
containing position data in their vertices. These are tessellated into
triangles before being indexed into buffers.

Here's a more substantial example that further manipulates multiple streams of
polygons:

```rust
use decorum::R32;
use nalgebra::Point3;
use plexus::buffer::MeshBuffer;
use plexus::prelude::*;
use plexus::primitive::cube::{Cube, Plane};
use plexus::primitive;

let unit: R32 = 10.0.into();
let cube = Cube::new();
let buffer = primitive::zip_vertices((
        cube.polygons_with_position()
            .map_vertices(|position| -> Point3<R32> { position.into() })
            .map_vertices(|position| position * unit),
        cube.polygons_with_plane(),
    ))
    .map_vertices(|(position, plane)| {
        (position, map_unit_uv(&position, &plane, unit))
    })
    .tetrahedrons()
    .collect::<MeshBuffer<u32, _>>();
```

Here, different geometric attributes are combined by `zip_vertices` into a
single stream of polygons containing both position and plane geometry. The
combined vertex data is used to generate texture coordinates with `map_unit_uv`
(not shown). Finally, the polygons are tessellated and collected into a
`MeshBuffer`. More on that later.

### Indexing: the Problem with Hashing

These kinds of iterator expressions aren't very useful on their own.
Aggregating the results of an expression often requires _indexing_, which
eliminates redundant geometry while maintaining topological structures.

Hashing is often a natural way to detect and discard duplicate data, but
there's a problem: it is a common need to represent real numbers in graphics
programming and that relies heavily on [IEEE-754
floating-point](https://en.wikipedia.org/wiki/IEEE_754) values like `f32` that
do **not** implement the `Hash` and `Eq` traits (for good reasons).

To get around this, Plexus uses the
[Decorum](https://github.com/olson-sean-k/decorum) crate, which I also spun off
from Bismuth. I plan to write a bit about Decorum in the future as well.
Plexus uses Decorum types in generators and provides conversions to improve
ergonomics. Here's an example:

```rust
use nalgebra::Point3;
use plexus::graph::Mesh;
use plexus::prelude::*;
use plexus::primitive::sphere::UvSphere;

let mut mesh = UvSphere::new(16, 16)
    .polygons_with_position()
    .collect::<Mesh<Point3<f32>>>(); // Magic!
```

This produces a graph where the vertex geometry is of type `Point3<f32>`.
Before these conversations occur, the topology stream is indexed over a
hashable type emitted by the generator. Enabling this kind of conversion only
requires implementing the `FromGeometry` trait and implementations are provided
out-of-the-box for commonly used types.

...

## Half-Edge Graphs

Plexus uses a [half-edge
graph](https://www.openmesh.org/media/Documentations/OpenMesh-6.3-Documentation/a00010.html)
representation of meshes. These graphs allow for effecient traversals of
topology, especially neighbor lookups, which are often important. They also
support associated geometry for vertices, edges, and faces. Implementing graphs
in Rust is an interesting excercise, as common approaches using pointers or
references do not satisfy the borrow checker if the graph is mutable.

...
