// Vertex shader program
var VSHADER_SOURCE =
    'attribute vec4 a_Position;\n' +
    'attribute vec4 a_Normal;\n' +
    'attribute vec2 a_TexCoords;\n' +
    'uniform mat4 u_ModelMatrix;\n' +
    'uniform mat4 u_NormalMatrix;\n' +
    'uniform mat4 u_ViewMatrix;\n' +
    'uniform mat4 u_ProjMatrix;\n' +
    'varying vec4 v_Color;\n' +
    'varying vec3 v_Normal;\n' +
    'varying vec2 v_TexCoords;\n' +
    'varying vec3 v_Position;\n' +

    'void main() {\n' +
    '  gl_Position = u_ProjMatrix * u_ViewMatrix * u_ModelMatrix * a_Position;\n' +
    '  v_Position = vec3(u_ModelMatrix * a_Position);\n' +
    '  v_Normal = normalize(vec3(u_NormalMatrix * a_Normal));\n' +
    '  v_TexCoords = a_TexCoords;\n' +
    '}\n';

// Fragment shader program
var FSHADER_SOURCE =
    '#ifdef GL_ES\n' +
    'precision mediump float;\n' +
    '#endif\n' +
    'uniform vec3 u_LightColor;\n' +
    'uniform vec3 u_LightPosition;\n' +
    'uniform sampler2D u_Sampler;\n' +
    'uniform vec3 u_AmbientLight;\n' +
    'varying vec3 v_Normal;\n' +
    'varying vec3 v_Position;\n' +
    'varying vec4 v_Color;\n' +
    'varying vec2 v_TexCoords;\n' +
    'void main() {\n' +

    '  vec3 normal = normalize(v_Normal);\n' +
    '  vec3 lightDirection = normalize(u_LightPosition - v_Position);\n' +
    '  float nDotL = max(dot(lightDirection, normal), 0.0);\n' +

    '  vec4 TexColor = texture2D(u_Sampler, v_TexCoords);\n' +
    '  vec3 diffuse = u_LightColor * TexColor.rgb * nDotL;\n' +

    '  vec3 ambient = u_AmbientLight * TexColor.rgb;\n' +
    '  gl_FragColor = vec4(diffuse + ambient, TexColor.a);\n' +

    '}\n';


// Webgl variables
var gl;
var u_ModelMatrix;
var u_NormalMatrix;
var modelMatrix = new Matrix4();
var viewMatrix = new Matrix4();
var projMatrix = new Matrix4();
var g_normalMatrix = new Matrix4();
var chair_step = 0.4;
var g_chair_displacement = 8;
var g_bulb_rotation = 45.0;

// Moving camera variables (Adapted from https://sites.google.com/site/csc8820/educational/move-a-camera)

// The increments of rotation angle (degrees)
var ANGLE_STEP = 1.0;
var Bulb_STEP = 8.0;

// The pitch angle (degrees)
var pitch_degrees = 25;
var max_pitch = 50;
var min_pitch = 20;

// The horizontal angle (degrees)
var roll = 60;
var fov = 180;

let textures_loaded = 0;

