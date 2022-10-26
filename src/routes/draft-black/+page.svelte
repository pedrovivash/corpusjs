<script>
    import fxp from "fast-xml-parser"
    import { onMount } from 'svelte';


    console.clear()

    //missing attrbibuteNamePrefix: "@_"
    const options = {
        ignoreAttributes: false,
        ignorePiTags: true
    }
    const parser = new fxp.XMLParser(options)

    function rsvpMeasure(word, wpm, avgLength = avgMemberLength) {
        const millisecond = 1000
        const avgEngWordLength = 5.2
        const avgWordLength = avgLength
        const chunkLength = word.length
        const reserveTime = (1/((wpm*avgWordLength)/60))*chunkLength
        return reserveTime*millisecond
    }

    let synsetDefines
    let synsetsLength = 0
    let sumMembersLengths = 0
    let avgMemberLength = 0
    let position = 0
    let member = ''
    let previousMember = ''
    let pos = ''
    let definition = ''

    async function displayDefines(defines, wpm = 500) {
        let ii = 1
        let jCopy = 0
        for (const [i, x] of defines.entries()) {    
            for (const [j, y] of x.entries()) {
                ii++
                [position, member, pos, definition] = [ii, y[0], y[1], y[2]]
                i != 0 && j > 0 ?
                previousMember = x[j-1][0] :
                j == 0 && i >= 1 ?
                previousMember = defines[i-1][jCopy][0] :
                null
                await new Promise(r => setTimeout(r, rsvpMeasure(member, wpm*2)))
                previousMember = ''
                await new Promise(r => setTimeout(r, rsvpMeasure(member, wpm*2)))
                jCopy = j
            }
        }
    }

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
            wpm: words per minute (target)
            avgLegnth: against what the word (param) was sourced from
        
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
        // gate params: see switch/case
        function getSynsetEntries(aoo, i, gate) {
            const synsetArray = flatSynset(aoo, i)
            let giveReturn = []
            let toReturn = []
            let copy = ''
            let giveReturnValues = []
            let giveReturnValue = ''
            let giveReturnKey = ''
            const keyConditions = ['Definition', 'SynsetRelation', 'Example', '@_', '#']
            const defineConditions = ['Definition', '@_members', '@_lexfile']
            switch (gate) {
                case 'keys':
                    synsetArray.filter(x => {                        
                        keyConditions.some(y => y == x ||  
                        y == x.slice(0, 2) || 
                        y == x[0]) ?
                        giveReturn.push(x) : 
                        null
                    })
                    break
                case 'values':
                    synsetArray.filter(x => {
                        keyConditions.some(y => y == x || 
                        y == x.slice(0, 2) || 
                        y == x[0]) ?
                        null : 
                        giveReturn.push(x)
                    })
                    break
                // maybe cutting something important out, like SynsetRelation had an aao and I lost it
                case 'entries':
                    synsetArray.filter(x => {
                        keyConditions.some(y => y == x ||  
                        y == x.slice(0, 2) || 
                        y == x[0]) ?
                        giveReturnKey = x : 
                        giveReturnValues.push(x) 
                        giveReturnKey != copy ?
                        [copy, giveReturnValues] = [giveReturnKey, []] :
                        giveReturn.length == 0 || 
                        giveReturn[giveReturn.length - 1][0] != giveReturnKey ?
                        giveReturn[giveReturn.length] = 
                        [giveReturnKey, giveReturnValues] :
                        null
                    })
                    break
                case 'define':
                    synsetArray.filter((x, i) => {
                        defineConditions.some(y => y == x) ?
                        giveReturnKey = x :
                        synsetArray[i - 1] == giveReturnKey ?
                        giveReturnValue = x :
                        null
                        giveReturnValue != '' ?
                        [giveReturn[giveReturn.length], giveReturnValue] = 
                        [[giveReturnKey, giveReturnValue], ''] :
                        null
                    })
            }
            
            if (gate != 'define') {
                toReturn = giveReturn
            }
            else {
                const membersSplit = giveReturn[1][1].
                replaceAll('-a-',"'").
                split(" ").
                map(x => x.slice(5, x.length - 2).replaceAll('_',' '))
                membersSplit.forEach(x => {
                    synsetsLength+=1
                    sumMembersLengths += x.length
                    toReturn.push([x,
                    giveReturn[2][1],
                    giveReturn[0][1]
                    ])}
                )
                avgMemberLength = sumMembersLengths/synsetsLength
            }
            return toReturn
        }
        
        // see range of Sysnets
        function getDecaFunc(off = 0, aao = synsets, func = flatSynset, gate = 'entries') {
            const range = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
            return range.map(x => x + off).map(y => func(aao, y, gate))
        }

        // see range of Sysnets
        function getAllFunc(aao = synsets, func = flatSynset, gate = 'entries') {
            const range = aao.map((x, i) => i)
            return range.map(y => func(aao, y, gate))
        }


        // function calls
        //console.log(getAllFunc(synsets, getSynsetEntries, 'keys'))
        synsetDefines = getAllFunc(synsets, getSynsetEntries, 'define')
        //console.log(wordnetArray = flatSynset(synsets,0))
        
        
    })
    
