<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';
	import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
	import { Line2 } from 'three/examples/jsm/lines/Line2.js';
	import { LineSegments2 } from 'three/examples/jsm/lines/LineSegments2.js';
	import { LineGeometry } from 'three/examples/jsm/lines/LineGeometry.js';
	import { LineMaterial } from 'three/examples/jsm/lines/LineMaterial.js';
	import { EdgesGeometry } from 'three/src/geometries/EdgesGeometry';
	import { Slider } from '$lib/components/ui/slider';
	import { TextGeometry } from 'three/addons/geometries/TextGeometry.js';
	import { FontLoader } from 'three/examples/jsm/loaders/FontLoader.js';

	let canvas: HTMLCanvasElement;

	let medalCounts = {
		USA: { gold: 40, silver: 44, bronze: 42, name: 'United States', borderColor: 0x051d43 },
		CHN: { gold: 40, silver: 27, bronze: 24, name: 'China', borderColor: 0xfdee00 },
		JPN: { gold: 20, silver: 12, bronze: 13, name: 'Japan', borderColor: 0xe63329 },
		AUS: { gold: 18, silver: 19, bronze: 16, name: 'Australia', borderColor: 0x0d3ca5 }, //0xffffff
		FRA: { gold: 16, silver: 26, bronze: 22, name: 'France', borderColor: 0xca112d },
		NED: { gold: 15, silver: 7, bronze: 12, name: 'Netherlands', borderColor: 0x0d3ca5 },
		GBR: { gold: 14, silver: 22, bronze: 29, name: 'Great Britain', borderColor: 0xffffff }, //0xdd062a
		KOR: { gold: 13, silver: 9, bronze: 10, name: 'South Korea', borderColor: 0xffffff },
		ITA: { gold: 12, silver: 13, bronze: 15, name: 'Italy', borderColor: 0x37854b },
		GER: { gold: 12, silver: 13, bronze: 8, name: 'Germany', borderColor: 0xf1cb00 }
	};
	let levelOfSeparation = 2;
	let displayTable = true;

	onMount(() => {
		if (canvas) createScene();
	});

	function makeText(font, textContents, scale, color, pos, rot) {
		const geometry = new TextGeometry(textContents, {
			font: font,
			size: 1 * scale,
			depth: 0.01 * scale,
			curveSegments: 10,
			// bevelEnabled: true,
			bevelThickness: 0.01 * scale,
			bevelSize: 0.1 * scale,
			bevelOffset: 0,
			bevelSegments: 2
		});
		//Move the text to the right place
		geometry.translate(pos[0], pos[1], pos[2]);
		//Rotate
		geometry.rotateZ(rot);

		const material = new THREE.MeshBasicMaterial({ color: color });
		const text = new THREE.Mesh(geometry, material);
		return text;
	}

	function createScene() {
		const scene = new THREE.Scene();
		const camera = new THREE.PerspectiveCamera(
			75,
			window.innerWidth / window.innerHeight,
			0.1,
			1000
		);

		const renderer = new THREE.WebGLRenderer({ canvas });
		renderer.setSize(window.innerWidth, window.innerHeight);

		const controls = new OrbitControls(camera, renderer.domElement);
		controls.enableDamping = true;
		controls.dampingFactor = 0.25;
		controls.enableZoom = true;
		controls.enableRotate = true;
		camera.position.x = 3;
		camera.position.y = 3;
		camera.position.z = 0;
		camera.rotation.x = (3 * Math.PI) / 4; // Rotate 45 degrees around the X-axis
		camera.rotation.y = Math.PI / 4; // Rotate 45 degrees around the Y-axis
		camera.rotation.z = 0; // No rotation around the Z-axis

		scene.add(camera);

		let countries = Object.keys(medalCounts);
		let numIncrements = 40;

		let medalGrids: Record<string, number[][]> = {};

		countries.forEach((country) => {
			medalGrids[country] = Array.from({ length: numIncrements }, () =>
				Array.from({ length: numIncrements }, () => 0)
			);
		});

		// Compute the medalGrids
		for (let i = 0; i < numIncrements; i++) {
			for (let j = 0; j < numIncrements; j++) {
				//When i and j are 0, gold = silver = bronze
				//When i is max, gold only
				//When i 0 j 1, gold = silver but bronze = 0

				//We use the falloff curve where 0->1, 0.5->1/2, 0.75->1/4, 0.875->1/8, etc.

				let goldSilverGridpoint = i / numIncrements;
				let silverBronzeGridpoint = j / numIncrements;
				let divideSilverBronze = 1 - goldSilverGridpoint;
				let divideBronze = 1 - silverBronzeGridpoint;
				//If this is the edge, make the division .00001
				if (i === numIncrements - 1) {
					divideSilverBronze = 0.00001;
				}
				if (j === numIncrements - 1) {
					divideBronze = 0.00001;
				}

				let medalCountsCopy = JSON.parse(JSON.stringify(medalCounts));
				for (let country in medalCountsCopy) {
					let medalCount = medalCountsCopy[country];
					let gold = medalCount.gold;
					let silver = medalCount.silver;
					let bronze = medalCount.bronze;
					let goldPoints = gold;
					let silverPoints = silver * divideSilverBronze;
					let bronzePoints = bronze * divideSilverBronze * divideBronze;
					medalGrids[country][i][j] = goldPoints + silverPoints + bronzePoints;
				}
				let countryRankings: [string, number][] = countries.map((country) => [
					country,
					medalGrids[country][i][j]
				]);
				countryRankings.sort((a: [string, number], b: [string, number]) => {
					if (a[1] === b[1]) {
						return a[0] < b[0] ? -1 : 1;
					}
					return b[1] - a[1];
				});
				countryRankings.forEach((countryRanking, index) => {
					medalGrids[countryRanking[0]][i][j] = index + 1;
				});
			}
		}

		countries.forEach((country, countryIndex) => {
			let countryGrid = medalGrids[country];
			// Smooth out the grid. Pin the corners. Each value should be weighted average of values within 1/5 of the grid
			// This is a simple blur filter
			let smoothedGrid = Array.from({ length: numIncrements }, () =>
				Array.from({ length: numIncrements }, () => 0)
			);
			let radius = Math.ceil(numIncrements / 2);
			for (let i = 0; i < numIncrements; i++) {
				for (let j = 0; j < numIncrements; j++) {
					let sum = 0;
					let count = 0;
					for (let x = i - radius; x <= i + radius; x++) {
						for (let y = j - radius; y <= j + radius; y++) {
							if (x >= 0 && x < numIncrements && y >= 0 && y < numIncrements) {
								sum += countryGrid[x][y] * (1 / (Math.abs(x - i) + Math.abs(y - j) + 1) ** 3);
								count += 1 / (Math.abs(x - i) + Math.abs(y - j) + 1) ** 3;
							}
						}
					}
					smoothedGrid[i][j] = sum / count;
				}
			}
			countryGrid = smoothedGrid;

			// Create each plane and set its z values
			let geometry = new THREE.PlaneGeometry(2, 2, numIncrements - 1, numIncrements - 1);
			let vertices = geometry.attributes.position.array;
			for (let i = 0; i < numIncrements; i++) {
				for (let j = 0; j < numIncrements; j++) {
					let vertexIndex = (i * numIncrements + j) * 3;
					let z = (countryGrid[i][j] / countries.length) * 2 - 1;
					vertices[vertexIndex + 2] = -z * levelOfSeparation;
				}
			}
			geometry.attributes.position.needsUpdate = true;
			geometry.computeVertexNormals();

			// Color is flag color
			let material = new THREE.MeshBasicMaterial({
				map: new THREE.TextureLoader().load(`/flags/${country}.png`),
				wireframe: true,
				transparent: true,
				opacity: 0.5
			});
			let mesh = new THREE.Mesh(geometry, material);
			scene.add(mesh);

			let edgesGeometry = new EdgesGeometry(geometry);
			let positions = edgesGeometry.attributes.position.array;

			//Create a dummy ring of positions for the edge of the plane
			let topRow = [];
			//Top border: y = 1, x = -1 to 1
			for (let i = 0; i < numIncrements; i++) {
				let vertexIndex = (0 * numIncrements + i) * 3;
				topRow.push(vertices[vertexIndex], vertices[vertexIndex + 1], vertices[vertexIndex + 2]);
				topRow.push(vertices[vertexIndex], vertices[vertexIndex + 1], vertices[vertexIndex + 2]);
			}
			topRow = topRow.slice(3, topRow.length - 3);

			let rightRow = [];
			//Right border: x = 1, y = 1 to -1
			for (let i = 0; i < numIncrements; i++) {
				let vertexIndex = (i * numIncrements + numIncrements - 1) * 3;
				rightRow.push(vertices[vertexIndex], vertices[vertexIndex + 1], vertices[vertexIndex + 2]);
				rightRow.push(vertices[vertexIndex], vertices[vertexIndex + 1], vertices[vertexIndex + 2]);
			}
			rightRow = rightRow.slice(3, rightRow.length - 3);

			let bottomRow = [];
			//Bottom border: y = -1, x = 1 to -1
			for (let i = numIncrements - 1; i >= 0; i--) {
				let vertexIndex = ((numIncrements - 1) * numIncrements + i) * 3;
				bottomRow.push(vertices[vertexIndex], vertices[vertexIndex + 1], vertices[vertexIndex + 2]);
				bottomRow.push(vertices[vertexIndex], vertices[vertexIndex + 1], vertices[vertexIndex + 2]);
			}
			bottomRow = bottomRow.slice(3, bottomRow.length - 3);

			let leftRow = [];
			//Left border: x = -1, y = -1 to 1
			for (let i = numIncrements - 1; i >= 0; i--) {
				let vertexIndex = (i * numIncrements + 0) * 3;
				leftRow.push(vertices[vertexIndex], vertices[vertexIndex + 1], vertices[vertexIndex + 2]);
				leftRow.push(vertices[vertexIndex], vertices[vertexIndex + 1], vertices[vertexIndex + 2]);
			}
			leftRow = leftRow.slice(3, leftRow.length - 3);

			let borderPositions = [...topRow, ...rightRow, ...bottomRow, ...leftRow];

			let lineGeometry = new LineGeometry();
			lineGeometry.setPositions(borderPositions);

			let borderMaterial = new LineMaterial({
				color: medalCounts[country].borderColor,
				wireframe: false,
				linewidth: 15 // Set an initial line width
			});
			borderMaterial.resolution.set(window.innerWidth, window.innerHeight);

			let borderMesh = new LineSegments2(lineGeometry, borderMaterial);
			scene.add(borderMesh);
		});

		//Create a vertical, gold glowing line
		let top = -((1 / countries.length) * 2 - 1) * levelOfSeparation;
		let bottom = -(1 * 2 - 1) * levelOfSeparation;

		// Create material with thickness
		let goldLineMaterial = new LineMaterial({
			color: 0xffd700,
			linewidth: 10
		});

		let silverLineMaterial = new LineMaterial({
			color: 0xc0c0c0,
			linewidth: 10
		});

		let bronzeLineMaterial = new LineMaterial({
			color: 0xcd7f32,
			linewidth: 10
		});

		let goldLinePos = [
			[-1.1, 1.1],
			[1.1, 1.1],
			[-1.1, -1.1],
			[1.1, -1.1]
		];
		let silverLinePos = [
			[-1.15, 1.15],
			[1.15, 1.15]
		];
		let bronzeLinePos = [[-1.2, 1.2]];

		goldLinePos.forEach((pos) => {
			let goldLine = new Line2(
				new LineGeometry().setPositions([pos[0], pos[1], top, pos[0], pos[1], bottom]),
				goldLineMaterial
			).computeLineDistances();
			scene.add(goldLine);
		});

		silverLinePos.forEach((pos) => {
			let silverLine = new Line2(
				new LineGeometry().setPositions([pos[0], pos[1], top, pos[0], pos[1], bottom]),
				silverLineMaterial
			).computeLineDistances();
			scene.add(silverLine);
		});

		bronzeLinePos.forEach((pos) => {
			let bronzeLine = new Line2(
				new LineGeometry().setPositions([pos[0], pos[1], top, pos[0], pos[1], bottom]),
				bronzeLineMaterial
			).computeLineDistances();
			scene.add(bronzeLine);
		});

		const loader = new FontLoader();

		loader.load('Inter_Regular.json', function (font) {
			//Create each of the font objects
			let textObjs = [
				// makeText(font, 'Increasing Silver Value →', 0.12, 0xc0c0c0, [-0.9, 1.2, top], Math.PI / 2),
				// makeText(font, '← Increasing Bronze Value', 0.12, 0xcd7f32, [-1.1, 1.2, top], 0),
				// makeText(font, 'Only Gold', 0.06, 0xffd700, [-0.2, -1.2, top], 0),
				// makeText(font, 'All Count', 0.06, 0xffd700, [-0.2, 1.8, top], Math.PI / 4),
				// makeText(font, 'Gold and Silver', 0.06, 0xffd700, [-0.3, 1.7, top], -Math.PI / 4),
				makeText(
					font,
					'Increasing Silver Value →',
					0.12,
					0xc0c0c0,
					[-0.9, 1.2, bottom],
					Math.PI / 2
				),
				makeText(
					font,
					'Increasing Silver Value →',
					0.12,
					0xc0c0c0,
					[-0.9, -1.3, bottom],
					Math.PI / 2
				),
				makeText(font, '← Increasing Bronze Value', 0.12, 0xcd7f32, [-1.1, 1.2, bottom], 0),
				makeText(font, '← Increasing Bronze Value', 0.12, 0xcd7f32, [-1.1, -1.4, bottom], 0),
				makeText(font, 'Only Gold', 0.06, 0xffd700, [-0.2, -1.2, bottom], 0),
				makeText(font, 'All Count', 0.06, 0xffd700, [-0.2, 1.8, bottom], Math.PI / 4),
				makeText(font, 'Gold and Silver', 0.06, 0xffd700, [-0.3, 1.7, bottom], -Math.PI / 4)
			];

			textObjs.forEach((textObj) => {
				scene.add(textObj);
			});
		});

		function animate() {
			requestAnimationFrame(animate);
			controls.update();
			renderer.render(scene, camera);
		}

		animate();

		const resize = () => {
			renderer.setSize(window.innerWidth, window.innerHeight);
			camera.aspect = window.innerWidth / window.innerHeight;
			camera.updateProjectionMatrix();
			if (window.innerWidth < 1200) {
				displayTable = false;
			} else {
				displayTable = true;
			}
		};

		window.addEventListener('resize', resize);
	}