function main() {
    // Retrieve <canvas> element
    var canvas = document.getElementById('webgl');

    // Get the rendering context for WebGL
    gl = getWebGLContext(canvas);
    if (!gl) {
        console.log('Failed to get the rendering context for WebGL');
        return;
    }

    // Initialize shaders
    if (!initShaders(gl, VSHADER_SOURCE, FSHADER_SOURCE)) {
        console.log('Failed to intialize shaders.');
        return;
    }

    // Set background colour to a light grey and Enable Depth Test (hidden surface removal).
    gl.clearColor(0.82, 0.82, 0.82, 1.0);
    gl.enable(gl.DEPTH_TEST);

    // Clear color and depth buffer
    gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);

    // Get the storage locations of uniform attributes
    u_ModelMatrix = gl.getUniformLocation(gl.program, 'u_ModelMatrix');
    var u_ViewMatrix = gl.getUniformLocation(gl.program, 'u_ViewMatrix');
    u_NormalMatrix = gl.getUniformLocation(gl.program, 'u_NormalMatrix');
    var u_ProjMatrix = gl.getUniformLocation(gl.program, 'u_ProjMatrix');
    var u_LightColor = gl.getUniformLocation(gl.program, 'u_LightColor');
    var u_LightPosition = gl.getUniformLocation(gl.program, 'u_LightPosition');
    var u_AmbientLight = gl.getUniformLocation(gl.program, 'u_AmbientLight');
    var u_Sampler = gl.getUniformLocation(gl.program, 'u_Sampler');

    if (!u_ModelMatrix || !u_ViewMatrix || !u_NormalMatrix ||
        !u_ProjMatrix || !u_AmbientLight || !u_LightPosition
        || !u_Sampler) {
        console.log('Failed to Get the storage locations of u_ModelMatrix, u_ViewMatrix, and/or u_ProjMatrix');
        //return;
    }

    document.onkeydown = function (ev) {
        keydown(ev, viewMatrix, u_ViewMatrix);
    };

    // Calculate the view matrix and the projection matrix
    projMatrix.setPerspective(30, canvas.width / canvas.height, 1, 1000);
    // Pass the model, view, and projection matrix to the uniform variable respectively

    gl.uniformMatrix4fv(u_ProjMatrix, false, projMatrix.elements);

    texture_floor = gl.createTexture();
    texture_floor.image = new Image();
    texture_floor.image.onload = function () { loadTexture(gl, texture_floor) };
    texture_floor.image.src = './img/Floor.jpg';

    texture_wall = gl.createTexture();
    texture_wall.image = new Image();
    texture_wall.image.onload = function () { loadTexture(gl, texture_wall) };
    texture_wall.image.src = './img/Walls.jpg';

    texture_wood = gl.createTexture();
    texture_wood.image = new Image();
    texture_wood.image.onload = function () { loadTexture(gl, texture_wood) };
    texture_wood.image.src = './img/Wood.jpg';

    texture_sofa = gl.createTexture();
    texture_sofa.image = new Image();
    texture_sofa.image.onload = function () { loadTexture(gl, texture_sofa) };
    texture_sofa.image.src = './img/Sofa.jpg';

    texture_tv = gl.createTexture();
    texture_tv.image = new Image();
    texture_tv.image.onload = function () { loadTexture(gl, texture_tv) };
    texture_tv.image.src = './img/TV.jpg';

    texture_light = gl.createTexture();
    texture_light.image = new Image();
    texture_light.image.onload = function () { loadTexture(gl, texture_light) };
    texture_light.image.src = './img/Light.jpg';

    //Setting up lighting parameters
    gl.uniform3f(u_AmbientLight, 0.2, 0.2, 0.2);
    gl.uniform3f(u_LightColor, 1.0, 1.0, 0.9);
    gl.uniform3f(u_LightPosition, -20, 30, -10);

    // Set Camera Position.
    setCamera(viewMatrix, u_ViewMatrix);
}

function isPowerOf2(value) {
    return (value & (value - 1)) === 0;
}

function loadTexture(gl, texture) {
    // Bind the texture object to the target
    gl.bindTexture(gl.TEXTURE_2D, texture);
    // Sets the texture image
    gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGB, gl.RGB, gl.UNSIGNED_BYTE, texture.image);

    //Handle powers of 2
    if (isPowerOf2(texture.image.width) && isPowerOf2(texture.image.height)) {
        //If PO2 generate mipmap and repeat as generic texture
        gl.generateMipmap(gl.TEXTURE_2D)
        gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.REPEAT);
        gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.REPEAT);
    } else {
        //If not clamp to edge as is wanted for a single object e.g. window/door
        // Flip the image's y axis
        gl.pixelStorei(gl.UNPACK_FLIP_Y_WEBGL, 1);
        gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
        gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);
        gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR);
    }
    textures_loaded += 1;

    if (textures_loaded === 6) {
        draw(gl, u_ModelMatrix, u_NormalMatrix);
    }
}

