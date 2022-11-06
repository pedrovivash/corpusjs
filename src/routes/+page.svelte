<script>
    import { onMount } from 'svelte'
    import {MagnifyingGlass, Gear} from 'radix-icons-svelte'
    import '../styles/global.css'
    import Nav from '$lib/navbar.svelte'
    import Footer from '$lib/footer.svelte'


    // synset handling

    // ...catches the lexical entries
    let lexPartArray = []    
    let lexicalSearchTerms

    // ...catches the synsets and calculates the mean characters per member
    let synsetDefines
    let synsetsLength = 0
    let sumMembersLengths = 0
    let avgMemberLength = 10
    
    // ... display stores used in the HTML
    let position = 0
    let member = ''
    let pos = ''
    let definition = ''

    // inputs

    // ...every UI element needs a temp variable to store its values. A number of the functions require shared variables. 
    let validateWPM = 60
    let desiredWPM = 60
    let validatePosition = 0
    let desiredPosition = 0
    let positionTransfer = 0
    let stepPositionUp = false
    let stepPositionDown = false 
    let activateReverse = false
    let activateStop = false
    
    // ...CSS handling for menus 
    let displayChapter = 'none'
    let displaySettings = 'none'
    let displaySearch = 'none'
    
    // ...switches to direct the counter and to remove menus from display
    let isResults = false
    let isResultsUp = false
    let isChapters = false 
    let isChaptersUp = false
    let isIndex = false
    let restop = false

    // search handling
    let searchTerm = ''

    // bindings

    // ...for timeout
    let delay

    // ...for HTML elements
    let stopCheck
    let searchCheck
    let searchBoxCheck
    let chaptersCheck
    let chaptersBoxCheck

    // ...handle mobile dropdown "blurs"
    function windowClick(a) {
        let target = (a && a.target) || (event && event.srcElement)

        if (target != searchBoxCheck &&  target != searchCheck) {    
            searchBoxCheck.style.display = 'none'
        }
        else
        {
            searchBoxCheck.style.display = 'initial'
        }
        if (target != chaptersBoxCheck && target != chaptersCheck) {
            chaptersBoxCheck.style.display = 'none' 
        }
        else 
        {              
            chaptersBoxCheck.style.display = 'flex'
        }
    }

    // animation

    // ...loading graphic
    const loaderRange = Array(65).fill('x',0,65).map( (x, i )=> [i+1,(i+1).toString().padStart(2, '0')])


    // functions, part 1 | core functions

    // ...measures the RSVP delay
    // -requires more precice measurement that calls the following members lengths up to n words per 60 seconds
    function rsvpMeasure(word, targetWPM, avgLength = avgMemberLength) {
        const millisecond = 1000
        const avgEngWordLength = 5.2
        const avgWordLength = avgLength
        const chunkLength = word.length
        const reserveTime = (1/((targetWPM*avgLength)/60))*chunkLength
        return reserveTime*millisecond
    }

    // ...fills the display stores
    function displayDefine(define, positionReq) {
        const x = define[positionReq];
        [member, pos, definition] = [x[0], x[1], x[2]]
    }

    // ...delay handling asynchronously
    function createDelay(intrpt = desiredWPM) {
        delay =  new Promise(r => setTimeout(r, rsvpMeasure(member, intrpt)))
    }
    
    // ...cancel delay
    function cancelDelay() {
        clearTimeout(delay)
    }

    // ...calls the input, forces the HTML to display a different array, handles the delay, handles the direction of the next position
    async function gatherUserInput(positionIn, definesIn) {    

        cancelDelay()
        displayDefine(definesIn, positionIn)
        createDelay()
        await delay

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


    // functions, part 2 | UI controls

    // ...index stepping +/- 1
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
    
    // ...jump to another topic
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

    // ...jump to the start of a synset range per member search request
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

    // ...jump to specified index
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

    // functions, part 3 | search

    // ...search members within the array of all synsets
    async function searchTermReturn(term, aoaoa, synsetsArray){
        const translatePOS = {
            'n': 'noun',
            'v': 'verb',
            'a': 'adjective',
            'r': 'adverb',
            's': 'adjective'
        }
        
        function lexicalLookup(lexicalTerm) {
           return term.trim() == lexicalTerm[0]
        }
        const searchReturn = aoaoa.filter(lexicalLookup)
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

    // ...formats indeces range for search results
    function formatIndecesRange(aon) {
        let tempString = ''
        aon[0] == aon[1] ?
        tempString = aon[0].toString() :
        tempString = aon[0].toString()+' - '+aon[1].toString()
        return tempString
    }
    
    // functions, part 4 | CSS and bindings

    // ...settings CSS
    function settingsDisplay() {
        displaySettings != 'none' ?
        displaySettings = 'none' :
        displaySettings = 'flex'
    }

    // ...topics CSS
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
    
    // ...search CSS
    function searchDisplayOn(){
        isResults = true
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

    // functions, part 5 | loading data

    // ...gets the data
    onMount(async () => {
        // used fast-xml-parser to extract data from LMF XML beforehand
        const synsResponse = await fetch('/01_synsetDefines.json')
        const lexSearchResponse = await fetch('/02_lexicalSearchTerms.json')
        const lexPartResponse = await fetch('/03_lexPartArray.json')
        const synsText = await synsResponse.text()
        const lexSearchText = await lexSearchResponse.text()
        const lexPartText = await lexPartResponse.text()
        synsetDefines = JSON.parse(synsText)
        lexicalSearchTerms = JSON.parse(lexSearchText)
        lexPartArray = JSON.parse(lexPartText)
        synsetDefines.forEach( (x,i) => {
            if (i == 0) {
                null
            }
            else {
            sumMembersLengths += x[0].length
            }
        })
        avgMemberLength = sumMembersLengths/(synsetDefines.length - 1)
        synsetsLength = synsetDefines.length
    })

</script>
<svelte:body on:click={windowClick}/>
<svelte:head>
    <!-- HTML Meta Tags -->
    <title>rsvp-synsets</title>
    <meta name="description" content="A way to quickly read open parts of speech organized by synset groupings.">

    <!-- Facebook Meta Tags -->
    <meta property="og:url" content="https://rsvp-synsets.vercel.app/">
    <meta property="og:type" content="website">
    <meta property="og:title" content="rsvp-synsets">
    <meta property="og:description" content="A way to quickly read open parts of speech organized by synset groupings.">
    <meta property="og:image" content="https://rsvp-synsets.vercel.app/rsvp-synsets-logo_v01.png">

    <!-- Twitter Meta Tags -->
    <meta name="twitter:card" content="summary_large_image">
    <meta property="twitter:domain" content="rsvp-synsets.vercel.app">
    <meta property="twitter:url" content="https://rsvp-synsets.vercel.app/">
    <meta name="twitter:title" content="rsvp-synsets">
    <meta name="twitter:description" content="A way to quickly read open parts of speech organized by synset groupings.">
    <meta name="twitter:image" content="https://rsvp-synsets.vercel.app/rsvp-synsets-logo_v01.png">

    <!-- Meta Tags Generated via https://opengraph.dev -->
</svelte:head>

<header class='nav-container'>
    <div>
        <Nav />
    </div>
</header>
<div style='display:flex; justify-content:center'>
    <main style='width:600px; height:auto; border:18px solid black; padding:18px; overflow:hidden'>
        <noscript><div style='text-align:center; padding:0px 0px 20px 0px'>This page requires JavaScript.</div></noscript>    
        {#if synsetDefines == undefined}                
            <div class='loading-anim' style='display:flex; width:100%; height:64px; justify-content:center;'>
                <div class='loading-img-container' style='display:flex;justify-content:center;position:absolute;width:96px;height:64px;'>
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
        {:else}
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
                        <button class='dropbtn' on:focus={()=> chaptersDisplayOn()} on:blur={()=> chaptersDisplayOff()} bind:this={chaptersCheck} name='partofSpeech'>TOPICS<div class='arrow' /></button>
                        <div class='dropdown-container' on:pointerenter={()=> isChapters = true} on:mouseleave={()=> chaptershMouseOff()} style='display:{displayChapter}' bind:this={chaptersBoxCheck}>
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
                        <div style='justify-content:space-between; width: 148px; align-items:center'>
                            <MagnifyingGlass style='top:3px; position:relative; color:#bbbbbb' />
                            <input type='search' class='search-input' autocomplete='off' placeholder='>>' on:focus={()=> searchDisplayOn()} on:blur={()=> searchDisplayOff()} bind:this={searchCheck} bind:value={searchTerm} name='SEARCH'>
                        </div>
                        <div class='search-results-container' on:pointerenter={()=> isResults=true} on:mouseleave={()=> searchMouseOff()} style='display:{displaySearch}' bind:this={searchBoxCheck}>
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
        {/if}
    </main>
</div>
    <footer>
        <Footer />
    </footer>
<style>

    progress {
        /* Reset the default appearance */
        -webkit-appearance: none;
        -moz-appearance:none;
        appearance: none
    }
    .this-prog {
        background-color:#eeeeee;
        border: none;
        width: 149px;
        height: 9px;
    }
    .this-prog::-webkit-progress-bar {
        background-color:#eeeeee
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
        z-index: 1;
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
        z-index: 2;
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
        border: none;
        outline: 1px solid #80d000;
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
        background-color: #eeeeee;
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
        color:black;
        line-height:1rem;
        letter-spacing:-0.05rem;
        font-size:20px
    }
</style>