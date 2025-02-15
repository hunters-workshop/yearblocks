<script>
  import { t } from "svelte-i18n";
  import {
    user,
    eventCreationInProgress,
    eventCreatedStatus,
  } from "$lib/flow/stores";
  import {
    authenticate,
    createEvent,
    getGroups,
    isSharedWithUser,
  } from "$lib/flow/actions";

  import { draftFloat, theme } from "$lib/stores";
  import { PAGE_TITLE_EXTENSION } from "$lib/constants";
  import { notifications } from "$lib/notifications";

  // import LibLoader from "$lib/components/LibLoader.svelte";
  // import { onMount } from "svelte";
  import Float from "$lib/components/Float.svelte";
  import { getKeysFromClaimCode } from "$lib/flow/utils";
  import { slide } from "svelte/transition";
  import Loading from "$lib/components/common/Loading.svelte";
  import { NFTStorage } from "nft.storage";

  const NFT_STORAGE_TOKEN = import.meta.env.VITE_NFT_STORAGE_ACCESS_TOKEN;
  const client = new NFTStorage({ token: NFT_STORAGE_TOKEN });

  let timezone = new Date()
    .toLocaleTimeString("en-us", { timeZoneName: "short" })
    .split(" ")[2];
  /* States related to image upload */
  // let ipfsIsReady = false;
  let uploading = false;
  // let uploadingPercent = 0;
  let uploadedSuccessfully = false;

  let imagePreviewSrc = null;

  let advancedOptions = false;

  // onMount(() => {
  //   ipfsIsReady = window?.IpfsHttpClient ?? false;
  // });

  let uploadToIPFS = async (e) => {
    uploading = true;
    // uploadingPercent = 0;

    let file = e.target.files[0];

    // function progress(len) {
    //   uploadingPercent = len / file.size;
    // }

    // const client = window.IpfsHttpClient.create({
    //   url: "https://api.pinata.cloud/psa",
    //   repo: "file-path" + Math.random(),
    // });

    const cid = await client.storeBlob(file);
    uploadedSuccessfully = true;
    uploading = false;
    $draftFloat.ipfsHash = cid;
    imagePreviewSrc = `https://nftstorage.link/ipfs/${cid}`;
    // imagePreviewSrc = `https://nftstorage.link/ipfs/${cid}`; // if CF is slow, use <-
  };

  // function ipfsReady() {
  //   console.log("ipfs is ready");
  //   ipfsIsReady = true;
  // }

  let minter = null;
  let claimCode = null;
  let certificateCode = "";
  async function initCreateFloat() {
    // check if all required inputs are correct
    let canCreateFloat = await checkInputs();

    if (!canCreateFloat) {
      return;
    }
    if (claimCode) {
      const { publicKey } = getKeysFromClaimCode(claimCode);
      $draftFloat.secretPK = publicKey;
    }

    createEvent(minter || $user?.addr, $draftFloat);
  }

  async function checkInputs() {
    let errorArray = [];
    let messageString = $t("errors.common.fields-missing");

    // add conditions here
    if (!$draftFloat.name) {
      errorArray.push($t("errors.events.name-missing"));
    }
    if (!$draftFloat.description) {
      errorArray.push($t("errors.events.desc"));
    }
    if (!imagePreviewSrc) {
      errorArray.push($t("errors.events.image"));
    }

    if ($draftFloat.challengeCertificate) {
      const parts = String(certificateCode).split(":");
      if (
        parts.length !== 3 ||
        !(parts[0].length === 18 && parts[0].startsWith("0x")) ||
        !(parseInt(parts[1]) > 0) ||
        !(parseInt(parts[2]) > 0)
      ) {
        messageString = $t("errors.events.invalid-certificate-code");
      } else {
        $draftFloat.challengeCertificate = certificateCode;
      }
    }

    if (minter) {
      let canMintFor = await isSharedWithUser(minter, $user?.addr);
      if (!canMintFor) {
        messageString = $t("errors.events.cannot-mint-for");
        errorArray.push(minter);
      }
    }

    if (errorArray.length > 0) {
      notifications.info(`${messageString}: ${errorArray.join(",")}`);
      return false;
    } else {
      return true;
    }
  }
</script>

<svelte:head>
  <title>Create a new {PAGE_TITLE_EXTENSION}</title>
</svelte:head>

