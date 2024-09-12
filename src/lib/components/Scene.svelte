<script>
	import { T, useRender, useThrelte } from '@threlte/core';
	import { OrbitControls, Text } from '@threlte/extras';
	import { Color, Mesh, PMREMGenerator, PlaneGeometry, Raycaster, Vector2, Vector3 } from 'three';
	import { onMount } from 'svelte';
	import Stars from './Stars.svelte';
	import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
	import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
	import { OutputPass } from 'three/addons/postprocessing/OutputPass.js';
	import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';
	import Spaceship2 from './models/Spaceship2.svelte'
	import Rocketisro from './models/rocketisro.svelte'
	import Chandrayan from './models/chandrayan.svelte'
	import Mangalayan from './models/mangalayan.svelte'

	const { scene, camera, renderer } = useThrelte();
	let spaceShipRef;
	let nextButtonRef;
	let intersectionPoint;
	let translY = 0;
	let translAccelleration = 0;
	let angleZ = 0;
	let angleAccelleration = 0;
	let pmrem = new PMREMGenerator(renderer);
	let envMapRT;
	let mouse = new Vector2();
	let ModelRef;
	

	let scrollPos = 0; // Define scrollPos in your main scene

	const composer = new EffectComposer(renderer);
	composer.setSize(innerWidth, innerHeight);

	const setupEffectComposer = () => {
		const renderPass = new RenderPass(scene, camera.current);
		composer.addPass(renderPass);

		const bloomPass = new UnrealBloomPass(new Vector2(innerWidth, innerHeight), 0.275, 1, 0);
		composer.addPass(bloomPass);

		const outputPass = new OutputPass();
		composer.addPass(outputPass);
	};

	// takes control of the render loop, unlike useFrame
	// https://threlte.xyz/docs/reference/core/use-render
	useRender(({ scene }) => {
		if (ModelRef) {
            ModelRef.rotation.y += 0.01; // Rotate around the Y-axis
        }
		if (intersectionPoint) {
			const targetY = intersectionPoint?.y || 0;
			translAccelleration += (targetY - translY) * 0.002; // stiffness
			translAccelleration *= 0.95; // damping
			translY += translAccelleration;

			const dir = intersectionPoint.clone().sub(new Vector3(0, translY, 0)).normalize();
			const dirCos = dir.dot(new Vector3(0, 1, 0));
			const angle = Math.acos(dirCos) - Math.PI * 0.5;
			angleAccelleration += (angle - angleZ) * 0.01; // stiffness
			angleAccelleration *= 0.85; // damping
			angleZ += angleAccelleration;
		}

		if (envMapRT) envMapRT.dispose();

		spaceShipRef.visible = false;
		ModelRef.visible = false
		scene.background = null;
		envMapRT = pmrem.fromScene(scene, 0, 0.1, 1000);
		scene.background = new Color('#598889').multiplyScalar(0.05);
		spaceShipRef.visible = true;
		ModelRef.visible = true;

		spaceShipRef.traverse((child) => {
			if (child?.material?.envMapIntensity) {
				child.material.envMap = envMapRT.texture;
				child.material.envMapIntensity = 50;
				child.material.normalScale.set(0.3, 0.3);
			}
		});
		ModelRef.traverse((child) => {
			if (child?.material?.envMapIntensity) {
				child.material.envMap = envMapRT.texture;
				child.material.envMapIntensity = 25;
				child.material.normalScale.set(0.3, 0.3);
			}
		});

		composer.render();
	});

	onMount(() => {
		setupEffectComposer();

		const planeGeo = new PlaneGeometry(20, 20);
		const mesh = new Mesh(planeGeo);

		const raycaster = new Raycaster();
		const pointer = new Vector2();

		function onPointerMove(event) {
			console.log(event)
			pointer.x = (event.clientX / window.innerWidth) * 2 - 1;
			pointer.y = -(event.clientY / window.innerHeight) * 2 + 1;

			raycaster.setFromCamera(pointer, $camera);
			const intersects = raycaster.intersectObject(mesh);
			intersectionPoint = intersects[0]?.point;

			if (intersectionPoint) {
				// this prevents the spring motion to be different while the pointer
				// spans the x axis
				intersectionPoint.x = 3;
			}
		}

		const onClick = (event) => {
			// Convert mouse coordinates to normalized device coordinates (-1 to +1)
			mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
			mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

			// Update raycaster with the camera and mouse position
			raycaster.setFromCamera(mouse, camera);

			// Check if the "Next" text was clicked
			const intersects = raycaster.intersectObject(nextButtonRef, true);
			if (intersects.length > 0) {
				handleNextText(); // Trigger the next text when clicked
			}
    	};

		window.addEventListener('pointermove', onPointerMove);
		window.addEventListener('click', onClick);
		return () => {
			window.removeEventListener('pointermove', onPointerMove);
			window.removeEventListener('click', onClick);
		};
	});

	let textSet = 0; // Tracks the current set of text being displayed
    let textData = [
        "The Journey of ISRO:\nThe Indian Space Research Organisation (ISRO) was established in\n1969 to harness space technology for national development while \npursuing space science research and planetary exploration. Founded \nunder the visionary leadership of Dr. Vikram Sarabhai, ISRO's journey \nbegan with humble beginnings but has grown to become one of the \nworld's leading space agencies. Its mission is to provide cost-effective \nand reliable space-based services while contributing to the peaceful \nexploration of outer space. ISRO's continuous efforts have transformed \nIndia into a spacefaring nation capable of independent satellite \nlaunches and advanced space missions.",
        "Chandrayaan: India’s Lunar Triumph:\nThe Chandrayaan mission marked a significant milestone in ISRO's \nspace exploration journey. Chandrayaan-1, launched in 2008, was \nIndia’s first mission to the Moon. It aimed to explore and map the \nlunar surface while searching for the presence of water molecules. \nChandrayaan-2, launched in 2019, was an ambitious follow-up mission, \nwith an orbiter, lander (Vikram), and rover (Pragyan). Though the lander \ndid not achieve a soft landing, the mission was considered a success as \nthe orbiter continues to relay valuable data about the Moon’s surface. \nChandrayaan missions positioned India as a major player in lunar exploration.",
        "Mangalyaan, Reaching for Mars:\nMangalyaan, or the Mars Orbiter Mission (MOM), is one of ISRO’s most \ncelebrated achievements. Launched in 2013, Mangalyaan made India the \nfirst country to reach Mars orbit on its maiden attempt, and the fourth \nspace agency globally to do so. MOM’s mission objectives were to study \nthe Martian surface, morphology, atmosphere, and mineral composition. \nIt is renowned for its cost-effectiveness, having been completed on a \nshoestring budget compared to other Mars missions. Mangalyaan’s success \nis a testament to ISRO’s ingenuity and resourcefulness in space exploration."
    ];

    const handleNextText = () => {
        if (textSet < textData.length - 1) {
            textSet += 1;
        } else {
            textSet = 0; // Loop back to the first text
        }
    };

