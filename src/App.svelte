<script>
    import {
        seed,
        getDate,
        yesterday,
        loadWords,
        makeGenerate,
        makeCheck,
        makeSolve,
        xmur3
    } from "./litterboxed.js";

    import { fade } from "svelte/transition";

    let message = "loading";
    let alert = (msg) => {
        message = msg;
        setTimeout(() => {
            message = "";
        }, 1000);
    };

    let minlength = 3;
    let side = 30;
    let x = 6;
    let y = 6;
    let circles = [];
    let letters_pos = [];
    let letter_offset = 3.5;
    let letter_size = 4;
    let letters = "";
    if (localStorage.getItem("date") != getDate()) {
        localStorage.removeItem("puzzle");
    }
    letters = localStorage.getItem("puzzle") || "            ";
    let generate, check, solve, solveAll;
    let displaySols = false;
    let showSolveLink = false;
    let solutions, allSolutions;
    (async () => {
        let y = window.location.hash == "#y";
        const urlParams = new URLSearchParams(window.location.search)
        const userSeed = urlParams.has('seed') ? urlParams.get('seed') : getDate()
        seed(userSeed);
        let easy = await loadWords("./easy.txt");
        generate = makeGenerate(easy);
        letters = generate(userSeed).join("").toUpperCase();
        localStorage.setItem("puzzle", letters);
        localStorage.setItem("date", getDate());
        message = "loading dict";
        let scrabble = await loadWords("./scrabble.txt");
        check = makeCheck(scrabble);
        message = " ";
        solve = makeSolve(easy);
        solveAll = makeSolve(scrabble);
    })();

    let hitboxes = [];
    for (let i = 0; i < 3; i++) {
        let offset = (side / 3) * (i + 0.5);
        circles.push({ x: x, y: y + offset });
        letters_pos.push({ x: x - letter_offset, y: y + offset });
        hitboxes.push({
            x: x - letter_offset - letter_size / 2,
            y: y + offset - letter_size / 2,
            width: letter_size + letter_offset,
            height: letter_size,
        });

        circles.push({ x: x + side, y: y + offset });
        letters_pos.push({ x: x + side + letter_offset, y: y + offset });
        hitboxes.push({
            x: x + side - letter_size / 2,
            y: y + offset - letter_size / 2,
            width: letter_size + letter_offset,
            height: letter_size,
        });

        circles.push({ x: x + offset, y: y });
        letters_pos.push({ x: x + offset, y: y - letter_offset });
        hitboxes.push({
            x: x + offset - letter_size / 2,
            y: y - letter_offset - letter_size / 2,
            width: letter_size,
            height: letter_size + letter_offset,
        });

        circles.push({ x: x + offset, y: y + side });
        letters_pos.push({ x: x + offset, y: y + side + letter_offset });
        hitboxes.push({
            x: x + offset - letter_size / 2,
            y: y + side - letter_size / 2,
            width: letter_size,
            height: letter_size + letter_offset,
        });
    }
    let stroke = 0.3;
    let index = (i) => (i % 4) * 3 + Math.floor(i / 4);
    let revindex = (i) => (i % 3) * 4 + Math.floor(i / 3);
    let currentWord = "";
    $: lastLetter = currentWord ? letters.indexOf(currentWord.slice(-1)) : -1;
    let selectLetter = (i) => {
        if (done) return;
        if (lastLetter == i) deleteLetter()
        if (Math.floor(lastLetter / 3) != Math.floor(i / 3))
            currentWord = currentWord + letters[i];
    };
    let deleteLetter = () => {
        currentWord = currentWord.slice(0, -1);
        if (currentWord == "")
            if (previousWords.length) {
                currentWord = previousWords.pop();
                previousWords = previousWords;
            }
    };

    let previousWords = [];
    $: done = [...Array(12).keys()].every(
        (i) => previousWords.join("").indexOf(letters[i]) > -1
    );
    let enterWord = () => {
        if (done) return;
        if (currentWord.length < minlength) return alert("Too short");
        if (!check(currentWord)) return alert("Not a word");
        previousWords = [...previousWords, currentWord];
        currentWord = currentWord.slice(-1);
        if (done) currentWord = "";
    };
    let generateNewLetters = () => {
        let generateRandomString = () => {
            const min = 'A'.charCodeAt(0)
            const max = 'Z'.charCodeAt(0)
            let choices = []
            for (let i=min; i<=max; i++) {
                choices.push(String.fromCharCode(i))
            }
            let result = ""
            for (let i=0; i<10; i++) {
                result += choices[Math.floor(Math.random()*choices.length)]
            }
            return result
        }
        const newRandomSeed = generateRandomString()
        // letters = generate(newRandomSeed).join("").toUpperCase()

        let urlParams = new URLSearchParams(window.location.search)
        urlParams.set('seed', newRandomSeed)
        window.location.search = `?${urlParams.toString()}`
        
        displaySols = false
    };
    let caret = "?";
    setInterval(() => {
        caret = caret ? "" : "?";
    }, 500);
    $: words = previousWords.join(" - ");
    let letterColor = (i) => {
        if (i == lastLetter) return "#ff3e00";
        if (previousWords.join("").indexOf(letters[i]) > -1) return "grey";
        if (currentWord.indexOf(letters[i]) > -1) return "black";
        return "white";
    };

    document.addEventListener("keydown", function (event) {
        if (event.key == "Enter") {
            enterWord();
        } else if (event.key == "Backspace") {
            deleteLetter();
        } else {
            let i = letters.indexOf(event.key.toUpperCase());
            if (i != -1) selectLetter(i);
        }
    });
    let help = false;
    let modal;

    window.showSolution = (p) => {
        console.debug(xmur3(p)())
        if (xmur3(p)() == 2126390614) {
            showSolveLink = true
        }
    }