</script>

<!-- //position absolute so it's full screen and behind the input -->
<canvas bind:this={canvas} class="absolute inset-0"></canvas>

<div class="absolute top-10 left-10 mx-auto text-white">
	<h1 class="text-4xl text-center">Olympic Medal Standings</h1>
	<p class="text-center">Drag and zoom to move</p>
	<p class="text-center">Falloff curve puts 4-2-1 point system in the center</p>
	<div class="p-4 flex flex-col justify-normal">
		<p class="text-center">Separation</p>

		<input
			type="range"
			step=".1"
			min=".1"
			max="4"
			bind:value={levelOfSeparation}
			on:input={() => {
				createScene();
			}}
		/>
	</div>
</div>

{#if medalCounts && displayTable}
	<div class="absolute bottom-10 right-10 bg-black bg-opacity-50 text-white p-4 rounded-lg">
		<table class="table-auto">
			<thead>
				<tr>
					<th class="px-4 py-2"></th>
					<!-- <th class="px-4 py-2"></th> -->
					<th class="px-4 py-2 text-yellow-400">G</th>
					<th class="px-4 py-2 text-gray-300">S</th>
					<th class="px-4 py-2 text-orange-400">B</th>
					<th class="px-4 py-2">T</th>
				</tr>
			</thead>
			<tbody>
				{#each Object.entries(medalCounts) as [country, data]}
					<tr>
						<td class="px-4 py-2">
							<img src="/flags/{country}.png" alt="{data.name} flag" class="w-8 h-4 object-cover" />
						</td>
						<!-- <td class="px-4 py-2">{data.name}</td> -->
						<td class="px-4 py-2 text-yellow-400">{data.gold}</td>
						<td class="px-4 py-2 text-gray-300">{data.silver}</td>
						<td class="px-4 py-2 text-orange-400">{data.bronze}</td>
						<td class="px-4 py-2">{data.gold + data.silver + data.bronze}</td>
					</tr>
				{/each}
			</tbody>
		</table>
	</div>
{/if}

<style>
	table {
		border-collapse: separate;
		border-spacing: 0 4px;
	}

	th,
	td {
		text-align: center;
	}

	tr:nth-child(even) {
		background-color: rgba(255, 255, 255, 0.1);
	}

	tr:hover {
		background-color: rgba(255, 255, 255, 0.2);
	}
</style>
