<script>
    import Scrolly from 'svelte-scrolly';
    import intro from '$lib/intro.json';

    let scrollyProgress = 0;

    const progressPerSection = 100 / intro.length;
    $: activeSectionIdx = Math.min(
        intro.length - 1,
        Math.floor(scrollyProgress / progressPerSection)
    );
</script>

<div class="scrolly-wrapper">
    <Scrolly 
    bind:progress={scrollyProgress}
    --scrolly-layout="overlap"
    --scrolly-gap="0"
    --scrolly-viz-top="0px">
        {#each intro as i}
            <section class="step">
                <div class="step-content">
                    <h3 style:color={i.color}>{i.title}</h3>
                    <p>{@html i.story}</p>
                </div>
            </section>
        {/each}

        <svelte:fragment slot="viz">
            <div class="section-detail">
                
                <img src={intro[activeSectionIdx].image} alt={intro[activeSectionIdx].alt ?? intro[activeSectionIdx].title} />
            </div>
        </svelte:fragment>
    </Scrolly>
</div>

<style>
h3 {
    text-align: center;
}
    .scrolly-wrapper {
    width: 100%;
    margin: 0 auto;
     background: linear-gradient(180deg, #ffffff 0%, #fbfdff 100%);
        box-shadow: 0 18px 36px rgba(15, 23, 42, 0.08);
        border-radius: 24px; 
}

/* The internal svelte-scrolly layout needs to be controlled globally */
.scrolly-wrapper :global(.story) {
    position: relative;
    z-index: 2;
}

.scrolly-wrapper :global(.viz) {
    z-index: 1;
    height: 100vh;
}

/* Each text step scrolls through the sticky viewport */
.step {
    min-height: 140vh;
    display: flex;
    align-items: flex-start;
    justify-content: center;
    padding: 6vh 1rem 0;
    /* padding: 10vh 1rem 45vh; more breathing space before next section begins  */
    box-sizing: border-box;
}

.step-content {
    width: min(760px, 100%);
    padding: 1.25rem 1.75rem;
    font-size: clamp(1.05rem, 2vw, 1.35rem);
    line-height: 1.75;
    color: rgb(59, 59, 59);

    /* optional, but useful if the text overlays the viz area */
    background: rgba(255, 255, 255, 0.86);
    backdrop-filter: blur(8px);
    border-radius: 18px;
}

/* This is inside the already-sticky viz slot */
.section-detail {
    height: 100vh;
    display: flex;
    align-items: flex-end;
    justify-content: center;
    /* leaves the top part of the viewport for text */
    padding: 32vh 2rem 6vh;
    box-sizing: border-box;
}

img {
    max-width: min(100%, 900px);
    max-height: 60vh;
    object-fit: contain;
}
</style>