function keydown(ev, viewMatrix, u_ViewMatrix) {

    switch (ev.keyCode) {
        case 40: // Down arrow key
            pitch_degrees = Math.max(min_pitch, (pitch_degrees - ANGLE_STEP) % 360);
            setCamera(viewMatrix, u_ViewMatrix);
            break;
        case 38: // Up arrow key
            pitch_degrees = Math.min(max_pitch, (pitch_degrees + ANGLE_STEP) % 360);
            setCamera(viewMatrix, u_ViewMatrix);
            break;
        case 39: // Right arrow key
            roll = (roll - ANGLE_STEP) % 360;
            setCamera(viewMatrix, u_ViewMatrix)
            break;
        case 37: // Left arrow key
            roll = (roll + ANGLE_STEP) % 360;
            setCamera(viewMatrix, u_ViewMatrix)
            break;
        case 90: // 'ｚ'key -> Moves Chairs In
            if (g_chair_displacement > 3.5)(g_chair_displacement -= chair_step);
            break;
        case 88: // 'x'key -> Moves Chairs Out
            if (g_chair_displacement < 7.5)(g_chair_displacement += chair_step);
            break;
        case 86: // 'v'key -> Rotates Lamp Clockwise
            g_bulb_rotation = (g_bulb_rotation + Bulb_STEP) % 360;
            break;
        case 67: // 'c'key -> Rotates Lamp Anti-Clockwise
            g_bulb_rotation = (g_bulb_rotation - Bulb_STEP) % 360;
            break;
        default:
            return;
    }

    draw(gl, u_ModelMatrix, u_NormalMatrix);

}

function radians(deg) {
    return deg * (Math.PI / 180);
}

// Camera Position (https://community.khronos.org/t/moving-the-camera-using-spherical-coordinates/49549)
function setCamera(viewMatrix, u_ViewMatrix) {

    // Works out x, y & z of camera.
    var x = fov * Math.cos(radians(roll)) * Math.cos(radians(pitch_degrees));
    var y = fov * Math.sin(radians(pitch_degrees));
    var z = fov * Math.cos(radians(pitch_degrees)) * Math.sin(radians(roll));

    // Sets initial camera in scene.
    viewMatrix.setLookAt(x, y, z, 0, 0, 0, 0, 1, 0);
    gl.uniformMatrix4fv(u_ViewMatrix, false, viewMatrix.elements);
}

function initCubeVertexBuffers(gl) {

    // 1x1x1 Cube Coordinates
    var vertices = new Float32Array([
        1, 1, 1, -1, 1, 1, -1, -1, 1, 1, -1, 1,
        1, 1, 1, 1, -1, 1, 1, -1, -1, 1, 1, -1,
        1, 1, 1, 1, 1, -1, -1, 1, -1, -1, 1, 1,
        -1, 1, 1, -1, 1, -1, -1, -1, -1, -1, -1, 1,
        -1, -1, -1, 1, -1, -1, 1, -1, 1, -1, -1, 1,
        1, -1, -1, -1, -1, -1, -1, 1, -1, 1, 1, -1
    ]);

    // Normal
    var normals = new Float32Array([
        0.0, 0.0, 1.0, 0.0, 0.0, 1.0, 0.0, 0.0, 1.0, 0.0, 0.0, 1.0,      // v0-v1-v2-v3 front
        1.0, 0.0, 0.0, 1.0, 0.0, 0.0, 1.0, 0.0, 0.0, 1.0, 0.0, 0.0,      // v0-v3-v4-v5 right
        0.0, 1.0, 0.0, 0.0, 1.0, 0.0, 0.0, 1.0, 0.0, 0.0, 1.0, 0.0,      // v0-v5-v6-v1 up
        -1.0, 0.0, 0.0, -1.0, 0.0, 0.0, -1.0, 0.0, 0.0, -1.0, 0.0, 0.0,  // v1-v6-v7-v2 left
        0.0, -1.0, 0.0, 0.0, -1.0, 0.0, 0.0, -1.0, 0.0, 0.0, -1.0, 0.0,  // v7-v4-v3-v2 down
        0.0, 0.0, -1.0, 0.0, 0.0, -1.0, 0.0, 0.0, -1.0, 0.0, 0.0, -1.0   // v4-v7-v6-v5 back
    ]);

    // Indices of the vertices
    var indices = new Uint8Array([
        0,  1,  2,  0,  2,  3,     // front
        4,  5,  6,  4,  6,  7,     // right
        8,  9,  10, 8,  10, 11,    // up
        12, 13, 14, 12, 14, 15,    // left
        16, 17, 18, 16, 18, 19,    // down
        20, 21, 22, 20, 22, 23     // back
    ]);

    // Write the vertex property to buffers (coordinates, colors and normals)
    if (!initArrayBuffer(gl, 'a_Position', vertices, 3, gl.FLOAT)) return -1;
    if (!initArrayBuffer(gl, 'a_Normal', normals, 3, gl.FLOAT)) return -1;

    // Write the indices to the buffer object
    let indexBuffer = gl.createBuffer();

    if (!indexBuffer) {
        console.log('Failed to create the buffer object');
        return false;
    }

    gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, indexBuffer);
    gl.bufferData(gl.ELEMENT_ARRAY_BUFFER, indices, gl.STATIC_DRAW);

    return indices.length;
}

