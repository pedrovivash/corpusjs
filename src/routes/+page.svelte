<script>
    import fxp from "fast-xml-parser"
    import { onMount } from 'svelte';

    console.clear()

    const parser = new fxp.XMLParser()
    
    onMount(async () => {
        const response = await fetch('/wn.xml')
        const text = await response.text()
        const wordnet = parser.parse(text)
        console.log(wordnet)
    })

    
</script>