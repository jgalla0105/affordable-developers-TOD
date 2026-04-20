<script>
    import tabs from '$lib/tabs.json';


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
    <p><b> Map goes here</b></p>
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
    border-style: solid;
    border-width: 3px;
    border-color:rgba(148, 163, 184, 0.55);
    background-color:#eee98577; 
    padding: 15px;
    font-size: clamp(1.05rem, 2vw, 1.35rem);

  }

  h4 {
    font-size: 2rem;
    text-align: center;
  }
</style>

