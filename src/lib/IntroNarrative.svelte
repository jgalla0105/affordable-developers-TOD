<script>

import Scrolly from "svelte-scrolly";
import intro from "$lib/intro.json";


let scrollyProgress = 0

let progressPerSection = 100 / intro.length;
$: activeSectionIdx = Math.min(intro.length-1, Math.floor(scrollyProgress / progressPerSection));


</script>



<div class='scrolly-wrapper'>
    <Scrolly bind:progress={ scrollyProgress }>
        <!-- Story here -->
        {#each intro as i}
            <section class='step'>
                <div class='step-content'>
                    <h3>{i.title}</h3>
                    <p>{i.story}</p>
                </div>
            </section>
        {/each}
        <svelte:fragment slot="viz" >
            <!-- Visualizations here (these will stay sticky) -->
            <div class='section-detail'>
                <img src={intro[activeSectionIdx].image} alt={intro[activeSectionIdx].alt}/>
            </div>
        </svelte:fragment>
    </Scrolly>
</div>



<style>

    .scrolly-wrapper {
        width:auto;
        position: relative;
        left: 45%;
        transform: translateX(-50%);
        }
    .step {
            min-height: 80vh;
            padding: 2rem;
        }

    .step-content {
        border-left-style: solid;
        border-left-color: rgb(59, 59, 59);
        border-left-width: 8px;
        border-left-style:dotted;
        padding: 1.5rem 2rem;
        font-size: clamp(1.05rem, 2vw, 1.35rem);
    }

    .section-detail {
        padding: 2rem;
        width: 100%;
    }

    img {
        max-width: 100%;
    }


</style>