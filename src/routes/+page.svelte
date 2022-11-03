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
    let previousPosition = -1
    let previousReverse = false

    // search handling
    let searchTerm = ''
    let displayChapter = 'none'
    let displayControls = 'none'
    let displaySettings = 'none'
    let displaySearch = 'none'
    let isResults = false
    let isChapters = false 
    let isChaptersUp = false
    let isIndex = false
    let isResultsUp = false

    // other
    let delay
    let stopCheck
    let searchCheck
    let chaptersCheck
    let restop = false
    let loaderIndex = 1

    const loaderRange = Array(65).fill('x',0,65).map( (x, i )=> [i+1,(i+1).toString().padStart(2, '0')])
    
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

    function chaptersDisplayOn(){
        displayChapter = 'flex'
    }

    function chaptersDisplayOff(){
        isChapters ?
        null :
        [displayChapter, isChapters] = ['none', false]
    }
    function chaptershMouseOff(){
        displayChapter = 'none'
        isChapters = false
        chaptersCheck.blur()
    }
    
    function searchDisplayOn(){
        displaySearch = 'initial'
    }
    function searchDisplayOff(){
        isResults  ?
        null :
        [displaySearch, isResults] = ['none', false]
        
    }
    function searchMouseOff(){
        displaySearch = 'none'
        isResults = false
        searchCheck.blur()
    }

    function displayDefine(define, positionReq) {
        const x = define[positionReq];
        [member, pos, definition] = [x[0], x[1], x[2]]
    }
   
    // delay handling in async
    function createDelay(intrpt = desiredWPM) {
        //intrpt == 0 ?
        //delay =  new Promise(r => setTimeout(r, rsvpMeasure(member, intrpt))):
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
            isResultsUp != true ?
            null :
            positionTransfer++
            desiredPosition = chapterIndex 
            displaySearch = 'none'}
        else {
            positionTransfer++
            desiredPosition = chapterIndex 
            displaySearch = 'none'
        }
        desiredPosition == positionTransfer ?
        positionTransfer++ :
        null       
        restop = true
        isChapters = false
        isChaptersUp = true
    }

    function searchSwitch(sindex) {
        const sindexInt = sindex[0]
        if (activateStop != false) {
            activateStop = false
            stopCheck.checked = false
            isResultsUp != true ?
            null :
            positionTransfer++
            desiredPosition = sindexInt 
            displaySearch = 'none'}
        else {
            positionTransfer++
            desiredPosition = sindexInt 
            displaySearch = 'none'
        }
        desiredPosition == positionTransfer ?
        positionTransfer++ :
        null       
        restop = true
        isResults = false
        isResultsUp = true
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
        restop = true
        isIndex = true
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
            if (position == desiredPosition && restop == true) {
                position++
                position--
                isIndex ?
                [isIndex, positionTransfer] = [false, desiredPosition] :
                isResultsUp ?
                [isResultsUp, positionTransfer] = [false, desiredPosition] :
                isChaptersUp ?
                [isChaptersUp, positionTransfer] = [false, desiredPosition] :
                null ; 
                [activateStop, restop, stopCheck.checked] = [true, false, true]
            }
            else if (positionTransfer != desiredPosition && position != desiredPosition) {
                position = desiredPosition
                positionTransfer = desiredPosition
                if (restop == true) {
                    [activateStop, restop, stopCheck.checked] = [true, false, true]
                }
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
    
    // loader animate

    async function animateLoader(frameIn) {    

        cancelDelay()
        createDelay(60)
        await delay
        let frame = frameIn
        frame == 2 ?
        frame = 1 :
        frame++
        loaderIndex = frame

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
            y[2].forEach( (z) => {
                function getIndex(synsetTest) {                    
                    return z == synsetTest[3] 
                }
                const count = synsetsArray.filter(getIndex).length
                const idToID = synsetsArray.find(getIndex)[4]
                tempIn.push([idToID, (idToID+count-1)])
            })
            realIDs.push(tempIn)
        })
        const giveReturn = searchResFormat.map((x, i) => x.concat([realIDs[i].sort((a, b)=> a[0] - b[0])]))
        return giveReturn.map(x => [x[1],x[3]])
    }

    function formatIndecesRange(aon) {
        let tempString = ''
        aon[0] == aon[1] ?
        tempString = aon[0].toString() :
        tempString = aon[0].toString()+' - '+aon[1].toString()
        return tempString
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
   
    function getLargest(synsets){
        let members = [] 
        synsets.forEach(x=> members.push(x[0]))
        let max = members[0].length
        
        members.map(y=> max = Math.max(max, y.length))
        console.log(max)
        let longest = members.filter(z=> z.length == max)
        console.log(longest)
        return longest
    }
    function getLargest2(synsets){
        let members = [] 
        synsets.forEach(x=> members.push(x[2]))
        let max = members[0].length
        
        members.map(y=> max = Math.max(max, y.length))
        console.log(max)
        let longest = members.filter(z=> z.length == max)
        console.log(longest)
        return longest
    }    
    function getLargest3(synsets){
        let members = [] 
        synsets.forEach(x=> members.push(x[0].split(' ')))
        members = members.flat()
        members = members.map(a=> a.split('-')).flat()
        let max = members[0].length
        
        members.map(y=> {
            y.length < 32 ?
            max = Math.max(max, y.length) :
            null
        })
        console.log(max)
        let longest = members.filter(z=> z.length == max)
        console.log(longest)
        return longest
    }
</script>

<div style='display:flex; justify-content:center'>
    <main style='width:600px; height:auto; border:18px solid black; padding:18px; overflow:hidden'>
        <noscript><div style='text-align:center; padding:0px 0px 20px 0px'>This page requires JavaScript.</div></noscript>    
        {#if synsetDefines == undefined}                
            <div class='loading-anim' style='display:flex; width:100%; height:64px; justify-content:center;'>
                <div class=loading-img-container style='display:flex;justify-content:center;position:absolute;width:96px;height:64px;'>
                    {#each loaderRange as frameIndex}
                        <img class='loading-img' id='loading-frame-{frameIndex[0]}' style='' src='rsvp-synsets-loader/rsvp-synsets-loader_{frameIndex[1]}.svg' alt='animation of letters per frame at 60hz'>
                    {/each}
                 </div>
            </div>   
            <div style='display:flex;flex-direction:column;align-items:center'>   
                <p class='loading' style='text-align:center'>
                    Loading Synsets
                </p>
            </div>
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
                    <button class='dropbtn' on:focus={()=> chaptersDisplayOn()} on:blur={()=> chaptersDisplayOff()} bind:this={chaptersCheck} name='partofSpeech'>CHAPTERS<div class='arrow' /></button>
                    <div class='dropdown-container' on:pointerenter={()=> isChapters = true} on:mouseleave={()=> chaptershMouseOff()} style='display:{displayChapter}'>
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
                        <input type='search' class='search-input' autocomplete='off' placeholder='>>' on:focus={()=> searchDisplayOn()} on:blur={()=> searchDisplayOff()} bind:this={searchCheck} bind:value={searchTerm} name='SEARCH'>
                    </div>
                    <div class='search-results-container' on:pointerenter={()=> isResults=true} on:mouseleave={()=> searchMouseOff()} style='display:{displaySearch}'>
                        {#await searchTermReturn(searchTerm, lexicalSearchTerms, synsetDefines) then result}
                            {#if result.length > 0} 
                                {#each result as searchDisp}
                                    <label for='memberIndex'>{searchDisp[0]}</label>
                                    <div name='memberIndex' class='member-index'>
                                        {#each searchDisp[1] as searchIndex}
                                            <button class='search-contents' on:click={()=> searchSwitch(searchIndex)}>@{formatIndecesRange(searchIndex)}</button>
                                        {/each}
                                    </div>
                                {/each}
                            {/if}
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
            <div class='member-container' >
                <h1 class='member' style=''>
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
        background-color: black
    }
    .this-prog[value]::-webkit-progress-value {
        background-color: black
    }
    .this-prog-container {
        border:none;
    }

    ::-webkit-search-cancel-button,
    ::-webkit-search-decoration,
    ::-webkit-search-results-decoration,
    ::-webkit-search-results-decoration {
        all:unset
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

    .settings:hover,
    input[type=number]:hover,
    .desired-button:hover,
    .step-button:hover,
    .dropbtn:hover,
    input[type=checkbox]:hover,
    .search-input:hover {
        outline: #c5f17e solid 3px;
        outline-offset: -1px;
    }

    .settings:focus,
    input[type=number]:focus,
    .desired-button:focus,
    .step-button:focus,
    .dropbtn:focus,
    input[type=checkbox]:focus,
    .search-input:focus {
        outline: #c5f17e solid 3px;
        outline-offset: -1px;
    }
    @keyframes animateFrames {
        0%    { opacity: 0; }
        96%    { opacity: 0;  }
        99%    { opacity: 1; } 
        100%  { opacity: 0; }
    }
    @keyframes fade {
        0%    { opacity: 0; }
        100%    { opacity: 1; }

    }
    .loading {
        animation: fade ease-in-out 1s infinite alternate;
    }

    .loading-img {
        opacity: 0;
        position:absolute;
        height:100%;
        width:100%;
        filter:brightness(1.1);
        z-index: -1;
        
    }

    @media only screen and (max-width: 98px) {

        .loading-img-container {
            max-width: 48px;
            max-height: 32px
        }
        .loading-anim {
            max-height: 32px
        }

    } 

    @media only screen and (max-width: 50px) {

        .loading-img-container {
            max-width: 48px;
            max-height: 32px
        }
        .loading-anim {
            max-height: 32px
        }

    }

    #loading-frame-1 {
        animation: animateFrames 4800ms steps(2, jump-none) 0s infinite reverse;
    }
    #loading-frame-2 {
        animation: animateFrames 4800ms steps(2, jump-none) 75ms infinite reverse;
    }
    #loading-frame-3 {
        animation: animateFrames 4800ms steps(2, jump-none) 150ms infinite reverse;
    }
    #loading-frame-4 {
        animation: animateFrames 4800ms steps(2, jump-none) 225ms infinite reverse;
    }
    #loading-frame-5 {
        animation: animateFrames 4800ms steps(2, jump-none) 300ms infinite reverse;
    }
    #loading-frame-6 {
        animation: animateFrames 4800ms steps(2, jump-none) 375ms infinite reverse;
    }
    #loading-frame-7 {
        animation: animateFrames 4800ms steps(2, jump-none) 450ms infinite reverse;
    }
    #loading-frame-8 {
        animation: animateFrames 4800ms steps(2, jump-none) 525ms infinite reverse;
    }
    #loading-frame-9 {
        animation: animateFrames 4800ms steps(2, jump-none) 600ms infinite reverse;
    }
    #loading-frame-10 {
        animation: animateFrames 4800ms steps(2, jump-none) 675ms infinite reverse;
    }
    #loading-frame-11 {
        animation: animateFrames 4800ms steps(2, jump-none) 750ms infinite reverse;
    }
    #loading-frame-12 {
        animation: animateFrames 4800ms steps(2, jump-none) 825ms infinite reverse;
    }
    #loading-frame-13 {
        animation: animateFrames 4800ms steps(2, jump-none) 900ms infinite reverse;
    }
    #loading-frame-14 {
        animation: animateFrames 4800ms steps(2, jump-none) 975ms infinite reverse;
    }
    #loading-frame-15 {
        animation: animateFrames 4800ms steps(2, jump-none) 1050ms infinite reverse;
    }
    #loading-frame-16 {
        animation: animateFrames 4800ms steps(2, jump-none) 1125ms infinite reverse;
    }
    #loading-frame-17 {
        animation: animateFrames 4800ms steps(2, jump-none) 1200ms infinite reverse;
    }
    #loading-frame-18 {
        animation: animateFrames 4800ms steps(2, jump-none) 1275ms infinite reverse;
    }
    #loading-frame-19 {
        animation: animateFrames 4800ms steps(2, jump-none) 1350ms infinite reverse;
    }
    #loading-frame-20 {
        animation: animateFrames 4800ms steps(2, jump-none) 1425ms infinite reverse;
    }
    #loading-frame-21 {
        animation: animateFrames 4800ms steps(2, jump-none) 1500ms infinite reverse;
    }
    #loading-frame-22 {
        animation: animateFrames 4800ms steps(2, jump-none) 1575ms infinite reverse;
    }
    #loading-frame-23 {
        animation: animateFrames 4800ms steps(2, jump-none) 1650ms infinite reverse;
    }
    #loading-frame-24 {
        animation: animateFrames 4800ms steps(2, jump-none) 1725ms infinite reverse;
    }
    #loading-frame-25 {
        animation: animateFrames 4800ms steps(2, jump-none) 1800ms infinite reverse;
    }
    #loading-frame-26 {
        animation: animateFrames 4800ms steps(2, jump-none) 1875ms infinite reverse;
    }
    #loading-frame-27 {
        animation: animateFrames 4800ms steps(2, jump-none) 1950ms infinite reverse;
    }
    #loading-frame-28 {
        animation: animateFrames 4800ms steps(2, jump-none) 2025ms infinite reverse;
    }
    #loading-frame-29 {
        animation: animateFrames 4800ms steps(2, jump-none) 2100ms infinite reverse;
    }
    #loading-frame-30 {
        animation: animateFrames 4800ms steps(2, jump-none) 2175ms infinite reverse;
    }
    #loading-frame-31 {
        animation: animateFrames 4800ms steps(2, jump-none) 2250ms infinite reverse;
    }
    #loading-frame-32 {
        animation: animateFrames 4800ms steps(2, jump-none) 2325ms infinite reverse;
    }
    #loading-frame-33 {
        animation: animateFrames 4800ms steps(2, jump-none) 2400ms infinite reverse;
    }
    #loading-frame-34 {
        animation: animateFrames 4800ms steps(2, jump-none) 2475ms infinite reverse;
    }
    #loading-frame-35 {
        animation: animateFrames 4800ms steps(2, jump-none) 2550ms infinite reverse;
    }
    #loading-frame-36 {
        animation: animateFrames 4800ms steps(2, jump-none) 2625ms infinite reverse;
    }
    #loading-frame-37 {
        animation: animateFrames 4800ms steps(2, jump-none) 2700ms infinite reverse;
    }
    #loading-frame-38 {
        animation: animateFrames 4800ms steps(2, jump-none) 2775ms infinite reverse;
    }
    #loading-frame-39 {
        animation: animateFrames 4800ms steps(2, jump-none) 2850ms infinite reverse;
    }
    #loading-frame-40 {
        animation: animateFrames 4800ms steps(2, jump-none) 2925ms infinite reverse;
    }
    #loading-frame-41 {
        animation: animateFrames 4800ms steps(2, jump-none) 3000ms infinite reverse;
    }
    #loading-frame-42 {
        animation: animateFrames 4800ms steps(2, jump-none) 3075ms infinite reverse;
    }
    #loading-frame-43 {
        animation: animateFrames 4800ms steps(2, jump-none) 3150ms infinite reverse;
    }
    #loading-frame-44 {
        animation: animateFrames 4800ms steps(2, jump-none) 3225ms infinite reverse;
    }
    #loading-frame-45 {
        animation: animateFrames 4800ms steps(2, jump-none) 3300ms infinite reverse;
    }
    #loading-frame-46 {
        animation: animateFrames 4800ms steps(2, jump-none) 3375ms infinite reverse;
    }
    #loading-frame-47 {
        animation: animateFrames 4800ms steps(2, jump-none) 3450ms infinite reverse;
    }
    #loading-frame-48 {
        animation: animateFrames 4800ms steps(2, jump-none) 3525ms infinite reverse;
    }
    #loading-frame-49 {
        animation: animateFrames 4800ms steps(2, jump-none) 3600ms infinite reverse;
    }
    #loading-frame-50 {
        animation: animateFrames 4800ms steps(2, jump-none) 3675ms infinite reverse;
    }
    #loading-frame-51 {
        animation: animateFrames 4800ms steps(2, jump-none) 3750ms infinite reverse;
    }
    #loading-frame-52 {
        animation: animateFrames 4800ms steps(2, jump-none) 3825ms infinite reverse;
    }
    #loading-frame-53 {
        animation: animateFrames 4800ms steps(2, jump-none) 3900ms infinite reverse;
    }
    #loading-frame-54 {
        animation: animateFrames 4800ms steps(2, jump-none) 3975ms infinite reverse;
    }
    #loading-frame-55 {
        animation: animateFrames 4800ms steps(2, jump-none) 4050ms infinite reverse;
    }
    #loading-frame-56 {
        animation: animateFrames 4800ms steps(2, jump-none) 4125ms infinite reverse;
    }
    #loading-frame-57 {
        animation: animateFrames 4800ms steps(2, jump-none) 4200ms infinite reverse;
    }
    #loading-frame-58 {
        animation: animateFrames 4800ms steps(2, jump-none) 4275ms infinite reverse;
    }
    #loading-frame-59 {
        animation: animateFrames 4800ms steps(2, jump-none) 4350ms infinite reverse;
    }
    #loading-frame-60 {
        animation: animateFrames 4800ms steps(2, jump-none) 4425ms infinite reverse;
    }
    #loading-frame-61 {
        animation: animateFrames 4800ms steps(2, jump-none) 4500ms infinite reverse;
    }
    #loading-frame-62 {
        animation: animateFrames 4800ms steps(2, jump-none) 4575ms infinite reverse;
    }
    #loading-frame-63 {
        animation: animateFrames 4800ms steps(2, jump-none) 4650ms infinite reverse;
    }
    #loading-frame-64 {
        animation: animateFrames 4800ms steps(2, jump-none) 4725ms infinite reverse;
    }
    #loading-frame-65 {
        animation: animateFrames 4800ms steps(2, jump-none) 4800ms infinite reverse;
    }

    .member-container {
        min-height:66px;
        background-color:black;
        text-align:center;
        padding:10px 0px 10px 0px;
        word-break:break-word;
    }

    @media only screen and (max-width: 185px) {

        .member-container {
            word-break:normal
        }

    } 

    .member {
        text-transform:none;
        color:white;
        line-height:1rem;
        letter-spacing:-0.05rem;
        font-size:20px
    }
</style>