function initArrayBuffer(gl, attribute, data, num) {
    // Create a buffer object
    var buffer = gl.createBuffer();
    if (!buffer) {
        console.log('Failed to create the buffer object.');
        return false;
    }

    // Write date into the buffer object
    gl.bindBuffer(gl.ARRAY_BUFFER, buffer);
    gl.bufferData(gl.ARRAY_BUFFER, data, gl.STATIC_DRAW);

    // Element size
    var FSIZE = data.BYTES_PER_ELEMENT;

    // Assign the buffer object to the attribute variable

    var a_attribute = gl.getAttribLocation(gl.program, attribute);
    if (a_attribute < 0) {
        console.log('Failed to get the storage location of ' + attribute);
        return false;
    }
    gl.vertexAttribPointer(a_attribute, num, gl.FLOAT, false, FSIZE * num, 0);
    // Enable the assignment of the buffer object to the attribute variable
    gl.enableVertexAttribArray(a_attribute);
    gl.bindBuffer(gl.ARRAY_BUFFER, null);

    return true;
}

// Array for storing matrix stack
var g_matrixStack = [];

// Stores the specified matrix to the array
function pushMatrix(m) {
    g_matrixStack.push(new Matrix4(m));
}

// Retrieves matrix from the array
function popMatrix() {
    return g_matrixStack.pop();
}

// Helper function for drawing all objects
function drawObject(gl, u_ModelMatrix, u_NormalMatrix, n) {
    pushMatrix(modelMatrix);
    // Pass the model matrix to the uniform variable
    gl.uniformMatrix4fv(u_ModelMatrix, false, modelMatrix.elements);

    // Calculate the normal transformation matrix and pass it to u_NormalMatrix
    g_normalMatrix.setInverseOf(modelMatrix);
    g_normalMatrix.transpose();
    gl.uniformMatrix4fv(u_NormalMatrix, false, g_normalMatrix.elements);
    // Draw the cube
    gl.drawElements(gl.TRIANGLES, n, gl.UNSIGNED_BYTE, 0);

    modelMatrix = popMatrix();
}

// Main draw function
function draw(gl, u_ModelMatrix, u_NormalMatrix) {

    // Clear color and depth buffer
    gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);

    var n = initCubeVertexBuffers(gl);
    if (n < 0) {
        console.log('Failed to set the vertex information');
        return;
    }

    // -- Floor & Walls --
    pushMatrix(modelMatrix);
    modelMatrix.translate(0, 0, 0);
    drawFloorWalls(gl, u_ModelMatrix, u_NormalMatrix, n);
    modelMatrix = popMatrix();

    // -- Table and Chairs --
    pushMatrix(modelMatrix)
    modelMatrix.translate(-15, 0, 30);

    // Table
    pushMatrix(modelMatrix);
    modelMatrix.translate(0, 0, 0);
    drawTable(gl, u_ModelMatrix, u_NormalMatrix, n);
    modelMatrix = popMatrix();

    // 4x Chairs
    for (let opposite = -1; opposite < 2; opposite += 2) {
        pushMatrix(modelMatrix);
        modelMatrix.translate(opposite * g_chair_displacement, 0, 0);
        for (let side = -1; side < 2; side += 2) {
            pushMatrix(modelMatrix);
            modelMatrix.translate(11 * opposite, 6, 10 * side);
            if (opposite === 1) modelMatrix.rotate(180, 0, 1, 0);
            drawChair(gl, u_ModelMatrix, u_NormalMatrix, n);
            modelMatrix = popMatrix();

        }
        modelMatrix = popMatrix();
    }

    modelMatrix = popMatrix();

    // -- Standing Lamp --
    pushMatrix(modelMatrix);
    modelMatrix.translate(-30, 0, -30);
    drawStandingLamp(gl, u_ModelMatrix, u_NormalMatrix, n);
    modelMatrix = popMatrix();

    // -- TV Corner --
    pushMatrix(modelMatrix);

    // TV
    pushMatrix(modelMatrix);
    modelMatrix.translate(35, 0, -30);
    modelMatrix.rotate(-45, 0, 1, 0);
    drawTV(gl, u_ModelMatrix, u_NormalMatrix, n);
    modelMatrix = popMatrix();

    // Sofa
    pushMatrix(modelMatrix);
    modelMatrix.translate(0, 3, -23);
    modelMatrix.rotate(-90, 0, 1, 0);
    drawSofa(gl, u_ModelMatrix, u_NormalMatrix, n);
    modelMatrix = popMatrix();

    // Sofa Chair 1
    pushMatrix(modelMatrix);
    modelMatrix.translate(27.5, 3, 5);
    drawSofaChair(gl, u_ModelMatrix, u_NormalMatrix, n);
    modelMatrix = popMatrix();

    modelMatrix = popMatrix();

}

