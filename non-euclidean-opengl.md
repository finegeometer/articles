# Non-Euclidean OpenGL

In this article, I plan to show you how to render a non-Euclidean scene in OpenGL.

## What I Mean by Non-Euclidean

"Non-Euclidean geometry" is a term that gets used in several different ways. Some people use the term to mean any weirdly connected space. Others, including myself, prefer a more specific definition. Rather than just having a few portals, the *entire space* should act differently than Euclidean space.

I will focus on two types of non-Euclidean geometry: *spherical* and *hyperbolic* geometry. The method I show is unlikely to work for other types of geometry.

In spherical geometry, lines tend to bend toward each other. Think of lines of longitude on the Earth; they seem parallel near the equator, but then bend toward each other until they meet at the poles. Squares have obtuse angles, and the larger the square, the bigger the angles. The entire space tends to close up on itself, so if you wander too far, you end up back where you started.

In hyperbolic geometry, the opposite is true. Lines tend to diverge, squares have acute angles, and there is an incredible amount of space within even a short distance of you. If you wander off and get lost, you almost certainly won't find your way home.
The best way I know of to get an intuition for hyperbolic space is to play [*HyperRogue*](https://www.roguetemple.com/z/hyper/), a game by ZenoRogue that takes place in the hyperbolic plane.

## Preview
*This is a preview, so feel free to skip over the code in this section. It is, however, a fully working example, so if you want to try it out, you can.*

Implementing spherical or hyperbolic geometry is surprisingly simple. In fact, it's no harder than Euclidean geometry!

Don't believe me? Here's a simple Euclidean renderer:
<details>

```html
<html>
<meta content="text/html;charset=utf-8" http-equiv="Content-Type" />

<body style="background-color: black;">
    <script>
        let canvas_size = 800;

        // MATRIX STUFF //

        // Multiply two matrices, stored as length 16 arrays in column-major order.
        function matrix_mul(m1, m2) {
            let out = Array(16).fill(0);
            for (let i = 0; i < 4; i++) {
                for (let j = 0; j < 4; j++) {
                    for (let k = 0; k < 4; k++) {
                        out[4 * i + k] += m1[4 * j + k] * m2[4 * i + j];
                    }
                }
            }
            return out;
        }

        let perspective = [
            1, 0, 0, 0,
            0, 1, 0, 0,
            0, 0, -1, -1,
            0, 0, -0.02, 0,
        ];

        let transform = [
            1, 0, 0, 0,
            0, 1, 0, 0,
            0, 0, 1, 0,
            0, 0, 0, 1
        ];

        // GL STUFF //
        let canvas = window.document.createElement("canvas");
        canvas.setAttribute("width", canvas_size);
        canvas.setAttribute("height", canvas_size);
        window.document.body.appendChild(canvas);

        let gl = canvas.getContext("webgl2");

        gl.clearColor(0.1, 0.1, 0.1, 1.0);
        gl.viewport(0, 0, canvas_size, canvas_size);
        gl.enable(gl.DEPTH_TEST);

        let vshader = gl.createShader(gl.VERTEX_SHADER);
        gl.shaderSource(vshader,
            '#version 300 es\n' +
            'in vec4 pos;\n' +
            'out vec4 vpos;\n' +
            'uniform mat4 transform;\n' +
            'uniform mat4 perspective;\n' +
            'void main() {\n' +
            '   vpos = transform * pos;\n' +
            '   gl_Position = perspective * vpos\n;' +
            '}\n'
        );
        gl.compileShader(vshader);

        let fshader = gl.createShader(gl.FRAGMENT_SHADER);
        gl.shaderSource(fshader,
            '#version 300 es\n' +
            'precision mediump float;\n' +
            'in vec4 vpos;\n' +
            'out vec4 color;\n' +
            'void main() {\n' +
            '   float distance = length(vpos / vpos.w - vec4(0., 0., 0., 1.));\n' +
            '   color = mix(vec4(0.1, 0.1, 0.1, 0.1), vec4(0.2, 0.8, 0.6, 1.0), exp(-distance));\n' +
            '}\n'
        );
        gl.compileShader(fshader);

        let program = gl.createProgram();
        gl.attachShader(program, vshader);
        gl.attachShader(program, fshader);
        gl.linkProgram(program);
        gl.useProgram(program);

        gl.deleteShader(vshader);
        gl.deleteShader(fshader);

        let vao = gl.createVertexArray();
        gl.bindVertexArray(vao);

        let vertex_buffer = gl.createBuffer();
        gl.bindBuffer(gl.ARRAY_BUFFER, vertex_buffer);

        let attrib_pos = gl.getAttribLocation(program, "pos");
        gl.enableVertexAttribArray(attrib_pos);
        gl.vertexAttribPointer(attrib_pos, 4, gl.FLOAT, false, 4 * 4, 0);

        // Mesh data. Annoyingly long, but it *is* 192 triangles.
        let data = Float32Array.from([
            +1.0, +1.1, +1.0, +1.0, /**/ +1.0, +1.0, +1.1, +1.0, /**/ -1.0, +1.0, +1.1, +1.0,
            -1.0, +1.0, +1.1, +1.0, /**/ -1.0, +1.1, +1.0, +1.0, /**/ +1.0, +1.1, +1.0, +1.0,
            +1.0, +1.0, +1.0, +1.1, /**/ +1.0, +1.1, +1.0, +1.0, /**/ -1.0, +1.1, +1.0, +1.0,
            -1.0, +1.1, +1.0, +1.0, /**/ -1.0, +1.0, +1.0, +1.1, /**/ +1.0, +1.0, +1.0, +1.1,
            +1.0, +1.0, +1.1, +1.0, /**/ +1.0, +1.0, +1.0, +1.1, /**/ -1.0, +1.0, +1.0, +1.1,
            -1.0, +1.0, +1.0, +1.1, /**/ -1.0, +1.0, +1.1, +1.0, /**/ +1.0, +1.0, +1.1, +1.0,
            +1.0, +1.1, +1.0, -1.0, /**/ +1.0, +1.0, +1.1, -1.0, /**/ -1.0, +1.0, +1.1, -1.0,
            -1.0, +1.0, +1.1, -1.0, /**/ -1.0, +1.1, +1.0, -1.0, /**/ +1.0, +1.1, +1.0, -1.0,
            +1.0, +1.0, +1.0, -1.1, /**/ +1.0, +1.1, +1.0, -1.0, /**/ -1.0, +1.1, +1.0, -1.0,
            -1.0, +1.1, +1.0, -1.0, /**/ -1.0, +1.0, +1.0, -1.1, /**/ +1.0, +1.0, +1.0, -1.1,
            +1.0, +1.0, +1.1, -1.0, /**/ +1.0, +1.0, +1.0, -1.1, /**/ -1.0, +1.0, +1.0, -1.1,
            -1.0, +1.0, +1.0, -1.1, /**/ -1.0, +1.0, +1.1, -1.0, /**/ +1.0, +1.0, +1.1, -1.0,
            +1.0, +1.1, -1.0, +1.0, /**/ +1.0, +1.0, -1.1, +1.0, /**/ -1.0, +1.0, -1.1, +1.0,
            -1.0, +1.0, -1.1, +1.0, /**/ -1.0, +1.1, -1.0, +1.0, /**/ +1.0, +1.1, -1.0, +1.0,
            +1.0, +1.0, -1.0, +1.1, /**/ +1.0, +1.1, -1.0, +1.0, /**/ -1.0, +1.1, -1.0, +1.0,
            -1.0, +1.1, -1.0, +1.0, /**/ -1.0, +1.0, -1.0, +1.1, /**/ +1.0, +1.0, -1.0, +1.1,
            +1.0, +1.0, -1.1, +1.0, /**/ +1.0, +1.0, -1.0, +1.1, /**/ -1.0, +1.0, -1.0, +1.1,
            -1.0, +1.0, -1.0, +1.1, /**/ -1.0, +1.0, -1.1, +1.0, /**/ +1.0, +1.0, -1.1, +1.0,
            +1.0, +1.1, -1.0, -1.0, /**/ +1.0, +1.0, -1.1, -1.0, /**/ -1.0, +1.0, -1.1, -1.0,
            -1.0, +1.0, -1.1, -1.0, /**/ -1.0, +1.1, -1.0, -1.0, /**/ +1.0, +1.1, -1.0, -1.0,
            +1.0, +1.0, -1.0, -1.1, /**/ +1.0, +1.1, -1.0, -1.0, /**/ -1.0, +1.1, -1.0, -1.0,
            -1.0, +1.1, -1.0, -1.0, /**/ -1.0, +1.0, -1.0, -1.1, /**/ +1.0, +1.0, -1.0, -1.1,
            +1.0, +1.0, -1.1, -1.0, /**/ +1.0, +1.0, -1.0, -1.1, /**/ -1.0, +1.0, -1.0, -1.1,
            -1.0, +1.0, -1.0, -1.1, /**/ -1.0, +1.0, -1.1, -1.0, /**/ +1.0, +1.0, -1.1, -1.0,
            +1.0, -1.1, +1.0, +1.0, /**/ +1.0, -1.0, +1.1, +1.0, /**/ -1.0, -1.0, +1.1, +1.0,
            -1.0, -1.0, +1.1, +1.0, /**/ -1.0, -1.1, +1.0, +1.0, /**/ +1.0, -1.1, +1.0, +1.0,
            +1.0, -1.0, +1.0, +1.1, /**/ +1.0, -1.1, +1.0, +1.0, /**/ -1.0, -1.1, +1.0, +1.0,
            -1.0, -1.1, +1.0, +1.0, /**/ -1.0, -1.0, +1.0, +1.1, /**/ +1.0, -1.0, +1.0, +1.1,
            +1.0, -1.0, +1.1, +1.0, /**/ +1.0, -1.0, +1.0, +1.1, /**/ -1.0, -1.0, +1.0, +1.1,
            -1.0, -1.0, +1.0, +1.1, /**/ -1.0, -1.0, +1.1, +1.0, /**/ +1.0, -1.0, +1.1, +1.0,
            +1.0, -1.1, +1.0, -1.0, /**/ +1.0, -1.0, +1.1, -1.0, /**/ -1.0, -1.0, +1.1, -1.0,
            -1.0, -1.0, +1.1, -1.0, /**/ -1.0, -1.1, +1.0, -1.0, /**/ +1.0, -1.1, +1.0, -1.0,
            +1.0, -1.0, +1.0, -1.1, /**/ +1.0, -1.1, +1.0, -1.0, /**/ -1.0, -1.1, +1.0, -1.0,
            -1.0, -1.1, +1.0, -1.0, /**/ -1.0, -1.0, +1.0, -1.1, /**/ +1.0, -1.0, +1.0, -1.1,
            +1.0, -1.0, +1.1, -1.0, /**/ +1.0, -1.0, +1.0, -1.1, /**/ -1.0, -1.0, +1.0, -1.1,
            -1.0, -1.0, +1.0, -1.1, /**/ -1.0, -1.0, +1.1, -1.0, /**/ +1.0, -1.0, +1.1, -1.0,
            +1.0, -1.1, -1.0, +1.0, /**/ +1.0, -1.0, -1.1, +1.0, /**/ -1.0, -1.0, -1.1, +1.0,
            -1.0, -1.0, -1.1, +1.0, /**/ -1.0, -1.1, -1.0, +1.0, /**/ +1.0, -1.1, -1.0, +1.0,
            +1.0, -1.0, -1.0, +1.1, /**/ +1.0, -1.1, -1.0, +1.0, /**/ -1.0, -1.1, -1.0, +1.0,
            -1.0, -1.1, -1.0, +1.0, /**/ -1.0, -1.0, -1.0, +1.1, /**/ +1.0, -1.0, -1.0, +1.1,
            +1.0, -1.0, -1.1, +1.0, /**/ +1.0, -1.0, -1.0, +1.1, /**/ -1.0, -1.0, -1.0, +1.1,
            -1.0, -1.0, -1.0, +1.1, /**/ -1.0, -1.0, -1.1, +1.0, /**/ +1.0, -1.0, -1.1, +1.0,
            +1.0, -1.1, -1.0, -1.0, /**/ +1.0, -1.0, -1.1, -1.0, /**/ -1.0, -1.0, -1.1, -1.0,
            -1.0, -1.0, -1.1, -1.0, /**/ -1.0, -1.1, -1.0, -1.0, /**/ +1.0, -1.1, -1.0, -1.0,
            +1.0, -1.0, -1.0, -1.1, /**/ +1.0, -1.1, -1.0, -1.0, /**/ -1.0, -1.1, -1.0, -1.0,
            -1.0, -1.1, -1.0, -1.0, /**/ -1.0, -1.0, -1.0, -1.1, /**/ +1.0, -1.0, -1.0, -1.1,
            +1.0, -1.0, -1.1, -1.0, /**/ +1.0, -1.0, -1.0, -1.1, /**/ -1.0, -1.0, -1.0, -1.1,
            -1.0, -1.0, -1.0, -1.1, /**/ -1.0, -1.0, -1.1, -1.0, /**/ +1.0, -1.0, -1.1, -1.0,
            +1.1, +1.0, +1.0, +1.0, /**/ +1.0, +1.0, +1.0, +1.1, /**/ +1.0, -1.0, +1.0, +1.1,
            +1.0, -1.0, +1.0, +1.1, /**/ +1.1, -1.0, +1.0, +1.0, /**/ +1.1, +1.0, +1.0, +1.0,
            +1.0, +1.0, +1.1, +1.0, /**/ +1.1, +1.0, +1.0, +1.0, /**/ +1.1, -1.0, +1.0, +1.0,
            +1.1, -1.0, +1.0, +1.0, /**/ +1.0, -1.0, +1.1, +1.0, /**/ +1.0, +1.0, +1.1, +1.0,
            +1.0, +1.0, +1.0, +1.1, /**/ +1.0, +1.0, +1.1, +1.0, /**/ +1.0, -1.0, +1.1, +1.0,
            +1.0, -1.0, +1.1, +1.0, /**/ +1.0, -1.0, +1.0, +1.1, /**/ +1.0, +1.0, +1.0, +1.1,
            +1.1, +1.0, -1.0, +1.0, /**/ +1.0, +1.0, -1.0, +1.1, /**/ +1.0, -1.0, -1.0, +1.1,
            +1.0, -1.0, -1.0, +1.1, /**/ +1.1, -1.0, -1.0, +1.0, /**/ +1.1, +1.0, -1.0, +1.0,
            +1.0, +1.0, -1.1, +1.0, /**/ +1.1, +1.0, -1.0, +1.0, /**/ +1.1, -1.0, -1.0, +1.0,
            +1.1, -1.0, -1.0, +1.0, /**/ +1.0, -1.0, -1.1, +1.0, /**/ +1.0, +1.0, -1.1, +1.0,
            +1.0, +1.0, -1.0, +1.1, /**/ +1.0, +1.0, -1.1, +1.0, /**/ +1.0, -1.0, -1.1, +1.0,
            +1.0, -1.0, -1.1, +1.0, /**/ +1.0, -1.0, -1.0, +1.1, /**/ +1.0, +1.0, -1.0, +1.1,
            +1.1, +1.0, +1.0, -1.0, /**/ +1.0, +1.0, +1.0, -1.1, /**/ +1.0, -1.0, +1.0, -1.1,
            +1.0, -1.0, +1.0, -1.1, /**/ +1.1, -1.0, +1.0, -1.0, /**/ +1.1, +1.0, +1.0, -1.0,
            +1.0, +1.0, +1.1, -1.0, /**/ +1.1, +1.0, +1.0, -1.0, /**/ +1.1, -1.0, +1.0, -1.0,
            +1.1, -1.0, +1.0, -1.0, /**/ +1.0, -1.0, +1.1, -1.0, /**/ +1.0, +1.0, +1.1, -1.0,
            +1.0, +1.0, +1.0, -1.1, /**/ +1.0, +1.0, +1.1, -1.0, /**/ +1.0, -1.0, +1.1, -1.0,
            +1.0, -1.0, +1.1, -1.0, /**/ +1.0, -1.0, +1.0, -1.1, /**/ +1.0, +1.0, +1.0, -1.1,
            +1.1, +1.0, -1.0, -1.0, /**/ +1.0, +1.0, -1.0, -1.1, /**/ +1.0, -1.0, -1.0, -1.1,
            +1.0, -1.0, -1.0, -1.1, /**/ +1.1, -1.0, -1.0, -1.0, /**/ +1.1, +1.0, -1.0, -1.0,
            +1.0, +1.0, -1.1, -1.0, /**/ +1.1, +1.0, -1.0, -1.0, /**/ +1.1, -1.0, -1.0, -1.0,
            +1.1, -1.0, -1.0, -1.0, /**/ +1.0, -1.0, -1.1, -1.0, /**/ +1.0, +1.0, -1.1, -1.0,
            +1.0, +1.0, -1.0, -1.1, /**/ +1.0, +1.0, -1.1, -1.0, /**/ +1.0, -1.0, -1.1, -1.0,
            +1.0, -1.0, -1.1, -1.0, /**/ +1.0, -1.0, -1.0, -1.1, /**/ +1.0, +1.0, -1.0, -1.1,
            -1.1, +1.0, +1.0, +1.0, /**/ -1.0, +1.0, +1.0, +1.1, /**/ -1.0, -1.0, +1.0, +1.1,
            -1.0, -1.0, +1.0, +1.1, /**/ -1.1, -1.0, +1.0, +1.0, /**/ -1.1, +1.0, +1.0, +1.0,
            -1.0, +1.0, +1.1, +1.0, /**/ -1.1, +1.0, +1.0, +1.0, /**/ -1.1, -1.0, +1.0, +1.0,
            -1.1, -1.0, +1.0, +1.0, /**/ -1.0, -1.0, +1.1, +1.0, /**/ -1.0, +1.0, +1.1, +1.0,
            -1.0, +1.0, +1.0, +1.1, /**/ -1.0, +1.0, +1.1, +1.0, /**/ -1.0, -1.0, +1.1, +1.0,
            -1.0, -1.0, +1.1, +1.0, /**/ -1.0, -1.0, +1.0, +1.1, /**/ -1.0, +1.0, +1.0, +1.1,
            -1.1, +1.0, -1.0, +1.0, /**/ -1.0, +1.0, -1.0, +1.1, /**/ -1.0, -1.0, -1.0, +1.1,
            -1.0, -1.0, -1.0, +1.1, /**/ -1.1, -1.0, -1.0, +1.0, /**/ -1.1, +1.0, -1.0, +1.0,
            -1.0, +1.0, -1.1, +1.0, /**/ -1.1, +1.0, -1.0, +1.0, /**/ -1.1, -1.0, -1.0, +1.0,
            -1.1, -1.0, -1.0, +1.0, /**/ -1.0, -1.0, -1.1, +1.0, /**/ -1.0, +1.0, -1.1, +1.0,
            -1.0, +1.0, -1.0, +1.1, /**/ -1.0, +1.0, -1.1, +1.0, /**/ -1.0, -1.0, -1.1, +1.0,
            -1.0, -1.0, -1.1, +1.0, /**/ -1.0, -1.0, -1.0, +1.1, /**/ -1.0, +1.0, -1.0, +1.1,
            -1.1, +1.0, +1.0, -1.0, /**/ -1.0, +1.0, +1.0, -1.1, /**/ -1.0, -1.0, +1.0, -1.1,
            -1.0, -1.0, +1.0, -1.1, /**/ -1.1, -1.0, +1.0, -1.0, /**/ -1.1, +1.0, +1.0, -1.0,
            -1.0, +1.0, +1.1, -1.0, /**/ -1.1, +1.0, +1.0, -1.0, /**/ -1.1, -1.0, +1.0, -1.0,
            -1.1, -1.0, +1.0, -1.0, /**/ -1.0, -1.0, +1.1, -1.0, /**/ -1.0, +1.0, +1.1, -1.0,
            -1.0, +1.0, +1.0, -1.1, /**/ -1.0, +1.0, +1.1, -1.0, /**/ -1.0, -1.0, +1.1, -1.0,
            -1.0, -1.0, +1.1, -1.0, /**/ -1.0, -1.0, +1.0, -1.1, /**/ -1.0, +1.0, +1.0, -1.1,
            -1.1, +1.0, -1.0, -1.0, /**/ -1.0, +1.0, -1.0, -1.1, /**/ -1.0, -1.0, -1.0, -1.1,
            -1.0, -1.0, -1.0, -1.1, /**/ -1.1, -1.0, -1.0, -1.0, /**/ -1.1, +1.0, -1.0, -1.0,
            -1.0, +1.0, -1.1, -1.0, /**/ -1.1, +1.0, -1.0, -1.0, /**/ -1.1, -1.0, -1.0, -1.0,
            -1.1, -1.0, -1.0, -1.0, /**/ -1.0, -1.0, -1.1, -1.0, /**/ -1.0, +1.0, -1.1, -1.0,
            -1.0, +1.0, -1.0, -1.1, /**/ -1.0, +1.0, -1.1, -1.0, /**/ -1.0, -1.0, -1.1, -1.0,
            -1.0, -1.0, -1.1, -1.0, /**/ -1.0, -1.0, -1.0, -1.1, /**/ -1.0, +1.0, -1.0, -1.1,
            +1.0, +1.0, +1.0, +1.1, /**/ +1.1, +1.0, +1.0, +1.0, /**/ +1.1, +1.0, -1.0, +1.0,
            +1.1, +1.0, -1.0, +1.0, /**/ +1.0, +1.0, -1.0, +1.1, /**/ +1.0, +1.0, +1.0, +1.1,
            +1.0, +1.1, +1.0, +1.0, /**/ +1.0, +1.0, +1.0, +1.1, /**/ +1.0, +1.0, -1.0, +1.1,
            +1.0, +1.0, -1.0, +1.1, /**/ +1.0, +1.1, -1.0, +1.0, /**/ +1.0, +1.1, +1.0, +1.0,
            +1.1, +1.0, +1.0, +1.0, /**/ +1.0, +1.1, +1.0, +1.0, /**/ +1.0, +1.1, -1.0, +1.0,
            +1.0, +1.1, -1.0, +1.0, /**/ +1.1, +1.0, -1.0, +1.0, /**/ +1.1, +1.0, +1.0, +1.0,
            +1.0, -1.0, +1.0, +1.1, /**/ +1.1, -1.0, +1.0, +1.0, /**/ +1.1, -1.0, -1.0, +1.0,
            +1.1, -1.0, -1.0, +1.0, /**/ +1.0, -1.0, -1.0, +1.1, /**/ +1.0, -1.0, +1.0, +1.1,
            +1.0, -1.1, +1.0, +1.0, /**/ +1.0, -1.0, +1.0, +1.1, /**/ +1.0, -1.0, -1.0, +1.1,
            +1.0, -1.0, -1.0, +1.1, /**/ +1.0, -1.1, -1.0, +1.0, /**/ +1.0, -1.1, +1.0, +1.0,
            +1.1, -1.0, +1.0, +1.0, /**/ +1.0, -1.1, +1.0, +1.0, /**/ +1.0, -1.1, -1.0, +1.0,
            +1.0, -1.1, -1.0, +1.0, /**/ +1.1, -1.0, -1.0, +1.0, /**/ +1.1, -1.0, +1.0, +1.0,
            -1.0, +1.0, +1.0, +1.1, /**/ -1.1, +1.0, +1.0, +1.0, /**/ -1.1, +1.0, -1.0, +1.0,
            -1.1, +1.0, -1.0, +1.0, /**/ -1.0, +1.0, -1.0, +1.1, /**/ -1.0, +1.0, +1.0, +1.1,
            -1.0, +1.1, +1.0, +1.0, /**/ -1.0, +1.0, +1.0, +1.1, /**/ -1.0, +1.0, -1.0, +1.1,
            -1.0, +1.0, -1.0, +1.1, /**/ -1.0, +1.1, -1.0, +1.0, /**/ -1.0, +1.1, +1.0, +1.0,
            -1.1, +1.0, +1.0, +1.0, /**/ -1.0, +1.1, +1.0, +1.0, /**/ -1.0, +1.1, -1.0, +1.0,
            -1.0, +1.1, -1.0, +1.0, /**/ -1.1, +1.0, -1.0, +1.0, /**/ -1.1, +1.0, +1.0, +1.0,
            -1.0, -1.0, +1.0, +1.1, /**/ -1.1, -1.0, +1.0, +1.0, /**/ -1.1, -1.0, -1.0, +1.0,
            -1.1, -1.0, -1.0, +1.0, /**/ -1.0, -1.0, -1.0, +1.1, /**/ -1.0, -1.0, +1.0, +1.1,
            -1.0, -1.1, +1.0, +1.0, /**/ -1.0, -1.0, +1.0, +1.1, /**/ -1.0, -1.0, -1.0, +1.1,
            -1.0, -1.0, -1.0, +1.1, /**/ -1.0, -1.1, -1.0, +1.0, /**/ -1.0, -1.1, +1.0, +1.0,
            -1.1, -1.0, +1.0, +1.0, /**/ -1.0, -1.1, +1.0, +1.0, /**/ -1.0, -1.1, -1.0, +1.0,
            -1.0, -1.1, -1.0, +1.0, /**/ -1.1, -1.0, -1.0, +1.0, /**/ -1.1, -1.0, +1.0, +1.0,
            +1.0, +1.0, +1.0, -1.1, /**/ +1.1, +1.0, +1.0, -1.0, /**/ +1.1, +1.0, -1.0, -1.0,
            +1.1, +1.0, -1.0, -1.0, /**/ +1.0, +1.0, -1.0, -1.1, /**/ +1.0, +1.0, +1.0, -1.1,
            +1.0, +1.1, +1.0, -1.0, /**/ +1.0, +1.0, +1.0, -1.1, /**/ +1.0, +1.0, -1.0, -1.1,
            +1.0, +1.0, -1.0, -1.1, /**/ +1.0, +1.1, -1.0, -1.0, /**/ +1.0, +1.1, +1.0, -1.0,
            +1.1, +1.0, +1.0, -1.0, /**/ +1.0, +1.1, +1.0, -1.0, /**/ +1.0, +1.1, -1.0, -1.0,
            +1.0, +1.1, -1.0, -1.0, /**/ +1.1, +1.0, -1.0, -1.0, /**/ +1.1, +1.0, +1.0, -1.0,
            +1.0, -1.0, +1.0, -1.1, /**/ +1.1, -1.0, +1.0, -1.0, /**/ +1.1, -1.0, -1.0, -1.0,
            +1.1, -1.0, -1.0, -1.0, /**/ +1.0, -1.0, -1.0, -1.1, /**/ +1.0, -1.0, +1.0, -1.1,
            +1.0, -1.1, +1.0, -1.0, /**/ +1.0, -1.0, +1.0, -1.1, /**/ +1.0, -1.0, -1.0, -1.1,
            +1.0, -1.0, -1.0, -1.1, /**/ +1.0, -1.1, -1.0, -1.0, /**/ +1.0, -1.1, +1.0, -1.0,
            +1.1, -1.0, +1.0, -1.0, /**/ +1.0, -1.1, +1.0, -1.0, /**/ +1.0, -1.1, -1.0, -1.0,
            +1.0, -1.1, -1.0, -1.0, /**/ +1.1, -1.0, -1.0, -1.0, /**/ +1.1, -1.0, +1.0, -1.0,
            -1.0, +1.0, +1.0, -1.1, /**/ -1.1, +1.0, +1.0, -1.0, /**/ -1.1, +1.0, -1.0, -1.0,
            -1.1, +1.0, -1.0, -1.0, /**/ -1.0, +1.0, -1.0, -1.1, /**/ -1.0, +1.0, +1.0, -1.1,
            -1.0, +1.1, +1.0, -1.0, /**/ -1.0, +1.0, +1.0, -1.1, /**/ -1.0, +1.0, -1.0, -1.1,
            -1.0, +1.0, -1.0, -1.1, /**/ -1.0, +1.1, -1.0, -1.0, /**/ -1.0, +1.1, +1.0, -1.0,
            -1.1, +1.0, +1.0, -1.0, /**/ -1.0, +1.1, +1.0, -1.0, /**/ -1.0, +1.1, -1.0, -1.0,
            -1.0, +1.1, -1.0, -1.0, /**/ -1.1, +1.0, -1.0, -1.0, /**/ -1.1, +1.0, +1.0, -1.0,
            -1.0, -1.0, +1.0, -1.1, /**/ -1.1, -1.0, +1.0, -1.0, /**/ -1.1, -1.0, -1.0, -1.0,
            -1.1, -1.0, -1.0, -1.0, /**/ -1.0, -1.0, -1.0, -1.1, /**/ -1.0, -1.0, +1.0, -1.1,
            -1.0, -1.1, +1.0, -1.0, /**/ -1.0, -1.0, +1.0, -1.1, /**/ -1.0, -1.0, -1.0, -1.1,
            -1.0, -1.0, -1.0, -1.1, /**/ -1.0, -1.1, -1.0, -1.0, /**/ -1.0, -1.1, +1.0, -1.0,
            -1.1, -1.0, +1.0, -1.0, /**/ -1.0, -1.1, +1.0, -1.0, /**/ -1.0, -1.1, -1.0, -1.0,
            -1.0, -1.1, -1.0, -1.0, /**/ -1.1, -1.0, -1.0, -1.0, /**/ -1.1, -1.0, +1.0, -1.0,
            +1.0, +1.0, +1.1, +1.0, /**/ +1.0, +1.1, +1.0, +1.0, /**/ +1.0, +1.1, +1.0, -1.0,
            +1.0, +1.1, +1.0, -1.0, /**/ +1.0, +1.0, +1.1, -1.0, /**/ +1.0, +1.0, +1.1, +1.0,
            +1.1, +1.0, +1.0, +1.0, /**/ +1.0, +1.0, +1.1, +1.0, /**/ +1.0, +1.0, +1.1, -1.0,
            +1.0, +1.0, +1.1, -1.0, /**/ +1.1, +1.0, +1.0, -1.0, /**/ +1.1, +1.0, +1.0, +1.0,
            +1.0, +1.1, +1.0, +1.0, /**/ +1.1, +1.0, +1.0, +1.0, /**/ +1.1, +1.0, +1.0, -1.0,
            +1.1, +1.0, +1.0, -1.0, /**/ +1.0, +1.1, +1.0, -1.0, /**/ +1.0, +1.1, +1.0, +1.0,
            -1.0, +1.0, +1.1, +1.0, /**/ -1.0, +1.1, +1.0, +1.0, /**/ -1.0, +1.1, +1.0, -1.0,
            -1.0, +1.1, +1.0, -1.0, /**/ -1.0, +1.0, +1.1, -1.0, /**/ -1.0, +1.0, +1.1, +1.0,
            -1.1, +1.0, +1.0, +1.0, /**/ -1.0, +1.0, +1.1, +1.0, /**/ -1.0, +1.0, +1.1, -1.0,
            -1.0, +1.0, +1.1, -1.0, /**/ -1.1, +1.0, +1.0, -1.0, /**/ -1.1, +1.0, +1.0, +1.0,
            -1.0, +1.1, +1.0, +1.0, /**/ -1.1, +1.0, +1.0, +1.0, /**/ -1.1, +1.0, +1.0, -1.0,
            -1.1, +1.0, +1.0, -1.0, /**/ -1.0, +1.1, +1.0, -1.0, /**/ -1.0, +1.1, +1.0, +1.0,
            +1.0, -1.0, +1.1, +1.0, /**/ +1.0, -1.1, +1.0, +1.0, /**/ +1.0, -1.1, +1.0, -1.0,
            +1.0, -1.1, +1.0, -1.0, /**/ +1.0, -1.0, +1.1, -1.0, /**/ +1.0, -1.0, +1.1, +1.0,
            +1.1, -1.0, +1.0, +1.0, /**/ +1.0, -1.0, +1.1, +1.0, /**/ +1.0, -1.0, +1.1, -1.0,
            +1.0, -1.0, +1.1, -1.0, /**/ +1.1, -1.0, +1.0, -1.0, /**/ +1.1, -1.0, +1.0, +1.0,
            +1.0, -1.1, +1.0, +1.0, /**/ +1.1, -1.0, +1.0, +1.0, /**/ +1.1, -1.0, +1.0, -1.0,
            +1.1, -1.0, +1.0, -1.0, /**/ +1.0, -1.1, +1.0, -1.0, /**/ +1.0, -1.1, +1.0, +1.0,
            -1.0, -1.0, +1.1, +1.0, /**/ -1.0, -1.1, +1.0, +1.0, /**/ -1.0, -1.1, +1.0, -1.0,
            -1.0, -1.1, +1.0, -1.0, /**/ -1.0, -1.0, +1.1, -1.0, /**/ -1.0, -1.0, +1.1, +1.0,
            -1.1, -1.0, +1.0, +1.0, /**/ -1.0, -1.0, +1.1, +1.0, /**/ -1.0, -1.0, +1.1, -1.0,
            -1.0, -1.0, +1.1, -1.0, /**/ -1.1, -1.0, +1.0, -1.0, /**/ -1.1, -1.0, +1.0, +1.0,
            -1.0, -1.1, +1.0, +1.0, /**/ -1.1, -1.0, +1.0, +1.0, /**/ -1.1, -1.0, +1.0, -1.0,
            -1.1, -1.0, +1.0, -1.0, /**/ -1.0, -1.1, +1.0, -1.0, /**/ -1.0, -1.1, +1.0, +1.0,
            +1.0, +1.0, -1.1, +1.0, /**/ +1.0, +1.1, -1.0, +1.0, /**/ +1.0, +1.1, -1.0, -1.0,
            +1.0, +1.1, -1.0, -1.0, /**/ +1.0, +1.0, -1.1, -1.0, /**/ +1.0, +1.0, -1.1, +1.0,
            +1.1, +1.0, -1.0, +1.0, /**/ +1.0, +1.0, -1.1, +1.0, /**/ +1.0, +1.0, -1.1, -1.0,
            +1.0, +1.0, -1.1, -1.0, /**/ +1.1, +1.0, -1.0, -1.0, /**/ +1.1, +1.0, -1.0, +1.0,
            +1.0, +1.1, -1.0, +1.0, /**/ +1.1, +1.0, -1.0, +1.0, /**/ +1.1, +1.0, -1.0, -1.0,
            +1.1, +1.0, -1.0, -1.0, /**/ +1.0, +1.1, -1.0, -1.0, /**/ +1.0, +1.1, -1.0, +1.0,
            -1.0, +1.0, -1.1, +1.0, /**/ -1.0, +1.1, -1.0, +1.0, /**/ -1.0, +1.1, -1.0, -1.0,
            -1.0, +1.1, -1.0, -1.0, /**/ -1.0, +1.0, -1.1, -1.0, /**/ -1.0, +1.0, -1.1, +1.0,
            -1.1, +1.0, -1.0, +1.0, /**/ -1.0, +1.0, -1.1, +1.0, /**/ -1.0, +1.0, -1.1, -1.0,
            -1.0, +1.0, -1.1, -1.0, /**/ -1.1, +1.0, -1.0, -1.0, /**/ -1.1, +1.0, -1.0, +1.0,
            -1.0, +1.1, -1.0, +1.0, /**/ -1.1, +1.0, -1.0, +1.0, /**/ -1.1, +1.0, -1.0, -1.0,
            -1.1, +1.0, -1.0, -1.0, /**/ -1.0, +1.1, -1.0, -1.0, /**/ -1.0, +1.1, -1.0, +1.0,
            +1.0, -1.0, -1.1, +1.0, /**/ +1.0, -1.1, -1.0, +1.0, /**/ +1.0, -1.1, -1.0, -1.0,
            +1.0, -1.1, -1.0, -1.0, /**/ +1.0, -1.0, -1.1, -1.0, /**/ +1.0, -1.0, -1.1, +1.0,
            +1.1, -1.0, -1.0, +1.0, /**/ +1.0, -1.0, -1.1, +1.0, /**/ +1.0, -1.0, -1.1, -1.0,
            +1.0, -1.0, -1.1, -1.0, /**/ +1.1, -1.0, -1.0, -1.0, /**/ +1.1, -1.0, -1.0, +1.0,
            +1.0, -1.1, -1.0, +1.0, /**/ +1.1, -1.0, -1.0, +1.0, /**/ +1.1, -1.0, -1.0, -1.0,
            +1.1, -1.0, -1.0, -1.0, /**/ +1.0, -1.1, -1.0, -1.0, /**/ +1.0, -1.1, -1.0, +1.0,
            -1.0, -1.0, -1.1, +1.0, /**/ -1.0, -1.1, -1.0, +1.0, /**/ -1.0, -1.1, -1.0, -1.0,
            -1.0, -1.1, -1.0, -1.0, /**/ -1.0, -1.0, -1.1, -1.0, /**/ -1.0, -1.0, -1.1, +1.0,
            -1.1, -1.0, -1.0, +1.0, /**/ -1.0, -1.0, -1.1, +1.0, /**/ -1.0, -1.0, -1.1, -1.0,
            -1.0, -1.0, -1.1, -1.0, /**/ -1.1, -1.0, -1.0, -1.0, /**/ -1.1, -1.0, -1.0, +1.0,
            -1.0, -1.1, -1.0, +1.0, /**/ -1.1, -1.0, -1.0, +1.0, /**/ -1.1, -1.0, -1.0, -1.0,
            -1.1, -1.0, -1.0, -1.0, /**/ -1.0, -1.1, -1.0, -1.0, /**/ -1.0, -1.1, -1.0, +1.0,
        ]);

        gl.bufferData(gl.ARRAY_BUFFER,
            data,
            gl.STATIC_DRAW,
        );

        // MAIN LOOP //
        let key_pressed = {};
        window.addEventListener("keydown", function (e) { key_pressed[e.code] = true });
        window.addEventListener("keyup", function (e) { key_pressed[e.code] = false });

        let time;

        function animation_frame_callback(t) {
            if (!time) { time = t; }
            // time since last frame, in seconds
            let dt = (t - time) / 1000;
            time = t;

            // Rotate up
            if (key_pressed["ArrowUp"]) {
                transform = matrix_mul([
                    1, 0, 0, 0,
                    0, Math.cos(dt), -Math.sin(dt), 0,
                    0, Math.sin(dt), Math.cos(dt), 0,
                    0, 0, 0, 1,
                ], transform);
            }
            // Rotate down
            if (key_pressed["ArrowDown"]) {
                transform = matrix_mul([
                    1, 0, 0, 0,
                    0, Math.cos(dt), Math.sin(dt), 0,
                    0, -Math.sin(dt), Math.cos(dt), 0,
                    0, 0, 0, 1,
                ], transform);
            }
            // Rotate left
            if (key_pressed["ArrowLeft"]) {
                transform = matrix_mul([
                    Math.cos(dt), 0, Math.sin(dt), 0,
                    0, 1, 0, 0,
                    -Math.sin(dt), 0, Math.cos(dt), 0,
                    0, 0, 0, 1,
                ], transform);
            }
            // Rotate right
            if (key_pressed["ArrowRight"]) {
                transform = matrix_mul([
                    Math.cos(dt), 0, -Math.sin(dt), 0,
                    0, 1, 0, 0,
                    Math.sin(dt), 0, Math.cos(dt), 0,
                    0, 0, 0, 1,
                ], transform);
            }


            // Move forward
            if (key_pressed["KeyW"]) {
                transform = matrix_mul([
                    1, 0, 0, 0,
                    0, 1, 0, 0,
                    0, 0, 1, 0,
                    0, 0, dt, 1
                ], transform);
            }
            // Move backward
            if (key_pressed["KeyS"]) {
                transform = matrix_mul([
                    1, 0, 0, 0,
                    0, 1, 0, 0,
                    0, 0, 1, 0,
                    0, 0, -dt, 1
                ], transform);
            }
            // Move down
            if (key_pressed["ShiftLeft"]) {
                transform = matrix_mul([
                    1, 0, 0, 0,
                    0, 1, 0, 0,
                    0, 0, 1, 0,
                    0, dt, 0, 1
                ], transform);
            }
            // Move up
            if (key_pressed["Space"]) {
                transform = matrix_mul([
                    1, 0, 0, 0,
                    0, 1, 0, 0,
                    0, 0, 1, 0,
                    0, -dt, 0, 1
                ], transform);
            }
            // Move right
            if (key_pressed["KeyD"]) {
                transform = matrix_mul([
                    1, 0, 0, 0,
                    0, 1, 0, 0,
                    0, 0, 1, 0,
                    -dt, 0, 0, 1
                ], transform);
            }
            // Move left
            if (key_pressed["KeyA"]) {
                transform = matrix_mul([
                    1, 0, 0, 0,
                    0, 1, 0, 0,
                    0, 0, 1, 0,
                    dt, 0, 0, 1
                ], transform);
            }

            gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);
            gl.uniformMatrix4fv(gl.getUniformLocation(program, "transform"), false, transform);
            gl.uniformMatrix4fv(gl.getUniformLocation(program, "perspective"), false, perspective);
            gl.drawArrays(gl.TRIANGLES, 0, data.length / 4);

            window.requestAnimationFrame(animation_frame_callback);
        };

        window.requestAnimationFrame(animation_frame_callback);
    </script>
</body>

</html>
```
</details>

Now, replace a few of the matrices in the movement code...
<details>

```js
// // Move forward
// if (key_pressed["KeyW"]) {
//     transform = matrix_mul([
//         1, 0, 0, 0,
//         0, 1, 0, 0,
//         0, 0, 1, 0,
//         0, 0, dt, 1
//     ], transform);
// }
// // Move backward
// if (key_pressed["KeyS"]) {
//     transform = matrix_mul([
//         1, 0, 0, 0,
//         0, 1, 0, 0,
//         0, 0, 1, 0,
//         0, 0, -dt, 1
//     ], transform);
// }
// // Move down
// if (key_pressed["ShiftLeft"]) {
//     transform = matrix_mul([
//         1, 0, 0, 0,
//         0, 1, 0, 0,
//         0, 0, 1, 0,
//         0, dt, 0, 1
//     ], transform);
// }
// // Move up
// if (key_pressed["Space"]) {
//     transform = matrix_mul([
//         1, 0, 0, 0,
//         0, 1, 0, 0,
//         0, 0, 1, 0,
//         0, -dt, 0, 1
//     ], transform);
// }
// // Move right
// if (key_pressed["KeyD"]) {
//     transform = matrix_mul([
//         1, 0, 0, 0,
//         0, 1, 0, 0,
//         0, 0, 1, 0,
//         -dt, 0, 0, 1
//     ], transform);
// }
// // Move left
// if (key_pressed["KeyA"]) {
//     transform = matrix_mul([
//         1, 0, 0, 0,
//         0, 1, 0, 0,
//         0, 0, 1, 0,
//         dt, 0, 0, 1
//     ], transform);
// }

// Move forward
if (key_pressed["KeyW"]) {
    transform = matrix_mul([
        1, 0, 0, 0,
        0, 1, 0, 0,
        0, 0, Math.cos(dt), -Math.sin(dt),
        0, 0, Math.sin(dt), Math.cos(dt)
    ], transform);
}
// Move backward
if (key_pressed["KeyS"]) {
    transform = matrix_mul([
        1, 0, 0, 0,
        0, 1, 0, 0,
        0, 0, Math.cos(dt), Math.sin(dt),
        0, 0, -Math.sin(dt), Math.cos(dt)
    ], transform);
}
// Move down
if (key_pressed["ShiftLeft"]) {
    transform = matrix_mul([
        1, 0, 0, 0,
        0, Math.cos(dt), 0, -Math.sin(dt),
        0, 0, 1, 0,
        0, Math.sin(dt), 0, Math.cos(dt)
    ], transform);
}
// Move up
if (key_pressed["Space"]) {
    transform = matrix_mul([
        1, 0, 0, 0,
        0, Math.cos(dt), 0, Math.sin(dt),
        0, 0, 1, 0,
        0, -Math.sin(dt), 0, Math.cos(dt)
    ], transform);
}
// Move right
if (key_pressed["KeyD"]) {
    transform = matrix_mul([
        Math.cos(dt), 0, 0, Math.sin(dt),
        0, 1, 0, 0,
        0, 0, 1, 0,
        -Math.sin(dt), 0, 0, Math.cos(dt)
    ], transform);
}
// Move left
if (key_pressed["KeyA"]) {
    transform = matrix_mul([
        Math.cos(dt), 0, 0, -Math.sin(dt),
        0, 1, 0, 0,
        0, 0, 1, 0,
        Math.sin(dt), 0, 0, Math.cos(dt)
    ], transform);
}
```
</details>

... change the perspective matrix ...
<details>

```js
// let perspective = [
//     1, 0, 0, 0,
//     0, 1, 0, 0,
//     0, 0, -1, -1,
//     0, 0, -0.02, 0,
// ];

let perspective = [
    1, 0, 0, 0,
    0, 1, 0, 0,
    0, 0, 0, -1,
    0, 0, -0.01, 0
];
```
</details>

... and slightly modify the shader code ...
<details>

```glsl
// float distance = length(vpos / vpos.w - vec4(0., 0., 0., 1.));
float distance = acos(normalize(vpos).w);
```
</details>

... and now we're in spherical geometry.

How is this possible?

## Projective Geometry

This is possible because OpenGL doesn't really work with Euclidean *or* spherical geometry. It works with what is called *projective geometry*.

In projective geometry, you have points, lines, and planes, but no concept of distance or angle. So concepts like *intersection* or *triangle* make sense, but not *midpoint* or *perpendicular*. The equivalent of a rigid motion is a *projective transformation*, which is just any transformation that turns lines into lines and planes into planes.
Projective geometry may seem limited, but it is enough for what OpenGL has to do -- figuring out which objects are hidden behind others. It is also convenient, because perspective matrices are projective transformations.


There is a useful coordinate system for three-dimensional projective space. A point in projective 3-space is represented by a non-zero 4-vector `[x,y,z,w]`. But two vectors represent the same point if one is a scaled version of the other. Only the *ratios* between `x`, `y`, `z`, and `w` matter, not their absolute sizes. To emphasize this, we usually write the coordinates as `[x:y:z:w]`.

For example, `[1:2:3:4]`, `[2:4:6:8]`, and `[-3:-6:-9:-12]` are the same point, but `[2:3:4:5]` is different.

A projective transformation is represented by an invertible 4x4 matrix. This is why you see `mat4`s all over OpenGL. The projective transformation `M`, as you might expect, transforms point `v` to point `M*v`.

### Getting Euclid Back

I described projective geometry as "missing" the concepts of distance and angle. We can turn that around: Euclidean geometry is projective geometry plus distances and angles.
But then I gave a representation of projective geometry in terms of 4-vectors. So we should be able to find Euclidean geometry "hidden" in that description. But where?

Suppose we have a point `[x:y:z:w]` in projective space. If `w ≠ 0`, it can also be written as `[x/w : y/w : z/w : 1]`. We can think of this as the point `(x/w, y/w, z/w)` in Euclidean space.
If you've seen points represented as `[x,y,z,1]` in OpenGL, this is why.

Then we can define distance and angle. For example, to calculate the Euclidean distance between projective points `[0:0:0:5]` and `[1:2:2:3]`:
- Rewrite them as `[0:0:0:1]` and `[1/3 : 2/3 : 2/3 : 1]`.
- Interpret these as Euclidean points `(0, 0, 0)` and `(1/3, 2/3, 2/3)`.
- Use the Pythagorean theorem to get `(1/3)^2 + (2/3)^2 + (2/3)^2 = 1`.

We also care about the *isometries*, or rigid motions, of Euclidean space. These are those projective transformations that do not change the distances between points. It takes an annoying amount of algebra to prove, but these matrices are isometries:
```
⎡ cos(t) sin(t) 0 0⎤   ⎡ cos(t) 0 sin(t) 0⎤   ⎡1      0      0  0⎤
⎢-sin(t) cos(t) 0 0⎥   ⎢     0  1     0  0⎥   ⎢0  cos(t) sin(t) 0⎥
⎢     0      0  1 0⎥   ⎢-sin(t) 0 cos(t) 0⎥   ⎢0 -sin(t) cos(t) 0⎥
⎣     0      0  0 1⎦   ⎣     0  0     0  1⎦   ⎣0      0      0  1⎦

⎡1 0 0 t⎤   ⎡1 0 0 0⎤   ⎡1 0 0 0⎤
⎢0 1 0 0⎥   ⎢0 1 0 t⎥   ⎢0 1 0 0⎥
⎢0 0 1 0⎥   ⎢0 0 1 0⎥   ⎢0 0 1 t⎥
⎣0 0 0 1⎦   ⎣0 0 0 1⎦   ⎣0 0 0 1⎦
```

You may recognise the first three as rotation matrices, and the last three as translations.

### Spherical Geometry

That's good to know, but we're here for *non-Euclidean* geometry. How do we do that?

Here's the trick: spherical geometry is *also* projective geometry plus distances and angles! Before, we noticed we could rewrite (almost) all of the points into the form `[x,y,z,1]`, and noticed that looks like a plane. But we can instead rewrite the points into the form `x*x + y*y + z*z + w*w = 1`, which looks like a sphere!

To calculate the spherical distance between points `[a:b:c:d]` and `[x:y:z:w]`:
- Rewrite the first point such that `a*a + b*b + c*c + d*d = 1`.
- Rewrite the second point such that `x*x + y*y + z*z + w*w = 1`.
- Return `acos(a*x + b*y + c*z + d*w)`.

As before, we care about the isometries, or rigid motions. The three rotation matrices still work, but the translation matrices don't. Instead, we get three new "translation matrices":
```
⎡ cos(t) 0 0 sin(t)⎤   ⎡1      0  0     0 ⎤   ⎡1 0      0      0 ⎤
⎢     0  1 0     0 ⎥   ⎢0  cos(t) 0 sin(t)⎥   ⎢0 1      0      0 ⎥
⎢     0  0 1     0 ⎥   ⎢0      0  1     0 ⎥   ⎢0 0  cos(t) sin(t)⎥
⎣-sin(t) 0 0 cos(t)⎦   ⎣0 -sin(t) 0 cos(t)⎦   ⎣0 0 -sin(t) cos(t)⎦
```

<details>
<summary>Technical issues</summary>

There are two technical issues here:
1.) There are two ways to rewrite a point `[a:b:c:d]` such that `a*a + b*b + c*c + d*d = 1`. This fact would end up "folding" our sphere so that opposite points are the same.
2.) OpenGL doesn't *exactly* use projective geometry; it distinguishes a point `p` from `-p`, even though they are the same in projective geometry.

