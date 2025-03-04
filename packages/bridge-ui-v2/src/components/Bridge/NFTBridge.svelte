<script lang="ts">
  import { onDestroy, tick } from 'svelte';
  import { t } from 'svelte-i18n';
  import { type Address, isAddress } from 'viem';

  import { ImportMethod } from '$components/Bridge/types';
  import ActionButton from '$components/Button/ActionButton.svelte';
  import { Card } from '$components/Card';
  import { OnAccount } from '$components/OnAccount';
  import { OnNetwork } from '$components/OnNetwork';
  import { Step, Stepper } from '$components/Stepper';
  import { hasBridge } from '$libs/bridge/bridges';
  import { BridgePausedError } from '$libs/error';
  import { ETHToken, type NFT } from '$libs/token';
  import { fetchNFTImageUrl } from '$libs/token/fetchNFTImageUrl';
  import { getCanonicalInfoForToken } from '$libs/token/getCanonicalInfo';
  import { getTokenWithInfoFromAddress } from '$libs/token/getTokenWithInfoFromAddress';
  import { isBridgePaused } from '$libs/util/checkForPausedContracts';
  import { type Account, account } from '$stores/account';
  import { type Network, network } from '$stores/network';

  import type AddressInput from './AddressInput/AddressInput.svelte';
  import type Amount from './Amount.svelte';
  import type IdInput from './IDInput/IDInput.svelte';
  import ConfirmationStep from './NFTBridgeSteps/ConfirmationStep.svelte';
  import ImportStep from './NFTBridgeSteps/ImportStep.svelte';
  import RecipientStep from './NFTBridgeSteps/RecipientStep.svelte';
  import ReviewStep from './NFTBridgeSteps/ReviewStep.svelte';
  import type { ProcessingFee } from './ProcessingFee';
  import {
    activeBridge,
    destNetwork as destinationChain,
    recipientAddress,
    selectedNFTs,
    selectedToken,
    selectedTokenIsBridged,
  } from './state';
  import { NFTSteps } from './types';

  let amountComponent: Amount;
  let recipientStepComponent: RecipientStep;
  let processingFeeComponent: ProcessingFee;
  let importMethod: ImportMethod;
  let nftIdArray: number[] = [];
  let contractAddress: Address | string = '';
  let bridgingStatus: 'pending' | 'done' = 'pending';

  let hasEnoughEth: boolean = false;

  function onNetworkChange(newNetwork: Network, oldNetwork: Network) {
    updateForm();
    activeStep = NFTSteps.IMPORT;
    if (newNetwork) {
      const destChainId = $destinationChain?.id;
      if (!$destinationChain?.id) return;
      // determine if we simply swapped dest and src networks
      if (newNetwork.id === destChainId) {
        destinationChain.set(oldNetwork);
        return;
      }
      // check if the new network has a bridge to the current dest network
      if (hasBridge(newNetwork.id, $destinationChain?.id)) {
        destinationChain.set(oldNetwork);
      } else {
        // if not, set dest network to null
        $destinationChain = null;
      }
    }
  }

  const runValidations = () => {
    if (amountComponent) amountComponent.validateAmount();
    if (addressInputComponent) addressInputComponent.validateAddress();
    isBridgePaused().then((paused) => {
      if (paused) {
        throw new BridgePausedError();
      }
    });
  };

  function onAccountChange(account: Account) {
    updateForm();
    if (account && account.isDisconnected) {
      $selectedToken = null;
      $destinationChain = null;
    }
  }

  function updateForm() {
    tick().then(() => {
      if (importMethod === ImportMethod.MANUAL) {
        // run validations again if we are in manual mode
        runValidations();
      } else {
        resetForm();
      }
    });
  }

  $: if ($selectedToken && amountComponent) {
    amountComponent.validateAmount();
  }

  const resetForm = () => {
    //we check if these are still mounted, as the user might have left the page
    if (amountComponent) amountComponent.clearAmount();
    if (processingFeeComponent) processingFeeComponent.resetProcessingFee();
    if (addressInputComponent) addressInputComponent.clearAddress();

    // Update balance after bridging
    if (amountComponent) amountComponent.updateBalance();
    if (nftIdInputComponent) nftIdInputComponent.clearIds();

    $recipientAddress = $account?.address || null;
    bridgingStatus = 'pending';
    $selectedToken = ETHToken;
    importMethod === null;
    scanned = false;
    canProceed = false;
    $selectedNFTs = [];
    activeStep = NFTSteps.IMPORT;
  };

  /**
   *   NFT Bridge specific
   */
  let activeStep: NFTSteps = NFTSteps.IMPORT;

  const nextStep = () => (activeStep = Math.min(activeStep + 1, NFTSteps.CONFIRM));
  const previousStep = () => {
    activeStep = Math.max(activeStep - 1, NFTSteps.IMPORT);
  };

  let nftStepTitle: string;
  let nftStepDescription: string;
  let nextStepButtonText: string;

  let addressInputComponent: AddressInput;
  let nftIdInputComponent: IdInput;

  let validatingImport: boolean = false;
  let scanned: boolean = false;

  let canProceed: boolean = false;
  let foundNFTs: NFT[] = [];

  const getStepText = () => {
    if (activeStep === NFTSteps.REVIEW) {
      return $t('common.confirm');
    }
    if (activeStep === NFTSteps.CONFIRM) {
      return $t('common.ok');
    } else {
      return $t('common.continue');
    }
  };

  const changeImportMethod = () => {
    importMethod = importMethod === ImportMethod.MANUAL ? ImportMethod.SCAN : ImportMethod.MANUAL;
    resetForm();
  };

  const prefetchImage = async () => {
    const srcChainId = $network?.id;
    const destChainId = $destinationChain?.id;
    if (!srcChainId || !destChainId) throw new Error('both src and dest chain id must be defined');
    await Promise.all(
      nftIdArray.map(async (id) => {
        const token = $selectedToken as NFT;
        if (token) {
          token.tokenId = id;
          fetchNFTImageUrl(token, srcChainId, destChainId).then((nftWithUrl) => {
            $selectedToken = nftWithUrl;
            $selectedNFTs = [nftWithUrl];
          });
        } else {
          throw new Error('no token');
        }
      }),
    );
  };
  const manualImportAction = async () => {
    if (!$network?.id) throw new Error('network not found');
    const srcChainId = $network.id;
    const destChainId = $destinationChain?.id;
    const tokenId = nftIdArray[0];

    if (!isAddress(contractAddress) || !srcChainId || !destChainId) {
      return;
    }

    try {
      const token = await getTokenWithInfoFromAddress({
        contractAddress,
        srcChainId,
        tokenId,
        owner: $account?.address,
      });
      if (!token) throw new Error('no token with info');

      $selectedToken = token;

      const [, info] = await Promise.all([
        prefetchImage(),
        getCanonicalInfoForToken({ token, srcChainId, destChainId }),
      ]);

      if (info) $selectedTokenIsBridged = contractAddress !== info.address;

      nextStep();
    } catch (err) {
      console.error(err);
    }
  };

  const handleTransactionDetailsClick = () => {
    activeStep = NFTSteps.RECIPIENT;
  };

  // Whenever the user switches bridge types, we should reset the forms
  $: $activeBridge && (resetForm(), (activeStep = NFTSteps.IMPORT));

  // Set the content text based on the current step
  $: {
    const stepKey = NFTSteps[activeStep].toLowerCase();
    if (activeStep === NFTSteps.CONFIRM) {
      nftStepTitle = '';
      nftStepDescription = '';
    } else {
      nftStepTitle = $t(`bridge.title.nft.${stepKey}`);
      nftStepDescription = $t(`bridge.description.nft.${stepKey}`);
    }
    nextStepButtonText = getStepText();
  }

  onDestroy(() => {
    resetForm();
  });

  $: activeStep === NFTSteps.IMPORT && resetForm();