// Draws Floor & Walls
function drawFloorWalls(gl, u_ModelMatrix, u_NormalMatrix, n) {

    // Adding Floor Texture to Texture Buffer.
    var texCoords = new Float32Array([
        1.0, 1.0, 0.0, 1.0, 0.0, 0.0, 1.0, 0.0,  // v0-v1-v2-v3 front
        0.0, 1.0, 0.0, 0.0, 1.0, 0.0, 1.0, 1.0,  // v0-v3-v4-v5 right
        4.0, 0.0, 4.0, 4.0, 0.0, 4.0, 0.0, 0.0,  // v0-v5-v6-v1 up
        1.0, 1.0, 0.0, 1.0, 0.0, 0.0, 1.0, 0.0,  // v1-v6-v7-v2 left
        0.0, 0.0, 1.0, 0.0, 1.0, 1.0, 0.0, 1.0,  // v7-v4-v3-v2 down
        0.0, 0.0, 1.0, 0.0, 1.0, 1.0, 0.0, 1.0   // v4-v7-v6-v5 back
    ]);
    if (!initArrayBuffer(gl, 'a_TexCoords', texCoords, 2)) return -1;
    gl.bindTexture(gl.TEXTURE_2D, texture_floor);

    // Floor
    pushMatrix(modelMatrix);
    modelMatrix.translate(0, 0, 0);
    modelMatrix.scale(50, 0.5, 50);
    drawObject(gl, u_ModelMatrix, u_NormalMatrix, n);
    modelMatrix = popMatrix();

    // Adding Wall Texture to Texture Buffer.
    var texCoords = new Float32Array([
        4.0, 4.0, 0.0, 4.0, 0.0, 0.0, 4.0, 0.0,  // v0-v1-v2-v3 front
        0.0, 4.0, 0.0, 0.0, 4.0, 0.0, 4.0, 4.0,  // v0-v3-v4-v5 right
        1.0, 0.0, 1.0, 1.0, 0.0, 1.0, 0.0, 0.0,  // v0-v5-v6-v1 up
        1.0, 1.0, 0.0, 1.0, 0.0, 0.0, 1.0, 0.0,  // v1-v6-v7-v2 left
        0.0, 0.0, 1.0, 0.0, 1.0, 1.0, 0.0, 1.0,  // v7-v4-v3-v2 down
        0.0, 0.0, 1.0, 0.0, 1.0, 1.0, 0.0, 1.0   // v4-v7-v6-v5 back
    ]);
    if (!initArrayBuffer(gl, 'a_TexCoords', texCoords, 2)) return -1;
    gl.bindTexture(gl.TEXTURE_2D, texture_wall);

    // Walls
    pushMatrix(modelMatrix);

    // Wall 1
    pushMatrix(modelMatrix);
    modelMatrix.translate(-50, 19, 0);
    modelMatrix.scale(.4, 20, 50);
    drawObject(gl, u_ModelMatrix, u_NormalMatrix, n);
    modelMatrix = popMatrix();

    // Wall 2
    pushMatrix(modelMatrix);
    modelMatrix.translate(0, 19, -50);
    modelMatrix.scale(50, 20, 0.4);
    drawObject(gl, u_ModelMatrix, u_NormalMatrix, n);
    modelMatrix = popMatrix();

    modelMatrix = popMatrix();

}

