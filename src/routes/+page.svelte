<script>
    import fxp from "fast-xml-parser"
    import { onMount } from 'svelte'
    import '../styles/global.css'
    import {MagnifyingGlass, Gear} from 'radix-icons-svelte'

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
    const lexPartList = [
        'adj.all',
        'adj.pert',
        'adj.ppl',
        'adv.all',
        'noun.act',
        'noun.animal',
        'noun.artifact',
        'noun.attribute',
        'noun.body',
        'noun.cognition',
        'noun.communication',
        'noun.event',
        'noun.feeling',
        'noun.food',
        'noun.group',
        'noun.location',
        'noun.motive',
        'noun.object',
        'noun.person',
        'noun.phenomenon',
        'noun.plant',
        'noun.possession',
        'noun.process',
        'noun.quantity',
        'noun.relation',
        'noun.shape',
        'noun.state',
        'noun.substance',
        'noun.time',
        'noun.Tops',
        'verb.body',
        'verb.change',
        'verb.cognition',
        'verb.communication',
        'verb.competition',
        'verb.consumption',
        'verb.contact',
        'verb.creation',
        'verb.emotion',
        'verb.motion',
        'verb.perception',
        'verb.possession',
        'verb.social',
        'verb.stative',
        'verb.weather']
    let lexPartArray = []    
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
    let displayChapter = 'none'
    let displayControls = 'none'
    let displaySettings = 'none'
    let displaySearch = 'none'
    let isResults = false

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
    function settingsDisplay() {
        displaySettings != 'none' ?
        displaySettings = 'none' :
        displaySettings = 'flex'
    }

    function chaptersDisplay(){
        displayChapter != 'none' ?
        displayChapter = 'none' :
        displayChapter = 'flex'
    }

    function searchDisplayOn(){
        displaySearch = 'initial'
    }
    function searchDisplayOff(){
        isSearch ?
        null :
        displaySearch = 'none'
    }

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
                stepPositionUp = true 
                break
            case 'down':
                activateStop = true
                stopCheck.checked = true
                stepPositionDown = true
                break
        }
    }
      
    function chapterSwitch(chapterIndex) {
        if (activateStop != false) {
            activateStop = false
            stopCheck.checked = false
            desiredPosition = chapterIndex
            displayChapter = 'none'
        }
        else {
            desiredPosition = chapterIndex
            displayChapter = 'none'
        }
    }
    function searchSwitch(sindex) {
        const sindexInt = parseInt(sindex)
        if (activateStop != false) {
            activateStop = false
            stopCheck.checked = false
            desiredPosition = sindexInt
            displaySearch = 'none'
        }
        else {
            desiredPosition = sindexInt
            displaySearch = 'none'
        }
    }

    function fillDesiredPosition(aPosition) {
        if (activateStop != false) {
            activateStop = false
            stopCheck.checked = false
            desiredPosition != aPosition ? 
            desiredPosition = aPosition : 
            positionTransfer++
        }
        else {
            desiredPosition != aPosition ? 
            desiredPosition = aPosition : 
            positionTransfer++
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

        const giveReturn = searchResFormat.map((x, i) => x.concat(realIDs[i].sort((a, b)=> a - b).toString().replaceAll(',',' | '))) 
        return giveReturn.map(x => [x[1],x[3].split(' | ')])
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
                const memberArray = x['@_members'].replaceAll('-ap-',"'").split(" ")
                const memberArrayTrim = memberArray.map(y => y.slice(5, y.length -2).replaceAll('_', ' '))
                for (const z of memberArrayTrim) {
                    giveReturn.push([z, x['@_lexfile'], x['Definition'], x['@_id'], countSyn])
                    countSyn++
                }
            })
            giveReturn > 0 ?
            isSearch = true :
            null
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
                replaceAll('-ap-',"'").
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
        lexPartArray = lexPartList.map(lexGroup=> {
            function testLexPart(synsetArray) {
                return synsetArray[1] == lexGroup
            }
            return [synsetDefines.find(testLexPart)[4], lexGroup]
        })
        
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
            <div style='cursor:pointer;display:flex;flex-direction: row;justify-content: flex-start;align-items:center'>
                <div class='settings-padding' style='' />
                <label for='SETTINGS:'>SETTINGS:</label>
                <button class='settings' name='SETTINGS' on:click={()=>settingsDisplay()}>
                    <Gear style='color:#80d000;width:12px;height:12px'/>
                </button>         
            </div>
            <div class='controls' style='display:{displaySettings}'>
                <div class='controls-padding' style='' />
                <div class='inputs' style=''>
                    <div style='display:flex;'>
                        <input type='number' name='WPM' min='60' max='3600' on:drop={() => false} on:paste={() => false} on:change={()=> {validateWPM > 3600 ? validateWPM = 3600 : validateWPM < 60 ? validateWPM = 60 : validateWPM == null ? validateWPM = 60 : null}} bind:value={validateWPM}>
                        <input type='button' class='desired-button' on:click={()=> desiredWPM = validateWPM} value='WPM'>
                    </div>
                    <div style='display:flex;'>
                        <input type='number' name='INDEX' min='0' max={synsetsLength - 1} on:drop={()=> false} on:paste={() => false}  on:change={()=> {validatePosition > synsetsLength - 1 ? validatePosition = synsetsLength - 1 : validatePosition < 0 ? validatePosition = 0 : validatePosition == null ? validatePosition = 0 : null}} bind:value={validatePosition}>
                        <input type='button' class='desired-button' on:click={()=> fillDesiredPosition(validatePosition) } value='INDEX'>
                    </div>            
                    <div style='display:flex; flex-direction:row;'>
                        <input type='button' class='step-button' name='STEPDOWN' on:click={()=> stepPosition('down')} value='STEP -1'>
                        <input type='button' class='step-button' name='STEPUP' on:click={()=> stepPosition('up')} value='STEP +1'>
                    </div>    
                    <button class='dropbtn' on:click={()=> chaptersDisplay()} name='partofSpeech'>CHAPTERS<div class='arrow' /></button>
                    <div class='dropdown-container' style='display:{displayChapter}'>
                        {#each lexPartArray as lexPart}
                            <button class='dropdown-content' style='' on:click={()=> chapterSwitch(lexPart[0])}>{lexPart[1]}</button>
                        {/each}
                    </div>
                    <div style='display:flex; flex-direction:row; justify-content:space-between; width: 148px; align-items:center'>
                        <label for='REVERSE'>REVERSE:</label> 
                        <input type='checkbox' name='REVERSE' on:click={(()=>(activateReverse = !activateReverse))}>
                        <label for='STOP'>STOP:</label> 
                        <input type='checkbox' name='STOP' on:click={(()=>(activateStop = !activateStop))} bind:this={stopCheck}>
                    </div>
                    <div style='display=flex; flex-direction:row; justify-content:space-between; width: 148px; align-items:center'>
                        <MagnifyingGlass style='top:3px; position:relative; color:#bbbbbb' />
                        <input type='search' class='search-input' placeholder='>>' on:focus={()=> searchDisplayOn()} on:blur={()=> searchDisplayOff()} bind:value={searchTerm} name='SEARCH'>
                    </div>
                    <div class='search-results-container' style='display:{displaySearch}'>
                        {#await searchTermReturn(searchTerm, lexicalSearchTerms, synsetDefines) then result}
                                {#each result as searchDisp}
                                    <label for='memberIndex'>{searchDisp[0]}</label>
                                    <div name='memberIndex' class='member-index'>
                                        {#each searchDisp[1] as searchIndex}
                                            <button class='search-contents' on:click={()=> searchSwitch(searchIndex)}>@{searchIndex}</button>
                                        {/each}
                                    </div>
                                {/each}
                        {/await}
                    </div>
                </div>
                <div class='controls-padding' style='' />
            </div>
            <div class='this-prog-containter'>
                <progress class='this-prog' style='' value={position + 1} max={synsetsLength} />
            </div>
            <p>
                <small>{pos}</small>
            </p>
            <div style='height:76px;background-color:black;text-align:center;padding:10px 0px 0px 0px;'>
                <h1 style='text-transform:none;color:white;line-height:1rem;letter-spacing:-.05rem;font-size:20px'>
                    {member}
                </h1>
            </div>
                <p style='text-align:right'>
                    <small>{position}</small>
                </p>
            <div style='min-height:236px'>
                <p>
                    {definition}
                </p>
            </div>


        {/await}

    </main>
</div>

<style>
    progress {
        /* Reset the default appearance */
        -webkit-appearance: none;
        -moz-appearance:none;
        appearance: none
    }
    .this-prog {
        background-color:#dddddd;
        border: none;
        width: 149px;
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
    .this-prog-container {
        border:none;
    }

    .search-input {
        height: 16px;
        width: 129px;
        color: #80d000;
        margin: 0px 0px 4px 0px;
        box-sizing: border-box;
        font-family: 'Arial';
        font-size: 10px;
        letter-spacing: .05em;
        line-height: 1.25em;
        font-weight: bold;
        text-transform: none;        
        background-color: white;
        border-radius: none;
        outline: #80d000 solid 1px;
        outline-offset: none;
        border-style: none;
    }
    
    .search-results-container {
        outline: #eeeeee solid 1px;
        width: 148px;
        max-height: 135px;
        background-color: white;
        box-shadow: 0px 2px 8px rgba(0, 0, 0, .2);
        position:absolute;
        overflow-y: scroll;
        z-index: 1;
    }
    .member-index {
        display: flex;
        flex-direction: column;
        justify-content: left;
    }
    .search-contents {
        text-align: left;
        padding: 0px 2px;
        box-sizing: border-box;
        color: #80d000;
        font-family: 'Arial';
        font-size: 10px;
        letter-spacing: .05em;
        font-weight: bold;
        text-transform: uppercase;        
        background-color: white;
        border-radius: none;
        outline-offset: none;
        border-style: none;
        cursor: pointer;
    }
    .search-contents:hover {
        color:white;
        background-color: black;
    }
    ::placeholder {
        opacity: 1.0;
        color: #80d000
    }
    input[type=number]::-webkit-inner-spin-button, 
    input[type=number]::-webkit-outer-spin-button { 
        -webkit-appearance: none; 
        margin: 0; 
    }
    input[type=number] {
        -moz-appearance:textfield;
    }  
    input[type=number] { 
        height: 16px;
        max-width:72px;
        min-width:72px;
        margin: 0px 4px 4px 0px;
        box-sizing: border-box;
        color: #80d000;
        font-family: 'Arial';
        font-size: 10px;
        letter-spacing: .05em;
        line-height: 1.25em;
        font-weight: bold;
        text-transform: uppercase;        
        background-color: white;
        border-radius: none;
        outline: #80d000 solid 1px;
        outline-offset: none;
        border-style: none;
    }
    .desired-button {
        height: 16px;
        min-width:72px;
        margin: 0px 4px 4px 0px;
        box-sizing: border-box;
        color: #80d000;
        font-family: 'Arial';
        font-size: 10px;
        letter-spacing: .05em;
        line-height: 1.25em;
        font-weight: bold;
        text-transform: uppercase;
        background-color: white;
        border-radius: none;
        outline: #80d000 solid 1px;
        outline-offset: none;
        border-style: none;
        cursor: pointer;
    }
    .step-button {
        height: 16px;
        min-width:72px;
        margin: 0px 4px 4px 0px;
        box-sizing: border-box;
        color: #80d000;
        font-family: 'Arial';
        font-size: 10px;
        letter-spacing: .05em;
        line-height: 1.25em;
        font-weight: bold;
        text-transform: uppercase;        
        background-color: white;
        border-radius: none;
        outline: #80d000 solid 1px;
        outline-offset: none;
        border-style: none;
        cursor: pointer;
    }
    .dropbtn {
        display:flex;
        flex-direction: row;
        justify-content: space-between;
        align-items: center;
        height: 16px;
        width: 148px;
        margin: 0px 4px 4px 0px;
        padding: 0px 16px 0px 15px;
        box-sizing: border-box;
        color: #80d000;
        font-family: 'Arial';
        font-size: 10px;
        letter-spacing: .05em;
        line-height: 1.25em;
        font-weight: bold;
        text-transform: uppercase;        
        background-color: white;
        border-radius: none;
        outline: #80d000 solid 1px;
        outline-offset: none;
        border-style: none;
        cursor: pointer;
    }
    .dropdown-container {
        flex-direction: column;
        outline: #eeeeee solid 1px;
        width: 148px;
        height: 135px;
        box-shadow: 0px 2px 8px rgba(0, 0, 0, .2);
        position:absolute;
        overflow-y: scroll;
        z-index: 1;
    }
   
    .dropdown-content {
        text-align: left;
        padding: 0px 2px;
        box-sizing: border-box;
        color: #80d000;
        font-family: 'Arial';
        font-size: 10px;
        letter-spacing: .05em;
        font-weight: bold;
        text-transform: uppercase;        
        background-color: white;
        border-radius: none;
        outline-offset: none;
        border-style: none;
        cursor: pointer;
    }
    .dropdown-content:hover {
        color:white;
        background-color: black;
    }
    .arrow {
        content: "";
        width: 0.8em;
        height: 0.5em;
        background-color: #80d000;
        clip-path: polygon(100% 0%, 0 0%, 50% 100%);

        
    }
    input[type="checkbox"] {
        -webkit-appearance: none !important;
        -moz-appearance: none !important;
        -ms-appearance: none !important;
        -o-appearance: none !important;
        appearance: none !important;
        background-color: white;
        border: 1px solid #80d000;
        padding: 6px;
        border-radius: none;
        align-items: center;

    }
    input[type='checkbox']:checked {
        display:grid;
        grid-template-areas: "select";
        padding: 4px;
    }
    input[type='checkbox']::after {
        content:"";
        width: 4px;
        height: 4px;
        clip-path: polygon(20% 0%, 0% 20%, 30% 50%, 0% 80%, 20% 100%, 50% 70%, 80% 100%, 100% 80%, 70% 50%, 100% 20%, 80% 0%, 50% 30%);
        background-color:#80d000;
        grid-area: select
    }
    .controls {
        display:none;
        flex-direction: row;
        justify-content: flex-start;

    }
    .settings {
        align-items: center;
        height: 16px;
        width: 16x;
        margin: 0px 0px 4px 4px;
        padding: 2px;
        box-sizing: border-box;
        color: #80d000;
        font-family: 'Arial';
        font-size: 10px;
        letter-spacing: .05em;
        line-height: 1.25em;
        font-weight: bold;
        text-transform: uppercase;        
        background-color: white;
        border-radius: none;
        outline: #80d000 solid 1px;
        outline-offset: none;
        border-style: none;
        cursor: pointer;
    }
    .controls-padding {
        width:225px;
        min-width:0%
    }
    .inputs {
        border:none;
    }
    .settings-padding {
        width:100%;
        min-width: 70px
    }

</style>