These two issues exactly cancel out, and everything works perfectly.

</details>

### Hyperbolic Geometry

Hyperbolic geometry is, once again, projective geometry plus distances and angles.
The distance function looks similar to spherical geometry, but with a few minus signs and an `h`:
- Rewrite the first point such that `a*a + b*b + c*c - d*d = -1`.
- Rewrite the second point such that `x*x + y*y + z*z - w*w = -1`.
- Return `acosh(-(a*x + b*y + c*z - d*w))`.

This time, we get these "translation matrices":
```
⎡cosh(t) 0 0 sinh(t)⎤   ⎡1      0  0      0 ⎤   ⎡1 0      0       0 ⎤
⎢     0  1 0      0 ⎥   ⎢0 cosh(t) 0 sinh(t)⎥   ⎢0 1      0       0 ⎥
⎢     0  0 1      0 ⎥   ⎢0      0  1      0 ⎥   ⎢0 0 cosh(t) sinh(t)⎥
⎣sinh(t) 0 0 cosh(t)⎦   ⎣0 sinh(t) 0 cosh(t)⎦   ⎣0 0 sinh(t) cosh(t)⎦
```

Also, notice that points are only in the hyperbolic space if `x*x + y*y + z*z < w*w`. Make sure to keep your polygons inside this range -- not because it breaks any of the code, but just because it's confusing to see objects that are "beyond infinity".

