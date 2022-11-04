<script>
    import Nav from '$lib/navbar.svelte'
    import Footer from '$lib/footer.svelte'
    import Meta from '$lib/headMeta.svelte'
    const titleData = "Error"
    import '../styles/global.css'
    import { errorStore } from './stores'
    import { page } from '$app/stores'
    errorStore.update((x) => x = $page.status)

</script>

<Meta pageTitle = {titleData} />
<header class='nav-container'>
    <div>
        <Nav />
    </div>
</header>
<div style='display:flex; justify-content:center'>
    <main style='width:600px; height:auto; border:18px solid black; padding:18px; overflow:hidden'>
        <div style="padding-top: 36px; overflow: hidden; width: 100%; display: flex; flex-direction: column; justify-content: center; align-items:center !important;">
            <h1>{$errorStore} Error.</h1>
            <p class = 'go-home'>
                Go <a href = 'https://rsvp-synsets.vercel.app'>home</a>.
            </p>
        </div>
    </main>
</div>
<footer>
    <Footer />
</footer>

