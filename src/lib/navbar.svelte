<script>
    import { DiscordLogo, GithubLogo, LinkedinLogo } from "radix-icons-svelte"
    import Logo  from "$lib/logo.svelte"
</script>

<nav class = "nav-container">
    <img class='rsvp' src='/rsvp-synsets-logo_v01.svg' alt='rsvp-synsets logo'>
    <div class = 'logo'>
        <a href="https://yoctobio.me" target="_blank" rel="noopener noreferrer"><Logo alt='YOCTOBIOME LLC, OFFICIAL LOGO 2022 CE'/></a>
    </div>
    <div style="width: 43px" />
    <a href="https://discord.com/users/1002927979111911445" target="_blank" rel="noopener noreferrer"><DiscordLogo /></a>
    <a href="https://github.com/pedrovivash/rsvp-synsets" target="_blank" rel="noopener noreferrer"><GithubLogo /></a>
    <a href="https://linkedin.com/in/pedrovivash" target="_blank" rel="noopener noreferrer"><LinkedinLogo /></a>
    <div style='width:45px' />
</nav>

<style>
</style>