## Putting it together

So we want to make a spherical renderer.

As I mentioned, it is similar to a Euclidean renderer. So start with a Euclidean renderer, and switch out everything specifically Euclidean.

For example, take my renderer from the preview. What parts of it are specific to Euclidean geometry?

Let's see...
- When you move, the transform matrix is multiplied by a translation.
- The fragment shader calculates the distance from you to an object, for the fog effect.

... And that's it.


So, replace the translation matrices with their spherical versions. Replace the Euclidean distance function with the spherical one. In a less contrived example, you might also have to switch out some `vec3`s for `vec4`s.


But there is one issue. The render distance only reaches a quarter of the way around the sphere.

Why? Suppose you're in spherical space, at `[0:0:0:1]`, looking towards an object at `[0 : 0 : -sin(d) : cos(d)]`. The object is distance `d` away.
The object's position can be rewritten as `[0 : 0 : -tan(d) : 1]`, so it will only be rendered if `tan(d) < z_far`.
So we need `z_far = tan(maximum_view_distance)`, instead of `z_far = maximum_view_distance`.

But there seems to be a problem. `tan(π/2)` is infinite. Surely we can't crank `z_far` any higher?

We can. If you make `z_far` negative, *nothing actually breaks*! Normally, we don't do this, because there's no point to pushing the render distance past infinity.
But we can just set `maximum_view_distance = π - 0.01`, `z_far = -0.01`, and everything just works.

