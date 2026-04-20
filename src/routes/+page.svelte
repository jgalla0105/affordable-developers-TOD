<script>
    import { onMount } from 'svelte';
    import { base } from '$app/paths';
    import WordCloud from '$lib/WordCloud.svelte';
    import IntroNarrative from '$lib/IntroNarrative.svelte';
    import TabSection from '../lib/TabSection.svelte';
    // import intro from '$lib/intro.json';

    const todFigureCandidates = ['tod-intro.png', 'tod-intro.jpg', 'tod-intro.jpeg', 'tod-intro.webp', 'tod-intro.svg'];
    let todFigureSrc = '';

    onMount(() => {
        let isCancelled = false;

        async function findTodFigure() {
            for (const fileName of todFigureCandidates) {
                const candidateSrc = `${base}/figures/${fileName}`;
                const exists = await new Promise((resolve) => {
                    const image = new Image();

                    image.onload = () => resolve(true);
                    image.onerror = () => resolve(false);
                    image.src = candidateSrc;
                });

                if (exists) {
                    if (!isCancelled) {
                        todFigureSrc = candidateSrc;
                    }

                    return;
                }
            }
        }

        findTodFigure();

        return () => {
            isCancelled = true;
        };
    });
</script>

<main class="page">
    <h1 class="section-heading">WHAT IS TRANSIT-ORIENTED DEVELOPMENT?</h1>
    <p>
       Transit-oriented development (TOD) is a planning approach that focuses housing, jobs, shops,
       and services around public transit stations so people can live in denser, mixed-use neighborhoods
       with less reliance on cars. By pairing zoning changes with walkable streets, bike access, and
       affordable housing, TOD aims to create more sustainable, connected, and equitable communities.
   </p>

   <p>
        In the Greater Boston area, the Massachusetts Department of Transportation (MassDOT) and the Massachusetts Bay Transportation Authority (MBTA) have established four foundational principles that provide guidance to their pursuit of transit-oriented development. These principles are as follows:  
   </p>

   <h2 class="section-heading section-heading--subsection">MBTA TOD PRINCIPLES</h2>
    <IntroNarrative />


    <h2 class="section-heading section-heading--subsection">HOW IS STATE FUNDING CURRENTLY GOING TOWARDS THESE PRINCIPLES?</h2>

    <p class="placeholder">** introduce the MBTA Communities Act briefly **</p>
    <p>
        The word cloud below offers a quick visual overview of how MAPC communities are advancing transit-oriented
        development. At the top level, larger words highlight the project categories that have the most communities investing funding in them; selecting one reveals the awardees (communities) behind it, where larger and darker names indicate higher
        funding per capita. It is intended to make patterns in local TOD investment easier to scan at a glance
        before digging into the details.
    </p>
    
    <WordCloud csvUrl='wordcloud-classified-data.csv' />

    <h2 class="section-heading section-heading--subsection">WHAT PLACES MIGHT BENEFIT MOST FROM TOD?</h2>

    <TabSection/>

   

</main>


<style>
    @font-face {
        font-family: 'DotFont';
        src: url('/DOTMATRI.TTF') format('truetype');
        font-weight: normal;
        font-style: normal;
        font-display: swap;
    }
    :global(body) {
     background-color: #fcfbec; /* Replace with your desired color */
    }

    .placeholder {
        color: red;
        font-weight: bold;
    }
    .page {
        --content-width: 72ch;
        --section-gap: clamp(1.25rem, 3vw, 2rem);
        max-width: 72rem;
        margin: 0 auto;
        padding: 1.5rem clamp(1rem, 4vw, 2.5rem) 3rem;
        display: grid;
        gap: var(--section-gap);
    }

    .section-heading {
        margin: 0;
        font-family: 'DotFont', sans-serif;
        font-size: clamp(2.5rem, 7vw, 3.75rem);
        line-height: 0.92;
        letter-spacing: 0.06em;
        text-align: center;
        color: rgb(59, 59, 59);
        text-shadow: 0 0 10px #e6e600;
    }

    .section-heading--subsection {
        /* font-size: clamp(2.5rem, 7vw, 3.75rem); */
        font-size: 3rem;
        margin-bottom: 1.5rem;
    }

    p {
        max-width: var(--content-width);
        margin: 0 auto;
        font-size: clamp(1.05rem, 2vw, 1.35rem);
        line-height: 1.6;
        text-align: left;
        color: rgb(59, 59, 59);
    }

    .tod-figure {
        width: min(100%, var(--content-width));
        margin: 0 auto;
    }

    .tod-figure img {
        display: block;
        width: 100%;
        max-width: 100%;
        height: auto;
        margin: 0 auto;
        border-radius: 16px;
        box-shadow: 0 12px 30px rgba(15, 23, 42, 0.12);
    }
    .placeholder {
        color:brown;
    }

</style>
