<script>
  import IconButton from "@smui/icon-button";
  import Button, { Label } from "@smui/button";
  import Checkbox from "@smui/checkbox";
  import Menu from "@smui/menu";
  import Select, { Option } from "@smui/select";
  import List, { Item, Separator, Text } from "@smui/list";
  import { mdiPlus, mdiTrashCan, mdiHelpCircleOutline, mdiSort } from "@mdi/js";
  import lodashUniq from "lodash/uniq";
  import lodashOrderBy from "lodash/orderBy";
  import { flip } from "svelte/animate";
	import { quintOut } from 'svelte/easing';
	import { crossfade } from 'svelte/transition';
  import {
    addFilter,
    removeFilter,
    selectedProfile,
    commitChange
  } from "../js/datasource";
  import { DISABLED_COLOR, PRIMARY_COLOR } from "../js/constants";
  import AutoComplete from "./Autocomplete.svelte";
  import MdiIcon from "./MdiIcon.svelte";
  import ResourceTypeMenu from "./ResourceTypeMenu.svelte";

  let selectedFilter;
  let dialog;
  let sortMenu;
  let clazz;
  let resourceTypeMenuLocation;
  export { clazz as class };

  function expandEditor(filter) {
    selectedFilter = filter;
    dialog.open();
  }

  function sort(field, order) {
    commitChange({
      filters: lodashOrderBy(filters, [field], [order])
    });
  }

  function openLink(link) {
    chrome.tabs.create({ url: link });
  }

  function toggleAll() {
    if (!allChecked) {
      filters.forEach(f => (f.enabled = true));
    } else {
      filters.forEach(f => (f.enabled = false));
    }
  }

  function refreshFilters() {
    commitChange({ filters });
  }

	const [send, receive] = crossfade({
		duration: d => Math.sqrt(d * 200),

		fallback(node, params) {
			const style = getComputedStyle(node);
			const transform = style.transform === 'none' ? '' : style.transform;

			return {
				duration: 600,
				easing: quintOut,
				css: t => `
					transform: ${transform} scaleY(${t});
					opacity: ${t}
				`
			};
		}
  });

  $: filters = $selectedProfile.filters;
  $: allChecked = filters.every(f => f.enabled);
  $: allUnchecked = filters.every(f => !f.enabled);
  $: knownUrlRegexes = lodashUniq(
    filters.map(f => f.urlRegex).filter(n => !!n)
  );
  $: knownFilterComments = lodashUniq(
    filters.map(f => f.comment).filter(n => !!n)
  );
</script>

<style scoped>
  :global(.filter-select) {
    height: 26px;
    flex-basis: 170px;
    flex-shrink: 0;
    margin-right: 5px;
  }

  :global(.filter-select-field) {
    width: 170px;
    font-size: 14px;
    margin: 0;
    padding: 0;
    height: 26px;
    border-bottom: 1px solid #ddd !important;
  }

  :global(.filter-select) :global(.mdc-select__dropdown-icon) {
    bottom: 0px;
  }

  :global(.data-table-value-cell) {
    width: 350px;
  }
</style>

