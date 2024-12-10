<script lang="ts">
  import Editor from "./Editor.svelte";
  import Tab from "./Tab.svelte";

  let tabs = ["Untitled.txt", "Untitled2.txt"];
  let files_data = ["This is the first file", "This is the second file"];
  let active_cursors = [
    { top: 0, left: 0 },
    { top: 0, left: 0 },
  ];
  let active_tab = 0;
</script>

<div class="container">
  <div class="tabs">
    {#each tabs as tab, i}
      <Tab
        file_name={tab}
        active={active_tab == i}
        on_click={() => {
          active_tab = i;
        }}
      />
    {/each}
  </div>
  {#each files_data as file_data, i}
    {#if active_tab == i}
      <Editor
        {file_data}
        save_data={(data) => {
          files_data[i] = data;
        }}
        cursor={active_cursors[i]}
        update_cursor={(cursor) => {
          active_cursors[i] = cursor;
        }}
      />
    {/if}
  {/each}
</div>

<style>
  .tabs {
    display: flex;
    flex-direction: row;
  }
</style>
