
# SVG path to triangle mesh

Have you ever found yourself wanting to use a simple SVG path in OpenGL or another graphics library? As the object shall be solid 2D, you need to generate a triangle mesh from the path, of course with holes being possible. That's exactly what this JavaScript code does, with the help of the amazing [Triangle](https://www.cs.cmu.edu/~quake/triangle.html) library.

## Usage
First, run `npm install` to build the underlying _Triangle_ library from C/C++ source and to install all dependencies, then process a demo file with a command like `node index.js demo/person.svg`. This project automatically detects the only path in this SVG file and outputs a JSON object containing a `vertices` attribute, which is a long list of 3D coordinates (the z-coordinate is always 0). That list of vertices may, for example, be used as a vertex buffer together with `GL_TRIANGLES`.

This project has only been used and tested on Linux. I'd be happy to hear reports about how it works on Windows or Mac. Additionally, all SVG files this project has been tested with have been generated by Inkscape, so files created by programs using other conventions may not work.

## Demo
The following is an image, rendered using WebGL, that contains nine SVG paths which have been converted to triangles using this project.

![svg-path-to-triangle-mesh example screenshot](/demo/example.png)

## How does it work?
The initial SVG path data is pipelined through several stages, which are explained in the following.
1. Read the file and parse its XML using [xml2js](https://www.npmjs.com/package/xml2js).
2. Find all `<path>` elements in the resulting structure. If there is only one, use it, otherwise ask the user to specify the desired path's `id` attribute as an additional command-line argument.
3. Parse the path's `d` attribute using [parse-svg-path](https://www.npmjs.com/package/parse-svg-path). This will, for example, turn `'m1 2 l3 4'` into `[['m',1,2], ['l',3,4]]`.
4. Convert this information into a more verbose version of itself. For example, turn `[['c',1,2,3,4,5,6], ['L',7,8]]` into

```
[
  {
    "positioning": "relative",
    "type": "curveto",
    "controlBegin": [1,2],
    "controlEnd": [3,4],
    "dest": [5,6]
  },
  {
    "positioning": "absolute",
    "type": "lineto",
    "dest": [7,8]
  }
]
```

5. Make all coordinates be absolute, i.e. follow relative coordinates and update them accordingly.
6. Convert all `curveto` entries into straight lines, i.e. evaluate the underlying cubic bezier curves. By default, five points are generated for each curve.
7. Correct all coordinates to lay inside the range between 0 and 1. The aspect ratio is kept, and only the "longer" dimension of the path (regarding the axis-aligned bounding box) will reach 1.
8. Split the verbose path on entries with type `closepath`. Use the first subpath as the main, surrounding path and treat the others as holes inside the main one. As _Triangle_ needs single points inside each hole, use a [simple algorithm](https://mathoverflow.net/a/56660) to find them.
9. Eventually, call the `triangulate` function of _Triangle_ (which is easily callable from JavaScript, as it has been wrapped inside a [N-API](https://github.com/nodejs/node-addon-api) module). The returned object contains `pointlist` and `trianglelist` attributes. Indices into `pointlist`, three for each triangle, are resolved and the final object is written to stdout. Note that the coordinates of triangles output by _Triangle_ are always in counter-clockwise order, so make sure to possibly consider face culling here.
