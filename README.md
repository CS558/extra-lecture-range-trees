CS558: Range trees
==================
Last time we talked about orthogonal range searching, and we saw a simple data structure for solving the problem using only O(n) space giving O(n^{(d-1/d)}+k) queries.  

# Range trees
Unlike kd-trees, range trees give gauranteed O(log^d(n) + k) query time for orthogonal range searches.  But this comes at a cost, as they use O(n log^{d-1}(n)) space.  The basis for range trees comes from a different generalization of 1D range searching.

## Review: 1D range searching on trees
Probably the most straightforward way to find all elements in a binary search tree that are contained within an interval is to recursively insert the end points of the interval into the tree.  If we ever come to a subtree which is completely contained in the interval, then we return that entire subtree and terminate.  This generates a partition of the interval into ranges, each of which contains 2^k elements.  In this case, to report all the elements in one of these subtrees we can just do an O(n) traversal of the tree:

```javascript
function allPoints(tree) {
  if(!tree) {
    return []
  }
  return [tree.p].concat(allPoints(tree.left)).concat(allPoints(tree.right))
}
```

The number of these ranges is at most log_2(n), and each pair of ranges in the returned result does not intersect.  In total, the tree stores O(n) distinct ranges, not including the internal nodes of the tree.

## Range tree construction
The basic idea behind a range tree is to recursively apply this splitting to each of the ranges.  The idea is that we first construct a balanced binary tree ordered by the x-coordinate, then for every subtree within this tree, we recursively construct a second tree on all points in the subtree ordered by y.  In JavaScript, we could do this as follows:

```javascript
function makeTree(d, points) {
  //Check base case
  if(d >= dimension || points.length === 0) {
    return null
  }
  //Find midpoint
  points.sort(function(a,b) {
    return a[d] - b[d]
  })
  var mid = points.length >> 1
  //Construct children
  return {
    d: d,
    p: points[mid],
    w: points.length
    subtree: makeTree(d+1, points.slice()),
    left: makeTree(d, points.slice(0, mid)),
    right: makeTree(d, points.slice(mid+1)),
    lo: points[0][d],
    hi: points[points.length-1][d]
  }
}
```

At first it might seem like we are building a lot of extra subtrees, but it turns out that it isn't too bad.  The key thing to realize is that at each level of the tree, there are exactly n extra nodes in all the subtrees.  So the total amount of storage (in 2D anyway) is proportional to n * height of the tree, or O(n log(n)).  We can continue this process out to higher dimensions, and we get a storage cost of O(n log^{d-1}(n)) overall.  The one slightly tricky thing here is how we select the mid point for each subtree.  If we use a fast median selection algorithm, then the overall construction time will reduce to merely O(n log^{d-1}(n)), which is optimal.  However, because we are using a sort() here to save some keystrokes the total cost is actually O(n log^{d}(n)).


## Range queries

To query a range tree, we first search along x to get a decomposition of the range into dyadic intervals.  Then for each of those subregions, we search again recursively until we have exhausted all of the points:

```javascript
function queryBox(tree, lo, hi) {
  //Check base case
  if(!tree) {
    return []
  }
  //Check if interval is completely contained
  var d = tree.d
  if(lo[d] <= tree.lo && tree.hi <= hi[d]) {
    if(tree.subtree) {
      return queryBox(tree.subtree, lo, hi)
    } else {
      return allPoints(tree)
    }
  }
  //Check if range completely to the left or to the right
  if(hi[d] < tree.p[d]) {
    return queryBox(tree.left, lo, hi)
  } else if(tree.p[d] < lo[d]) {
    return queryBox(tree.right, lo, hi)
  }
  //Check if point is contained in range
  var result = queryBox(tree.left, lo, hi).concat(queryBox(tree.right, lo, hi))
  for(var i=d; i<dimension; ++i) {
    if(!(lo[i] <= tree.p[i] && tree.p[i] <= hi[i])) {
      return result
    }
  }
  result.push(tree.p)
  return result
}
```

