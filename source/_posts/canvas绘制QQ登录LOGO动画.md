title: canvas绘制QQ登录LOGO动画
date: 2015-11-16 22:37:51
tags: [canvas]
---

基于[flat-surface-shader](https://github.com/wagerfield/flat-surface-shader) 库绘制动态的低多边形效果,仿QQ登录界面动画

<!-- more -->


```html
    <div id="container" class="container dragable">
        <div id="output" class="container">
        </div>
    </div>
```

```css
    .container {
        position: absolute;
        height: 37.5%;
        width: 100%;
    }
```

```js
        (function () {

            //------------------------------
            // Mesh Properties
            //------------------------------
            var MESH = {
                width: 2,
                height: 4,
                depth: 30,
                segments: 15,
                slices: 8,
                xRange: 1,
                yRange: 0.1,
                zRange: 1.0,
                ambient: '#ababab',
                diffuse: '#ffffff',
                speed: 0.0002
            };

            //------------------------------
            // Light Properties
            //------------------------------
            var LIGHT = {
                count: 1,
                xyScalar: 1,
                zOffset: 200,
                ambient: '#0080c1',
                diffuse: '#0069ad',
                speed: 0.000001,
                gravity: 400,
                dampening: 0.8,
                minLimit: 10,
                maxLimit: 30,
                minDistance: 100,
                maxDistance: 600,
                autopilot: true,
                draw: false,
                bounds: FSS.Vector3.create(),
                step: FSS.Vector3.create(
                        Math.randomInRange(0.2, 1.0),
                        Math.randomInRange(0.2, 1.0),
                        Math.randomInRange(0.2, 1.0)
                )
            };

            //------------------------------
            // Render Properties
            //------------------------------

            var RENDER = {
                renderer: 'canvas'
            };

            //------------------------------
            // Global Properties
            //------------------------------
            var now, start = Date.now();
            var center = FSS.Vector3.create();
            var attractor = FSS.Vector3.create();
            var container = document.getElementById('container');
            var output = document.getElementById('output');
            var renderer, scene, mesh, geometry, material;
            var canvasRenderer;
            var gui, autopilotController;

            //------------------------------
            // Methods
            //------------------------------
            function initialise() {
                createRenderer();
                createScene();
                createMesh();
                createLights();
                addEventListeners();
                resize(container.offsetWidth, container.offsetHeight);
                animate();
            }

            function createRenderer() {
                canvasRenderer = new FSS.CanvasRenderer();
                setRenderer(RENDER.renderer);
            }

            function setRenderer(index) {
                if (renderer) {
                    output.removeChild(renderer.element);
                }

                renderer = canvasRenderer;

                renderer.setSize(container.offsetWidth, container.offsetHeight);
                output.appendChild(renderer.element);
            }

            function createScene() {
                scene = new FSS.Scene();
            }

            function createMesh() {
                scene.remove(mesh);
                renderer.clear();
                geometry = new FSS.Plane(MESH.width * renderer.width, MESH.height * renderer.height, MESH.segments, MESH.slices);
                material = new FSS.Material(MESH.ambient, MESH.diffuse);
                mesh = new FSS.Mesh(geometry, material);
                scene.add(mesh);

                // Augment vertices for animation
                var v, vertex;
                for (v = geometry.vertices.length - 1; v >= 0; v--) {
                    vertex = geometry.vertices[v];
                    vertex.anchor = FSS.Vector3.clone(vertex.position);
                    vertex.step = FSS.Vector3.create(
                            Math.randomInRange(0.2, 1.0),
                            Math.randomInRange(0.2, 1.0),
                            Math.randomInRange(0.2, 1.0)
                    );
                    vertex.time = Math.randomInRange(0, Math.PIM2);
                }
            }

            function createLights() {
                var l, light;
                for (l = scene.lights.length - 1; l >= 0; l--) {
                    light = scene.lights[l];
                    scene.remove(light);
                }
                renderer.clear();
                for (l = 0; l < LIGHT.count; l++) {
                    light = new FSS.Light(LIGHT.ambient, LIGHT.diffuse);
                    light.ambientHex = light.ambient.format();
                    light.diffuseHex = light.diffuse.format();
                    scene.add(light);

                    // Augment light for animation
                    light.mass = Math.randomInRange(0.5, 1);
                    light.velocity = FSS.Vector3.create();
                    light.acceleration = FSS.Vector3.create();
                    light.force = FSS.Vector3.create();

                    // Ring SVG Circle
                    light.ring = document.createElementNS(FSS.SVGNS, 'circle');
                    light.ring.setAttributeNS(null, 'stroke', light.ambientHex);
                    light.ring.setAttributeNS(null, 'stroke-width', '0.5');
                    light.ring.setAttributeNS(null, 'fill', 'none');
                    light.ring.setAttributeNS(null, 'r', '10');

                    // Core SVG Circle
                    light.core = document.createElementNS(FSS.SVGNS, 'circle');
                    light.core.setAttributeNS(null, 'fill', light.diffuseHex);
                    light.core.setAttributeNS(null, 'r', '4');
                }
            }

            function resize(width, height) {
                renderer.setSize(width, height);
                FSS.Vector3.set(center, renderer.halfWidth, renderer.halfHeight);
                createMesh();
            }

            function animate() {
                now = Date.now() - start;
                update();
                render();
                requestAnimationFrame(animate);
            }

            function update() {
                var ox, oy, oz, l, light, v, vertex, offset = MESH.depth / 2;

                // Update Bounds
                FSS.Vector3.copy(LIGHT.bounds, center);
                FSS.Vector3.multiplyScalar(LIGHT.bounds, LIGHT.xyScalar);

                // Update Attractor
                FSS.Vector3.setZ(attractor, LIGHT.zOffset);

                // Overwrite the Attractor position
                if (LIGHT.autopilot) {
                    ox = Math.sin(LIGHT.step[0] * now * LIGHT.speed);
                    oy = Math.cos(LIGHT.step[1] * now * LIGHT.speed);
                    FSS.Vector3.set(attractor,
                            LIGHT.bounds[0] * ox,
                            LIGHT.bounds[1] * oy,
                            LIGHT.zOffset);
                }

                // Animate Lights
                for (l = scene.lights.length - 1; l >= 0; l--) {
                    light = scene.lights[l];

                    // Reset the z position of the light
                    FSS.Vector3.setZ(light.position, LIGHT.zOffset);

                    // Calculate the force Luke!
                    var D = Math.clamp(FSS.Vector3.distanceSquared(light.position, attractor), LIGHT.minDistance, LIGHT.maxDistance);
                    var F = LIGHT.gravity * light.mass / D;
                    FSS.Vector3.subtractVectors(light.force, attractor, light.position);
                    FSS.Vector3.normalise(light.force);
                    FSS.Vector3.multiplyScalar(light.force, F);

                    // Update the light position
                    FSS.Vector3.set(light.acceleration);
                    FSS.Vector3.add(light.acceleration, light.force);
                    FSS.Vector3.add(light.velocity, light.acceleration);
                    FSS.Vector3.multiplyScalar(light.velocity, LIGHT.dampening);
                    FSS.Vector3.limit(light.velocity, LIGHT.minLimit, LIGHT.maxLimit);
                    FSS.Vector3.add(light.position, light.velocity);
                }

                // Animate Vertices
                for (v = geometry.vertices.length - 1; v >= 0; v--) {
                    vertex = geometry.vertices[v];
                    ox = Math.sin(vertex.time + vertex.step[0] * now * MESH.speed);
                    oy = Math.cos(vertex.time + vertex.step[1] * now * MESH.speed);
                    oz = Math.sin(vertex.time + vertex.step[2] * now * MESH.speed);
                    FSS.Vector3.set(vertex.position,
                            MESH.xRange * geometry.segmentWidth * ox,
                            MESH.yRange * geometry.sliceHeight * oy,
                            MESH.zRange * offset * oz - offset);
                    FSS.Vector3.add(vertex.position, vertex.anchor);
                }

                // Set the Geometry to dirty
                geometry.dirty = true;
            }

            function render() {
                renderer.render(scene);

                // Draw Lights
                if (LIGHT.draw) {
                    var l, lx, ly, light;
                    for (l = scene.lights.length - 1; l >= 0; l--) {
                        light = scene.lights[l];
                        lx = light.position[0];
                        ly = light.position[1];
                        renderer.context.lineWidth = 0.5;
                        renderer.context.beginPath();
                        renderer.context.arc(lx, ly, 10, 0, Math.PIM2);
                        renderer.context.strokeStyle = light.ambientHex;
                        renderer.context.stroke();
                        renderer.context.beginPath();
                        renderer.context.arc(lx, ly, 4, 0, Math.PIM2);
                        renderer.context.fillStyle = light.diffuseHex;
                        renderer.context.fill();

                    }
                }
            }

            function addEventListeners() {
                window.addEventListener('resize', onWindowResize);
                //container.addEventListener('mousemove', onMouseMove);
            }

            //------------------------------
            // Callbacks
            //------------------------------

            function onMouseMove(event) {
                FSS.Vector3.set(attractor, event.x, renderer.height - event.y);
                FSS.Vector3.subtract(attractor, center);
            }

            function onWindowResize(event) {
                resize(container.offsetWidth, container.offsetHeight);
                render();
            }


            // Let there be light!
            initialise();

        })();
```



[demo](http://jsfiddle.net/wky31pw7/)