// Draws Table
function drawTable(gl, u_ModelMatrix, u_NormalMatrix, n) {

    // Adding Wood Texture to Texture Buffer.
    var texCoords = new Float32Array([
        1.0, 1.0, 0.0, 1.0, 0.0, 0.0, 1.0, 0.0,  // v0-v1-v2-v3 front
        0.0, 1.0, 0.0, 0.0, 1.0, 0.0, 1.0, 1.0,  // v0-v3-v4-v5 right
        1.0, 0.0, 1.0, 1.0, 0.0, 1.0, 0.0, 0.0,  // v0-v5-v6-v1 up
        1.0, 1.0, 0.0, 1.0, 0.0, 0.0, 1.0, 0.0,  // v1-v6-v7-v2 left
        0.0, 0.0, 1.0, 0.0, 1.0, 1.0, 0.0, 1.0,  // v7-v4-v3-v2 down
        0.0, 0.0, 1.0, 0.0, 1.0, 1.0, 0.0, 1.0   // v4-v7-v6-v5 back
    ]);
    if (!initArrayBuffer(gl, 'a_TexCoords', texCoords, 2)) return -1;
    gl.bindTexture(gl.TEXTURE_2D, texture_wood);

    // Base
    pushMatrix(modelMatrix);
    modelMatrix.translate(0, 10, 0);
    modelMatrix.scale(10, 0.5, 20);
    drawObject(gl, u_ModelMatrix, u_NormalMatrix, n);
    modelMatrix = popMatrix();

    // Under-Base
    pushMatrix(modelMatrix);
    modelMatrix.translate(0, 5, 0);
    modelMatrix.scale(5, 0.5, 8);
    drawObject(gl, u_ModelMatrix, u_NormalMatrix, n);
    modelMatrix = popMatrix();

    // Legs
    for (let x = -1; x <= 1; x += 2) {
            pushMatrix(modelMatrix);
            modelMatrix.translate(0, 5, 7.5 * x);
            modelMatrix.scale(5, 5, 1);
            drawObject(gl, u_ModelMatrix, u_NormalMatrix, n);
            modelMatrix = popMatrix();
    }

}

// Draws Chair
function drawChair(gl, u_ModelMatrix, u_NormalMatrix, n) {

    // Chair base.
    pushMatrix(modelMatrix);
    modelMatrix.scale(5, 0.5, 5);
    drawObject(gl, u_ModelMatrix, u_NormalMatrix, n);
    modelMatrix = popMatrix();

    // Chair Legs.
    for (let opposite = -1; opposite < 2; opposite += 2) {
        for (let sides = -1; sides < 2; sides += 2) {
            pushMatrix(modelMatrix);
            modelMatrix.translate(3.5 * opposite, -4.5, 3.5 * sides);
            modelMatrix.scale(0.6, 4.5, 0.6);
            drawObject(gl, u_ModelMatrix, u_NormalMatrix, n);
            modelMatrix = popMatrix();
        }
    }

    // Chair Back.
    pushMatrix(modelMatrix);
    modelMatrix.translate(-5, 7, 0);
    modelMatrix.scale(0.6, 7, 5);
    drawObject(gl, u_ModelMatrix, u_NormalMatrix, n);
    modelMatrix = popMatrix();
}