</script>

<main>
    <h1>soi ya loiter 🅱️ox</h1>

    {#if help}
        <div
            class="modal"
            bind:this={modal}
            on:click={(evt) => {
                help = evt.target != modal;
            }}
        >
            <div class="modal-content">
                <h3 style="margin-top: .5em">How to Play</h3>
                <ul>
                    <li>Connect letters to spell words</li>
                    <li>Words must be at least 3 letters long</li>
                    <li>Letters can be reused</li>
                    <li>Consecutive letters cannot be from the same side</li>
                    <li>
                        The last letter of a word becomes the first letter of
                        the next word <br />
                        e.g. THY > YES > SINCE
                    </li>
                    <li>Words cannot be proper nouns or hyphenated</li>
                    <li>Use all letters to solve!</li>
                    <li>
                        There is always a solution with two words that repeats
                        only the common letter <br />
                        e.g. LANDS > SECURITY
                    </li>
                    <p>
                        Inspired by the <a
                            href="https://www.nytimes.com/puzzles/letter-boxed"
                            >NYT's Letter Boxed</a
                        >.
                    </p>
                    <p>
                        Fork created by spencer. Not open source (yet)
                    </p>
                    <p>
                        <a href="https://github.com/louisabraham/litterboxed"
                            >Original source code by louisabraham and trivia</a
                        >
                    </p>
                </ul>
            </div>
        </div>
    {/if}
    <div class="current">
        {#if message}
            <span
                style="position: absolute; left: 0; right: 0; font-size: medium;"
                transition:fade
                class="message">{message}</span
            >
        {/if}
        {#if !done}
            <span style="display: inline">{currentWord}</span><span
                class="unselectable"
                style={(currentWord ? "width: 0em;" : "") +
                    "display: inline-block;"}>{caret}</span
            >
        {:else}
            <span class="blink" style="display: inline">YOU WIN!</span>
        {/if}
        <hr
            style="min-width: 10em; max-width: 13em;
			border:1px solid black; margin-top: 0"
        />
    </div>
    <div class="words">
        <p
            style="width: fit-content; margin: auto"
            on:click={() => {
                navigator.clipboard.writeText(words), alert("copied");
            }}
        >
            {words}
        </p>
    </div>
    <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 42 42">
        <rect
            {x}
            {y}
            width={side}
            height={side}
            stroke="black"
            stroke-width={stroke}
            fill="none"
        />
        {#each [...previousWords, currentWord] as word, pos}
            {#each word as l, i}
                {#if i + 1 < word.length}
                    <line
                        x1={circles[revindex(letters.indexOf(l))].x}
                        y1={circles[revindex(letters.indexOf(l))].y}
                        x2={circles[revindex(letters.indexOf(word[i + 1]))].x}
                        y2={circles[revindex(letters.indexOf(word[i + 1]))].y}
                        stroke={pos == previousWords.length
                            ? "#ff3e00"
                            : "#ff3e0080"}
                        stroke-width={stroke}
                        stroke-dasharray={pos == previousWords.length ? 2 : 0}
                    />
                {/if}
            {/each}
        {/each}

        {#each circles as c, i}
            <circle
                cx={c.x}
                cy={c.y}
                r="1"
                fill={(lastLetter, currentWord, letterColor(index(i)))}
                stroke="black"
                stroke-width={stroke}
            />
            <text
                text-anchor="middle"
                dominant-baseline="central"
                x={letters_pos[i].x}
                y={letters_pos[i].y}
                font-size={letter_size}>{letters[index(i)]}</text
            >
            <rect
                x={hitboxes[i].x}
                y={hitboxes[i].y}
                width={hitboxes[i].width}
                height={hitboxes[i].height}
                fill="none"
                stroke="none"
                stroke-width=".1"
                pointer-events="fill"
                on:click={() => selectLetter(index(i))}
            />
        {/each}
    </svg>
    <div class="buttons">
        <button on:click={deleteLetter}>Delete</button>
        <button on:click={enterWord}>Enter</button>
        <button on:click={generateNewLetters}>Random</button>
    </div>
    {#if displaySols}
        <div class="solutions">
            <p>Solutions ({allSolutions.length}):</p>
            {#each solutions as sol}
                <p class={sol.length==16 ? "unique-letter-sol" : ""} style="font-weight: bold">{sol}</p>
            {/each}
            {#each allSolutions as sol}
                {#if !solutions.includes(sol)}
                    <p class={sol.length==16 ? "unique-letter-sol" : ""}>{sol}</p>
                {/if}
            {/each}
        </div>
    {:else}
        <p
            class="link"
            on:click={() => {
                displaySols = true;
                let yesterdayPuzzle = generate(yesterday())
                    .join("")
                    .toUpperCase();
                letters = yesterdayPuzzle;
                previousWords = [];
                currentWord = "";
                let format = (s) => `${s[0]} - ${s[1]}`;
                solutions = solve(yesterdayPuzzle).map(format);
                allSolutions = solveAll(yesterdayPuzzle).map(format);
            }}
        >
            Yesterday
        </p>
        {#if showSolveLink}
        <p
            class="link"
            on:click={() => {
                displaySols = true;
                previousWords = [];
                currentWord = "";
                let format = (s) => `${s[0]} - ${s[1]}`;
                solutions = solve(letters).map(format);
                allSolutions = solveAll(letters).map(format);
            }}
        >
            Solve
        </p>
        {/if}
        <p
            class="link"
            on:click={() => {
                help = true;
            }}
        >
            Help
        </p>
    {/if}
</main>

<style>
    main {
        text-align: center;
        padding: 1em;
        margin: 0 auto;
    }

    h1 {
        color: #ff3e00;
        text-transform: uppercase;
        font-size: 3em;
        font-weight: 100;
        margin-top: 0;
        margin-bottom: 0.4em;
    }
    .unselectable {
        -webkit-touch-callout: none;
        -webkit-user-select: none;
        -khtml-user-select: none;
        -moz-user-select: none;
        -ms-user-select: none;
        user-select: none;
    }
    .current {
        font-size: x-large;
        position: relative;
    }

    .words {
        height: 1em;
        font-size: large;
    }
    .buttons > button {
        width: 100px;
        text-align: center;
    }

    svg {
        width: 100%;
        max-width: 400px;
        margin-top: 1em;
        margin-bottom: 1em;
    }

    .message {
        margin-top: -1.4em;
    }

    .blink {
        animation: blinker 1s infinite;
    }

    .link {
        display: inline;
        margin: 0.5em;
        color: rgb(0, 100, 200);
    }

    .link:hover {
        text-decoration: underline;
    }

    .modal {
        position: fixed; /* Stay in place */
        z-index: 1; /* Sit on top */
        left: 0;
        top: 0;
        width: 100%; /* Full width */
        height: 100%; /* Full height */
        overflow: auto; /* Enable scroll if needed */
        background-color: rgb(0, 0, 0); /* Fallback color */
        background-color: rgba(0, 0, 0, 0.4); /* Black w/ opacity */
    }

    /* Modal Content/Box */
    .modal-content {
        background-color: #fefefe;
        margin: 6em auto;
        padding: 10px;
        border: 1px solid #888;
        border-radius: 10px;
        max-width: 400px; /* Could be more or less, depending on screen size */
    }
    .modal-content > ul {
        text-align: left;
    }

    .unique-letter-sol {
        color: #ff3e00; 
    }

    @keyframes blinker {
        50% {
            opacity: 0;
        }
    }
    @media (max-width: 350px) {
        h1 {
            font-size: 2.5em;
        }
        .modal-content {
            margin: 5.5em auto; /* 15% from the top and centered */
        }
        .message {
            margin-top: -1.2em;
        }
    }
</style>
