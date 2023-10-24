<template>
  <div ref="container" id="container"></div>
</template>

<script>
import * as THREE from 'three';

export default {
  name: 'TheSphere',
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
      camera.position.z = 5;  // Adjust this value to fit the sphere within 500px diameter

      // Create renderer
      const renderer = new THREE.WebGLRenderer({ antialias: true });
      renderer.setSize(width, height);
      container.appendChild(renderer.domElement);

      // Create dots
      const dotsGroup = this.createDots();
      scene.add(dotsGroup);

      // Resize listener
      window.addEventListener('resize', this.onWindowResize.bind(this, camera, renderer));

      // Animation loop
      const animate = () => {
        requestAnimationFrame(animate);
        renderer.render(scene, camera);
      };
      animate();
    },
    createDots() {
      const DOT_COUNT = 72000;
      const dotMaterial = new THREE.MeshBasicMaterial({ color: 0x0077ff });
      const vector = new THREE.Vector3();
      const sceneDots = new THREE.Group();  // Create a group to hold all dots

      for (let i = DOT_COUNT; i >= 0; i--) {
        const phi = Math.acos(-1 + (2 * i) / DOT_COUNT);
        const theta = Math.sqrt(DOT_COUNT * Math.PI) * phi;

        vector.setFromSphericalCoords(2.5, phi, theta);  // Adjust radius to match the sphere radius

        const dotGeometry = new THREE.CircleGeometry(0.01, 5);  // Adjust dot size as needed
        const dot = new THREE.Mesh(dotGeometry, dotMaterial);  // Create a mesh for each dot
        dot.position.set(vector.x, vector.y, vector.z);  // Set the position of each dot
        dot.lookAt(new THREE.Vector3(0, 0, 0));  // Make each dot face the center of the sphere

        sceneDots.add(dot);  // Add each dot to the group
      }

      return sceneDots;  // Return the group containing all dots
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
  position: fixed;
  top: 0;
  left: 0;
}
</style>