// Draws Standing Lamp
function drawStandingLamp(gl, u_ModelMatrix, u_NormalMatrix, n) {

    // Base
    pushMatrix(modelMatrix);
    modelMatrix.scale(5, 2, 5);
    drawObject(gl, u_ModelMatrix, u_NormalMatrix, n);
    modelMatrix = popMatrix();

    // Column
    pushMatrix(modelMatrix);
    modelMatrix.translate(0, 15, 0);
    modelMatrix.scale(2, 15, 2);
    drawObject(gl, u_ModelMatrix, u_NormalMatrix, n);
    modelMatrix = popMatrix();

    // Adding Light Texture to Texture Buffer.
    var texCoords = new Float32Array([
        1.0, 1.0, 0.0, 1.0, 0.0, 0.0, 1.0, 0.0,  // v0-v1-v2-v3 front
        0.0, 1.0, 0.0, 0.0, 1.0, 0.0, 1.0, 1.0,  // v0-v3-v4-v5 right
        1.0, 0.0, 1.0, 1.0, 0.0, 1.0, 0.0, 0.0,  // v0-v5-v6-v1 up
        1.0, 1.0, 0.0, 1.0, 0.0, 0.0, 1.0, 0.0,  // v1-v6-v7-v2 left
        0.0, 0.0, 1.0, 0.0, 1.0, 1.0, 0.0, 1.0,  // v7-v4-v3-v2 down
        0.0, 0.0, 1.0, 0.0, 1.0, 1.0, 0.0, 1.0   // v4-v7-v6-v5 back
    ]);
    if (!initArrayBuffer(gl, 'a_TexCoords', texCoords, 2)) return -1;
    gl.bindTexture(gl.TEXTURE_2D, texture_light);

    // Bulb
    pushMatrix(modelMatrix);
    modelMatrix.translate(0, 30, 0);
    modelMatrix.scale(3, 6, 3);
    modelMatrix.rotate(g_bulb_rotation, 0.0, 1.0, 0);
    drawObject(gl, u_ModelMatrix, u_NormalMatrix, n);
    modelMatrix = popMatrix();

}

// Draws TV
function drawTV(gl, u_ModelMatrix, u_NormalMatrix, n) {

    // Adding Wood Texture to Texture Buffer.
    var texCoords = new Float32Array([
        1.0, 1.0, 0.0, 1.0, 0.0, 0.0, 1.0, 0.0,  // v0-v1-v2-v3 front
        0.0, 1.0, 0.0, 0.0, 1.0, 0.0, 1.0, 1.0,  // v0-v3-v4-v5 right
        1.0, 0.0, 1.0, 1.0, 0.0, 1.0, 0.0, 0.0,  // v0-v5-v6-v1 up
        1.0, 1.0, 0.0, 1.0, 0.0, 0.0, 1.0, 0.0,  // v1-v6-v7-v2 left
        0.0, 0.0, 1.0, 0.0, 1.0, 1.0, 0.0, 1.0,  // v7-v4-v3-v2 down
        0.0, 0.0, 1.0, 0.0, 1.0, 1.0, 0.0, 1.0   // v4-v7-v6-v5 back
    ]);
    if (!initArrayBuffer(gl, 'a_TexCoords', texCoords, 2)) return -1;
    gl.bindTexture(gl.TEXTURE_2D, texture_wood);

    // Drawer
    pushMatrix(modelMatrix);
    modelMatrix.translate(0, 4, 0);
    modelMatrix.scale(15, 4, 4);
    drawObject(gl, u_ModelMatrix, u_NormalMatrix, n);
    modelMatrix = popMatrix();

    // Adding Tv Texture to Texture Buffer.
    var texCoords = new Float32Array([
        1.0, 1.0, 0.0, 1.0, 0.0, 0.0, 1.0, 0.0,  // v0-v1-v2-v3 front
        0.0, 1.0, 0.0, 0.0, 1.0, 0.0, 1.0, 1.0,  // v0-v3-v4-v5 right
        1.0, 0.0, 1.0, 1.0, 0.0, 1.0, 0.0, 0.0,  // v0-v5-v6-v1 up
        1.0, 1.0, 0.0, 1.0, 0.0, 0.0, 1.0, 0.0,  // v1-v6-v7-v2 left
        0.0, 0.0, 1.0, 0.0, 1.0, 1.0, 0.0, 1.0,  // v7-v4-v3-v2 down
        0.0, 0.0, 1.0, 0.0, 1.0, 1.0, 0.0, 1.0   // v4-v7-v6-v5 back
    ]);
    if (!initArrayBuffer(gl, 'a_TexCoords', texCoords, 2)) return -1;
    gl.bindTexture(gl.TEXTURE_2D, texture_tv);

    // Mount
    pushMatrix(modelMatrix);
    modelMatrix.translate(0, 8, 0);
    modelMatrix.scale(6, 1, 2);
    drawObject(gl, u_ModelMatrix, u_NormalMatrix, n);
    modelMatrix = popMatrix();

    // TV
    pushMatrix(modelMatrix);
    modelMatrix.translate(0, 18, 0);
    modelMatrix.scale(18, 9, 0.25);
    drawObject(gl, u_ModelMatrix, u_NormalMatrix, n);
    modelMatrix = popMatrix();

}

