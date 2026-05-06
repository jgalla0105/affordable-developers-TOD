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

    <div class="intro">
    <div class="section">
    <h1 class="section-heading" style="margin-top:6rem; margin-bottom:2rem">TRANSIT-ORIENTED DEVELOPMENT IN MASSACHUSETTS</h1>
    <p>
       Transit-oriented development (TOD) is a planning approach that focuses housing, jobs, shops,
       and services around public transit stations so people can live in denser, mixed-use neighborhoods
       with less reliance on cars. By pairing zoning changes with walkable streets, bike access, and
       affordable housing, TOD aims to create more sustainable, connected, and equitable communities.
   </p>

   <p>
        In the Greater Boston area, the Massachusetts Department of Transportation (MassDOT) and the Massachusetts Bay Transportation Authority (MBTA) have established four foundational principles that provide guidance to their pursuit of transit-oriented development. These principles are as follows:  
   </p>

   <h2 class="section-heading section-heading--subsection" style="margin-top:2rem">MBTA TOD PRINCIPLES</h2>
    <IntroNarrative />
    <p class="placeholder"> transition to next section</p>
   </div>
   </div>

   <div class="wordcloud">
   <div class="section">
    <h2 class="section-heading section-heading--subsection">HOW IS STATE FUNDING CURRENTLY GOING TOWARDS THESE PRINCIPLES?</h2>

    <p>Established in 2021, the MBTA Communities Act is a state-issued ordinance focused on building multi-family housing supply near mass transit services. Based on proximity to MBTA transit service, the Communities Act requires cities to possess a stated quantity of multi-family housing to provide additional funding to transit-oriented development projects. Failure to comply prevents access to certain areas of state funding. Learn more about the MBTA Communities and its different grants listed in Section 3A, <a href="https://www.mass.gov/info-details/multi-family-zoning-requirement-for-mbta-communities#what-is-an-mbta-community" target="_blank">here.</a></p>
    <p>
        The word cloud below offers an overview of how transit-oriented
        development is being advanced in compliant MBTA communities. At the top level, larger words highlight the project categories that have the most communities investing funding in them; select a category to reveal the awardees (communities) spending on it, and learn more about what they're developing.
    </p>
    
    <WordCloud csvUrl='wordcloud-classified-data.csv' />
    <p class="placeholder"> transition to next section</p>
    </div>
    </div>
    
    <div class="map">
    <div class="section">
    <h2 class="section-heading section-heading--subsection" style="margin-top:3rem">WHAT COMMUNITIES MIGHT BENEFIT MOST FROM TOD?</h2>
    <TabSection/>
    </div>
    

    <hr  style="margin-top:10rem">
        <p class="refs">This project was developed with guidance and feedback from the <a href=https://www.mapc.org target=blank>Metropolitan Area Planning Commission (MAPC)</a></p>
        <p class="refs" style="font-weight:bold">DATA SOURCES</p>
        <p class="placeholder">**insert rest of referneces here**</p>
    </div>
   



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
     margin: 0;
    }
    
    hr {
        width: 50%;
        background-color:rgb(158, 158, 158);
        height:1px;
    }
    .refs {
        text-align: center;
        font-style:italic;
        color:rgb(158, 158, 158); 
        font-size: 1rem;
    }
    .intro {
        background-color: rgb(255, 255, 255);
        width: 100%;
        
    }
    .wordcloud {
        background-color:#f5ebbd;
    }
    .map {
        background-color:rgb(255, 255, 255);
        /* height: 100 vh; */
    }
    .placeholder {
        color: red;
        font-weight: bold;
    }
    .section {
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
        margin-top:3rem;
    }

    p {
        max-width: var(--content-width);
        margin: 0 auto;
        font-size: clamp(1.05rem, 2vw, 1.35rem);
        line-height: 1.6;
        text-align: left;
        color: rgb(59, 59, 59);
    }

    .placeholder {
        color:brown;
    }

    

</style>
