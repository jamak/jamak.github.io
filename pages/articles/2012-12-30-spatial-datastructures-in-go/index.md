---
title: "Spatial Datastructures in Go"
date: "2012-12-30T22:12:03.284Z"
layout: post
path: "/spatial-datastructures-1/"
category: "Go Quadtrees"
description: "A short introduction to writing spatial datastructures in Go."
---

Over this holiday break, I’ve been fooling around with Go, and using it for a side project that requires a quadtree. After implementing the quadtree, I thought it might be good practice to implement a few other spatial data structures with Go.

## Quadtrees

a [point quadtree](http://en.wikipedia.org/wiki/Quadtree#Point_quadtree) is basically a binary tree that is adapted to contain data about points, partitioning a 2-d space into quadrants, each quadrant having a max number of points. When a quadrant is full, it is divided into quadrants, et cetera et cetera.

so lets set up the basics we’re going to need before we even implement the quadtree. The first thing we’re going to need is a simple point structure to hold the location and the data:

```go
//a basic point struct
type Point struct {
    X,Y float64
}
```

and now we’ll need a boundary box (this is 2-D after all) and some functions to tell when it contains a point or intersects another box:

```go
//axis-aligned bounding box with half-dimension and center
type AABB struct {
  center, halfDimension Point
}

func (ab *AABB) containsPoint(p Point) bool {
  xRight := ab.center.X + ab.halfDimension.X
  xLeft := ab.center.X - ab.halfDimension.X
  yUp := ab.center.Y + ab.halfDimension.Y
  yDown := ab.center.Y - ab.halfDimension.Y
  return (p.X < xRight && p.X > xLeft && p.Y < yUp && p.Y > yDown)
}

func (ab *AABB) intersects(aabb AABB) bool {
  xRight := ab.center.X + ab.halfDimension.X
  xLeft := ab.center.X - ab.halfDimension.X
  yUp := ab.center.Y + ab.halfDimension.Y
  yDown := ab.center.Y - ab.halfDimension.Y
  pNE := Point{xRight, yUp}
  pNW := Point{xLeft, yUp}
  pSE := Point{xRight, yDown}
  pSW := Point{xLeft, yDown}
  return aabb.containsPoint(pNE) || aabb.containsPoint(pNW) ||
      aabb.containsPoint(pSE) || aabb.containsPoint(pSW) ||
      aabb.containsPoint(ab.center) || ab.containsPoint(aabb.center)
}
```

and with that, we can now define the quadtree struct in terms of these two structures.

```go
type QuadTree struct {
  //The capacity of each QT
  capacity int
  //The boundary of the QT
  boundary AABB
  //children
  northWest, northEast, southWest, southEast *QuadTree
  //elements in this QT
  points []Point
  ancestor *QuadTree
}
```

Now that we have the quadtree struct, we need to define methods to insert, delete, query and subdivide it.

For insertion, we need to first make sure the point we want to insert is contained in the view of the quadtree we’re trying to insert into, then see if the point will fit in this quadtree. if it can’t, we subdivide this quadrant and stick the point where it fits in.

```go
func (q *QuadTree) insert(p Point) bool {
  if !q.boundary.containsPoint(p) {
      return false
  }
  if len(q.points) < q.capacity && !q.hasChildren() {
      q.points = append(q.points, p)
      return true
  }
  if q.northWest == nil {
      q.subdivide()
  }
  if q.northWest.insert(p) {
      return true
  }
  if q.northEast.insert(p) {
      return true
  }
  if q.southWest.insert(p) {
      return true
  }
  if q.southEast.insert(p) {
      return true
  }
  return false
}
```

Querying is similar, in that you get a view and you query all the parts of the subtree that are encompassed by that view, and return all points contained in that query range.

```go
func (q QuadTree) queryRange(view AABB) []Point {
    t := *new([]Point)
    if !q.boundary.intersects(view) {
        return t
    }
    for _, point := range q.points {
        if view.containsPoint(point) {
            t = append(t, point)
        }
    }
    if q.northWest == nil {
        return t
    } else {
        t = append(t, q.northWest.queryRange(view)...)
        t = append(t, q.northEast.queryRange(view)...)
        t = append(t, q.southWest.queryRange(view)...)
        t = append(t, q.southEast.queryRange(view)...)
    }
    return t
}
```

When you subdivide, you split the current quadtree into 4 children, and then redistribute the points in the quadtree to the newly formed children.

```go
func (q *QuadTree) subdivide() {
    halfHalfD := Point{X: q.boundary.halfDimension.X * 0.5, Y: q.boundary.halfDimension.Y * 0.5}
    cNW := Point{q.boundary.center.X - halfHalfD.X, q.boundary.center.Y + halfHalfD.Y}
    cNE := Point{q.boundary.center.X + halfHalfD.X, q.boundary.center.Y + halfHalfD.Y}
    cSW := Point{q.boundary.center.X - halfHalfD.X, q.boundary.center.Y - halfHalfD.Y}
    cSE := Point{q.boundary.center.X + halfHalfD.X, q.boundary.center.Y - halfHalfD.Y}
    q.northWest = &QuadTree{capacity: CAPACITY, boundary: AABB{center: cNW, halfDimension: halfHalfD}}
    q.northEast = &QuadTree{capacity: CAPACITY, boundary: AABB{center: cNE, halfDimension: halfHalfD}}
    q.southWest = &QuadTree{capacity: CAPACITY, boundary: AABB{center: cSW, halfDimension: halfHalfD}}
    q.southEast = &QuadTree{capacity: CAPACITY, boundary: AABB{center: cSE, halfDimension: halfHalfD}}
    for _, point := range q.points {
        q.insert(point)
    }
    q.points = make([]Point, len(q.points))
}
```

at this point, you have all the stuff necessary for a simple implementation of a quadtree (this in particlar is very simple, in that it only holds points), but I’ll do a follow up post on the other side of the new year about a more complete and robust implementation of a quadtree (similar to the one I’m actually using!)