In the first situation, we get O(log(n)) ranges, each of subsequently partitions into at most O(log(n)) subranges and so on.  So the total cost of this algorithm is O(log^d(n) + k) in general.  Compare this to a kdtree, where range searching could take up to O(n^{1-1/d}+k), and it is clear that we get a pretty big benefit.  However, there are still a few tricks that we can do to speed this up.

## Advanced topics

### Fractional cascading

We can save an extra log factor on the query time in range trees without substantially increasing the space costs.  The key idea behind this is an algorithmic technique called *fractional cascading*.  The general version of fractional cascading is a way to speed up successive binary searches in sorted lists using the same key.  In the context of range trees, this is applied to the specific task of finding the upper and lower bounds on the y-coordinate.

#### Cascading
There are two main parts of fractional cascading.  The first part, or "cascading" speeds up binary searches by storing pointers to the indices of the elements in the next list.  This means that we can reuse the work we do on the first search to speed up the search on the next phase.  The problem with this is that we might end up in a situation where all the pointers are clumped together.  If this happens, then the speed up from the cascade will be very small.

#### Fractional propagation
To avoid this, we can propagate some fraction (say half) of the second array into the first, and then use these elements to speed up the binary search.  That way when we search the first array we are gaunateed that the next pointer will be within +/-1 of the exact index.  Using this trick we can reduce the cost of all binary searches after the first to an O(1) array look up.

### Dynamization

The above data structure is perfectly adequate for static data, but what if we want to insert or remove points?  It seems like it could be a really hard problem.  Generalizations of data structures like red/black or AVL trees to range trees and kdtrees are known, but they are hideously complicated.  However, there is a simple trick that gives us optimal *amortized* performance at no extra cost.

The main insight is to realize that it is trivial to update a range tree if we don't care about maintaining balance.  To insert, just walk the tree and insert into all the appropriate subtrees.  Deletion requires a bit more work, but basically boils down to walking a subtree to find an appropriate ancestor and moving it to the right position.

The problem with doing all these ad-hoc updates though is that the tree will over time become unbalanced, and so we lose the efficiency of our queries.  However, the idea behind BB-alpha trees is that this is ok providing we don't let it get too out of whack.  The basic concept is that we pick some constant 0 < alpha < 1/2 that determines how much sloppiness we will allow in our tree.  The goal is to enforce the invariant that subtrees are nearly balanced:

w(left(p)) <= (1 - alpha) * w(p)
w(right(p)) <= (1 - alpha) * w(p)

One can show that in this situation, the height of the tree is at most log_(1-1/alpha)(n), which for alpha>0 is still O(log(n)).  So what happens if a tree becomes unbalanced such that these invariants aren't satisified anymore?  In a BB-alpha tree we just rebuild it!  It turns out that amortized the cost of this rebuilding is completely paid for by the updates themself.  Here is some JavaScript which shows how to implement the BB-alpha rebalancing for insertion into a range tree:

```javascript
function insert(tree, p) {
  var d = tree.d
  //Figure out which subtree to insert
  var insertLeft = (p <= tree.p[d])
  var child = insertLeft ? tree.left : tree.right
  //Get number of children in subtree
  var wchild = 0
  if(child) {
    wchild = child.w
  }
  //Check if we need to rebuild tree
  if(4 * (wchild + 1) > 3 * (tree.w + 1)) {
    var points = allPoints(tree)
    points.push(p)
    var ntree = makeTree(d, points)
    for(var id in ntree) {
      tree[id] = ntree[id]
    }
    return
  }
  //Increment weight
  ++tree.w
  //Insert into child
  if(child) {
    insert(child, p)
  } else if(insertLeft) {
    tree.left = makeTree(d, [p])
  } else {
    tree.right = makeTree(d, [p])
  }
  //Insert into associated data structure
  if(d < dimension-1) {
    if(tree.subtree) {
      insert(tree.subtree, p)
    } else {
      tree.subtree = makeTree(d+1, [p])
    }
  }
}
```

A similar trick is possible for removing points.  It is also worth pointing out that this concept applies equally well to kdtrees.

### Layered range trees
We can also replace the tree data structure here with an array, though life becomes more complicated.  For an example of how this can be done see the following module:

[https://github.com/mikolalysenko/static-range-query](https://github.com/mikolalysenko/static-range-query)