</script>

<div class=" gap-0 w-full md:w-[524px]">
  <Stepper {activeStep}>
    <Step stepIndex={NFTSteps.IMPORT} currentStepIndex={activeStep} isActive={activeStep === NFTSteps.IMPORT}
      >{$t('bridge.nft.step.import.title')}</Step>
    <Step stepIndex={NFTSteps.REVIEW} currentStepIndex={activeStep} isActive={activeStep === NFTSteps.REVIEW}
      >{$t('bridge.nft.step.review.title')}</Step>
    <Step stepIndex={NFTSteps.CONFIRM} currentStepIndex={activeStep} isActive={activeStep === NFTSteps.CONFIRM}
      >{$t('bridge.nft.step.confirm.title')}</Step>
  </Stepper>

  <Card class="md:mt-[32px] w-full md:w-[524px]" title={nftStepTitle} text={nftStepDescription}>
    <div class="space-y-[30px]">
      <!-- IMPORT STEP -->
      {#if activeStep === NFTSteps.IMPORT}
        <ImportStep
          bind:importMethod
          bind:canProceed
          bind:nftIdArray
          bind:contractAddress
          bind:foundNFTs
          bind:scanned
          bind:validating={validatingImport} />
        <!-- REVIEW STEP -->
      {:else if activeStep === NFTSteps.REVIEW}
        <ReviewStep on:editTransactionDetails={handleTransactionDetailsClick} bind:hasEnoughEth />
        <!-- RECIPIENT STEP -->
      {:else if activeStep === NFTSteps.RECIPIENT}
        <RecipientStep bind:this={recipientStepComponent} bind:hasEnoughEth />
        <!-- CONFIRM STEP -->
      {:else if activeStep === NFTSteps.CONFIRM}
        <ConfirmationStep bind:bridgingStatus />
      {/if}
      <!-- 
        User Actions
      -->
      {#if activeStep === NFTSteps.REVIEW}
        <div class="f-col w-full gap-[16px]">
          <ActionButton priority="primary" disabled={!canProceed} on:click={() => (activeStep = NFTSteps.CONFIRM)}>
            <span class="body-bold">{nextStepButtonText}</span>
          </ActionButton>
          <button on:click={previousStep} class="flex justify-center py-3 link">
            {$t('common.back')}
          </button>
        </div>
      {:else if activeStep === NFTSteps.IMPORT}
        {#if importMethod === ImportMethod.MANUAL}
          <div class="h-sep" />

          <div class="f-col w-full">
            <ActionButton
              priority="primary"
              disabled={!canProceed}
              loading={validatingImport}
              on:click={manualImportAction}><span class="body-bold">{nextStepButtonText}</span></ActionButton>

            <button on:click={() => changeImportMethod()} class="flex justify-center py-3 link">
              {$t('common.back')}
            </button>
          </div>
        {:else if scanned && foundNFTs.length > 0}
          <div class="f-col w-full">
            <div class="h-sep" />

            <ActionButton priority="primary" disabled={!canProceed} on:click={nextStep}
              ><span class="body-bold">{nextStepButtonText}</span></ActionButton>

            <button on:click={resetForm} class="flex justify-center py-3 link">
              {$t('common.back')}
            </button>
          </div>
        {/if}
      {:else if activeStep === NFTSteps.RECIPIENT}
        <div class="f-col w-full">
          <ActionButton priority="primary" disabled={!canProceed} on:click={() => (activeStep = NFTSteps.REVIEW)}
            ><span class="body-bold">{nextStepButtonText}</span>
          </ActionButton>

          <button on:click={previousStep} class="flex justify-center py-3 link">
            {$t('common.back')}
          </button>
        </div>
      {:else if activeStep === NFTSteps.CONFIRM}
        <div class="f-col w-full">
          {#if bridgingStatus === 'done'}
            <ActionButton priority="primary" on:click={resetForm}
              ><span class="body-bold">{$t('bridge.nft.step.confirm.button.back')}</span>
            </ActionButton>
          {:else}
            <button on:click={() => (activeStep = NFTSteps.REVIEW)} class="flex justify-center py-3 link">
              {$t('common.back')}
            </button>
          {/if}
        </div>
      {/if}
    </div>
  </Card>
</div>

<OnNetwork change={onNetworkChange} />
<OnAccount change={onAccountChange} />
