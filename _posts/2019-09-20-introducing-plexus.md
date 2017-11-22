---
layout: post
title: "Introducing Plexus"
excerpt: "A Rust library for polygonal mesh processing."
tag:
  - graphics
  - mesh
  - plexus
  - rust
feature: assets/img/features/introducing-plexus.jpg
comments: false
published: false
---

I've been following Rust for a long time and in 2015 I decided to dive in and
write some code. I started with a project called
[Bismuth](https://github.com/olson-sean-k/bismuth), which took inspiration from
the [Cube engine](https://en.wikipedia.org/wiki/cube_(video_game)) and its
legacy: representing a dynamic game world as a deformable
[oct-tree](https://en.wikipedia.org/wiki/octree).

It didn't take long for some modules to become candidates for independent
crates. One such module generated polygonal mesh data for primitives like cubes.
As I spent more and more time on it, I spun that code into its own crate called
[Plexus](https://github.com/olson-sean-k/plexus). Plexus now provides an
assortment of tools for processing polygonal meshes while providing some unique
APIs and remaining agnostic to geometry and dimension.

This post is an introduction to the main features and goals of Plexus, but I
plan to discuss some of this in more detail in future posts. There is a lot to
discuss in all of the code examples!

## Philosophy and Goals

First, I should get this out of the way:

![I have no idea what I'm doing.](http://m.quickmeme.com/img/1c/1c491f71b689e82d6e838b5d8ce5cbdfef41723662d1ce5e5cf34f32ae60a7a3.jpg)

I'm easily distracted and it turns out that polygonal mesh processing is an
interesting topic! I don't have very much experience with graphics programming
or meshes as either a programmer or an artist. My hope is that coming to this
fresh may introduce some interesting and useful perspectives.

Below are a few high level goals of the Plexus project:

- **Flexibility and Genericness**

  Plexus attempts to be flexible, especially with respect to geometry. Plexus is
  opinionated about _topology_, but allows arbitrary data to be used for
  _geometry_. Geometry is important, so Plexus uses
  [Theon](https://github.com/olson-sean-k/theon) to provide rich features so
  long as it understands the positional data of some geometry.

- **Correctness and Consistency**

  Plexus provides a graph representation for polygonal meshes. Maintaining the
  topological consistency of such a data structure can be difficult, and Plexus
  attempts to provide an API that does not allow user code to invalidate graphs;
  errors, panics, and bugs are contained within the library.

- **Ergonomics**

  APIs for non-trivial data structures can sometimes be unwieldy. Plexus
  attempts to avoid this (with varying levels of success). In particular, the
  APIs for graphs are unique and provide _views_ into its structure that I hope
  are easier to understand and reason about than most other graph APIs.

- **Documentation**

  I sometimes find useful crates and libraries difficult to use solely due to a
  lack of documentation and examples.  Plexus is very incomplete, but I've tried
  to document what I can via `rustdoc` and a (very scrappy)
  [website](https://plexus.rs) and [user
  guide](https://plexus.rs/user-guide/graphs).

In the future, I think it would be interesting to use Plexus as the basis for
modeling software. I also find the propsect of features like rigging and
animation to be an exciting idea.

At this point, Plexus does not provide the necessary features to be very useful,
but the infrastructure built to power its graph implementation seems promising
and provides a basis for implementing common algorithms.

## Iterator Expressions

One of the first problems I encountered in Bismuth was generating vertex data
for simple polygonal meshes. Each non-empty partition of the oct-tree is
represented by deformations to a unit cube. For each partition, these
deformations are applied to the positional data of a unit cube.

Initially inspired by [`genmesh`](https://crates.io/crates/genmesh), I began
with generators that produce iterators over primitive topological structures
like triangles. These structures can be manipulated using iterator expressions
which can be arbitrarily complex and aggregated in various ways. Here's an
example:

```rust
use decorum::R64;
use nalgebra::Point3;
use plexus::prelude::*;
use plexus::primitive::generate::Position;
use plexus::primitive::sphere::UvSphere;
use plexus::index::{Flat3, HashIndexer};

type E3 = Point3<R64>;

let (indices, positions) = UvSphere::new(16, 16)
    .polygons::<Position<E3>>()
    .triangulate()
    .index_vertices::<Flat3, _>(HashIndexer::default());
```

The above example generates _raw buffers_ from a UV-sphere. These raw buffers
are commonly known as _index_ and _vertex buffers_ and are typically used for
[indexed drawing](https://learnopengl.com/getting-started/hello-triangle). The
`polygons` function produces an iterator over the polygons that form a primitive
and accepts a type parameter that determines the geometric attribute contained
in the vertices of those polygons. These polygons are
[tessellated](https://graphics.fandom.com/wiki/tesselation) into triangles
before being indexed into the raw buffers.

Here's a more substantial example that further manipulates iterators over
polygons:

```rust
use decorum::R64;
use nalgebra::Point3;
use plexus::buffer::MeshBuffer3;
use plexus::prelude::*;
use plexus::primitive::cube::{Bounds, Cube};
use plexus::primitive::generate::{Normal, Position};
use plexus::primitive;

use crate::Vertex; // Omitted for brevity.

type E3 = Point3<R64>;

let cube = Cube::new();
let buffer: MeshBuffer3<usize, Vertex> = primitive::zip_vertices((
    cube.polygons_from::<Position<E3>>(Bounds::with_width(10.0.into())),
    cube.polygons::<Normal<E3>>(),
))
    .map_vertices(|(position, normal)| {
        Vertex::new(position, normal)
    })
    .triangulate()
    .collect();
```

In this example, different geometric attributes are combined by `zip_vertices`
into a single iterator over polygons containing both position and normal
geometry. This is used to construct a `Vertex` and finally the polygons are
tessellated and collected into a `MeshBuffer`.

Iterator expressions like these can be used to collect polygons into linear and
graph data structures as well as raw buffers as seen above. It is also possible
to generate iterators over individual vertices alongside indexing polygons
(which avoids the use of an indexer).

### Tangent: Indexing, Hashing, and Floating-Point

As seen above, aggregating an iterator expression often requires _indexing_,
which eliminates redundant geometry while maintaining topological structure. You
may have noticed the use of the `R64` type rather than the primitive
floating-point types `f32` or `f64`. What's that about?

Hashing is often a natural way to detect and discard duplicate data, but
there's a problem: it is a common need to represent real numbers in graphics
programming and that relies heavily on [IEEE-754
floating-point](https://en.wikipedia.org/wiki/IEEE_754) values like `f32` that
do **not** implement the `Hash` and `Eq` traits (for good reasons).

To get around this, Plexus supports the
[Decorum](https://github.com/olson-sean-k/decorum) crate, which I also spun off
from Bismuth. Decorum provides proxy (wrapper) types for floating-point
primitives that provide a total ordering and therefore can implement `Hash` and
`Eq`. Plexus can index non-`Hash` data, but it requires a bit of additional
code. I plan to write a bit about Decorum in the future as well.

## Graphs

Plexus provides a [half-edge
graph](https://www.openmesh.org/media/Documentations/OpenMesh-6.3-Documentation/a00010.html)
representation of polygonal meshes. These graphs allow for efficient traversals
of topology. Graphs also support arbitrary geometry associated with vertices,
arcs (half-edges), edges, and faces. Implementing graphs in Rust is an
interesting excercise, as common approaches using pointers or references do not
provide clear ownship semantics and do not satisfy the borrow checker,
especially in a mutable context.

Here's a short example:

```rust
use cgmath::Point3;
use plexus::encoding::ply::{FromPly, PositionEncoding};
use plexus::graph::MeshGraph;
use plexus::prelude::*;
use std::fs::File;

let ply = File::open("cube.ply").unwrap();
let encoding = PositionEncoding::<Point3<f64>>::default();
let (mut graph, _) = MeshGraph::<Point3<f64>>::from_ply(encoding, ply).unwrap();

let keys = graph.faces().keys().collect::<Vec<_>>();
for key in keys {
    graph.face_mut(key).unwrap().poke_with_offset(1.0);
}
```

This code loads mesh data from a file into a graph using the
[PLY](https://en.wikipedia.org/wiki/ply_(file_format)) encoding. It then
[pokes](https://docs.blender.org/manual/en/latest/modeling/meshes/editing/faces.html#poke-faces)
every face, inserting a vertex at the centroid and forming a triangle fan.
`poke_with_offset` is able to compute the centroid of each face and then
translate the position of the inserted vertex along the faces normal, because
`Point3` implements the necessary geometric traits.

Poking a face with an offset involves several geometric operations. To provide
this functionality, Plexus integrates with
[Theon](https://github.com/olson-sean-k/theon), which abstracts Euclidean
spaces. Today, this provides immediate support for the
[`cgmath`](https://crates.io/crates/cgmath),
[`mint`](https://crates.io/crates/mint), and
[`nalgebra`](https://crates.io/crates/nalgebra) crates. Non-Euclidean or
otherwise unsupported types can be used, but the computation of geometry must be
provided when necessary.

I think one of the most interesting things about `MeshGraph`'s API is its use of
_views_. Views behave a lot like references in Rust and expose a rich API on top
of the underlying storage abstractions used to represent graphs. Combined with
traits, it is even possible to write very generic code that operates exclusively
with views:

```rust
use plexus::graph::{EdgeMidpoint, FaceView, GraphGeometry, MeshGraph};
use plexus::prelude::*;
use plexus::AsPosition;
use smallvec::SmallVec;

pub fn circumscribe<G>(face: FaceView<&mut MeshGraph<G>, G>) -> FaceView<&mut MeshGraph<G>, G>
where
    G: EdgeMidpoint + GraphGeometry,
    G::Vertex: AsPosition,
{
    // Split each edge, stashing the vertex key and moving to the next arc.
    let arity = face.arity();
    let mut arc = face.into_arc();
    let mut splits = SmallVec::<[_; 4]>::with_capacity(arity);
    for _ in 0..arity {
        let vertex = arc.split_at_midpoint();
        splits.push(vertex.key());
        arc = vertex.into_outgoing_arc().into_next_arc();
    }
    // Split faces along the vertices from each arc split.
    let mut face = arc.into_face().unwrap();
    for (a, b) in splits.into_iter().perimeter() {
        face = face.split(ByKey(a), ByKey(b)).unwrap().into_face().unwrap();
    }
    // Return the central face of the subdivision.
    face
}
```

This more advanced example subdivides a face by circumscribing a similar polygon
within its perimeter. Because this requires geometric operations, the
`EdgeMidpoint` trait is used as a bound, specifying that the graph's geometry
must support the computation of midpoints.
...
