<script lang="ts">
  let { file_name, active, on_click } = $props<{
    file_name: string;
    active: boolean;
    on_click: () => void;
  }>();
</script>

<button onclick={on_click} class={`tab ${active ? `active` : ``}`}>
  {file_name}
</button>

<style>
  .tab {
    padding-top: 2vh;
    padding-bottom: 2vh;
    padding-left: 4vw;
    padding-right: 5vw;
    color: var(--text);
    background-color: var(--tab-color);
    cursor: pointer;
    border: none;
    outline: none;
  }
  .active {
    background-color: var(--tab-color-active);
  }
</style>
