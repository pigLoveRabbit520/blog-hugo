title: WebGL Shader
author: pigLoveRabbit
tags:
  - WebGL
categories:
  - OpenGL
date: 2023-07-02 16:00:00
---
##### 顶点着色器
vertex.glsl
```
varying vec3 vPosition;

void main() {
    vPosition = position;
    // MVP
    vec4 modelViewPosition = modelViewMatrix * vec4( position, 1.0 );
    vec4 projectedPosition = projectionMatrix * modelViewPosition;
    gl_Position = projectedPosition;
}
```
##### 片段着色器
fragment.glsl
```
varying vec3 vPosition;

void main() {
    gl_FragColor = vec4(vPosition, 1);
}
```
#### 效果
用了PlaneGeometry
```
// meshes
const geometry = new THREE.PlaneGeometry(2, 2)
const material = new THREE.ShaderMaterial({
  vertexShader: vertexShader,
  fragmentShader: fragmentShader,
})
```

![upload successful](\\images\shader-1.png)