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
    initThree() {
      const container = this.$refs.container;
      const width = window.innerWidth;
      const height = window.innerHeight;

      // Create scene
      const scene = new THREE.Scene();

      // Create camera
      const camera = new THREE.PerspectiveCamera(75, width / height, 0.1, 1000);
      // Adjust this value to fit the sphere within 500px diameter
      camera.position.z = 5;

      // Create renderer
      const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
      renderer.setSize(width, height);
      container.appendChild(renderer.domElement);

      // // Add coreSphere
      const coreSphere = this.createCoreSphere();
      scene.add(coreSphere);

      // Add dotsSphere
      const dotsSphere = this.createDotsSphere();
      scene.add(dotsSphere);


      const light = new THREE.DirectionalLight(0xffffff, 1);
      light.position.set(1, 1, 1).normalize();
      scene.add(light);

      // Resize listener
      window.addEventListener('resize', this.onWindowResize.bind(this, camera, renderer));

      // Render order
      renderer.sortObjects = true;
      coreSphere.renderOrder = 1;
      dotsSphere.renderOrder = 2;

      // Animation loop
      const animate = () => {
        requestAnimationFrame(animate);
        renderer.render(scene, camera);
      };
      animate();

      // 🐞 Debug console
      console.log('🐞 Debug console');
      console.log(dotsSphere);
      console.log(coreSphere);
      console.log(renderer);

    },
    createDotsSphere() {
      // Dots number
      const DOT_COUNT = 18000;

      const dotMaterial = new THREE.MeshBasicMaterial({
        color: 0x1b2d53ff,
        depthTest: false,
        transparent: true,
      });

      // The XYZ coordinate of each dot
      const positions = [];

      // A random identifier for each dot
      const rndId = [];

      // The country border each dot falls within
      const countryIds = [];


      const vector = new THREE.Vector3();
      // Create a group to hold all dots
      const sceneDots = new THREE.Group();
      for (let i = DOT_COUNT; i >= 0; i--) {
        const phi = Math.acos(-1 + (2 * i) / DOT_COUNT);
        const theta = Math.sqrt(DOT_COUNT * Math.PI) * phi;

        vector.setFromSphericalCoords(2.5, phi, theta);  // Adjust radius to match the sphere radius

        // Create dots
        const dotGeometry = new THREE.CircleGeometry(0.02, 5);
        const dot = new THREE.Mesh(dotGeometry, dotMaterial);
        dot.position.set(vector.x, vector.y, vector.z);
        dot.lookAt(new THREE.Vector3(0, 0, 0));

        // Add each dot to the group
        sceneDots.add(dot);
      }

      return sceneDots;
    },
    createCoreSphere() {
      /* 
      Opaque objects are drawn first, followed by transparent objects.
      Therefore, set transparent: true for all objects in this project.
      */
      const coreMaterial = new THREE.MeshPhongMaterial({
        color: 0x1a2c52ff,
        transparent: true,
        opacity: 0.1,
        depthTest: true,
      });

      const sphereRadius = 2.48;
      const sphereGeometry = new THREE.SphereGeometry(sphereRadius, 32, 32);
      const coreSphere = new THREE.Mesh(sphereGeometry, coreMaterial);

      return coreSphere;
    },
    onWindowResize(camera, renderer) {
      const width = window.innerWidth;
      const height = window.innerHeight;
      camera.aspect = width / height;
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

