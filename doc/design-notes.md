# Introduction

Why another persistent vector library?

Here are my goals:

1. I wanted a high performance implementation with both good asymptotic
   performance, i.e. good "big O" performance, but also with constant
   factors that are at least close to the best that anyone knows how to
   achieve.
2. O(log N) worst case run time implementations of vector concatenation
   and sub-vector creation operations, as RRB trees enable.
3. Tight integration with Clojure's built in persistent vectors, such
   that instances of those vectors can be reused almost completely as
   instances of the new vector type, with O(1) time to convert a
   Clojure persistent vector to the new type.
4. Support for O(1) conversion to a Clojure transient, or a transient
   back to a persistent vector.
5. Support the 'normal' Clojure vector behavior, in which every
   element can contain a value/object of arbitrary type, independently
   of the other elements, but also vectors like those created via
   Clojure's `vector-of` where every element is restricted to the same
   Java primitive type.  These can save significant memory for large
   vectors when their elements are all the same primitive type, and
   potentially have better constant factors for run time performance
   of operations.  As of Clojure 1.10.1 these primitive vectors do not
   have transient implementations.  I want those, too.

Summary of the properties of implementations I have examined:

| Goal | Clojure 1.10.1 | core.rrb-vector 0.0.14 | Bifurcan List 0.1.0 |
| ---- | -------------- | ---------------------- | ------------------- |
| 1. high performance | good | poor | best I have found |
| 2. big O run time   | good | good | good |
| 3. tight Clojure integration | good | good | needs additional code, and perhaps also modifications |
| 4. O(1) conversion to transient and back | yes | yes | yes |
| 5. primitive vectors | yes, but no transient | yes | no |