<div class="data-table {clazz}">
  <div class="data-table-row data-table-title-row">
    <Checkbox
      class="data-table-cell flex-fixed-icon"
      bind:checked={allChecked}
      indeterminate={!allChecked && !allUnchecked}
      on:click={toggleAll}
      disabled={$selectedProfile.filters.length === 0} />
    <h3 class="data-table-title data-table-cell flex-grow">Filters</h3>
    <div class="data-table-cell">
      <Button on:click={() => addFilter()} class="small-text-button">
        <MdiIcon size="20" icon={mdiPlus} color={PRIMARY_COLOR} middle />
        Add
      </Button>  
    </div>
    <div class="data-table-cell">
      <Button
        on:click={e => {
          sortMenu.setFixedPosition(true);
          const rect = e.target.getBoundingClientRect();
          sortMenu.setAbsolutePosition(rect.left + window.scrollX, e.target.offsetParent.offsetTop + rect.top + window.scrollY);
          sortMenu.setOpen(true);
        }}
        disabled={$selectedProfile.filters.length === 0}
        class="small-text-button">
        <MdiIcon
          size="20"
          icon={mdiSort}
          color={$selectedProfile.filters.length === 0 ? DISABLED_COLOR : PRIMARY_COLOR}
          middle />
        Sort
      </Button>
      <Menu bind:this={sortMenu} quickOpen>
        <List>
          <Item on:SMUI:action={() => sort('type', 'asc')}>
            <Text>Type - ascending</Text>
          </Item>
          <Item on:SMUI:action={() => sort('type', 'desc')}>
            <Text>Type - descending</Text>
          </Item>
          <Item on:SMUI:action={() => sort('urlRegex', 'asc')}>
            <Text>URL regex - ascending</Text>
          </Item>
          <Item on:SMUI:action={() => sort('urlRegex', 'desc')}>
            <Text>URL regex - descending</Text>
          </Item>
          {#if !$selectedProfile.hideComment}
            <Item on:SMUI:action={() => sort('comment', 'asc')}>
              <Text>Comment - ascending</Text>
            </Item>
            <Item on:SMUI:action={() => sort('comment', 'desc')}>
              <Text>Comment - descending</Text>
            </Item>
          {/if}
        </List>
      </Menu>
    </div>
    <div class="data-table-cell">
      <Button
        on:click={() => commitChange({ filters: [] })}
        disabled={$selectedProfile.filters.length === 0}
        class="small-text-button">
        <MdiIcon
          size="20"
          icon={mdiTrashCan}
          color={$selectedProfile.filters.length === 0 ? DISABLED_COLOR : 'red'}
          middle />
        Clear
      </Button>
    </div>
  </div>
  {#each filters as filter, filterIndex (filterIndex)}
    <div 
      in:receive="{{key: filterIndex}}"
      animate:flip
      class="data-table-row {filter.enabled ? '' : 'data-table-row-unchecked'}">
      <Checkbox bind:checked={filter.enabled} indeterminate={false} class="data-table-cell flex-fixed-icon" />
      <Select
        bind:value={filter.type}
        class="data-table-cell filter-select"
        input$class="filter-select-field"
        on:change={refreshFilters}>
        <Option value="urls" selected={filter.type === 'urls'}>
          URL Pattern
        </Option>
        <Option
          value="excludeUrls"
          selected={filter.type === 'excludeUrls'}>
          Exclude URL Pattern
        </Option>
        <Option value="types" selected={filter.type === 'types'}>
          Resource Type
        </Option>
      </Select>
      {#if filter.type === 'urls' || filter.type === 'excludeUrls'}
        <AutoComplete
          className="mdc-text-field__input filter-text-field data-table-cell flex-grow"
          items={knownUrlRegexes}
          bind:value={filter.urlRegex}
          bind:selectedItem={filter.urlRegex}
          on:change={refreshFilters}
          placeholder=".*://.*.google.com/.*" />
      {:else}
        <ResourceTypeMenu
          bind:resourceType={filter.resourceType}
          {resourceTypeMenuLocation}
          on:change={refreshFilters} />
      {/if}
      {#if !$selectedProfile.hideComment}
        <AutoComplete
          className="mdc-text-field__input data-table-cell flex-grow"
          items={knownFilterComments}
          bind:value={filter.comment}
          bind:selectedItem={filter.comment}
          on:change={refreshFilters}
          placeholder="Comment" />
      {/if}
      <IconButton
        dense
        aria-label="Delete"
        class="small-icon-button data-table-cell flex-fixed-icon"
        on:click={() => removeFilter(filterIndex)}>
        <MdiIcon size="24" icon={mdiTrashCan} color="red" />
      </IconButton>
    </div>
  {/each}
</div>