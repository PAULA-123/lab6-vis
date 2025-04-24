<script>
  import projects from "$lib/projects.json";
  import Project from "$lib/Project.svelte";


  // let profileData = fetch("https://api.github.com/users/juan1t0");

  import { onMount } from "svelte";

  let githubData = null;
  let loading = true;
  let error = null;

  onMount(async () => {
      try {
          const response = await fetch("https://api.github.com/users/PAULA-123");
          githubData = await response.json();
      } catch (err) {
          error = err;
      }
      loading = false;
  });


</script>

<svelte:head>
  <title>Mike Wazowski: Personal site and portfolio</title>
</svelte:head>
<h1> Mike Wazowski</h1>
   
<img src="./images/mike.png" alt="mike" width="500px">

<p>Michael "Mike" Thomas Wazowski is the deuteragonist of the 2001 Disney Pixar animated film Monsters, Inc. and the protagonist of its 2013 prequel.
    Depicted as a diminutive, one-eyed monster with a wisecracking veneer, Mike is an employee of Monsters, Incorporated, where he works closely with his longtime partner/best friend Sulley.
    Mike's world gets turned upside down when a human girl (nicknamed "Boo") enters the monster world.
    Teaming up with Sulley to return Boo to her world, Mike uncovers a company conspiracy and helps solve an energy crisis that plagues the entire city of Monstropolis
</p>

{#if loading}
    <p>Loading...</p>
{:else if error}
    <p class="error">Something went wrong: {error.message}</p>
{:else}
    <section>
        <h2>My GitHub Stats</h2>
        <dl>
            <dt>Followers</dt>
            <dd>{githubData.followers}</dd>
            <dt>Following</dt>
            <dd>{githubData.following}</dd>
            <dt>Public Repositories</dt>
            <dd>{githubData.public_repos}</dd>
        </dl>
    </section>
{/if}


<h2>
  Latest Projects
</h2>
<div class="projects">
{#each projects.slice(0, 3) as p}
  <Project data={p} hLevel="3"/>
{/each}
</div>



<style>
  dl{
    display: grid;
    grid-template-columns: auto;
  }
  dt{
    grid-row: 1;
    font-family: inherit;
    font-weight: bold;
    color: var(--border-gray);
    text-transform: uppercase;
  }
  dd{
    font-family: inherit;
    font-weight: bold;
  }
  section{
    border-width:0.15em;
  	border-style:solid;
	  border-color:var(--border-gray);
    padding-left: 1em;
    padding-right: 1em;
  }
</style>