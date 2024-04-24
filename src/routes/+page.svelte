<script lang="ts">
	import { base } from '$app/paths';
</script>

<div class="min-h-screen bg-black flex flex-col items-center p-5">
	<div
		class="bg-opacity-60 shadow-lg rounded-xl p-8 w-full max-w-xl border bg-slate-900 transition-colors relative"
		id="questionBox"
	>
		<h1 id="title" class="text-4xl font-bold mb-2 text-pink-400">百 - Hyaku</h1>
		<h2 id="subTitle" class="text-md font-semibold text-purple-500 tracking-wider">
			This site allows you to learn Japanese and Esperanto vocabulary in a fun and easy way.
		</h2>
	</div>

	<div class="flex justify-center mt-8 gap-8 bg-slate-900 p-8 rounded-lg">
		<a
			href="{base}/japanese"
			class="bg-purple-600 text-white px-6 py-3 rounded-md hover:bg-pink-600 focus:outline-none focus:border-pink-700 focus:ring focus:ring-purple-400 active:bg-pink-700 transition duration-150 ease-in-out"
			>Japanese</a
		>

		<a
			href="{base}/japanese-kanji"
			class="bg-purple-600 text-white px-6 py-3 rounded-md hover:bg-pink-600 focus:outline-none focus:border-pink-700 focus:ring focus:ring-purple-400 active:bg-pink-700 transition duration-150 ease-in-out"
			>Kanji</a
		>

		<a
			href="{base}/esperanto"
			class="bg-purple-600 text-white px-6 py-3 rounded-md hover:bg-pink-600 focus:outline-none focus:border-pink-700 focus:ring focus:ring-purple-400 active:bg-pink-700 transition duration-150 ease-in-out"
			>Esperanto</a
		>
	</div>
</div>
