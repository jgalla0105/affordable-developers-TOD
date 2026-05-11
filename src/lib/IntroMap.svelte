<script>
    import intromap from '$lib/map-intro.json';
    import { base } from '$app/paths';
    import { slide } from 'svelte/transition';

    let expanded = {};

    function toggle(id) {
        expanded[id] = !expanded[id];
    }

    function resolveAssetUrl(path = '') {
        if (!path || /^(?:[a-z]+:)?\/\//i.test(path)) {
            return path;
        }

        if (path.startsWith(base)) {
            return path;
        }

        return path.startsWith('/') ? `${base}${path}` : `${base}/${path}`;
    }
</script>

<div class="tod-container">
    <div class="tod-grid">
        {#each intromap as item, i}
            <div class="tod-card" style="border-top-color: {item.color}; --theme-color: {item.color};">

                <button
                    class="card-header-trigger"
                    on:click={() => toggle(i)}
                    aria-expanded={expanded[i]}
                >
                    <div class="image-wrapper">
                        <img src={resolveAssetUrl(item.image)} alt={item.alt} />
                        <div class="badge" style="background-color: {item.color};">
                            {item['TOD-Principle']}
                        </div>

                        <div class="status-indicator {expanded[i] ? 'is-open' : ''}">
                            <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="3" stroke-linecap="round" stroke-linejoin="round"><path d="m6 9 6 6 6-6"/></svg>
                        </div>
                    </div>
                </button>

                {#if expanded[i]}
                    <div class="card-content" transition:slide={{ duration: 300 }}>
                        <div class="text-block problem">
                            <h3>Problem</h3>
                            <p>{@html item.problem}</p>
                        </div>

                        <div class="text-block solution">
                            <h3>Solution</h3>
                            <p>{@html item.solution}</p>
                        </div>
                    </div>
                {/if}
            </div>
        {/each}
    </div>
</div>

<style>
    /* 1. APPLY THIS TO PREVENT JUMPING */
    :global(html) {
        /* Reserves space for the scrollbar even when it's not there */
        scrollbar-gutter: stable;
    }

    .tod-container {
        font-family: system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
        max-width: 1200px;
        margin: 0 auto;
        padding: 2rem 1rem;
    }

    .tod-grid {
        display: grid;
        /* 2. FORCE 2 COLUMNS WITH FIXED RATIOS */
        grid-template-columns: repeat(2, minmax(0, 1fr));
        gap: 2rem;
        align-items: start;
    }

    @media (max-width: 768px) {
        .tod-grid {
            grid-template-columns: 1fr;
        }
    }

    .tod-card {
        background-color: #ffffff;
        border-radius: 12px;
        overflow: hidden;
        box-shadow: 0 4px 20px rgba(0, 0, 0, 0.08);
        border-top: 6px solid #ccc;
        transition: transform 0.2s ease, box-shadow 0.2s ease;
        /* Ensures card width is calculated from the grid track, not content */
        width: 100%;
    }

    .card-header-trigger {
        display: block;
        width: 100%;
        padding: 0;
        margin: 0;
        border: none;
        background: none;
        cursor: pointer;
        text-align: left;
    }

    .image-wrapper {
        position: relative;
        width: 100%;
        height: 150px;
        background-color: #f3f4f6;
        border-bottom: 3px solid var(--theme-color);
    }

    .image-wrapper img {
        width: 100%;
        height: 100%;
        object-fit: cover;
        display: block;
    }

    /* ... Remaining styles (badge, status-indicator, etc.) stay the same ... */

    .badge {
        position: absolute;
        bottom: 0;
        left: 0;
        color: white;
        padding: 0.5rem 1rem;
        font-size: 0.85rem;
        font-weight: 700;
        text-transform: uppercase;
        border-top-right-radius: 8px;
    }

    .status-indicator {
        position: absolute;
        bottom: 8px;
        right: 12px;
        color: white;
        background: rgba(0,0,0,0.3);
        border-radius: 50%;
        width: 28px;
        height: 28px;
        display: flex;
        align-items: center;
        justify-content: center;
        transition: transform 0.3s ease;
    }

    .status-indicator.is-open {
        transform: rotate(180deg);
    }

    .card-content {
        padding: 1.5rem;
        display: flex;
        flex-direction: column;
        gap: 1.25rem;
    }

    .text-block h3 {
        margin: 0;
        font-size: 0.9rem;
        text-transform: uppercase;
        color: #6b7280;
        border-bottom: 1px solid #e5e7eb;
        padding-bottom: 0.25rem;
    }

    .text-block p {
        margin: 0.5rem 0 0;
        font-size: 0.95rem;
        line-height: 1.5;
        color: #4b5563;
    }

    .problem p, .solution p {
        border-left: 3px solid #6b7280;
        padding-left: 0.75rem;
    }
</style>