<!-- 
<LibLoader
  url="https://cdn.jsdelivr.net/npm/ipfs-http-client@56.0.0/index.min.js"
  on:loaded={ipfsReady}
  uniqueId={+new Date()} />
-->

<div class="container">
  <article>
    <h1 class="mb-1" style="text-align: center;">Create a new YearBlock</h1>

    <label for="name">
      YearBlock Name
      <input type="text" id="name" name="name" bind:value={$draftFloat.name} />
    </label>

    <label for="name">
      YearBlock URL
      <input type="text" id="name" name="name" bind:value={$draftFloat.url} />
    </label>

    <label for="description">
      YearBlock Description
      <textarea
        id="description"
        name="description"
        bind:value={$draftFloat.description}
      />
    </label>

    {#await getGroups($user?.addr)}
      <Loading />
    {:then groupsWeCanAddTo}
      {#if Object.keys(groupsWeCanAddTo).length > 0}
        <div class="input-group">
          <label for="groups">Add Event to Group</label>
          <div class="input-button-group" id="names" name="names">
            <select
              bind:value={$draftFloat.initialGroup}
              id="addToGroup"
              required
            >
              {#each Object.keys(groupsWeCanAddTo) as groupName}
                <option value={groupName}>{groupName}</option>
              {/each}
            </select>
          </div>
        </div>
      {/if}
    {/await}

    <br />

    <!-- {#if ipfsIsReady} -->
    <label for="image">
      Event Image
      <input
        aria-busy={!!uploading}
        on:change={function (e) {
          uploadToIPFS(e);
        }}
        type="file"
        id="image"
        name="image"
        accept="image/png, image/gif, image/jpeg"
      />
      {#if uploading}
        <progress indeterminate />
      {/if}

      {#if uploadedSuccessfully}
        <small>✓ Image uploaded successfully to IPFS!</small>
      {/if}
    </label>
    <!--
    {:else}
      <p>IPFS not loaded</p>
    {/if}  -->

    <!-- <div class="alert alert-danger text-center info">
      <strong>IMPORTANT!</strong> <br />Our web3 image pinning provider is
      currently broken (Infura). <br />
      We are working on a solution/alternative. We will remove this warning once
      we resolve the issue.
    </div> -->

    {#if imagePreviewSrc}
      <h3>Preview</h3>
      <Float
        float={{
          eventName: $draftFloat.name,
          eventImage: $draftFloat.ipfsHash,
          totalSupply: "SERIAL_NUM",
          eventHost: $user?.addr || "0x0000000000",
        }}
      />
      <div class="mb-2" />
    {/if}

    <!-- 
      
      Claimable: Yes vs No (No = host must distribute manually, similar to custom above; Yes = quantity, time and claimcode appears)
      Quantity: UNLIMITED vs LIMITED (toggles FLOAT quantity limit input)
      Time: UNLIMITED vs LIMITED (toggles start /end time inputs)
      Requires Claim Code: Yes vs No (btw so are we going with hash or code after the event?) 
    -->
    <h3 class="mb-1">Configure your Yearblock</h3>

    <h5>Can be changed later.</h5>
    <!-- GIFTABLE -->
    <div class="grid no-break mb-1">
      <button
        class:secondary={!$draftFloat.transferrable}
        class="outline"
        on:click={() => ($draftFloat.transferrable = true)}
      >
        Tradeable
        <span>This Yearblock can be traded and/or transferred.</span>
      </button>
      <button
        class:secondary={$draftFloat.transferrable}
        class="outline"
        on:click={() => {
          $draftFloat.transferrable = false;
        }}
      >
        Soulbound
        <span>
          This Yearblock <strong>cannot</strong> be traded (soulbound).
        </span>
      </button>
    </div>

    <div class="grid no-break mb-1">
      <button
        class:secondary={!$draftFloat.claimable}
        class="outline"
        on:click={() => ($draftFloat.claimable = true)}
      >
        Claimable
        <span>
          Users can mint their own Yearblock based on the parameters defined below.
        </span>
      </button>
      <button
        class:secondary={$draftFloat.claimable}
        class="outline"
        on:click={() => ($draftFloat.claimable = false)}
      >
        Not Claimable
        <span
          >You will be responsible for distributing the Yearblock to accounts.</span
        >
      </button>
    </div>

    <!-- {#if !$draftFloat.claimable}
      <h5>This is how you would distribute your FLOAT to a user in Cadence:</h5>
      <xmp class={$theme === "light" ? "xmp-light" : "xmp-dark"}
        >{distributeCode}</xmp>
    {/if} -->

    <h5>Cannot be changed later.</h5>
    <!-- QUANTITY -->
    <div class="grid no-break mb-1">
      <button
        class:secondary={$draftFloat.quantity}
        class="outline"
        on:click={() => ($draftFloat.quantity = false)}
      >
        Unlimited Quantity
        <span>
          Select this if you don't want your YearBlock to have a limited quantity.
        </span>
      </button>
      <button
        class:secondary={!$draftFloat.quantity}
        class="outline"
        on:click={() => ($draftFloat.quantity = true)}
      >
        Limited Quantity
        <span>
          You can set the maximum number of times the YearBlock can be minted.
        </span>
      </button>
    </div>
    {#if $draftFloat.quantity}
      <label for="quantity">
        How many can be claimed?
        <input
          type="number"
          name="quantity"
          bind:value={$draftFloat.quantity}
          min="1"
          placeholder="ex. 100"
        />
      </label>
      <hr />
    {/if}

    <!-- TIME -->
    <div class="grid no-break mb-1">
      <button
        class:secondary={$draftFloat.timelock}
        class="outline"
        on:click={() => ($draftFloat.timelock = false)}
      >
        No Time Limit
        <span>Can be minted at any point in the future.</span>
      </button>
      <button
        class:secondary={!$draftFloat.timelock}
        class="outline"
        on:click={() => ($draftFloat.timelock = true)}
      >
        Time Limit
        <span>Can only be minted between a specific time interval.</span>
      </button>
    </div>
    {#if $draftFloat.timelock}
      <div class="grid">
        <!-- Date -->
        <label for="start"
          >Start ({timezone})
          <input
            type="datetime-local"
            id="start"
            name="start"
            bind:value={$draftFloat.startTime}
          />
        </label>

        <!-- Date -->
        <label for="start"
          >End ({timezone})
          <input
            type="datetime-local"
            id="start"
            name="start"
            bind:value={$draftFloat.endTime}
          />
        </label>
      </div>
      <hr />
    {/if}

    <!-- SECRET -->
    <div class="grid no-break mb-1">
      <button
        class:secondary={$draftFloat.claimCodeEnabled}
        class="outline"
        on:click={() => ($draftFloat.claimCodeEnabled = false)}
      >
        No Secret Code
        <span>Your YearBlock can be minted without a secret code.</span>
      </button>
      <button
        class:secondary={!$draftFloat.claimCodeEnabled}
        class="outline"
        on:click={() => ($draftFloat.claimCodeEnabled = true)}
      >
        Use Secret Code
        <span>
          Your YearBlock can only be minted if people know the secret code.
        </span>
      </button>
    </div>
    {#if $draftFloat.claimCodeEnabled}
      <label for="claimCode">
        Enter a claim code (<i>case-sensitive</i>)
        <input
          type="text"
          name="claimCode"
          bind:value={claimCode}
          placeholder="ex. mySecretCode"
        />
      </label>
      <hr />
    {/if}

    <!-- MINIMUM BALANCE -->
    <div class="grid no-break mb-1">
      <button
        class:secondary={$draftFloat.minimumBalance}
        class="outline"
        on:click={() => ($draftFloat.minimumBalance = false)}
      >
        No Required Balance
        <span>You do not have to have a minimum $FLOW balance.</span>
      </button>
      <button
        class:secondary={!$draftFloat.minimumBalance}
        class="outline"
        on:click={() => {
          $draftFloat.minimumBalance = true;
        }}
      >
        Minimum Balance
        <span>This YearBlock requires a minimum $FLOW balance.</span>
      </button>
    </div>
    {#if $draftFloat.minimumBalance}
      <div class="grid">
        <label for="minimum"
          >Amount (in $FLOW)
          <input
            type="number"
            id="minimum"
            name="minimum"
            min="1.00"
            placeholder="ex. 10.0"
            step="0.01"
            bind:value={$draftFloat.minimumBalance}
          />
        </label>
      </div>
      <hr />
    {/if}

    <!-- FLOWTOKEN PURCHASE -->
    <div class="grid no-break mb-1">
      <button
        class:secondary={$draftFloat.flowTokenPurchase}
        class="outline"
        on:click={() => ($draftFloat.flowTokenPurchase = false)}
      >
        Free
        <span>This YearBlock is free for users to claim.</span>
      </button>
      <button
        class:secondary={!$draftFloat.flowTokenPurchase}
        class="outline"
        on:click={() => {
          $draftFloat.flowTokenPurchase = true;
        }}
      >
        Payment
        <span>
          This YearBlock costs $FLOW to claim. Suitable for things like tickets.
        </span>
      </button>
    </div>
    {#if $draftFloat.flowTokenPurchase}
      <div class="grid">
        <!-- Date -->
        <label for="cost">
          Cost (in $FLOW)
          <input
            type="number"
            id="cost"
            name="cost"
            min="1.00"
            placeholder="ex. 10.0"
            step="0.01"
            bind:value={$draftFloat.flowTokenPurchase}
          />
          <small>
            Note: 5% of each purchase will go to the YearBlock DAO treasury.
          </small>
        </label>
      </div>
      <hr />
    {/if}

    <!-- CHALLENGE CERTIFICATE -->
    <div class="grid no-break mb-1">
      <button
        class:secondary={$draftFloat.challengeCertificate}
        class="outline"
        on:click={() => ($draftFloat.challengeCertificate = false)}
      >
        {$t("events.common.verifier.label-challenge-false")}
        <span>{$t("events.common.verifier.label-challenge-false-desc")}</span>
      </button>
      <button
        class:secondary={!$draftFloat.challengeCertificate}
        class="outline"
        on:click={() => {
          $draftFloat.challengeCertificate = true;
        }}
      >
        {$t("events.common.verifier.label-challenge-true")}
        <span>
          {$t("events.common.verifier.label-challenge-true-desc")}
        </span>
      </button>
    </div>
    {#if $draftFloat.challengeCertificate}
      <label for="certificateCode">
        {$t("events.create.hint.enter-cert-code")}
        <input
          type="text"
          name="certificateCode"
          required
          bind:value={certificateCode}
        />
      </label>
      <hr />
    {/if}

    {#if advancedOptions}
      <div transition:slide>
        <h4 class="">Create on behalf of another account (shared minting)</h4>
        <div class="input-button-group">
          <input
            placeholder="0x00000000000"
            type="text"
            id="minters"
            name="minters"
            bind:value={minter}
          />
        </div>
        <p class="small mt-1">
          Input an address to create an event as that account. This will only
          work if that account has given you shared minting rights.
        </p>
      </div>
    {/if}
    <button
      class="secondary outline text-center mt-2"
      on:click={() => (advancedOptions = !advancedOptions)}
    >
      {advancedOptions ? "Hide" : "Show"} advanced options
    </button>

    <footer>
      {#if !$user?.loggedIn}
        <div class="mt-2 mb-2">
          <button class="contrast small-button" on:click={authenticate}>
            {$t("common.btn.connectWallet")}
          </button>
        </div>
      {:else if $eventCreationInProgress}
        <button aria-busy="true" disabled> Creating YearBlock </button>
      {:else if $eventCreatedStatus.success}
        <a
          role="button"
          class="d-block"
          href="/{$user.addr}/?tab=events"
          style="display:block"
        >
          Event created successfully!
        </a>
      {:else if !$eventCreatedStatus.success && $eventCreatedStatus.error}
        <button class="error" disabled>
          {$eventCreatedStatus.error}
        </button>
      {:else}
        <button on:click|preventDefault={initCreateFloat}>
          Create YearBlock
        </button>
      {/if}
    </footer>
  </article>
</div>

<style>
  h4 {
    margin: 0;
  }
  .outline {
    text-align: left;
  }

  .outline span {
    display: block;
    font-size: 0.75rem;
    line-height: 1.2;
    font-weight: 400;
    opacity: 0.6;
  }

  .small {
    font-size: 0.75rem;
    line-height: 1.2;
  }

  /* .image-preview {
    max-width: 150px;
    height: auto;
  }

  xmp {
    position: relative;
    width: 100%;
    font-size: 12px;
    padding: 10px;
    overflow: scroll;
    border-radius: 5px;
    color: white;
  }

  .xmp-dark {
    background: rgb(56, 232, 198, 0.25);
  }

  .xmp-light {
    background: rgb(27, 40, 50);
  }
 */
  h5 {
    margin-bottom: 5px;
  }

  .error {
    background-color: red;
    border-color: white;
    color: white;
    opacity: 1;
  }
</style>