Things *do* break if `maximum_view_distance ≥ π`. So I stopped there. But there is a workaround for that.
*Warning: I have not tested this workaround.*

As you look outward, your lines of sight diverge. But since we're in spherical space, they reconverge at the antipode. After crossing the antipode, they diverge again, until they reconverge once more back where they started.

Notice that the things you see at distances `π` to `2π` are the same things you would see at distances `0` to `π` if you were standing at the antipode.

So if want a view distance greater than `π`, do two render passes.
1.) Render the far things. (Pass `-transform` to your shader.)
2.) Render the near things. (Pass `transform` to your shader.)

## Warning about hyperbolic space

<details>
If you implement hyperbolic geometry, floating point error builds up extremely fast. This manifests itself in two ways.

First, if you wander too far, the simulation will break down. Since hyperblic space expands exponentially, you quickly *literally run out of coordinates*!
For example, a sphere of radius 100 has a volume of over 2^289, while a vector of four 64-bit floats only has 2^256 possible values.

Second, if you represent your position and orientation as a transformation matrix, rounding error may cause it to drift.
Let's say your `transform` matrix is the identity. There are sixteen independent "directions" it can drift.

If it drifts in this direction, you're fine. It will act exactly the same.
```
⎡+ 0 0 0⎤
⎢0 + 0 0⎥
⎢0 0 + 0⎥
⎣0 0 0 +⎦
```

