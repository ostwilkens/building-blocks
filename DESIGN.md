# Design Philosophy and Architecture

## Principles

The architecture of Building Blocks is driven by a few guiding principles:

- **Real-Time Performance**
  - The primary use case for Building Blocks is using voxel technology within a
    video game. This means it needs to be fast. For example, we want to be able
    to generate meshes for millions of voxels per frame (16.6 ms).
  - Critical algorithms must be benchmarked with Criterion so we can guide
    optimization with evidence.
- **Composable APIs**
  - APIs are more powerful when they are generic. You will find many examples of
    generic APIs that require the input types to implement some traits.
  - This is most prevalent in the storage crate, where we desire all of the
    lattice map types to be accessible through the same core set of traits. This
    results in a very powerful function, `copy_extent`, which works with all
    implementations of the `ReadExtent` and `WriteExtent` traits.
- **KISS (Keep It Simple Stupid)**
  - There are *many* complex data structures and algorithms used in voxel
    technology. While they certainly all serve a purpose, it is not feasible for
    contributors to understand and implement all of them at the very inception
    of this project.
  - Any increase in complexity of this library, usually by adding a new data
    structure or algorithm, must solve a specific problem for end users. And we
    should find the simplest solution to that problem such that the increase in
    complexity is minimal.
  - You might find your favorite voxel technology missing from the existing
    feature set or road map. This is probably just because no one has found a
    need for it yet. We are open to contributions, but keep the KISS principle
    in mind when considering what problem you are solving.
- **Minimal Dependencies**
  - Rather than taking on large dependencies like nalgebra, ncollide, ndarray,
    etc, we will first try to implement the simplest version of what is needed
    ourselves.
  - Integrations with 3rd party dependencies are exposed under feature flags to
    avoid bloat.
  - There is always a judgement call when determining if we should take on a
    dependency. The main considerations are build time, difficulty of "rolling
    our own," and of course the full feature set and performance of the
    dependency.

## Architecture

Noting the above principles, here is a quick summary of the design decisions
which brought about the current feature set:

- **Mathematical Data Types**
  - A voxel world can be modeled mathematically as a function over a
    3-dimensional integer lattice, i.e. with domain Z^3. Thus we have a
    `Point3i` type which serves as the main coordinate set.
  - When considering a subset of the voxel world, in coordinates, we very
    commonly desire an axis-aligned bounding box, which may contain any bounded
    subset. It is very simple to iterate over all of the points in such a box,
    find the intersection of two boxes, or check if the box contains a point. We
    call these boxes by the name `Extent3i`.
- **Storage**
  - Any extent of the voxel world function can be simply represented as a
    3-dimensional array. We call it `Array3`. Arrays contain some data at every
    point in some `Extent3i`. The most common operations on arrays are iteration
    over extents and random access. These operations should be very fast, and we
    have benchmarks for them.
  - Obviously we can't store an infinite voxel world, so we partition the
    lattice into chunks, each of which is an `Array3` of the same cubic shape.
    The container for these chunks is called a `ChunkMap3`.
  - With both the `Array3` and `ChunkMap3` serving similar purposes, we've made
    them implement a common set of traits for data access. This includes random
    access, iteration, and copying, using the `Get*`, `ForEach*`, `ReadExtent`,
    and `WriteExtent` traits.
  - When you have large voxel worlds, it's not feasible to store a lot of unique
    data for every voxel. A common strategy is to have each voxel labeled with
    some "type." If you only want to use a single byte for each voxel's type,
    then you can have up to 256 types of voxels. Then each type can have a large
    amount of data associated with it in a "palette" container. But we still
    want to be able to use our common set of access traits to read the voxel
    type data. Thus, we have a type called `TransformMap` which implements those
    traits. `TransformMap` wraps some other lattice map, referred to as the
    "delegate," and any access will first go through the delegate, then be
    transformed by an arbitrary `Fn(T) -> S` chosen by the programmer. This
    transformation closure is where we can access the palette based on the voxel
    type provided by the delegate map.
  - Even with only a couple bytes per voxel, we can still use up lots of memory
    on large voxel maps. The simplest way to save memory without changing the
    underlying array containers was to use compression inside of the `ChunkMap`.
    To make this optional, we made `ChunkMap` take a new chunk storage type
    parameter that can implement `ChunkReadStorage` or `ChunkWriteStorage`. This
    makes it so `ChunkMap`s can work with any kind of backing storage, be it a
    simple hash map or an LRU-cache of chunks that can be compressed.
- **Meshing**
  - There are many ways of generating meshes from voxel data. You can make each
    occupied voxel into a cube, which gives the classis Minecraft aesthetic. Or
    you can store geometric information, commonly referred to as "signed
    distances" or "hermite data," at each voxel in order to approximate smooth
    surfaces. We would like to support both schemes.
  - For cubic meshes, the fastest algorithm we know of to produce efficient
    meshes is coined as "Greedy Meshing" by the 0fps blog.
  - For smooth meshes, the most pervasive algorithm is Marching Cubes which is
    referred to as a "primal method." Marching Cubes places vertices on the
    edges of a voxel and constructs an independent mesh component for each
    voxel. There are also "dual methods," which place vertices on the interior
    of each voxel and connect them using a subset of the dual graph of the voxel
    lattice. We have found (Naive) Surface Nets, a dual method, to be simpler to
    implement and just as efficient, if not moreso, as Marching Cubes.
  - Another dual method, "Dual Contouring of Hermite Data," is essentially the
    same as Surface Nets, but it works on an octree and optimizes quadratic
    error functions (QEFs) to place vertices in more accurate locations, which
    helps with reproducing sharp features. This algorithm is rather difficult to
    optimize, so we are consciously taking it off the table in the near term.
  - While 3D voxel data is required for meshes with arbitrary topologies, one
    can choose to constrain themselves to a simpler planar topology and reap
    performance benefits, both in terms of space and time. A surface with planar
    topology can be modeled with a 2-dimensional function commonly referred to
    as a "height map." While we could represent a height map with an `Array3`
    where one of the dimensions has size 1, it leads to awkward code. Thus, we
    generalized all of the core data types to work in both 2 and 3 dimensions,
    which gives us the `Array2` type, capable of cleanly representing a height
    map. We've also implemented a specialized meshing algorithm for height maps.
- **Accelerated Spatial Queries**
  - Our first voxel game prototypes utilized the ncollide3d crate and it's
    `DBVT` (dynamic bounding volume tree) structure for doing raycasting.
    Unfortunately, storing an `AABB<f32>` for every voxel cost us 6 `f32`s or 24
    bytes per voxel. That simply doesn't scale. So as a replacement, we
    implemented the `OctreeSet` and `OctreeDBVT` types. The `OctreeSet` is
    essentially a hierarchical bitset, making it very memory efficient; it
    doesn't contain any voxel data, but it can tell you whether a `Point3i` is
    contained in the set. More importantly, it supports a visitor API that can
    be used for spatial queries like raycasting. Since the `OctreeSet` has a
    limited size, we also provide the `OctreeDBVT`, which is essentially a
    `DBVT` which may contain an arbitrary number of `OctreeSet`s.
  - `OctreeDBVT` requires taking on the `ncollide3d` dependency. We decided this
    was acceptable for the time being, since we don't have spare time to
    implement our own efficient `DBVT`.
