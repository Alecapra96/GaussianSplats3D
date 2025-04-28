
# GaussianSplats3D - Guía de Implementación de Efectos

Este documento explica cómo implementar efectos visuales en splats 3D usando Three.js y shaders GLSL.

## Estructura de los Shaders

### Fragment Shader

El fragment shader es donde se implementan los efectos visuales. Aquí está la estructura básica:

```glsl
// Declaración de uniforms para los efectos
uniform float rainbowEffect;
uniform float time;
uniform float pulseEffect;
uniform float pulseTime;
uniform float glowEffect;
uniform float invertEffect;

// Funciones de efectos
vec3 applyRainbowEffect(vec3 color, float time) {
    float hue = time;
    vec3 rainbow = vec3(
        sin(hue * 6.28318) * 0.5 + 0.5,
        sin((hue + 0.333) * 6.28318) * 0.5 + 0.5,
        sin((hue + 0.666) * 6.28318) * 0.5 + 0.5
    );
    return mix(color, rainbow, 0.5);
}

float applyPulseEffect(float time) {
    return sin(time) * 0.5 + 0.5;
}

vec3 applyGlowEffect(vec3 color) {
    return color * 1.5;
}

vec3 applyInvertEffect(vec3 color) {
    return vec3(1.0) - color;
}

void main() {
    // Cálculo básico del splat
    float A = dot(vPosition, vPosition);
    if (A > 8.0) discard;
    vec3 color = vColor.rgb;
    float opacity = exp(-0.5 * A) * vColor.a;

    vec3 finalColor = color;
    
    // Aplicación de efectos
    if (rainbowEffect > 0.0) {
        finalColor = applyRainbowEffect(finalColor, time);
    }
    
    if (pulseEffect > 0.0) {
        float pulse = applyPulseEffect(pulseTime);
        finalColor = mix(finalColor, finalColor * pulse, 0.5);
    }
    
    if (glowEffect > 0.0) {
        finalColor = applyGlowEffect(finalColor);
    }
    
    if (invertEffect > 0.0) {
        finalColor = applyInvertEffect(finalColor);
    }
    
    gl_FragColor = vec4(finalColor, opacity);
}
```

### Vertex Shader

El vertex shader maneja la geometría y la posición de los splats:

```glsl
// Variables de entrada
attribute vec3 position;
attribute vec4 color;
attribute vec2 uv;

// Variables de salida
varying vec4 vColor;
varying vec2 vUv;
varying vec2 vPosition;

void main() {
    vColor = color;
    vUv = uv;
    vPosition = position.xy;
    
    // Transformación de posición
    gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
}
```

## Implementación en JavaScript

### 1. Configuración del Material

```javascript
static build(dynamicMode = false, enableOptionalEffects = false, antialiased = false, maxScreenSpaceSplatSize = 2048,
             splatScale = 1.0, pointCloudModeEnabled = false, maxSphericalHarmonicsDegree = 0, kernel2DSize = 0.3) {
    
    // ... configuración previa ...

    const uniforms = {
        // ... otros uniforms ...
        rainbowEffect: { value: 0 },
        time: { value: 0 },
        pulseEffect: { value: 0 },
        pulseTime: { value: 0 },
        glowEffect: { value: 0 },
        invertEffect: { value: 0 }
    };

    const material = new THREE.ShaderMaterial({
        uniforms: uniforms,
        vertexShader: vertexShaderSource,
        fragmentShader: fragmentShaderSource,
        transparent: true,
        alphaTest: 1.0,
        blending: THREE.NormalBlending,
        depthTest: true,
        depthWrite: false,
        side: THREE.DoubleSide
    });

    return material;
}
```

### 2. Manejo de Efectos

