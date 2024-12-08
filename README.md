# WebGL2 Spinning and Moving Textured Cube

This project demonstrates a 3D textured cube that spins and moves around the screen using WebGL2. The cube can be textured with an image uploaded by the user.

## Features
- 3D textured cube
- Spinning and moving animation
- User-uploaded texture

## Prerequisites
- A modern web browser that supports WebGL2
- Internet connection to load the `gl-matrix` library

## Getting Started

### 1. Clone the Repository
Clone this repository to your local machine using:
```sh
git clone <repository-url>
```

### 2. Open the Project
Open the 

index.html

 file in your web browser.

### 3. Upload a Texture
Use the file input to upload an image that will be used as the texture for the cube.

## Code Overview

### HTML Structure
The HTML file includes a canvas element for rendering the WebGL content and an input element for uploading images.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>WebGL2 Textured Cube</title>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/gl-matrix/2.8.1/gl-matrix-min.js"></script>
</head>
<body>
  <input type="file" id="imageUpload" accept="image/*" />
  <canvas id="glCanvas" width="800" height="600"></canvas>
  <script>
    // JavaScript code goes here
  </script>
</body>
</html>
```

### Vertex Shader
The vertex shader is responsible for transforming the vertex positions, normals, and texture coordinates.

```glsl
#version 300 es
in vec4 aPosition;
in vec3 aNormal;
in vec2 aTexCoord;

out vec3 vNormal;
out vec3 vFragPos;
out vec2 vTexCoord;

uniform mat4 uModelViewMatrix;
uniform mat4 uProjectionMatrix;
uniform mat4 uModelMatrix;

void main() {
    gl_Position = uProjectionMatrix * uModelViewMatrix * aPosition;
    vFragPos = vec3(uModelMatrix * aPosition);
    vNormal = mat3(transpose(inverse(uModelMatrix))) * aNormal;
    vTexCoord = aTexCoord;
}
```

### Fragment Shader
The fragment shader handles the lighting and texture mapping.

```glsl
#version 300 es
precision highp float;

in vec3 vNormal;
in vec3 vFragPos;
in vec2 vTexCoord;

out vec4 fragColor;

uniform vec3 uLightPos;
uniform sampler2D uTexture;

