<script>
	import { themes } from '../config.js';
	import { getContext } from 'svelte';

	export let theme = getContext('theme');
</script>

<footer style="color: {themes[theme]['text']}; background-color: {themes[theme]['pale']};">
	
</footer>

<style>
	footer {
		padding-bottom: 72px;
		padding-top: 36px;
		font-size: 18px;
	}
</style>