// Draws Sofa
function drawSofa(gl, u_ModelMatrix, u_NormalMatrix, n) {

    // Adding Sofa Texture to Texture Buffer.
    var texCoords = new Float32Array([
        1.0, 1.0, 0.0, 1.0, 0.0, 0.0, 1.0, 0.0,  // v0-v1-v2-v3 front
        0.0, 1.0, 0.0, 0.0, 1.0, 0.0, 1.0, 1.0,  // v0-v3-v4-v5 right
        1.0, 0.0, 1.0, 1.0, 0.0, 1.0, 0.0, 0.0,  // v0-v5-v6-v1 up
        1.0, 1.0, 0.0, 1.0, 0.0, 0.0, 1.0, 0.0,  // v1-v6-v7-v2 left
        0.0, 0.0, 1.0, 0.0, 1.0, 1.0, 0.0, 1.0,  // v7-v4-v3-v2 down
        0.0, 0.0, 1.0, 0.0, 1.0, 1.0, 0.0, 1.0   // v4-v7-v6-v5 back
    ]);
    if (!initArrayBuffer(gl, 'a_TexCoords', texCoords, 2)) return -1;
    gl.bindTexture(gl.TEXTURE_2D, texture_sofa);

    // Bottom
    pushMatrix(modelMatrix);
    modelMatrix.translate(0, 2.5, 0);
    modelMatrix.scale(15, 4, 7);
    drawObject(gl, u_ModelMatrix, u_NormalMatrix, n);
    modelMatrix = popMatrix();

    // Back
    pushMatrix(modelMatrix);
    modelMatrix.translate(0, 5, 9);
    modelMatrix.scale(20, 8, 2);
    drawObject(gl, u_ModelMatrix, u_NormalMatrix, n);
    modelMatrix = popMatrix();

    // Right Armrest
    pushMatrix(modelMatrix);
    modelMatrix.translate(17.5, 5, 0);
    modelMatrix.scale(2.5, 8, 7);
    drawObject(gl, u_ModelMatrix, u_NormalMatrix, n);
    modelMatrix = popMatrix();

    // Left Armrest
    pushMatrix(modelMatrix);
    modelMatrix.translate(-17.5, 5, 0);
    modelMatrix.scale(2.5, 8, 7);
    drawObject(gl, u_ModelMatrix, u_NormalMatrix, n);
    modelMatrix = popMatrix();

}

// Draws Sofa Chair
function drawSofaChair(gl, u_ModelMatrix, u_NormalMatrix, n) {

    // Bottom
    pushMatrix(modelMatrix);
    modelMatrix.translate(0, 2.5, 0);
    modelMatrix.scale(7.5, 4, 7);
    drawObject(gl, u_ModelMatrix, u_NormalMatrix, n);
    modelMatrix = popMatrix();

    // Back
    pushMatrix(modelMatrix);
    modelMatrix.translate(0, 5, 9);
    modelMatrix.scale(9, 8, 2);
    drawObject(gl, u_ModelMatrix, u_NormalMatrix, n);
    modelMatrix = popMatrix();

    // Right Armrest
    pushMatrix(modelMatrix);
    modelMatrix.translate(7.5, 5, 0);
    modelMatrix.scale(1.5, 8, 7);
    drawObject(gl, u_ModelMatrix, u_NormalMatrix, n);
    modelMatrix = popMatrix();

    // Left Armrest
    pushMatrix(modelMatrix);
    modelMatrix.translate(-7.5, 5, 0);
    modelMatrix.scale(1.5, 8, 7);
    drawObject(gl, u_ModelMatrix, u_NormalMatrix, n);
    modelMatrix = popMatrix();

}
