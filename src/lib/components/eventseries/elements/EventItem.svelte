<script>
  import { createEventDispatcher } from "svelte";
  import { resolveAddressObject, getEvent } from "$lib/flow/actions";
  import Badge from "../svgs/badge.svelte";
  import PrimaryTag from "./PrimaryTag.svelte";

  // dispatcher
  const dispatch = createEventDispatcher();

  // For empty part
  export let empty = false;

  // for none empty part
  export let preview = true;
  /** @type {import('../types').EventSeriesSlot} */
  export let item = { event: null, required: false };
  /** for pending slots */
  export let pending = false;
  /** for ghost slots */
  export let ghost = false;
  /** for owned items */
  export let owned = false;

  $: itemRequired = item.required ?? false;

  /** @type {import('../types').FloatEvent} */
  let eventData = null;

  /** @type {Promise<import('../types').FloatEvent>} */
  const floatEventCallback = async () => {
    if (!item.event) return null;

    let hostAddress;
    if (item.event?.host.startsWith("0x")) {
      hostAddress = item.event?.host;
    } else {
      let resolvedNameObject = await resolveAddressObject(item.event?.host);
      hostAddress = resolvedNameObject.address;
    }
    eventData = await getEvent(hostAddress, String(item.event?.id));
    if (!eventData) {
      return null;
    }
    return eventData;
  };

  const handleClick = (e) => {
    if (!empty) {
      if (item.event) {
        dispatch("clickItem", {
          host: item.event?.host,
          id: item.event?.id,
        });
      }
    } else {
      dispatch("clickEmpty", e.detail);
    }
  };
</script>

<article class="card-item" class:ghost class:pending on:click={handleClick}>
  {#if !empty}
    {#await floatEventCallback()}
      <div class="content center" aria-busy="true" />
    {:then floatEvent}
      <div class="content">
        {#if !floatEvent}
          <h3 class="empty unclaimed">Empty<br />Slot</h3>
        {:else}
          <img
            src="https://nftstorage.link/ipfs/{floatEvent.image}"
            alt="{floatEvent.name} Image"
            class:unclaimed={!preview && !owned}
          />
          <a
            class="no-style"
            href="/{item.event?.host}/event/{floatEvent.eventId}"
            target="_blank"
            data-tooltip="#{floatEvent.eventId} by {item.event?.host}"
          >
            <h3 class:unclaimed={!preview && !owned}>{floatEvent.name}</h3>
          </a>
        {/if}
      </div>
    {/await}
  {:else}
    <div class="content center pointer" on:click={handleClick}>
      <svg
        xmlns="http://www.w3.org/2000/svg"
        viewBox="0 0 20 20"
        fill="currentColor"
      >
        <path
          fill-rule="evenodd"
          d="M10 3a1 1 0 011 1v5h5a1 1 0 110 2h-5v5a1 1 0 11-2 0v-5H4a1 1 0 110-2h5V4a1 1 0 011-1z"
          clip-rule="evenodd"
        />
      </svg>
    </div>
  {/if}

  {#if !empty}
    <span class="badge">
      <Badge {owned} />
    </span>
  {/if}
  {#if itemRequired}
    <div class="primary" class:unclaimed={!preview && !owned}>
      <PrimaryTag full={true} />
    </div>
  {/if}
</article>

<slot name="after" {eventData} />

<style>
  .card-item {
    position: relative;

    width: 96px;
    height: 140px;

    padding: 8px 8px 20px 8px;
    margin: 0;

    border: 1px solid transparent;
    border-radius: 10px;
  }

  .card-item.ghost {
    opacity: 0.2;
  }

  .card-item.pending {
    border: 1px dotted var(--primary);
  }

  .card-item:hover {
    border: 1px solid var(--primary);
  }

  .card-item > .content > * {
    max-width: 80px;
    max-height: 80px;
  }

  .card-item > .content {
    display: flex;
    flex-direction: column;
    justify-content: end;
    align-items: center;
    text-align: center;

    width: 100%;
    height: 100%;
  }
  .card-item > .content.pointer {
    cursor: pointer;
  }

  .card-item > .content.center {
    justify-content: center;
  }

  .card-item > .content.center > svg {
    width: 64px;
    height: 64px;
  }

  .card-item h3 {
    font-size: 0.8rem;
    flex-grow: 1;

    text-overflow: ellipsis;
    overflow: hidden;
    white-space: nowrap;
  }

  .card-item img {
    display: block;
    margin: 0 auto;
    margin-bottom: 6px;
  }

  .card-item .badge {
    display: block;
    position: absolute;
    top: -12px;
    left: -4px;
  }

  .card-item .primary {
    display: block;
    width: 100%;
    position: absolute;
    bottom: -2px;
    left: 0;
    padding: 0;
  }

  .unclaimed {
    opacity: 0.3;
  }
</style>
