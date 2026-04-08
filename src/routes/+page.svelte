<script> 
import WordCloud from '$lib/WordCloud.svelte';

</script>

<h1>WHAT IS TRANSIT ORIENTED DEVELOPMENT?</h1>
<p>**insert some text explaining what TOD is**</p>
<h4>Okay, but why should I care?</h4>
<p>Because look how MAPC communities are improving to promote TOD!</p>
<WordCloud csvURL='../static/wordcloud-classified-data.csv' />


<style>
    @font-face {
        font-family: 'DotFont';
        src: url('/DOTMATRI.TTF') format('truetype');
        font-weight: normal;
        font-style: normal;
        font-display: swap;
    }

    h1 {
        font-family: 'DotFont', sans-serif;
        font-size: 60px;
    }
    h1, p, h2, h3, h4 {
        color:rgb(59, 59, 59);
    }

</style>
