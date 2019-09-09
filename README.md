# data.rrbv

A Clojure implementation of immutable vectors, with efficient
concatenation and subvector implementations, based upon Relaxed Radix
Balanced Trees described in these papers:

* Phil Bagwell, Tiark Rompf, "RRB-Trees: Efficient Immutable Vectors",
  EPFL-REPORT-169879, September, 2011.  [[Semantic
  Scholar]](https://www.semanticscholar.org/paper/RRB-Trees-%3A-Efficient-Immutable-Vectors-Phil-Tiark-Bagwell-Rompf/30c8c562f6421ab6b00d0b7faebd897c407de69c),
  [[PDF]](http://infoscience.epfl.ch/record/169879/files/RMTrees.pdf)
* Nicolas Stucki, Tiark Rompf, Vlad Ureche, Phil Bagwell, "RRB vector:
  a practical general purpose immutable sequence", ICFP 2015,
  Proc. 20th ACM SIGPLAN International Conference on Functional
  Programming, 2015, [[ACM Digital
  Library]](https://dl.acm.org/citation.cfm?id=2784739),
  [[PDF]](https://infoscience.epfl.ch/record/213452/files/rrbvector.pdf)


## Usage

This will be fleshed out when the library is more complete, but the
current intention is to make the usage identical to the
[`core.rrb-vector`](https://github.com/clojure/core.rrb-vector)
library, but with better performance in terms of constant factors.


## License

Copyright © 2019 Zach Tellman, Andy Fingerhut

TBD: review the license choices more carefully later.

* What are changes between EPL 1.0 and 2.0?
* Do we want this dual licensed as GPL, or any other license?  I am
  fine leaving that out, personally.

This program and the accompanying materials are made available under the
terms of the Eclipse Public License 2.0 which is available at
http://www.eclipse.org/legal/epl-2.0.
