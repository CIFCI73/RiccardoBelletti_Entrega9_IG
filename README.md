# RiccardoBelletti_Entrega9_IG

[Codigo de sandbox]([https://thebookofshaders.com](https://codesandbox.io/p/sandbox/entrega-ig-s9-p94vp8))

Para esta entrega he trabajado en la integración de shaders dentro de mi sistema planetario en Three.js.
Decidí elegir la alternativa que consiste en añadir shaders personalizados y modificar funciones ya existentes para mejorar el aspecto general sin cambiar la estructura del sistema.
Mi objetivo principal era conseguir dos cosas: primero, que el Sol tuviera un aspecto más vivo, con un brillo que no se viera tan artificial como una simple esfera con textura; y segundo, crear un fondo de estrellas que se sintiera más dinámico, más cercano a un cielo real, usando puntos generados por código en vez de una imagen fija.

Cuando empecé a mejorar el Sol, la idea era hacer que no fuera solo una esfera con una textura estática, sino que tuviera un poco de movimiento y un efecto como de “pulsación” o actividad solar. Para eso modifiqué tanto el vertex shader como el fragment shader. En el vertex shader añadí una pequeña deformación basada en el tiempo, justo en esta parte:
```javascript
float pulse = sin(u_time * 2.5) * 0.1;
vec3 newPosition = position + normal * pulse;
```
Lo que hago aquí es usar un sin() para generar un valor que sube y baja con el tiempo, y este valor lo multiplico por la normal del vértice. Como las normales apuntan hacia fuera, lo que pasa es que la superficie de la esfera se expande y se contrae muy ligeramente. Es solo un 0.1, que es muy poco, pero lo suficiente para que el Sol parezca más vivo, como si respirara o latiera. Para que esto funcione tuve que declarar vNormal = normal; y también pasar el tiempo desde el JavaScript, porque si no el shader se quedaría totalmente estático.

El fragment shader es donde ocurre la parte más interesante. Allí mezclo la textura del Sol con un efecto de ruido Perlin 2D que genera patrones más orgánicos. La primera cosa que hago es cargar el color de la textura:
```javascript
vec4 textureColor = texture2D(u_texture, vUv);
```
Así puedo usar los colores reales del Sol como base. Después preparo las coordenadas multiplicando las UV para que el ruido tenga más detalle:
```javascript
vec2 st = vUv * 8.0;
```
Y aquí viene la parte clave, donde calculo el ruido:
```javascript
float n = noise(st + u_time * 1.0);
```
La función noise() que uso no es built-in, sino que la escribí arriba, y es el típico pseudo-Perlin construido combinando randoms suavizados. La parte importante es que le sumo el tiempo, así los patrones de ruido se van desplazando poco a poco, como remolinos de gas caliente sobre la superficie. Con ese valor genero un color entre un naranja más oscuro y un amarillo casi blanco, usando mix:
```javascript
vec3 noiseColor = mix(vec3(1.0, 0.5, 0.0), vec3(1.0, 1.0, 0.8), n);
```
Cuando n es bajo, gana el naranja; cuando es alto, aparece el amarillo. Esto da el efecto de regiones más calientes que se mueven. Después combino la textura original del Sol con este color generado por el ruido:
```javascript
vec3 finalColor = textureColor.rgb * 0.6 + noiseColor * 0.6;
```
El resultado es una mezcla al 60-40: suficiente para que la textura siga reconociéndose, pero con la distorsión añadida encima. Y al final aplico un pequeño multiplicador para que el Sol sea un poco más luminoso:
```javascript
gl_FragColor = vec4(finalColor * 1.2, 1.0);
```
Todo este shader lo usé porque quería un Sol que no solo cambiara de color sino que también tuviera movimiento físico, y por eso combiné movimiento geométrico en el vertex shader y movimiento visual en el fragment shader. La parte de la deformación por la normal le da un efecto como de expansión, mientras que el ruido animado hace que los colores nunca se queden fijos.

Para agregar el fondo estrellado decidí crear un shader personalizado también para las estrellas. El objetivo era conseguir estrellas que no solo estuvieran distribuidas de forma aleatoria, sino que también parpadearan ligeramente.
Lo primero que hago en el shader de estrellas es generar un patrón pseudoaleatorio usando las coordenadas UV. Para eso multiplico las UV por un factor grande en starsFragmentShaders:
```javascript
vec2 st = vUv * 500.0;
vec2 ipos = floor(st);
float r = random(ipos);
```
Lo que pasa aquí es que vUv normalmente va de 0 a 1, así que sin multiplicarlo todas las estrellas acabarían demasiado juntas. Con vUv * 500 divido la superficie en 500×500 celdas imaginarias, y a cada celda le asigno un valor aleatorio usando la función random(). Esta función:
```javascript
float random(vec2 st) {
    return fract(sin(dot(st.xy, vec2(12.9898,78.233))) * 43758.5453123);
}
```
Después decido si en esa celda hay una estrella o no, usando un step:
```javascript
float star = step(0.995, r);
```
Esto significa que solo si r es mayor que 0.995 aparece una estrella. La probabilidad es muy baja, así que salen pocas estrellas, pero distribuidas por todo el fondo. De esta forma evito tener que generar miles de puntos desde JavaScript: el propio shader se encarga de decidir dónde hay estrellas.

Una vez que sé si en esa posición hay una estrella, añado el efecto de parpadeo. Quería que cada estrella brillara con un ritmo un poco distinto, así que uso el tiempo y también el valor aleatorio de la celda:
```javascript
float twinkle = 0.8 + 0.2 * sin(u_time * 3.0 + r * 100.0);
```
Al principio intenté que cada una fuera totalmente caótica, pero me di cuenta de que, al usar valores aleatorios muy cercanos para filtrar las estrellas, el parpadeo resultaba casi sincronizado.
La animación es sutil, porque solo sube hasta 1.0 y baja a 0.8, pero eso lo hace más natural. Finalmente calculo el color:
```javascript
vec3 color = vec3(1.0) * star * twinkle;
gl_FragColor = vec4(color, 1.0);
```
Aquí el color es literalmente blanco, porque no necesitaba colores diferentes. star asegura que solo los puntos seleccionados se iluminen y twinkle añade la variación en el brillo. Las zonas donde no hay estrellas simplemente quedan en negro.
En cuanto a la función createStars(), en JavaScript la usé solo para crear la esfera grande que usa este shader. No genera estrellas individualmente, sino que simplemente crea la geometría base y asigna el ShaderMaterial.
El shader es el que lo hace todo. Básicamente dentro de la función creo una esfera muy grande que rodea la escena o lo pongo detrás de la cámara, y lo asigno así:
```javascript
const starsMaterial = new THREE.ShaderMaterial({
    uniforms: { u_time: { value: 0 }},
    vertexShader: starsVertexShader,
    fragmentShader: starsFragmentShader,
    transparent: false
});
```
Y luego en el loop actualizo el tiempo:
```javascript
starsMaterial.uniforms.u_time.value = clock.getElapsedTime();
```
El resto de la función simplemente posiciona la esfera y lo añade a la escena.
El resultado es un fondo que da bastante profundidad a la escena y que combina bien con el Sol animado.

Para el desarrollo de estos shaders me he basado en:

The Book of Shaders (Patricio González Vivo & Jen Lowe): Especialmente los capítulos sobre Noise (Ruido) y Random (Aleatoriedad) para la generación de las estrellas y el plasma solar.

Apuntes de Clase: Referencias sobre cómo pasar uniforms (como u_time) de Three.js a GLSL.

Documentación de Three.js: Para la configuración correcta de ShaderMaterial y SphereGeometry.






