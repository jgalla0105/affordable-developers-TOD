<script>
    import { onMount } from 'svelte';
    import { base } from '$app/paths';
    import WordCloud from '$lib/WordCloud.svelte';

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
    <h1 class="section-heading">WHAT IS TOD?</h1>
    <p>
        Transit-oriented development (TOD) is a planning approach that focuses housing, jobs, shops,
        and services around public transit stations so people can live in denser, mixed-use neighborhoods
        with less reliance on cars. By pairing zoning changes with walkable streets, bike access, and
        affordable housing, TOD aims to create more sustainable, connected, and equitable communities.
    </p>

    {#if todFigureSrc}
        <figure class="tod-figure">
            <img src={todFigureSrc} alt="Transit-oriented development figure" />
        </figure>
    {/if}

    <p>
        In Greater Boston, this idea is closely tied to the MBTA Communities Act, which encourages more
        housing near transit while also raising debates about density, affordability, and neighborhood change.
    </p>

    <h2 class="section-heading section-heading--subsection">WHY SHOULD I CARE?</h2>
    <p>
        The word cloud below offers a quick visual overview of how MAPC communities are advancing transit-oriented
        development. At the top level, larger words highlight the project categories that appear most often in
        the dataset; selecting one reveals the awardees behind it, where larger and darker names indicate higher
        funding per capita. It is meant to make patterns in local TOD investment easier to scan at a glance
        before digging into the details.
    </p>

    <WordCloud csvUrl='wordcloud-classified-data.csv' />
</main>


<style>
    @font-face {
        font-family: 'DotFont';
        src: url('/DOTMATRI.TTF') format('truetype');
        font-weight: normal;
        font-style: normal;
        font-display: swap;
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
    }

    .section-heading--subsection {
        font-size: clamp(2.5rem, 7vw, 3.75rem);
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

</style>
