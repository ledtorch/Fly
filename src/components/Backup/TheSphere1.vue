<template>
  <div class="frame" ref="frame">
    <div ref="container" id="container"></div>
  </div>
</template>

<script>
import * as THREE from 'three';

export default {
  mounted() {
    this.initThree();
  },
  beforeDestroy() {
    window.removeEventListener('resize', this.onWindowResize);
  },
  methods: {
    async initThree() {
      const container = this.$refs.container;
      const width = window.innerWidth;
      const height = window.innerHeight;

      // Create scene
      const scene = new THREE.Scene();

      // Create camera
      const camera = new THREE.PerspectiveCamera(60, width / height, 0.1, 1000);
      // Adjust this value to fit the sphere within 500px diameter
      camera.position.z = 6;

      // Create renderer
      // const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
      // ↓ Hight performance mode
      const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, powerPreference: "high-performance" });

      renderer.setSize(width, height);
      container.appendChild(renderer.domElement);

      // // Add coreSphere
      const coreSphere = this.createCoreSphere();
      scene.add(coreSphere);

      // Add dotsSphere
      const dotsSphere = this.createDotsSphere();
      scene.add(dotsSphere);

      // Wrap model to a unit
      const earthGroup = new THREE.Group();
      // earthGroup.add(coreSphere);
      earthGroup.add(dotsSphere);
      scene.add(earthGroup);

      try {
        const dotsSphere = await this.createDotsSphere();
        earthGroup.add(dotsSphere);
      } catch (error) {
        console.error('Error creating dots sphere:', error);
      }

      const light = new THREE.DirectionalLight(0xffffff, 1);
      light.position.set(1, 1, 1).normalize();
      scene.add(light);

      // Resize listener
      window.addEventListener('resize', this.onWindowResize.bind(this, camera, renderer));

      // Render order
      renderer.sortObjects = true;
      // coreSphere.renderOrder = 1;
      dotsSphere.renderOrder = 2;

      // Animation loop
      const animate = () => {
        requestAnimationFrame(animate);
        earthGroup.rotation.y += 0.007;
        renderer.render(scene, camera);
      };
      animate();

      // 🐞 Debug console
      console.log('🐞 Debug console');
      console.log(dotsSphere);
      // console.log(coreSphere);
      console.log(renderer);

    },



    async createDotsSphere() {
      // Dots number
      const DOT_COUNT = 30000;

      const dotMaterial = new THREE.MeshBasicMaterial({
        color: 0x1b2d53ff,
        depthTest: false,
        transparent: true,
        side: THREE.DoubleSide
      });

      // Load the color-coded image then get each pixel’s color
      const loader = new THREE.TextureLoader();
      const texture = await new Promise((resolve, reject) => {
        loader.load(
          '/Image/Map3.png',
          resolve,  // onLoad callback
          undefined,  // onProgress callback (not needed)
          reject  // onError callback
        );
      });

      const image = texture.image;
      const canvas = document.createElement('canvas');
      canvas.width = image.width;
      canvas.height = image.height;
      const context = canvas.getContext('2d');
      context.drawImage(image, 0, 0);
      const imageData = context.getImageData(0, 0, canvas.width, canvas.height);

      const vector = new THREE.Vector3();
      const sceneDots = new THREE.Group();

      for (let i = DOT_COUNT; i >= 0; i--) {
        const phi = Math.acos(-1 + (2 * i) / DOT_COUNT);
        const theta = Math.sqrt(DOT_COUNT * Math.PI) * phi;
        vector.setFromSphericalCoords(2.5, phi, theta);

        // Convert spherical coordinates to UV coordinates
        const uv = {
          // x: 1 - ((theta + Math.PI) / (2 * Math.PI)),
          x: ((theta + Math.PI) / (2 * Math.PI)),
          y: phi / Math.PI
        };

        // Get the corresponding pixel color on the map
        const pixelIndex = ((Math.floor(uv.y * (canvas.height - 1)) * canvas.width) + Math.floor(uv.x * (canvas.width - 1))) * 4;
        const r = imageData.data[pixelIndex];
        const g = imageData.data[pixelIndex + 1];
        const b = imageData.data[pixelIndex + 2];

        // Check if the pixel color represents land
        if (r === 0 && g === 0 && b === 0) {  // Assuming white represents land
          const dotGeometry = new THREE.CircleGeometry(0.02, 5);
          const dot = new THREE.Mesh(dotGeometry, dotMaterial);
          dot.position.set(vector.x, vector.y, vector.z);
          dot.lookAt(new THREE.Vector3(0, 0, 0));
          sceneDots.add(dot);
          // console.log(dot.position);
        }
      }

      return sceneDots;
    },

    createCoreSphere() {
      /* 
      Opaque objects are drawn first, followed by transparent objects.
      Therefore, set transparent: true for all objects in this project.
      */
      const sphereMaterial = new THREE.MeshPhongMaterial({
        color: 0xFFFFFF,
        transparent: true,
        opacity: 0.5,
        depthTest: true,
      });

      const sphereRadius = 2;

      // // Test
      // const textureLoader = new THREE.TextureLoader();
      // const mapTexture = textureLoader.load('/Image/Map3.png');
      // const sphereGeometry = new THREE.SphereGeometry(2.5, 32, 32);
      // const sphereMaterial = new THREE.MeshBasicMaterial({
      //   map: mapTexture,
      //   color: 0x1a2c52ff,
      //   transparent: true,
      //   opacity: 0.5,
      //   depthTest: true,
      // });
      // const coreSphere = new THREE.Mesh(sphereGeometry, sphereMaterial);

      const sphereGeometry = new THREE.SphereGeometry(sphereRadius, 32, 32);
      const coreSphere = new THREE.Mesh(sphereGeometry, sphereMaterial);

      return coreSphere;
    },
    onWindowResize(camera, renderer) {
      const width = window.innerWidth;
      const height = window.innerHeight;
      camera.aspect = width / height;
      camera.aspect = height / width;
      camera.updateProjectionMatrix();
      renderer.setSize(width, height);
    },
  },
};
</script>

<style scoped>
#container {
  position: absolute;
  /* Changed from fixed to absolute */
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 1;
  /* Ensure this element is rendered on top of .frame */
}

.frame {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: url('/Image/Nebula.jpg') no-repeat center center fixed;
  background-size: cover;
  z-index: -1;
  /* Ensure this element is rendered behind #container */
}
</style>