```javascript
handleEffectChange(effectName, active) {
    switch(effectName) {
        case 'rainbow':
            this.applyRainbowEffect(active);
            break;
        case 'pulse':
            this.applyPulseEffect(active);
            break;
        case 'glow':
            this.applyGlowEffect(active);
            break;
        case 'invert':
            this.applyInvertEffect(active);
            break;
    }
}

applyRainbowEffect(active) {
    if (active) {
        this.splatMesh.material.uniforms.rainbowEffect = { value: 1 };
        this.splatMesh.material.uniforms.time = { value: 0 };
        
        const updateTime = () => {
            if (this.splatMesh.material.uniforms.rainbowEffect.value === 1) {
                this.splatMesh.material.uniforms.time.value += 0.01;
                requestAnimationFrame(updateTime);
            }
        };
        updateTime();
    } else {
        this.splatMesh.material.uniforms.rainbowEffect = { value: 0 };
    }
    this.splatMesh.material.uniformsNeedUpdate = true;
}

applyPulseEffect(active) {
    if (active) {
        this.splatMesh.material.uniforms.pulseEffect = { value: 1 };
        this.splatMesh.material.uniforms.pulseTime = { value: 0 };
        
        const updatePulse = () => {
            if (this.splatMesh.material.uniforms.pulseEffect.value === 1) {
                this.splatMesh.material.uniforms.pulseTime.value += 0.05;
                requestAnimationFrame(updatePulse);
            }
        };
        updatePulse();
    } else {
        this.splatMesh.material.uniforms.pulseEffect = { value: 0 };
    }
    this.splatMesh.material.uniformsNeedUpdate = true;
}

applyGlowEffect(active) {
    this.splatMesh.material.uniforms.glowEffect = { value: active ? 1 : 0 };
    this.splatMesh.material.uniformsNeedUpdate = true;
}

applyInvertEffect(active) {
    this.splatMesh.material.uniforms.invertEffect = { value: active ? 1 : 0 };
    this.splatMesh.material.uniformsNeedUpdate = true;
}
```

## Consejos para Implementar Nuevos Efectos

1. **Declaración de Uniforms**: 
   - Agrega nuevos uniforms en el shader y en la configuración del material
   - Usa tipos de datos apropiados (float, vec3, etc.)

2. **Funciones de Efectos**:
   - Implementa la lógica del efecto en una función separada
   - Usa operaciones matemáticas de GLSL para manipular colores
   - Considera el rendimiento al implementar efectos complejos

3. **Animación de Efectos**:
   - Usa requestAnimationFrame para efectos animados
   - Actualiza los uniforms en cada frame
   - Controla la velocidad de la animación con valores de incremento apropiados

4. **Interfaz de Usuario**:
   - Agrega controles para activar/desactivar efectos
   - Proporciona feedback visual del estado del efecto
   - Considera la usabilidad al diseñar la interfaz

## Ejemplo de Nuevo Efecto

Para implementar un nuevo efecto, por ejemplo, un efecto de "blur":

1. Agrega el uniform en el shader:
```glsl
uniform float blurEffect;
uniform float blurAmount;
```

2. Implementa la función del efecto:
```glsl
vec3 applyBlurEffect(vec3 color, float amount) {
    // Implementación del efecto de blur
    return color * (1.0 - amount) + vec3(0.5) * amount;
}
```

3. Agrega la lógica de aplicación:
```glsl
if (blurEffect > 0.0) {
    finalColor = applyBlurEffect(finalColor, blurAmount);
}
```

4. Implementa el control en JavaScript:
```javascript
applyBlurEffect(active) {
    this.splatMesh.material.uniforms.blurEffect = { value: active ? 1 : 0 };
    this.splatMesh.material.uniforms.blurAmount = { value: 0.5 };
    this.splatMesh.material.uniformsNeedUpdate = true;
}
```

## Consideraciones de Rendimiento

1. Minimiza el número de operaciones en el fragment shader
2. Usa tipos de datos de precisión apropiada
3. Evita bucles y operaciones complejas en tiempo real
4. Considera el impacto en el rendimiento al combinar múltiples efectos

## Recursos Adicionales

- [Documentación de GLSL](https://www.khronos.org/opengl/wiki/OpenGL_Shading_Language)
- [Three.js ShaderMaterial](https://threejs.org/docs/#api/en/materials/ShaderMaterial)
- [WebGL Fundamentals](https://webglfundamentals.org/)