void main() {
    vec3 lightColor = vec3(1.0, 1.0, 1.0);
    vec3 objectColor = texture(uTexture, vTexCoord).rgb;

    float ambientStrength = 0.1;
    vec3 ambient = ambientStrength * lightColor;

    vec3 norm = normalize(vNormal);
    vec3 lightDir = normalize(uLightPos - vFragPos);
    float diff = max(dot(norm, lightDir), 0.0);
    vec3 diffuse = diff * lightColor;

    vec3 lighting = (ambient + diffuse) * objectColor;
    fragColor = vec4(lighting, 1.0);
}
```

### JavaScript
The JavaScript code initializes WebGL2, sets up the shaders, buffers, and handles the rendering loop.

#### Initialization
```javascript
function initGL() {
  const canvas = document.getElementById('glCanvas');
  gl = canvas.getContext('webgl2');

  if (!gl) {
    alert('Unable to initialize WebGL2. Your browser may not support it.');
    return;
  }

  const shaderProgram = initShaderProgram(gl, vsSource, fsSource);
  programInfo = {
    program: shaderProgram,
    attribLocations: {
      vertexPosition: gl.getAttribLocation(shaderProgram, 'aPosition'),
      vertexNormal: gl.getAttribLocation(shaderProgram, 'aNormal'),
      vertexTexCoord: gl.getAttribLocation(shaderProgram, 'aTexCoord'),
    },
    uniformLocations: {
      projectionMatrix: gl.getUniformLocation(shaderProgram, 'uProjectionMatrix'),
      modelViewMatrix: gl.getUniformLocation(shaderProgram, 'uModelViewMatrix'),
      modelMatrix: gl.getUniformLocation(shaderProgram, 'uModelMatrix'),
      lightPos: gl.getUniformLocation(shaderProgram, 'uLightPos'),
      texture: gl.getUniformLocation(shaderProgram, 'uTexture'),
    },
  };

  buffers = initBuffers(gl);
  texture = initTexture(gl);

  document.getElementById('imageUpload').addEventListener('change', (event) => {
    const file = event.target.files[0];
    if (file) {
      const image = new Image();
      image.onload = () => {
        updateTexture(gl, texture, image);
      };
      image.src = URL.createObjectURL(file);
    }
  });

  gl.enable(gl.DEPTH_TEST);
  gl.depthFunc(gl.LEQUAL);

  render();
}
```

#### Rendering
The `render` function updates the model matrix to spin and move the cube, and then draws it.

```javascript
function render() {
  gl.clearColor(0.0, 0.0, 0.0, 1.0);
  gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);

  const fieldOfView = 45 * Math.PI / 180;
  const aspect = gl.canvas.clientWidth / gl.canvas.clientHeight;
  const zNear = 0.1;
  const zFar = 100.0;
  const projectionMatrix = mat4.create();
  mat4.perspective(projectionMatrix, fieldOfView, aspect, zNear, zFar);

  const viewMatrix = mat4.create();
  const eye = [0, 2, 5];
  const center = [0, 0, 0];
  const up = [0, 1, 0];
  mat4.lookAt(viewMatrix, eye, center, up);

  const modelMatrix = mat4.create();
  const now = performance.now() / 1000;
  mat4.translate(modelMatrix, modelMatrix, [Math.sin(now) * 2, Math.cos(now) * 2, 0]);
  mat4.rotate(modelMatrix, modelMatrix, now, [0, 1, 0]);
  mat4.rotate(modelMatrix, modelMatrix, now * 0.7, [1, 0, 0]);

  const modelViewMatrix = mat4.create();
  mat4.multiply(modelViewMatrix, viewMatrix, modelMatrix);

  gl.useProgram(programInfo.program);
  gl.uniformMatrix4fv(programInfo.uniformLocations.projectionMatrix, false, projectionMatrix);
  gl.uniformMatrix4fv(programInfo.uniformLocations.modelViewMatrix, false, modelViewMatrix);
  gl.uniformMatrix4fv(programInfo.uniformLocations.modelMatrix, false, modelMatrix);
  gl.uniform3fv(programInfo.uniformLocations.lightPos, [5.0, 5.0, 5.0]);

  gl.bindBuffer(gl.ARRAY_BUFFER, buffers.position);
  gl.vertexAttribPointer(programInfo.attribLocations.vertexPosition, 3, gl.FLOAT, false, 0, 0);
  gl.enableVertexAttribArray(programInfo.attribLocations.vertexPosition);

  gl.bindBuffer(gl.ARRAY_BUFFER, buffers.texCoord);
  gl.vertexAttribPointer(programInfo.attribLocations.vertexTexCoord, 2, gl.FLOAT, false, 0, 0);
  gl.enableVertexAttribArray(programInfo.attribLocations.vertexTexCoord);

  gl.bindBuffer(gl.ARRAY_BUFFER, buffers.normal);
  gl.vertexAttribPointer(programInfo.attribLocations.vertexNormal, 3, gl.FLOAT, false, 0, 0);
  gl.enableVertexAttribArray(programInfo.attribLocations.vertexNormal);

  gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, buffers.indices);
  gl.activeTexture(gl.TEXTURE0);
  gl.bindTexture(gl.TEXTURE_2D, texture);
  gl.uniform1i(programInfo.uniformLocations.texture, 0);

  gl.drawElements(gl.TRIANGLES, 36, gl.UNSIGNED_SHORT, 0);

  requestAnimationFrame(render);
}
```

## Additional Information
- **Vertex Shader**: Transforms vertex positions, normals, and texture coordinates.
- **Fragment Shader**: Handles lighting and texture mapping.
- **Buffers**: Define positions, texture coordinates, normals, and indices for the cube.
- **Texture Handling**: Initializes a texture and updates it when an image is uploaded.
- **Rendering**: The `render` function applies rotation and translation transformations to the model matrix based on the current time, making the cube spin and move in a circular path.

#### Prompting
Help me to create a moving and spinning cube that applies graphical computer theory that can receive an image using webgl