</script>

<T.PerspectiveCamera makeDefault position={[-5, 6, 10]} fov={25}>
	<OrbitControls enableDamping target={[0, 0, 0]} enableZoom={false} enablePan={false} enableRotate={false} />
</T.PerspectiveCamera>

<Spaceship2
bind:ref={spaceShipRef}
	scale={.025}
	position={[4, translY, 0]}
	rotation={[angleZ, 30, angleZ, 'ZXY']}
/>

<Stars/>

<Text
    position={[-4,2+(translY*.1),-1]}  
    fontSize={0.2}          
    color="white"
    text={textData[textSet]}
/>

{#if textSet === 1}
	<Chandrayan 
		bind:ref={ModelRef}  
		scale={.5}
		position={[-1.5, -2+(translY*.1), -1.5]}
	/>
{/if}

{#if textSet === 0}
	<Rocketisro 
		bind:ref={ModelRef}  
		scale={.075}
		position={[-1, -3.5+(translY*.1), -1.5]}
	/>
{/if}

{#if textSet === 2}
	<Mangalayan 
		bind:ref={ModelRef}  
		scale={1}
		position={[-1, -2+(translY*.1), -1.5]}
	/>
{/if}

<Text
    bind:ref={nextButtonRef}  
    position={[1, -1+(translY*.1), -1]}    
    fontSize={0.3}           
    color="yellow"
    text="Next"               
    cursor="pointer"         
/>