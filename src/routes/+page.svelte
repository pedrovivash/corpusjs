<script>
    import fxp from "fast-xml-parser"
    import { onMount } from 'svelte';
	import { each, xlink_attr } from "svelte/internal";

    console.clear()

    const options = {
        ignoreAttributes: false,
        ignorePiTags: true
    }
    const parser = new fxp.XMLParser(options)

    let wordnetArray = []

    onMount(async () => {
        
        // fetching xml and converting > text > JSobj
        const response = await fetch('/wn.xml')
        const text = await response.text()
        const wordnet = parser.parse(text)
        const lexicals = wordnet.LexicalResource.Lexicon.LexicalEntry
        const synsets = wordnet.LexicalResource.Lexicon.Synset
        const syntax = wordnet.LexicalResource.Lexicon.SyntacticBehaviour
        
        /*
        legend:
            aao: array of objects
            off: offset
            i: index
            gate: return value
        
        note: 
            * I don't have to say this to myself, but some of thease are to learn
            the LMF format.
        */

    

        // validate

        // update wordnet, RestAPI needed

        // see individual Sysnet
        function flatSynset(aoo, i) {
            return ((Object.entries(aoo[i])).flat(2).map(x => {
                return typeof(x) == 'object' ? Object.entries(x) : x
            }).flat(2))
        }
        
        // extract nested keys
        function getSynsetEntries(aoo, i, gate) {
            const synsetArray = flatSynset(aoo, i)
            let giveReturn = []
            let copy = ''
            let giveReturnValues = []
            let giveReturnKey = ''
            const keyConditions = ['Definition', 'SynsetRelation', 'Example', '@_', '#']
            switch (gate) {
                case 'keys':
                    synsetArray.filter(x => {                        
                        keyConditions.some(y => y == x ||  
                        y == x.slice(0,2) || 
                        y == x[0]) ?
                        giveReturn.push(x) : 
                        null
                    })
                    break
                case 'values':
                    synsetArray.filter(x => {
                        keyConditions.some(y => y == x || 
                        y == x.slice(0,2) || 
                        y == x[0]) ?
                        null : 
                        giveReturn.push(x)
                    })
                    break
                // maybe cutting something important out, like SynsetRelation had an aao and I lost it
                case 'entries':
                    synsetArray.filter(x => {
                        keyConditions.some(y => y == x ||  
                        y == x.slice(0,2) || 
                        y == x[0]) ?
                        giveReturnKey = x : 
                        giveReturnValues.push(x) 
                        giveReturnKey != copy ?
                        [copy, giveReturnValues] = [giveReturnKey, []] :
                        giveReturn.length == 0 || giveReturn[giveReturn.length - 1][0] != giveReturnKey ?
                        giveReturn[giveReturn.length] = [giveReturnKey, giveReturnValues] :
                        null
                    })
                    break
            }
            return giveReturn
        }
        
        // see range of Sysnets
        function getDecaFunc(off = 0, aao = synsets, func = flatSynset, gate = 'entries') {
            const range = [0,1,2,3,4,5,6,7,8,9]
            return range.map(x => x + off).map(y => func(aao, y, gate))
        }

        // see range of Sysnets
        function getAllFunc(aao = synsets, func = flatSynset, gate = 'entries') {
            const range = aao.map((x,i) => i)
            return range.map(y => func(aao, y, gate))
        }


        // function calls
        //console.log(getDecaFunc(50, synsets, getSynsetEntries, 'keys'))
        console.log(wordnetArray = getAllFunc(synsets, getSynsetEntries, 'values'))
        //console.log(synsets[0])

    })
    
</script>

{#each wordnetArray as x}
    <p>
        {(x)}
    </p>
{/each}