If it drifts in one of these six directions, you're still fine. You're just in a slightly different position or orientation.
```
⎡0 + 0 0⎤ ⎡0 0 + 0⎤ ⎡0 0 0 0⎤
⎢- 0 0 0⎥ ⎢0 0 0 0⎥ ⎢0 0 + 0⎥
⎢0 0 0 0⎥ ⎢- 0 0 0⎥ ⎢0 - 0 0⎥
⎣0 0 0 0⎦ ⎣0 0 0 0⎦ ⎣0 0 0 0⎦

⎡0 0 0 +⎤ ⎡0 0 0 0⎤ ⎡0 0 0 0⎤
⎢0 0 0 0⎥ ⎢0 0 0 +⎥ ⎢0 0 0 0⎥
⎢0 0 0 0⎥ ⎢0 0 0 0⎥ ⎢0 0 0 +⎥
⎣+ 0 0 0⎦ ⎣0 + 0 0⎦ ⎣0 0 + 0⎦
```

But that leaves nine directions that are more problematic. If it drifts in one of these, `transform` is no longer a rigid transformation, and things will start to break.
```
⎡0 + 0 0⎤ ⎡0 0 + 0⎤ ⎡0 0 0 0⎤
⎢+ 0 0 0⎥ ⎢0 0 0 0⎥ ⎢0 0 + 0⎥
⎢0 0 0 0⎥ ⎢+ 0 0 0⎥ ⎢0 + 0 0⎥
⎣0 0 0 0⎦ ⎣0 0 0 0⎦ ⎣0 0 0 0⎦

⎡0 0 0 +⎤ ⎡0 0 0 0⎤ ⎡0 0 0 0⎤
⎢0 0 0 0⎥ ⎢0 0 0 +⎥ ⎢0 0 0 0⎥
⎢0 0 0 0⎥ ⎢0 0 0 0⎥ ⎢0 0 0 +⎥
⎣- 0 0 0⎦ ⎣0 - 0 0⎦ ⎣0 0 - 0⎦

⎡+ 0 0 0⎤ ⎡+ 0 0 0⎤ ⎡+ 0 0 0⎤
⎢0 + 0 0⎥ ⎢0 - 0 0⎥ ⎢0 - 0 0⎥
⎢0 0 - 0⎥ ⎢0 0 + 0⎥ ⎢0 0 - 0⎥
⎣0 0 0 -⎦ ⎣0 0 0 -⎦ ⎣0 0 0 +⎦
```

Both issues can be dealt with, but you probably will run into them.

</details>

## Thanks for reading!
