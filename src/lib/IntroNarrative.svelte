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
    <Scrolly bind:progress={scrollyProgress}>
        {#each intro as i}
            <section class="step">
                <div class="step-content" style:border-left-color={i.color}>
                    <h3 style:color={i.color}>{i.title}</h3>
                    <p>{i.story}</p>
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
    .scrolly-wrapper {
        width: auto;
        position: relative;
        left: 45%;
        transform: translateX(-50%);
    }

    .step {
        min-height: 80vh;
        padding: 2rem;
    }

    .step-content {
        border-left-width: 8px;
        border-left-style: dotted;
        padding: 1.5rem 2rem;
        font-size: clamp(1.05rem, 2vw, 1.35rem);
    }

    .section-detail {
        padding: 2rem;
        width: 100%;
        min-height: 100vh;
        display: flex;
        align-items: center;      /* vertical centering */
        justify-content: center;  /* horizontal centering */
    }

    img {
        max-width: 100%;
        max-height: calc(100vh - 4rem);
        object-fit: contain;
        border: 2px solid rgba(148, 163, 184, 0.32);
        background: linear-gradient(180deg, #ffffff 0%, #fbfdff 100%);
        box-shadow: 0 18px 36px rgba(15, 23, 42, 0.08);
        border-radius: 24px;
    }
</style>
