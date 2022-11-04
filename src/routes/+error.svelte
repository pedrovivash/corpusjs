<script>
    import Meta from '$lib/headMeta.svelte'
    const titleData = "Error"
    import '../styles/global.css'
    import { errorStore } from './stores'
    import { page } from '$app/stores'
    errorStore.update((x) => x = $page.status)

</script>

<Meta pageTitle = {titleData} />

<div style="padding-top: 36px; overflow: hidden; width: 100%; display: flex; flex-direction: column; justify-content: center; align-items:center !important;">
    <h1>{$errorStore} Error.</h1>
    <p class = 'go-home'>
        Go <a href = 'https://yoctobio.me'>home</a>.
    </p>
</div>
