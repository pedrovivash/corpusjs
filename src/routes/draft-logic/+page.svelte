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
    let lexicalSearchTerms
    let synsetDefines
    let synsetsLength = 0
    let sumMembersLengths = 0
    let avgMemberLength = 10
    let wordnet
    let position = 0
    let member = ''
    let previousMember = ''
    let pos = ''
    let definition = ''

    // inputs
    let validateWPM = 60
    let desiredWPM = 60
    let wpmTransfer = 60
    let validatePosition = 0
    let desiredPosition = 0
    let positionTransfer = 0
    let stepPositionUp = false
    let stepPositionDown = false 
    let activateReverse = false
    let activateStop = false
    let previousWPM = 0
    let previousPosition = 0
    let previousReverse = false

    // search handling
    let searchTerm = ''

    // other
    let delay
    let stopCheck


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
        const x = define[positionReq];
        [member, pos, definition] = [x[0], x[1], x[2]]
    }
   
    // delay handling in async
    function createDelay(intrpt = desiredWPM) {
        intrpt == 0 ?
        delay =  new Promise(r => setTimeout(r, rsvpMeasure(member, intrpt))):
        delay =  new Promise(r => setTimeout(r, rsvpMeasure(member, intrpt)))
        
    }
    function cancelDelay() {
        clearTimeout(delay)
    }

    
    function stepPosition(gate) {
        switch (gate) {
            case 'up':
                activateStop = true
                stopCheck.checked = true
                wpmTransfer = desiredWPM
                desiredWPM = 0
                stepPositionUp = true
                
                break
            case 'down':
                activateStop = true
                stopCheck.checked = true
                wpmTransfer = desiredWPM
                desiredWPM = 0
                stepPositionDown = true
                break
        }
    }
      
    

    // calling the input
    async function gatherUserInput(positionIn, definesIn) {    

        cancelDelay()
        displayDefine(definesIn, positionIn)
        createDelay()
        await delay
        
        /* incorrect, the step cannot interrupt the display like this
        if (stepPositionUp == true || stepPositionDown == true) {
            wpmTransfer = desiredWPM
            desiredWPM = 0            
            if (stepPositionUp) {
                position++ 
                stepPositionUp = false
            }
            else if (stepPositionDown == true) { 
                position--
                stepPositionDown = false 
            }            
        }
        */
        if (activateStop == false) {
            // user input index handling
            if (positionTransfer != desiredPosition ) {
                position = desiredPosition
                positionTransfer = desiredPosition
            }
            // looping the display 
            else if (position == synsetsLength - 1) {
                activateReverse == false ?
                position = 0 :
                position = synsetsLength - 2
            }
            // forward and reverse
            else {   
                activateReverse == false ?
                position++ :
                position != 0 ?
                position-- :
                position = synsetsLength - 1
            }
        }
        // pause
        else {
            if (stepPositionUp == true) {
                position != synsetsLength - 1 ? 
                position++ :
                position = 0
                stepPositionUp = false
            }
            else if (stepPositionDown == true) {
                position != 0 ?
                position-- :
                position = synsetsLength - 1
                stepPositionDown = false
            }
            else {
                position++
                position--
            }
        }       
        
    }
    
    

    // search members
    async function searchTermReturn(term, aoaoa, synsetsArray){
        const translatePOS = {
            'n': 'noun',
            'v': 'verb',
            'a': 'adjective',
            'r': 'adverb',
            's': 'adjective'
        }
        
        function testSearch(lexicalTerm) {
           return term.trim() == lexicalTerm[0]
        }
        const searchReturn = aoaoa.filter(testSearch)
        const searchResFormat = searchReturn.map(x => [x[0], translatePOS[x[1]], x[2]])
        let realIDs = []
        
        searchResFormat.forEach(y => {
            let tempIn = []
            y[2].forEach(z => {
                function getIndex(synsetTest) {
                    return z == synsetTest[3] 
                }
                tempIn.push(synsetsArray.find(getIndex)[4])
            })
            realIDs.push(tempIn)
        })

        return searchResFormat.map((x, i) => x.concat(realIDs[i].sort((a, b)=> a - b).toString().replaceAll(',',' | '))) 
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
            aoo: array of objects
            aoaoa: array (of arrays) ** 2
            off: offset
            i: index
            gate: return value
            wpm: words per minute (target)
            avgLegnth: against what the word (param) was sourced from
        */


        // validate

        // update wordnet, RestAPI needed

        // see individual Sysnet
        function flatSynset(aoo, i) {
            return ((Object.entries(aoo[i])).flat(2).map(x => {
                return typeof(x) == 'object' ? Object.entries(x) : x
            }).flat(2))
        }
        
        // extract nested
        function getLexicals (aoo){
            return lexicals.map(x => {
                const lemma = x['Lemma']['@_writtenForm']
                const lexPOS = x['Lemma']['@_partOfSpeech']
                let synsetIDs = []
                if (x['Sense'] instanceof Array) {
                    x['Sense'].forEach(y => synsetIDs.push(y['@_synset']))
                }
                else {
                    synsetIDs.push(x['Sense']['@_synset'])
                }
                return [lemma, lexPOS, synsetIDs]
            })
        }
               
        function getSynsets(aoo) {
            let giveReturn = []
            let countSyn = 1
            aoo.forEach(x => {
                const memberArray = x['@_members'].replaceAll('-a-',"'").split(" ")
                const memberArrayTrim = memberArray.map(y => y.slice(5, y.length -2).replaceAll('_', ' '))
                for (const z of memberArrayTrim) {
                    giveReturn.push([z, x['@_lexfile'], x['Definition'], x['@_id'], countSyn])
                    countSyn++
                }
            })
            return giveReturn
        }

        // words per entry with getAllFunc gate params: see switch/case
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
        
        // see range of synsets
        function getDecaFunc(off = 0, aoo = synsets, func = flatSynset, gate = 'entries') {
            const range = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
            return range.map(x => x + off).map(y => func(aoo, y, gate))
        }

        // see all of synsets
        function getAllFunc(aoo = synsets, func = flatSynset, gate = 'entries') {
            const range = aoo.map((x, i) => i)
            return range.map(y => func(aoo, y, gate))
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
        //synsetsPerEntry = getAllFunc(synsets, getSynsetEntries, 'define')
        synsetDefines = getSynsets(synsets)
        //synsetDefines = flattenInterior(synsetsPerEntry)
        synsetDefines.forEach(x => sumMembersLengths += x[0].length)
        avgMemberLength = sumMembersLengths/synsetDefines.length
        synsetDefines.unshift(['a word', 'intro.this', 'a feature of language with often comparable meaning(s) to others like it'])
        synsetsLength = synsetDefines.length
        lexicalSearchTerms = getLexicals(lexicals)
        //console.log(lexicalSearchTerms)
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
            <progress class='this-prog' style='position:absolute; top:36px;' value={position + 1} max={synsetsLength} />
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
                <input class='bar-input' type='number' name='WPM' min='60' max='3600' on:drop={() => false} on:paste={() => false} on:change={()=> {validateWPM > 3600 ? validateWPM = 3600 : validateWPM < 60 ? validateWPM = 60 : validateWPM == null ? validateWPM = 60 : null}} bind:value={validateWPM}>
                <input type='button' on:click={()=> desiredWPM = validateWPM} value='WPM'>
            </div>
            <div style='display:flex;'>
                <input class='bar-input' type='number' name='INDEX' min='0' max={synsetsLength - 1} on:drop={()=> false} on:paste={() => false}  on:change={()=> {validatePosition > synsetsLength - 1 ? validatePosition = synsetsLength - 1 : validatePosition < 0 ? validatePosition = 0 : validatePosition == null ? validatePosition = 0 : null}} bind:value={validatePosition}>
                <input type='button' on:click={()=> desiredPosition != validatePosition ? desiredPosition = validatePosition : positionTransfer++ } value='INDEX'>
            </div>
            <div style='display:flex; flex-direction:row;'>
                <input type='button' name='STEPDOWN' on:click={()=> stepPosition('down')} value='STEP -1'>
                <input type='button' name='STEPUP' on:click={()=> stepPosition('up')} value='STEP +1'>
            </div>           
             <div style='display:flex;'>
                <input type='checkbox' name='REVERSE' on:click={(()=>(activateReverse = !activateReverse))}>
                <label for='REVERSE'>::REVERSE</label>
            </div>            
            <div style='display:flex;'>
                <input type='checkbox' name='STOP' on:click={(()=>(activateStop = !activateStop))} bind:this={stopCheck}>
                <label for='STOP'>::STOP</label>
            </div>
            <hr />
            <div style='display=flex;'>
                <input type='search' placeholder='>>' bind:value={searchTerm} name='SEARCH'>
                <label for='SEARCH'>::SEARCH</label>
            </div>            

        {/await}
        {#await searchTermReturn(searchTerm, lexicalSearchTerms, synsetDefines) then result}
            <div>
                {#each result as searchDisp}
                    <p>
                        <small>Part of speech:</small> {searchDisp[1]}
                        <br/>
                        <small>Index:</small> {searchDisp[3]} 
                    </p>
                {/each}
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