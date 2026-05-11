<script>
    import intromap from '$lib/map-intro.json';
</script>

<div class="tod-container">
    <div class="tod-grid">
        {#each intromap as item}
            <div class="tod-card" style="border-top-color: {item.color}; --theme-color: {item.color};">

                <div class="image-wrapper">
                    <img src="/{item.image}" alt={item.alt} />
                    <div class="badge" style="background-color: {item.color};">
                        {item['TOD-Principle']}
                    </div>
                </div>

                <div class="card-content">
                    <div class="text-block problem">
                        <h3>Problem</h3>
                        <p>{@html item.problem}</p>
                    </div>

                    <div class="text-block solution">
                        <h3>Solution</h3>
                        <p>{@html item.solution}</p>
                    </div>
                </div>

            </div>
        {/each}
    </div>
</div>

<style>
    .tod-container {
        font-family: system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
        max-width: 1200px;
        margin: 0 auto;
        padding: 2rem 1rem;
    }

    .tod-grid {
        display: grid;
        /* Increased min-width from 320px to 360px to make the overall cards wider */
        grid-template-columns: repeat(auto-fit, minmax(360px, 1fr));
        gap: 2rem;
    }

    .tod-card {
        background-color: #ffffff;
        border-radius: 12px;
        overflow: hidden;
        box-shadow: 0 4px 20px rgba(0, 0, 0, 0.08);
        border-top: 6px solid #ccc;
        display: flex;
        flex-direction: column;
        transition: transform 0.2s ease, box-shadow 0.2s ease;
    }

    .tod-card:hover {
        transform: translateY(-4px);
        /* * Combines the dynamic theme color for a colored glow
         * and a subtle dark shadow to maintain depth
         */
        box-shadow: 0 10px 25px -5px var(--theme-color), 0 8px 10px -6px rgba(0, 0, 0, 0.1);
    }

    .image-wrapper {
        position: relative;
        width: 100%;
        /* Reduced height from 220px to 120px so the image acts more like a header strip */
        height: 150px;
        background-color: #f3f4f6;
        border-bottom: 3px solid var(--theme-color);
    }

    .image-wrapper img {
        width: 100%;
        height: 100%;
        object-fit: cover;
    }

    .badge {
        position: absolute;
        bottom: 0;
        left: 0;
        color: white;
        padding: 0.5rem 1rem;
        font-size: 0.85rem;
        font-weight: 600;
        letter-spacing: 0.5px;
        text-transform: uppercase;
        border-top-right-radius: 8px;
    }

    .card-content {
        padding: 1.5rem;
        display: flex;
        flex-direction: column;
        gap: 1.25rem;
        flex-grow: 1;
    }

    .card-content h2 {
        margin: 0;
        font-size: 1.4rem;
        color: #1f2937;
        line-height: 1.2;
    }

    .text-block {
        display: flex;
        flex-direction: column;
        gap: 0.5rem;
    }

    .text-block h3 {
        margin: 0;
        font-size: 0.9rem;
        text-transform: uppercase;
        letter-spacing: 0.5px;
        color: #6b7280;
        border-bottom: 1px solid #e5e7eb;
        padding-bottom: 0.25rem;
    }

    .text-block p {
        margin: 0;
        font-size: 0.95rem;
        line-height: 1.5;
        color: #4b5563;
    }

    .problem p {
        border-left: 3px solid #6b7280;
        padding-left: 0.75rem;
    }

    .solution p {
        border-left: 3px solid #6b7280;
        padding-left: 0.75rem;
    }
</style>
