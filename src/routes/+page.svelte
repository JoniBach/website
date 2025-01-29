<script>
	import { onMount } from 'svelte';
	import * as THREE from 'three';
	import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
	import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader';
	import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer';
	import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass';
	import { UnrealBloomPass } from 'three/examples/jsm/postprocessing/UnrealBloomPass';
	import { Raycaster, Vector2 } from 'three';

	let raycaster = new Raycaster();
	let mouse = new Vector2();
	let canvas;
	let clickedObject = null;
	let hoveredObject = null;
	let haseMoved = false;

	onMount(() => {
		window.addEventListener('mouseenter', (event) => {
			// Convert screen coordinates to normalized device coordinates (-1 to 1)
			mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
			mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

			// Update raycaster
			raycaster.setFromCamera(mouse, camera);

			// Check for intersections
			const intersects = raycaster.intersectObjects(scene.children, true);

			if (intersects.length > 0) {
				hoveredObject = intersects[0].object;
			}
		});

		window.addEventListener('mousemove', () => {
			haseMoved = true;
			hoveredObject = null;
		});
		window.addEventListener('mousedown', (event) => {
			haseMoved = false;
			// Convert screen coordinates to normalized device coordinates (-1 to 1)
			mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
			mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

			// Update raycaster
			raycaster.setFromCamera(mouse, camera);

			// Check for intersections
			const intersects = raycaster.intersectObjects(scene.children, true);

			if (intersects.length > 0) {
				clickedObject = intersects[0].object;
			}
		});
		window.addEventListener('mouseup', (event) => {
			if (clickedObject) {
				// Convert screen coordinates to normalized device coordinates (-1 to 1)
				mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
				mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

				// Update raycaster
				raycaster.setFromCamera(mouse, camera);

				// Check for intersections
				const intersects = raycaster.intersectObjects(scene.children, true);

				if (intersects.length > 0) {
					let newClickedObject = intersects[0].object;

					if (
						!haseMoved &&
						clickedObject.userData.groupName === newClickedObject.userData.groupName
					) {
						clickedObject = null;
						// Find the parent sign node
						while (newClickedObject.parent && !newClickedObject.userData.groupName) {
							newClickedObject = newClickedObject.parent;
						}

						if (newClickedObject.userData.groupName) {
							const groupName = newClickedObject.userData.groupName;
							const url = `${decodeURIComponent(groupName.replace('sign_', ''))}`;

							window.open(url, '_blank');
						}
					} else {
						clickedObject = null;
					}
				}
			}
		});

		// Scene, camera, and renderer setup
		const scene = new THREE.Scene();
		const camera = new THREE.PerspectiveCamera(
			75,
			window.innerWidth / window.innerHeight,
			0.1,
			1000
		);

		// Add fog to the scene to fade the floor into the distance
		scene.fog = new THREE.Fog(0x0a0a2a, 10, 100);

		// Create a large plane for the floor
		const floorGeometry = new THREE.CircleGeometry(150, 64);
		const floorMaterial = new THREE.MeshStandardMaterial({
			color: 0x222222,
			roughness: 1
		});
		const floor = new THREE.Mesh(floorGeometry, floorMaterial);
		floor.rotation.x = -Math.PI / 2; // Rotate to be horizontal
		scene.add(floor);
		// Move the floor down by 10 units
		floor.position.y = -10;
		const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
		renderer.setSize(window.innerWidth, window.innerHeight);
		renderer.setPixelRatio(window.devicePixelRatio);

		// Set the background color of the scene to sky blue
		scene.background = new THREE.Color(0x0a0a2a);
		// scene.background = new THREE.Color(0x000000);
		// Set the floor color to the same as the background color
		floor.material.color.set(0x666666);

		// OrbitControls setup
		const controls = new OrbitControls(camera, renderer.domElement);
		controls.enableDamping = true;
		controls.dampingFactor = 0.1;
		controls.minPolarAngle = Math.PI / 4; // Prevent going below the ground
		controls.maxPolarAngle = Math.PI / 2; // Prevent flipping the view
		controls.enablePan = false; // Disable panning
		controls.screenSpacePanning = false;
		controls.minAzimuthAngle = 0; // Restrict horizontal rotation to -90 degrees
		controls.maxAzimuthAngle = Math.PI / 2; // Restrict horizontal rotation to 90 degrees
		controls.minDistance = 10; // Set the minimum zoom distance
		controls.maxDistance = 50; // Set the maximum zoom distance
		// Camera position
		camera.position.set(0, 0, 25);
		controls.update();

		// Postprocessing setup (EffectComposer)
		const composer = new EffectComposer(renderer);
		const renderPass = new RenderPass(scene, camera);
		composer.addPass(renderPass);

		// Add UnrealBloomPass for glow
		const bloomPass = new UnrealBloomPass(
			new THREE.Vector2(window.innerWidth, window.innerHeight),
			0.2, // Strength of the bloom
			0.4, // Radius
			0.3 // Threshold
		);
		composer.addPass(bloomPass);

		// Load the GLTF model
		const loader = new GLTFLoader();
		loader.load(
			'/DIORAMA 3.2.gltf',
			(gltf) => {
				// Set position and add the scene to the main scene
				gltf.scene.position.set(0, -8, 0);
				scene.add(gltf.scene);

				// Initialize an array to store the materials
				const materials = [];
				const nodes = [];
				// Function to recursively traverse the GLTF scene graph
				gltf.scene.traverse((child) => {
					if (child.isMesh) {
						if (Array.isArray(child.material)) {
							// If the mesh has multiple materials
							child.material.forEach((mat) => {
								if (!materials.includes(mat)) {
									materials.push(mat);
								}
							});
						} else if (child.material && !materials.includes(child.material)) {
							// If the mesh has a single material
							materials.push(child.material);
						}
					}
					// Add the node to the nodes array
					nodes.push(child);
				});

				const neonNodes = nodes.filter(
					(node) => node.material && node.material.name === 'sign_neon'
				);

				const materialNames = materials.map((material) => material.name);
				const nodeNames = nodes.map((node) => node.name);
				const lanternNodes = nodes.filter(
					(node) => node.material && node.material.name === 'lantern_red'
				);
				const streetLightNodes = nodes.filter(
					(node) => node.material && node.material.name === 'street_light'
				);

				const signNodes = nodes.filter((node) => node.name && node.name.startsWith('sign_'));
				signNodes.forEach((group) => {
					group.children.forEach((child) => {
						child.userData.groupName = group.name; // Store the group name in the child's userData

						child.addEventListener('click', (event) => {
							console.log('Child clicked:', event.target); // Debugging statement
							const groupName = event.target.userData.groupName;
							const url = groupName.replace('sign_', '');
							console.log('Opening URL:', `https://${url}`); // Debugging statement
							window.open(`https://${url}`, '_blank');
						});
					});
				});

				// Add a spotlight to the scene
				const spotlight = new THREE.SpotLight(0xffffff, 5);
				spotlight.target.position.set(3.5, 12, -9);
				spotlight.position.set(3.5, 9, -8);
				spotlight.angle = Math.PI / 3; // Set the angle of the spotlight
				spotlight.penumbra = 0.1; // Set the penumbra of the spotlight
				spotlight.decay = 2; // Set the decay of the spotlight
				spotlight.distance = 5; // Set the distance of the spotlight
				spotlight.castShadow = true; // Enable shadows for the spotlight
				// Rotate the spotlight to face diagonally up instead of down
				scene.add(spotlight.target);
				// Set up shadow properties for the spotlight
				spotlight.shadow.mapSize.width = 100;
				spotlight.shadow.mapSize.height = 100;
				spotlight.shadow.camera.near = 10;
				spotlight.shadow.camera.far = 200;
				spotlight.shadow.camera.fov = 30;

				// Create a helper to visualize the spotlight
				// const spotlightHelper = new THREE.SpotLightHelper(spotlight);
				// scene.add(spotlightHelper);

				scene.add(spotlight);

				console.log('Sign Nodes:', signNodes);
				console.log('Street Light Nodes:', streetLightNodes);
				console.log('Lantern Nodes:', lanternNodes);

				console.log('Extracted Materials:', materials);
				console.log('Material Nodes:', nodes);
				console.log('Material Names:', materialNames);
				console.log('Node Names:', nodeNames);

				lanternNodes.forEach((node) => {
					if (node.material) {
						// Clone the material to ensure each node has its own unique instance
						node.material = node.material.clone();
						// Increase the emissive intensity for the lantern nodes
						node.material.emissiveIntensity = 4; // Adjust the intensity as needed
					}
				});

				// Neon glow effect
				const neonColors = [
					new THREE.Color(0xff0000), // Red
					new THREE.Color(0x00ff00), // Green
					new THREE.Color(0x0000ff), // Blue
					new THREE.Color(0xffff00), // Yellow
					new THREE.Color(0xff00ff), // Magenta
					new THREE.Color(0x00ffff), // Cyan
					new THREE.Color(0xffa500), // Orange
					new THREE.Color(0x800080) // Purple
				];
				neonNodes.forEach((node, i) => {
					if (node.material) {
						// Clone the material to ensure each node has its own unique instance
						node.material = node.material.clone();
						node.material.color = new THREE.Color(0x000000); // Set the base color to black
						// Set the neon color
						node.material.emissive = neonColors[i % neonColors.length];
						node.material.emissiveIntensity = 10; // Set the intensity of the glow

						// Add a point light at the position of the neon sign
						const neonLight = new THREE.PointLight(neonColors[i % neonColors.length], 2, 10);
						// Get the world position of the node
						const worldPosition = new THREE.Vector3();
						node.getWorldPosition(worldPosition);
						neonLight.position.copy(worldPosition);
						scene.add(neonLight);
					}
				});

				// Add red lights for lanterns
				lanternNodes.forEach((node) => {
					const lanternLight = new THREE.PointLight(0xff0000, 2, 10); // Red light
					// Get the world position of the node
					const worldPosition = new THREE.Vector3();
					node.getWorldPosition(worldPosition);
					lanternLight.position.copy(worldPosition);
					scene.add(lanternLight);
				});

				// Add white lights for street lights
				streetLightNodes.forEach((node) => {
					const streetLight = new THREE.PointLight(0xffffff, 10, 30); // White light
					// Get the world position of the node
					const worldPosition = new THREE.Vector3();
					node.getWorldPosition(worldPosition);
					streetLight.position.copy(worldPosition);
					worldPosition.y -= 2; // Move the light down by 2 units
					streetLight.position.copy(worldPosition);
					streetLight.position.x -= 2; // Move the light to the right by 2 units
					streetLight.position.z -= 2; // Move the light forward by 2 units
					scene.add(streetLight);
				});
			},

			undefined,
			(error) => {
				console.error('Error loading GLTF model:', error);
			}
		);

		// Handle resizing
		const onResize = () => {
			camera.aspect = window.innerWidth / window.innerHeight;
			camera.updateProjectionMatrix();
			renderer.setSize(window.innerWidth, window.innerHeight);
			composer.setSize(window.innerWidth, window.innerHeight);
		};
		window.addEventListener('resize', onResize);

		// Animation loop
		const animate = () => {
			requestAnimationFrame(animate);
			controls.update();
			composer.render(); // Use composer instead of renderer
		};
		animate();

		// Cleanup
		return () => {
			window.removeEventListener('resize', onResize);
			renderer.dispose();
			controls.dispose();
			composer.dispose();
		};
	});
</script>

<canvas bind:this={canvas}></canvas>

<style>
	canvas {
		display: block;
		width: 100vw;
		height: 100vh;
		margin: 0;
		overflow: hidden;
	}
	:global(body) {
		margin: 0;
		overflow: hidden;
	}
</style>