I started by fixing bugs in
[core.rrb-vector](https://github.com/clojure/core.rrb-vector).  The
results of that work are in the branch with a now out of date name
`proposed-fixes-for-4-issues` in [my fork of the core.rrb-vector
repository](https://github.com/jafingerhut/core.rrb-vector).

At that point I started doing performance comparisons of the
Clojure/Java implementation of `core.rrb-vector` against the `List`
class in Zach Tellman's
[`bifurcan`](https://github.com/lacuna/bifurcan), which despite its
name is also based upon the [RRB (Relaxed Radix Balanced) Trees data
structure](https://infoscience.epfl.ch/record/169879/files/RMTrees.pdf),
and thus supports efficient lookup of arbitrary elements by integer
index, as well as concatenation (aka splicing) and creating subvectors
(aka splitting).

I found several ways in which `core.rrb-vector`'s Clojure/Java
implementation to be slower than most of the other implementations
compared in `bifurcan`'s performance tests, and improved upon some of
those by tweaking a little bit of `core.rrb-vector`'s Clojure
implementation.  However, when I found that `nth` was about 5 times
slower than `bifurcan`'s, and could not find a way to make this
significantly faster by tweaking the Clojure implementation, it seemed
like a good idea to try for the maximum performance possible by
writing an implementation in Java.

I considered using `bifurcan`'s `List` class as is, or with small
modifications.  However, one nice feature of `core.rrb-vector` that
would be nice to preserve is that any built-in Clojure persistent
vector that has already been constructed in memory can be used as
(most of) a `core.rrb-vector` vector.  Only a small new "base" object
with a different class needs to be allocated and initialized, and it
then points at the same tree data structure in memory that the
`PersistentVector` class points at.  This makes for the most seamless
interoperability of `core.rrb-vector` vectors with Clojure's
`PersistentVector` that I can imagine.  I do not see how to make small
modifications to the `bifurcan` `List` class that could achieve that.

Other possibly desirable properties of a new implementation:

* Implement all of the same Java interfaces that Clojure
  PersistentVector does, e.g. implement IObj so that metadata can be
  attached to the new type.
* Consistently use named constants throughout the code related to the
  branch factor.  The value will almost certainly be 32, but it would
  be nice to be able to easily change the value in one line of the
  code, recompile, and be able to run performance tests with other
  values.
* It may be nice if it were straightforward for a third party to
  provide a new implementation for the "bottom level vectors".  This
  would enable others to extend the implementation to other types of
  vector elements other than the existing ones of "Object, or any Java
  primitive type'.  For example, vectors of single bits, with very
  efficient packed storage, would be good to enable, but note that
  particular vector element type would probably imply a different API
  than one that did not enable that, because of the need to twiddle
  bits in the middle of an int or long in the implementation.

Also note that for the last goal, it requires a bit of thought for
what the interface is between the RRB vector library and the writers
of "array managers".

I have thought of this as a performance experiment, too.
Unfortuantely I suspect that the most straightforward way to try it
would be to have lots of little tedious changes throughout the source
code.  I suspect it might make the interface between an "array
manager" implementer and the rest of the library a little bit messier.

* Create a version of the data structure that has one
  pointer/reference per tree level, rather than the two that Clojure
  PersistentVector, core.rrb-vector, and bifurcan List have, to see if
  there is any measurable performance improvement.  Right now PV and
  core.rrb-vector create nodes with Java's `final` modifier on small
  "node" objects that have only two references: one to the array of
  child nodes (or vector elements at the tree leaves), and the other
  to an "edit" object used for the transient implementation.

If this is a significant performance win, great.  One potential issue
that I am not sure about is whether this would violate any of the
ability of the data structure to be passed from one thread to another,
while making changes to transients by the first thread visible to
later threads that modify it.  Given the things I have learned and
described in this document, it seems that might not be a real issue


# Some detailed notes on core.rrb-vector and bifurcan List data structures

Some terms and abbreviations used in this section:

* _persistent_ - All data structures referred to as persistent here
  are immutable, and have the ability to efficiently do update
  operations on any such persistent data structure, even if it is an
  "older version" on which updates have been done before.  This is
  what is called "full persistence" on the [Wikipedia page for
  persistent data
  structures](https://en.wikipedia.org/wiki/Persistent_data_structure).
  This is called a forked data structure in the Bifurcan
  documentation.
* _transient_ - This term comes from the Clojure implementation of
  converting persistent data structures into a mutable data structure
  in O(1) time, making one or more mutations to it efficiently, with
  the ability to convert it back to a persistent data structure in
  O(1) time whenever you wish.  In the Bifurcan library, the
  corresponding method to convert from persistent to transient is
  called `linear`, and from transient back to persistent `forked`.
* _PV_: persistent vector. `clojure.lang.PersistentVector` = `(class
  [1 2 3])`, probably the most common type for vectors that most
  people use, when they use vectors in Clojure.
* _TV_: transient vector.
  `clojure.lang.PersistentVector$TransientVector` = `(class (transient
  [1 2 3]))`.  The type you get when you create a transient version of
  a Clojure vector.
* _Vec_: `clojure.core.Vec` = `(class (vector-of :long 1 2 3))`.  This
  is the class of vectors that are restricted to contain only elements
  of the same Java primitive type, e.g. `long`, `double`, etc., with
  the benefit of significantly lower memory usage and perhaps (TBD)
  also some performance benefits.
* _RPV_: core.rrb-vector relaxed radix persistent vector.
  `clojure.core.rrb_vector.rrbt.Vector` = `(class
  (clojure.core.rrb-vector/vector 1 2 3))`.  This is the default class
  of vectors you get if you use core.rrb-vector functions like
  `vector`, `subvec`, or `catvec`.
* _RTV_: core.rrb-vector relaxed radix transient vector.
  `clojure.core.rrb_vector.rrbt.Transient` = `(class (transient
  (clojure.core.rrb-vector/vector 1 2 3)))`.  This is the transient
  version of core.rrb-vector's vectors.
* _BPV_: Bifurcan persistent vector.  `io.lacuna.bifurcan.List` is the
  class for the persistent vector implementation in the bifurcan
  library.  Like RPV, it is also a relaxed radix vector.

These data structures have a lot in common, but also some differences.
They all have a "base object" that points directly at:

* a `root` node of a tree, and
* a `tail` array that contains the last few vector elements, up to 32.

BPV calls its corresponding field `suffix` instead of `tail`, and it
also has a corresponding `prefix` field pointing at an array of up to
32 elements, which are the first several vector elements.  I believe
it has this in order to more efficiently implement update operations
that prepend a few elements at the beginning of a large vector,
similar to how the tail enables all of these data structures to
implement appending one new element at the end in "usually O(1)" time
(except when the tail array is already full, in which case it is O(log
N) to put the tail into the tree at the proper location).

All of them have a tree of nodes, with the root field mentioned above
pointing ot the root node.

There are several invariants (i.e. conditions that are true always)
that are maintained about the structure of these trees, that all of
these implementations have in common.  First a little more
terminology:

* A _leaf node_ is one that does not have any nodes as children.  It
  contains up to 32 consecutive vector elements.
* An _internal node_ is one that only has other tree nodes as children
  (if it has any children at all), and never contains vector elements.
* The _right edge_ of a tree consists of the root node, the root
  node's rightmost child node C1, node C1's rightmost child node C2,
  etc. until you reach a leaf node.  The _left edge can be defined in
  an analogous way, through leftmost child nodes.  The _right edge_
  and _left edge_ of any tree node can be defined similarly, starting
  from that node instead of the root node, and proceeding downwards
  from there.

Some invariants:

* The root node is always an internal node, no matter how few elements
  are in the vector.  While one may think this wastes memory for
  storing small vectors, note that all vectors with 32 or fewer
  elements share the same identical root node in memory (named
  `EMPTY_NODE` in the Clojure implementation), and have all of their
  vector elements in the tail array.  Only vectors with 33 or more
  elements have a root node that is not this shared empty node object.
* All leaf nodes are at the same depth in the tree, i.e. the same
  number of steps from the root node.
* The PV and TV implementations are further restricted so that their
  trees are _strict_.  _Strict_ is the name used in the BPV
  implementation, which I like better than the term _regular_ used in
  the core.rrb-vector implementation.  Strict trees have all leaf
  nodes containing 32 vector elements, and every internal node that is
  not in the right edge set of nodes has 32 children.  Only tree nodes
  in the right edge set are allowed to have fewer than 32 children.
* We can also categorize a subtree rooted at a particular node N as
  strict.  This means that the subtree satisfies the conditions of
  being a strict tree, where the right edge set of nodes is calculated
  starting from that node and going downwards to the rightmost
  children.  A strict subtree restricts the leaf nodes beneath node N
  to contain 32 children, but places no restriction on leaf nodes that
  are not benath N in the tree.

If we know a subtree rooted at node N is strict, from that node
downwards we can do lookup operations more quickly, always choosing
child `(lookup_index >>> shift) & 0x1f` to go to next.  This cannot be
done for a non-strict subtree -- there we may need to do a little bit
more calculation or traversing of array elements to determine which
child node contains the `lookup_index`.

All of the implementations considered here have a Java class to
represent tree nodes that is a different class than the "base object".
All tree nodes have these fields:

* an `array` field.  For internal nodes each element of this array
  points at a child nodes.  For leaf nodes, the array consists of
  vector elements, which is an Object array for vectors that can
  contain elements of arbitrary type, or a Java array of primitives
  for vectors restricted to elements of one primitive type.  BPV names
  this field `nodes`, and I believe has no variant for vectors of only
  the same primitive type.
* an `edit` field, whose value is really only important when the node
  is part of a tree in a transient vector.  In that case, it indicates
  whether the node is owned by this transient, and thus is safe to
  mutate, or is not owned by this transient, and thus possibly shared
  with one or more other persistent vectors, and must remain
  unmutated.  BPV names this field `editor`.

The BPV implementation has several more fields in tree nodes that the
other implementations do not (class
`io.lacuna.bifurcan.nodes.ListNodes.Node`):

* `shift`, a byte.  This is stored explicitly in tree nodes in BPV,
  but is calculated on the fly in all other implementations considered
  here.  It is always 0 for leaf nodes, and because of the invariant
  that all leaf nodes are at the same depth from the root, every
  internal node has children all at the same depth, and therefore the
  same shift value.  The parent's shift value is exactly 5 more than
  their children's shift value.  In these data structures, every node
  remains at the same height from the leaves after it is created,
  although it can in some cases be included in a larger tree, due to
  concat operations, such that it can be at different depths from the
  roots of the different trees.

* `isStrict` - a boolean.  See the definition of strict above.  The
  core.rrb-vector does not store this value as a separate field.
  Instead it represents this distinction by all strict nodes having an
  array of 32 or fewer elements for its child nodes, and always has an
  array of exactly 33 elements for non-strict nodes, where the 33rd
  elements points at an array of index values.

  nodes except the "right spine" with a full 32 children.  A regular
  tree is the only kind of tree possible with the Clojure persistent
  vector implementation.  core.rrb-vector has no such field in its
  tree nodes, instead using arrays of 33 objects to indicate
  not-regular nodes, where the 33rd element points at an array of
  index values.

* `numNodes`

* `offsets` - an array of ints.  core.rrb-vector has a similar array,
  needed for implementing a relaxed radix search tree, except it puts
  them into the 33rd element of the array of up to 32 children.


All of them have one or two Java classes to represent tree nodes.

I will use the term "leaf node" not for a vector element, but for the
level of tree nodes that point at the vector elements directly, when
the elements are arbitrary objects, or they contain the array of
primitive elements, when the vector elements are Java primitive
values.

I will use the term "internal node" for a tree node that 

Both use the name `shift` for a non-negative integer value that is
used as the logical right shift amount, in number of bit positions,
for the vector index, in order to determine which child index to
descend into next in the tree.  This value is 0 for leaf
