<script>
    import fxp from "fast-xml-parser"
    import { onMount } from 'svelte';


    console.clear()

    // missing attrbibuteNamePrefix: "@_"
    const options = {
        ignoreAttributes: false,
        ignorePiTags: true
    }
    const parser = new fxp.XMLParser(options)

    
    function rsvpMeasure(word, targetWPM, avgLength = avgMemberLength) {
        const millisecond = 1000
        const avgEngWordLength = 5.2
        const avgWordLength = avgLength
        const chunkLength = word.length
        const reserveTime = (1/((targetWPM*avgLength)/60))*chunkLength
        return reserveTime*millisecond
    }
    
    // synset handling
    let synsetDefines
    let synsetsLength = 0
    let sumMembersLengths = 0
    let avgMemberLength = 10
    let position = 0
    let member = ''
    let previousMember = ''
    let pos = ''
    let definition = ''

    // inputs
    let desiredWPM = 60
    let desiredPosition = 0
    let positionTransfer = 0
    let activateReverse = false
    let previousWPM = 0
    let previousPosition = 0
    let previousReverse = false
    
    // other
    let delay

    // serves the data
    /* BUGS
    async function displayAllDefines(definesProc, wpmReq, positionReq, reverseReq) {
        let ii = 1 + positionReq
        let jCopy = 0
        let defines
        reverseReq != true ?
        defines = definesProc.slice(ii - 1) :
        defines = definesProc.slice(ii).reverse()
        for (const [i, x] of defines.entries()) {    
            for (const [j, y] of x.entries()) {
                ii++
                [position, member, pos, definition] = [ii, y[0], y[1], y[2]]
                i != 0 && j > 0 ?
                previousMember = x[j-1][0] :
                j == 0 && i >= 1 ?
                previousMember = defines[i-1][jCopy][0] :
                null
                await new Promise(r => setTimeout(r, rsvpMeasure(member, wpmReq)))
                jCopy = j
            }
        }
    }
    */

    function displayDefine(define, positionReq) {
        let x = define[positionReq];
        [position, member, pos, definition] = [positionReq, x[0], x[1], x[2]]
    }
   
    // delay handling in async
    function createDelay() {
        delay =  new Promise(r => setTimeout(r, rsvpMeasure(member, desiredWPM)))
        
    }
    function cancelDelay() {
        clearTimeout(delay)
    }

    // calling the input
    async function gatherUserInput(positionIn, definesIn) {
        //if (previousWPM != wpmIn || previousPosition != 0 || reverseIn != true ) {
        cancelDelay()
        displayDefine(definesIn, positionIn)
        createDelay()
        await delay
        if (positionTransfer != desiredPosition) {
            position = desiredPosition
            positionTransfer = desiredPosition
        }
        else if (position == synsetsLength - 1) {
            position = 0
        }
        else
        { 
            activateReverse == false ?
            position++ :
            position--
        }
        //}
        //else {
        //await displayDefine(definesIn[previousPosition], previousReverse)
        //
        //}
        //previousWPM = wpmIn
        //previousPosition = positionIn
        //previousReverse = reverseIn
    }

    // gets the data
    onMount(async () => {
        
        // fetching xml and converting > text > JSobj
        const response = await fetch('/wn.xml')
        const text = await response.text()
        const wordnet = parser.parse(text)
        const lexicals = wordnet.LexicalResource.Lexicon.LexicalEntry
        const synsets = wordnet.LexicalResource.Lexicon.Synset
        const syntax = wordnet.LexicalResource.Lexicon.SyntacticBehaviour
        let synsetsPerEntry
        /*
        legend:
            aao: array of objects
            aoaoa: array (of arrays) ** 2
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
            
            // this effectively pulls whatever you need per array within the flat parent
            // so, searching?
            if (gate != 'define') {
                toReturn = giveReturn
            }
            else {
                //console.log(giveReturn)
                const membersSplit = giveReturn[1][1].
                replaceAll('-a-',"'").
                split(" ").
                map(x => x.slice(5, x.length - 2).replaceAll('_',' '))
                membersSplit.forEach(x => {
                    toReturn.push([x,
                    giveReturn[2][1],
                    giveReturn[0][1]
                    ])}
                )
            }
            return toReturn
        }
        
        // see range of Sysnets
        function getDecaFunc(off = 0, aao = synsets, func = flatSynset, gate = 'entries') {
            const range = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
            return range.map(x => x + off).map(y => func(aao, y, gate))
        }

        // see all of Sysnets
        function getAllFunc(aao = synsets, func = flatSynset, gate = 'entries') {
            const range = aao.map((x, i) => i)
            return range.map(y => func(aao, y, gate))
        }
        
        // for an array of arrays ** 2 return a one level
        function flattenInterior(aoaoa) {
            let aoFlatA = []
            for (const x of aoaoa) {
                for (const y of x) {
                    aoFlatA.push(y)
                }
            }
            return aoFlatA
        }
        
        // function calls
        //console.log(getAllFunc(synsets, getSynsetEntries, 'keys'))
        synsetsPerEntry = getAllFunc(synsets, getSynsetEntries, 'define')
        synsetDefines = flattenInterior(synsetsPerEntry)
        synsetDefines.forEach(x => sumMembersLengths += x[0].length)
        avgMemberLength = sumMembersLengths/synsetDefines.length
        synsetsLength = synsetDefines.length
        //console.log(wordnetArray = flatSynset(synsets,0))
                
    })
    /*
        the progress bar needs to take another value when reversed or there's something wrong with position
        the array may need to be flattened for easier traversal
        end state required
        gathering user input
    */ 
</script>

<div style='display:flex; justify-content:center'>
    <main style='width:600px; height:auto; border:18px solid black; padding:18px; overflow:hidden'>
        <noscript>This page requires JavaScript.</noscript>
        {#if synsetDefines == undefined}    
            <p style='text-align:center'>
                Loading Synsets
            </p>
        {/if}
        {#await gatherUserInput(position, synsetDefines)}
            <small>{pos}</small>
            <br/>
            <progress class='this-prog' style='position:absolute; top:36px;' value={position} max={synsetsLength} />
            <div style='height:64px'>
                <h1 style='text-transform:none;text-align:center;color:black;letter-spacing:-.05rem'>
                    {member}
                </h1>
            </div>
            <p style='text-align:right'>
                <small>{position}</small>
            </p>
            <div style='height:180px'>
                <p>
                    {definition}
                </p>
            </div>
            <p/>
            <div style='display:flex;'>
                <input class='bar-input' name='WPM' type='number' bind:value={desiredWPM}>
                <label for='WPM'>WPM</label>
            </div>
            <div style='display:flex;'>
                <input class='bar-input' name='INDEX' type='number' bind:value={desiredPosition}>
                <label for='INDEX'>INDEX</label>
            </div>           
             <div style='display:flex;'>
                <input name='REVERSE' type='checkbox' on:click={(()=>(activateReverse = !activateReverse))}>
                <label for='REVERSE'>REVERSE</label>
            </div>
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

    .bar-input {
        width: 68px;

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
    small, label {
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