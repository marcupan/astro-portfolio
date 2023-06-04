<script>
	import { onMount } from 'svelte'

	import * as THREE from 'three'
	import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js'
	import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js'

	import Model from '../assets/models/donut_web.glb'

	let el
	let wrapper

	let donutColor = '#f59042'
	let icingColor = '#e042f5'
	let sphereColor = '#42e0f5'

	let sprinklesAmount = 145
	let sprinklesColorsAmount = 1
	let sprinkleColors = [...Array(sprinklesColorsAmount).fill('#ffffff')]

	let sceneRef

	const traverseModel = (scene) => {
		scene.traverse((object) => {
			if (object.isMesh) {
				if (object.name === 'Icing') {
					object.material = new THREE.MeshStandardMaterial({
						color: icingColor,
						metalness: 0.25,
						roughness: 0
					})
				} else if (object.name === 'Donut') {
					object.material = new THREE.MeshStandardMaterial({
						color: donutColor,
						metalness: 0,
						roughness: 0.5
					})
				} else if (object.name.match('Sphere')) {
					object.material = new THREE.MeshStandardMaterial({
						color: sphereColor,
						metalness: 0.1,
						roughness: 0
					})
				} else if (object.name.match('Sprinkle')) {
					const sprinkleIndex = parseInt(object.name.match(/\d+/gi)[0], 10)

					if (sprinkleIndex >= Math.min(Math.max(sprinklesAmount, 0), 145)) {
						object.visible = false
					} else {
						object.visible = true
						object.material = new THREE.MeshStandardMaterial({
							color: sprinkleColors[Math.floor(Math.random() * sprinkleColors.length)],
							metalness: 0.1,
							roughness: 0
						})
					}
				} else {
					object.visible = false
				}
			}
		})
	}

	onMount(() => {
		let renderer
		const scene = new THREE.Scene()

		const camera = new THREE.PerspectiveCamera(
			75,
			wrapper.clientWidth / wrapper.clientHeight,
			0.1,
			1000
		)
		camera.position.z = 0.35

		const directionalLight = new THREE.DirectionalLight(0x9090aa)
		directionalLight.position.set(-10, 10, -10).normalize()
		scene.add(directionalLight)

		const hemisphereLight = new THREE.HemisphereLight(0xffffff, 0x444444)
		hemisphereLight.position.set(1, 1, 1)
		scene.add(hemisphereLight)

		// Instantiate a loader
		const loader = new GLTFLoader()

		// Optional: Provide a DRACOLoader instance to decode compressed mesh data
		const dracoLoader = new DRACOLoader()
		dracoLoader.setDecoderPath('https://www.gstatic.com/draco/versioned/decoders/1.5.6/')
		loader.setDRACOLoader(dracoLoader)

		let mixer

		// Load a glTF resource
		loader.load(
			// resource URL
			Model,
			// called when the resource is loaded
			function (gltf) {
				sceneRef = gltf.scene
				scene.add(sceneRef)

				mixer = new THREE.AnimationMixer(sceneRef)
				const clips = gltf.animations

				clips.forEach((clip) => {
					const action = mixer.clipAction(clip)
					action.play()
				})

				traverseModel(sceneRef)

				gltf.animations // Array<THREE.AnimationClip>
				gltf.scene // THREE.Group
				gltf.scenes // Array<THREE.Group>
				gltf.cameras // Array<THREE.Camera>
				gltf.asset // Object
			},

			// called while loading is progressing
			function (xhr) {
				console.log((xhr.loaded / xhr.total) * 100 + '% loaded')
			},
			// called when loading has errors
			function (error) {
				console.log('An error happened', error)
			}
		)

		const clock = new THREE.Clock()
		const animate = () => {
			requestAnimationFrame(animate)

			if (mixer) {
				mixer.update(clock.getDelta())
			}

			renderer.render(scene, camera)
		}

		const resize = () => {
			renderer.setSize(wrapper.clientWidth, wrapper.clientHeight)
			camera.aspect = wrapper.clientWidth / wrapper.clientHeight
			camera.updateProjectionMatrix()
		}

		const createScene = (el) => {
			renderer = new THREE.WebGLRenderer({ alpha: false, antialias: true, canvas: el })
			renderer.setClearColor(0x000000, 0)
			resize()
			animate()
		}

		// if (renderer) {
		// 	renderer.setAnimationLoop(animate)
		// }

		window.addEventListener('resize', resize)

		if (el) {
			createScene(el)
		}
	})
