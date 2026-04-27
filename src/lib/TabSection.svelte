<script>
    import tabs from '$lib/tabs.json';
    import ChoroplethMap from '$lib/ChoroplethMap.svelte';

    $: activeTab = 0;
</script>

<div class="tabs">
  {#each tabs as tab, i}
    <button
      class:active={activeTab === i}
      on:click={() => activeTab = i}>
      {tab.heading}
    </button>
  {/each}
</div>

<div class="content">
    <h4>{tabs[activeTab].heading}</h4>

    <div class="map-wrapper">
      {#if activeTab === 0}
        <ChoroplethMap
          styleUrl="mapbox://styles/jgalla0611/cmoenjnxr000p01sa0m670jim"
          valueProperty="MULTI%"
          townNameProperty="NAME"
          criteriaName="Multifamily Housing"
          valueLabel="% Multifamily"
          unit="%"
          legendBins={[
            { label: "0 – 5%",   color: "#f7fbff" },
            { label: "5 – 20%",  color: "#deebf7" },
            { label: "20 – 35%", color: "#6baed6" },
            { label: "35 – 55%", color: "#2171b5" },
            { label: "55 – 73%", color: "#08519c" },
            { label: "73%+",     color: "#08306b" },
          ]}
        />
      {:else}
        <p><b>Choropleth Map coming soon for this tab.</b></p>
      {/if}
    </div>

  <p class="justification">{tabs[activeTab].justification}</p>
</div>





<style>
  .active {
    background-color: #ccc;
    font-weight: bold;
  }
  button {
    padding: 10px;
    cursor: pointer;
    justify-self: center;
    display: inline-flex;
    align-items: center;
    gap: 0.65rem;
    border: 1px solid rgba(148, 163, 184, 0.55);
    background: linear-gradient(180deg, #ffffff 0%, #f8fbff 100%);
    color: rgb(59, 59, 59);
    border-radius: 999px;
    padding: 0.8rem 1.15rem;
    font: inherit;
    font-weight: 700;
    cursor: pointer;
    box-shadow: 0 10px 20px rgba(15, 23, 42, 0.08);
    transition: transform 160ms ease, box-shadow 160ms ease, border-color 160ms ease, background 160ms ease;
  }
  button:hover {
    transform: translateY(-1px);
    border-color: rgba(71, 85, 105, 0.35);
    background: #bfbfbf;
    box-shadow: 0 14px 24px rgba(15, 23, 42, 0.12);
  }
  .justification {
     border: 2px solid rgba(148, 163, 184, 0.32);
        background: linear-gradient(180deg, #ffffff 0%, #fbfdff 100%);
        box-shadow: 0 18px 36px rgba(15, 23, 42, 0.08);
        border-radius: 24px;
      padding: 15px;
      font-size: clamp(1.05rem, 2vw, 1.35rem);

  }

  h4 {
    font-size: 2rem;
    text-align: center;
  }

  .map-wrapper {
    position: relative;
    margin-bottom: 1.5rem;
  }

  p, .content {
    color: rgb(59, 59, 59);
  }
</style>