</script>

<div style='display:flex; justify-content:center'>
    <main style='width:600px; height:300px; border:18px solid black; padding:18px; overflow:hidden'>
        <noscript>This page requires JavaScript.</noscript>
        {#if synsetDefines == undefined}    
            <p style='text-align:center'>
                Loading Synsets
            </p>
        {/if}
        {#await displayDefines(synsetDefines,600)}
            <small>{pos}</small>
            <br/>
            <progress class=this-prog style='position:absolute; top:36px;' value={position} max={synsetsLength} />
            <div style='display:flex;flex-direction:column;align-items:center; overflow:hidden'>
                <h1 style='position:absolute;text-transform:none;color:#cccccc;letter-spacing:-.1rem;text-align: center;'>
                    {previousMember}
                </h1>
                <h1 style='position:absolute;text-transform:none;color:black;letter-spacing:-.1rem;text-align: center;'>
                    {member}
                </h1>
            </div>
            <p style='text-align:right;margin-top:64px'>
                <small>{position}</small>
            </p>
            <p>
                {definition}
            </p>
        {/await}
    </main>
</div>

<style>
    :global(*) {
        -ms-overflow-style: none; /* for Internet Explorer, Edge */
        scrollbar-width: none; /* for Firefox */
    }

    :global(*::-webkit-scrollbar) {
        display: none; /* for Chrome, Safari, and Opera */
    }

    progress {
        /* Reset the default appearance */
        -webkit-appearance: none;
        -moz-appearance:none;
        appearance: none
    }
    .this-prog {
        background-color:#dddddd;
        border: none;
        min-width: 1rem;
        max-width: 10%;
        height: 9px;
    }
    .this-prog::-webkit-progress-bar {
        background-color:#dddddd
    }
    .this-prog[value]::-moz-progress-bar {
        background-color: #80d000
    }
    .this-prog[value]::-webkit-progress-value {
        background-color: #80d000
    }

    h1 {
        color: #808080;
        font-family: 'Arial';
        font-size: 32px;
        letter-spacing: .05em;
        line-height: 1.5rem;
        font-weight: bold;
        text-transform: uppercase;
    }
    h2 {
        color: #808080;
        font-family: 'Arial';
        font-size: 20px;
        letter-spacing: .05em;
        line-height: 1.5em;
        font-weight: bold;
        text-transform: uppercase;
    }
    p {
        color: #808080;
        font-family: 'Arial';
        font-size: 18px;
        letter-spacing: .11em;
        line-height: 1.5em;
        font-weight: normal;
    }
    small {
        color: #bbbbbb;
        font-family: 'Arial';
        font-size: 10px;
        letter-spacing: .05em;
        line-height: 1.25em;
        font-weight: bold;
        text-transform: uppercase;
    }
    br {
        line-height: 24px;
    }
    hr {
        border: 9px solid black;
    }
    a {
        text-decoration: none;
    }
    a:link {
        color: #80d000
    }
    a:hover {
        background-color: black;
        color: white !important;
    }
    a:visited {
        color: black
        
    }
    a:active {
        color: #c5f17e !important;
    }
</style>