</script>

<div class="card absolute right-0 top-0 max-w-xs p-4">
	<div class="card-bg" />

	<div class="z-10">
		<label for="donut-color">Donut Color</label>
		<input
			id="donut-color"
			type="color"
			bind:value={donutColor}
			on:input={() => traverseModel(sceneRef)}
			class="w-full outline-0"
		/>

		<label for="icing-color">Icing color</label>
		<input
			id="icing-color"
			type="color"
			bind:value={icingColor}
			on:input={() => traverseModel(sceneRef)}
			class="w-full"
		/>

		<label for="donut-color">Sphere color</label>
		<input
			id="sphere-color"
			type="color"
			bind:value={sphereColor}
			on:input={() => traverseModel(sceneRef)}
			class="w-full"
		/>

		<div class="flex flex-col">
			<fieldset>
				<label for="sprinkles-amount">Sprinkles amount (max: 145)</label>
				<input
					id="sprinkles-amount"
					type="number"
					min="0"
					max="145"
					bind:value={sprinklesAmount}
					on:input={() => traverseModel(sceneRef)}
					class="w-full border-2 invalid:border-red-400"
				/>
				<p class="error-message text-sm text-red-400">
					{#if sprinklesAmount <= 1}min: 0{/if}
					{#if sprinklesAmount > 10}max: 145{/if}
				</p>
			</fieldset>

			<fieldset>
				<label for="sprinkles-colors-amount">Sprinkles colors amount (max: 10)</label>
				<input
					id="sprinkles-colors-amount"
					type="number"
					min="1"
					max="10"
					bind:value={sprinklesColorsAmount}
					on:input={() => {
						sprinkleColors = [
							...Array(Math.min(Math.max(sprinklesColorsAmount, 1), 10)).fill('#ffffff')
						]
						traverseModel(sceneRef)
					}}
					class="w-full border-2 invalid:border-red-400"
				/>

				<p class="error-message text-sm text-red-400">
					{#if sprinklesAmount <= 1}min: 1{/if}
					{#if sprinklesAmount > 10}max: 10{/if}
				</p>
			</fieldset>

			<div class="flex flex-wrap">
				{#each sprinkleColors as opt}
					<input type="color" bind:value={opt} on:input={() => traverseModel(sceneRef)} />
				{/each}
			</div>
		</div>
	</div>
</div>
<div bind:this={wrapper} class="h-full w-full">
	<canvas bind:this={el} />
</div>

<style>
	.card {
		display: flex;
		border-radius: 0.6rem;
		background-position: 100%;
		transition: background-position 0.6s cubic-bezier(0.22, 1, 0.36, 1);
		box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -2px rgba(0, 0, 0, 0.1);
		overflow: hidden;
	}

	.card-bg {
		position: absolute;
		height: 100%;
		width: 100%;
		left: 0;
		top: 0;
		background: rgba(255, 255, 255, 0.3);
		border-radius: 0.6rem;
		box-shadow: 0 4px 30px rgba(0, 0, 0, 0.1);
		backdrop-filter: blur(5px);
		-webkit-backdrop-filter: blur(5px);
	}

	input {
		outline: none;
		padding: 0.25rem;
		margin: 0.5rem 0;
	}

	.error-message {
		display: none;
	}
	fieldset:invalid .error-message {
		display: block;
	}
</style>
