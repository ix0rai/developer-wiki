<script>
	export let href = "";
	export let title = "";
	export let text = "";
</script>

<a {href} {title} data-sveltekit-preload-data="false">
	{text}
</a>
