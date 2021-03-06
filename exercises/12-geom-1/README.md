# Clip coordinates

## Exercise

In this exercise, you will implement a vertex shader which applies the sequence of operations discussed below to a 3D vertex. Your vertex shader should take a `vec3` attribute called `position` and `mat4` uniforms called `model`, `view` and `projection`. To get started a vertex shader called `transforms.glsl` has been created <a href="/open/12-geom-1" target="_blank">in this project's directory</a>, which you should edit to create your solution.

***

This lesson requires a bit more mathematics than the previous lessons.  However, it is necessary to understand this material in order to work with 3D graphics.

## Projective geometry

You may have noticed from the first example that `gl_Position` is a 4D vector. At first this might seem a bit funny, but there is a good reason for this. Objects on the GPU are represented in what is known as a *homogeneous* coordinate system. While this may seem abstract, understanding homogeneous coordinates - and more broadly projective geometry - is essential for graphics programming.

The basic idea in projective geometry is to replace points a space with lines passing through the origin in a space which is one dimension higher. One nice property of lines passing through the origin is that they can be parameterized by vectors.  For example, in a 3+1D projective geometry, the vector `[2, 0, 0, 1]` generates a line with points `[4, 0, 0, 2]`,  `[1, 0, 0, 0.5]`, `[2000, 0, 0, 1000]` and so on.  That is, we say that two vectors `[x0,y0,z0,w0]` and `[x1,y1,z1,w1]`, identify the same line if they are related by some non-zero scalar multiple `t != 0`:

```
        [x0,y0,z0,w0]  ~  [x1,y1,z1,w1]

            if and only if

[t*x0,t*y0,t*z0,t*w0]  ~  [x1,y1,z1,w1]
```


Lines through the origin can be identified with points in a normal Euclidean space by intersecting them with a hyperplane that does not pass through the origin.

In WebGL this hyperplane is taken to be the solution set to `w=1`.  To see how this works concretely, let us intersect a line generated by the vector `[0.2, 0.3, 0, 0.5]` with this hyperplane.  That is, it is the point `t * [0.2, 0.3, 0, 0.1]` where,

```
0.1 * t = 1
```

Solving for `t` gives `t=10`, so this line is identified with the 3D point `[2, 3, 0]`.

More generally, in WebGL any vector `[x,y,z,w]`, corresponds to the 3D point `[x/w,y/w,z/w]`.

### Points at infinity

However, projective geometry is a little bit different than plane geometry. Specifically, there are a few extra points which correspond to the lines that do not pass through the hyperplane.  In WebGL, these are the points where `w=0`. Geometers call these extra points, the *points at infinity*.  While these things may seem a bit funny at first, adding them greatly simplifies the problem of handling degeneracies and certain extraordinary situations, much like the `Infinity` number in JavaScript. One way to think of points at infinity is that they act like idealized direction vectors instead of proper points.

## Clip coordinates on the GPU

Clip coordinates, which are the coordinate system in which the GPU interprets `gl_Position` are related to screen coordinates according to the following relation:

```
screenColumn = 0.5 * screenWidth  * (gl_Position.x / gl_Position.w + 1.0)
screenRow    = 0.5 * screenHeight * (1.0 - gl_Position.y / gl_Position.w)
```

A similar equation holds for the depth value of the vertex:

```
depth = gl_Position.z / gl_Position.w
```

The main motivation (and also the origin of the name "clip" coordinates) comes from the fact that this rule greatly simplifies the problem of testing if a given point is visible or not. Specifically, all of the drawable geometry is "clipped" against a viewable frustum which is define by 6 inequalities:

```
-w < x < w
-w < y < w
 0 < z < w
```

Working backwards from the formula for projecting points in clip coordinates to the screen, you can check that the first two inequations are sufficient to determine that no points are drawn outside of the viewable bounds. The last inequation constrains that no objects behind the camera are drawn and that distant points where `z>=1` are clipped.

## Transformations

Working in projective coordinates also has the advantage that it greatly simplifies coordinate transformations. Specifically, any change of reference frame in plane geometry can be encoded as a 4x4 matrix in homogeneous coordinates.  These matrices are used throughout graphics to control the position and orientation of the camera, the shape of the viewable frustum and the location of objects within the scene.  We will cover the different types of transformations in more detail in the coming lessons, but for now it is enough to understand that applying a transformation encoded in a 4x4 matrix `m` to vector `p` can be done using the following convetion:

```glsl
vec4 transformPoint(mat4 transform, vec4 point) {
  return transform * point;
}
```

And that the concatenation of transformations is equivalent to the product of their matrices.  That is,

```glsl
transformPoint(A, transformPoint(B, p)) == transformPoint(A * B, p)
```

That this is true is a consequence of the fact that matrix multiplication is associative.

### The model-view-projection factorization

Many 3D graphical applications make use of 4 different coordinate systems:

* Data coordinates: Which are the coordinates of the vertices in the model
* World coordinates: Which are the coordinates of objects in the scene
* View coordinates: Which are the unprojected coordinates of the camera
* Clip coordinates: Which are the coordinates used by the GPU to render all primitives

The relationship between these coordinate systems is usually specified using 3 different transformations:

* `model`: Which transforms the object from data coordinates to world coordinates.  This controls the location of the object in the world.
* `view`: Which transforms world coordinate system to a viewing coordinate system.  This controls the position and orientation of the camera.
* `projection`: Which transforms the view coordinate system into device clip coordinates.  This controls whether the view is orthographic or perspective, and also controls the aspect ratio of the camera.

While it would in theory be sufficient to pass just one matrix which is the full data -> clip coordinate transformation, factoring the coordinate transformation into 3 phases can simplify various effects.  For example, some lighting operations must be applied in world coordinates, and some effects like billboarding for sprites need to be applied in a view coordinate system.
