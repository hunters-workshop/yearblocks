<script context="module">
  import { t } from "svelte-i18n";
  import { getEventSeries, resolveAddressObject } from "$lib/flow/actions";

  export const prerender = true;

  export async function load({ url, params, stuff }) {
    // fix app.css.map
    if (String(params.seriesId).endsWith("map")) {
      return { status: 200 };
    }

    const resolvedNameObject = await resolveAddressObject(params.address);
    let response;
    try {
      response = await getEventSeries(
        resolvedNameObject.address,
        params.seriesId
      );
    } catch (err) {}
    return {
      status: 200,
      props: {
        resolvedNameObject,
        preloadEventSeries: response,
      },
      stuff: {
        title: response?.basics?.name + " | EventSeries by " + params.address,
        description: response?.basics?.description,
        author: response?.identifier?.host,
        removeTitleSuffix: true,
      },
    };
  }
</script>

<script>
  import { page } from "$app/stores";
  import SeriesCardDisplay from "$lib/components/eventseries/elements/SeriesCardDisplay.svelte";
  import SeriesSummary from "$lib/components/eventseries/partials/SeriesSummary.svelte";
  import SeriesTreasury from "$lib/components/eventseries/partials/SeriesTreasury.svelte";
  import SeriesSettings from "$lib/components/eventseries/partials/SeriesSettings.svelte";
  import { seriesTab } from "$lib/stores";
  import { getResolvedName } from "$lib/flow/utils";
  import { user } from "$lib/flow/stores";

  export let resolvedNameObject;
  export let preloadEventSeries;

  seriesTab.set("summary");

  $: isIndexPage = $seriesTab === "summary";
  $: isRewardsPage = $seriesTab === "rewards";
  $: isSettingsPage = $seriesTab === "settings";

  // merged eventSeries
  $: eventSeries = localEventSeries ?? preloadEventSeries;

  let localEventSeries;
  async function refreshEventSeries() {
    localEventSeries = await getEventSeries(
      resolvedNameObject.address,
      $page.params.seriesId
    );
  }
</script>

<div class="container">
  <article>
    {#if !resolvedNameObject?.address}
      <p>
        {$t("common.hint.invalid-address", {
          values: {
            address: $page.params.address,
          },
        })}
      </p>
    {:else if !eventSeries}
      <p>
        {$t("errors.challenges.failed-to-load", {
          values: {
            id: $page.params.seriesId,
            host: $page.params.host,
          },
        })}
      </p>
    {:else}
      <header>
        <SeriesCardDisplay
          eventSeriesData={{
            ...eventSeries,
            owner: getResolvedName(resolvedNameObject),
          }}
        />
      </header>
      <ul class="tabs">
        <li
          on:click={function () {
            if (!isIndexPage) {
              seriesTab.set("summary");
            }
          }}
          class:animatedlink={!isIndexPage}
          class:selected={isIndexPage}
        >
          {$t("common.tabs.summary")}
        </li>
        <li
          on:click={function () {
            if (!isRewardsPage) {
              seriesTab.set("rewards");
            }
          }}
          class:animatedlink={!isRewardsPage}
          class:selected={isRewardsPage}
        >
          {$t("common.tabs.rewards")}
        </li>
        {#if $user?.addr == resolvedNameObject.address}
          <li
            on:click={function () {
              if (!isSettingsPage) {
                seriesTab.set("settings");
              }
            }}
            class:animatedlink={!isSettingsPage}
            class:selected={isSettingsPage}
          >
            {$t("common.tabs.settings")}
          </li>
        {/if}
      </ul>

      {#if isIndexPage}
        <SeriesSummary {eventSeries} on:seriesUpdated={refreshEventSeries} />
      {:else if isRewardsPage}
        <SeriesTreasury {eventSeries} on:seriesUpdated={refreshEventSeries} />
      {:else if isSettingsPage && $user?.addr == resolvedNameObject.address}
        <SeriesSettings {eventSeries} on:seriesUpdated={refreshEventSeries} />
      {/if}
    {/if}
  </article>
</div>

<!-- FIXME: COPY FROM [address]/index, maybe set in app.css is better  -->
<style>
  .animatedlink {
    display: inline-block;
    position: relative;
  }

  .animatedlink:after {
    content: "";
    position: absolute;
    width: 100%;
    transform: scaleX(0);
    height: 2px;
    bottom: 0;
    left: 0;
    background-color: var(--primary);
    transform-origin: bottom right;
    transition: transform 0.25s ease-out;
  }

  .animatedlink:hover:after {
    transform: scaleX(1);
    transform-origin: bottom left;
  }

  .selected {
    color: var(--primary);
    border-bottom: 2px solid var(--primary);
  }

  .tabs {
    display: flex;
    justify-content: space-around;
  }

  .tabs li {
    list-style-type: none;
    font-size: 18px;
    text-transform: uppercase;
    font-weight: bold;
    cursor: pointer;
    padding: 0px 10px;
  }

  .tabs li.selected {
    cursor: default;
  }

  @media screen and (max-width: 767px) {
    .tabs {
      margin: 0px;
    }
    .tabs li {
      font-size: 15px;
    }
  }
</style>
