<script lang="ts">
	import { onMount } from 'svelte';

  export let username: string = "kalitsune";

  let data = {};
  onMount(() => {
    fetch(`https://status.cafe/users/${username}/status.json`)
    .then( r => r.json() )
    .then( r => data = r)
  });
</script>
<div class="statuscafe">
  <div>
    <a href="https://status.cafe/users/{username}" target="_blank">{data.author || username}</a>
    {data.face || "☕"} {data.timeAgo || "seconds ago"}
  </div>
  <div>{data.content || "I may be depressed but at least I have an iced coffe."}</div>
  <div class="statusball ballone"></div>
  <div class="statusball balltwo"></div>
  <div class="statusball ballthree"></div>
</div>
<style>
.statuscafe {
  background-color: var(--surface-1);
  border-radius: var(--radius-4);
  width: var(--size-13);
  position: relative;
  right: -2rem;
  bottom: -1.5rem;
  padding: var(--size-3);
  filter: drop-shadow(1px 1px var(--border)) drop-shadow(-1px -1px var(--border)) drop-shadow(1px -1px var(--border)) drop-shadow(-1px 1px var(--border)) drop-shadow(1px 0px var(--border)) drop-shadow(0 1px var(--border)) drop-shadow(-1px 0px var(--border)) drop-shadow(0px -1px var(--border));
}

@media (min-width: 768px) {
  .statuscafe {
      right: auto;
      bottom: -1rem;
      width: var(--size-14);
  }
}

.statusball {
  background-color: var(--surface-1);
  border-radius: var(--radius-round);
  position: absolute;
}

.ballone {
  width: var(--size-6);
  height: var(--size-6);
  right: 1.5rem;
}

.balltwo {
  width: var(--size-4);
  height: var(--size-4);
  right: 3.25rem;
  bottom: -1.75rem;
}

.ballthree {
  width: var(--size-2);
  height: var(--size-2);
  right: 5rem;
  bottom: -2rem;
}
</style>
