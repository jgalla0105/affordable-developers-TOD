<script>
    import { onMount } from 'svelte';
    import { base } from '$app/paths';
    import WordCloud from '$lib/WordCloud.svelte';
    import IntroNarrative from '$lib/IntroNarrative.svelte';
    import TabSection from '../lib/TabSection.svelte';
    import IntroMap from '$lib/IntroMap.svelte'
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
    <h1 class="section-heading" style="margin-top:6rem; margin-bottom:2rem">Transit-Oriented Development in Massachusetts</h1>
    <p>
       Transit-oriented development (TOD) is a planning approach that focuses housing, jobs, shops,
       and services around public transit stations so people can live in denser, mixed-use neighborhoods
       with less reliance on cars. By pairing zoning changes with walkable streets, bike access, and
       affordable housing, TOD aims to create more sustainable, connected, and equitable communities.
   </p>


   <p>
        In the Greater Boston area, the Massachusetts Department of Transportation (MassDOT) and the Massachusetts Bay Transportation Authority (MBTA) have established four foundational principles that provide guidance to their pursuit of transit-oriented development. These principles are: <span style="color: #7C878E; font-weight: bold;">density and mix of uses</span>, <span style="color: #ED8B00; font-weight: bold;">equitable development</span>, <span style="color: #003DA5; font-weight: bold;">a great public realm</span>, and <span style="color: #80276C; font-weight: bold;">a TOD approach to parking</span>.
   </p>

   <p class="transition">Scroll to learn what these principles mean <span class="arrow">&darr;</span></p>
   </div>
    </div>

    <div class="scrolly">
    <div class="section">
   <h2 class="section-heading section-heading--subsection" style="margin-top:2rem">MassDOT/MBTA TOD Principles</h2>
    <IntroNarrative />
   </div>
   </div>

   <div class="wordcloud">
   <div class="section">
    <h2 class="section-heading section-heading--subsection">How is state funding currently going towards these principles?</h2>

    <p>Established in 2021, the <strong>MBTA Communities Act</strong> encourages communities near transit to create more opportunities for multi-family housing. The goal is to support vibrant, walkable neighborhoods where more people can live close to trains, buses, jobs, shops, and other daily needs.
        Under the Act, cities and towns served by the MBTA are asked to plan for a certain amount of multi-family housing based on their proximity to transit. </p>
    <p>Communities that meet these requirements <strong>remain eligible for important state funding programs that can help support transit-oriented development</strong> and other local improvements. Learn more about the act <a href="https://www.mass.gov/info-details/multi-family-zoning-requirement-for-mbta-communities#what-is-an-mbta-community" target="_blank" style="text-decoration:none; color:black;">here &#8599;</a>, and more on how some of that state funding is being used by awardee communities below </p>

    <WordCloud csvUrl='wordcloud-classified-data.csv' />
    <p class="transition"> After seeing how some communities are already benefitting, you might be wondering what TOD investment could have in store for others. Let's explore <span class="arrow">&darr;</span></p>
    </div>
    </div>

    <div class="intro_map">
    <div class="section">
    <h2 class="section-heading section-heading--subsection" style="margin-top:3rem">What communities might benefit most from TOD?</h2>
    <IntroMap/>
    </div>
    </div>

    <div class="map">
    <div class="section">
        <h2 class="section-heading section-heading--subsection" style="margin-top:3rem">Explore on your own</h2>
        <TabSection/>
    </div>
    </div>

    <div class="footer">
    <hr  style="margin-top:10rem">
        <p class="refs">This project was developed with guidance and feedback from the <a href=https://www.mapc.org target=blank>Metropolitan Area Planning Commission (MAPC)</a></p>
        
        <div class="citations">
            <div>
                <p class="refTitle">DATA SOURCES</p>
                <ul style="text-align:left;">
                    <li>MassDot & MBTA Transit-Oriented Development (TOD) Policies and Guidelines <a class="arrow-link" href="https://mbtarealty.com/wp-content/uploads/2022/09/MassDOT-and-MBTA-TOD-Policy-20170619.pdf" target="_blank">&#8599;</a></li>
                    <li>U.S. Census Bureau and American Community Surveys <a class="arrow-link" href="https://data.census.gov/" target="_blank">&#8599;</a></li>
                    <li>Property Tax Parcels Land Use <a class="arrow-link" href="https://www.mass.gov/info-details/massgis-data-property-tax-parcels" target="_blank">&#8599;</a> </li>
                    <li>Massachusetts Property Types Classification Code <a class="arrow-link" href="https://www.mass.gov/files/documents/2016/08/wr/classificationcodebook.pdf" target="_blank">&#8599;</a> </li>
                </ul>
            </div>
            <div>
                    <p class="refTitle">TEAM</p>
                    <ul style="text-align:left;">
                        <li>Avril Matute Cruz · Eri-ife Olayinka · Esther Magbagbeola · Jabes Gallardo</li>
                    </ul>
            </div>
                
            
        </div>
        <p class="refs" style="font-weight:bold; margin-bottom:20px; margin-top:20px;">6.C35/C85 MIT Interactive Visualization & Society | Spring 2026</p>
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
     background-color: #ffffff; /* Replace with your desired color */
     margin: 0;
     font-family: 'Inter', sans-serif;
    }
    
    .transition {
        font-family:'DotFont', sans-serif;
        margin-top:100px; 
        color:darkgrey;
        max-width: 100%;
        text-align: center;
    }
    @keyframes bounce {
        0%, 20%, 50%, 80%, 100% {
            transform: translateY(0); /* Return to original position */
        }
        40% {
            transform: translateY(-20px); /* Bounce up */
        }
        60% {
            transform: translateY(-10px); /* Smaller bounce up */
        }
        }
    
    .arrow {
        animation: bounce 2s infinite;
        display: inline-block;
    }


    .arrow-link {
    text-decoration: none;
    font-weight:bolder;
    font-size: 18px;
    color: rgb(255, 213, 0);
    transition: 0.3s;
  }
  .arrow-link:hover {
    color: rgb(139, 134, 0);
    padding-left: 10px; /* Moves the arrow slightly when hovered */
  }

    ul {
        list-style:none;
        padding: 0;
    }

    li {
        margin-bottom: 5px;
    }


    .refTitle {
        text-align: center;
        font-weight: bold;
        color:rgb(0, 0, 0); 
        font-size: 1rem;
        text-align: left;
        font-family: 'DotFont', sans-serif;
    }
    .citations {
        display: grid;
        grid-template-columns: 2fr 1fr;
        gap: 10px;
        max-width: 50%;
        /* place-items: center; */
        margin: 0 auto;
        color:rgb(158, 158, 158); 
        font-size: 1rem;
        margin-top: 25px;
    }
    hr {
        width: 75%;
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
        height: 100vh;
        padding-bottom: 50px;
    }

    .scrolly {
        background-color: #fcfbec;
        padding-bottom: 50px;

    }
    .wordcloud {
        background-color:#ffffff;
        margin-bottom: 50px;
        /* height: 100vh; */
    }
    .intro_map {
        background-color:#fcfbec;
        /* height: 100vh; */
    }

    .map {
        background-color:#ffffff;
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
        margin-bottom: 3rem;
        margin-top:2rem;
    }

    p {
        max-width: var(--content-width);
        margin: 0 auto;
        font-size: clamp(1.05rem, 2vw, 1.35rem);
        line-height: 1.6;
        text-align: left;
        color: rgb(59, 59, 59);
    }

    


</style>
