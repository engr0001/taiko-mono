<script lang="ts">
  import '../app.css';
  import '../i18n';

  import { onDestroy, onMount } from 'svelte';

  import { AccountConnectionToast } from '$components/AccountConnectionToast';
  import { BridgePausedModal } from '$components/BridgePausedModal';
  import { Header } from '$components/Header';
  import { NotificationToast } from '$components/NotificationToast';
  import { SideNavigation } from '$components/SideNavigation';
  import { SwitchChainModal } from '$components/SwitchChainModal';
  import { WelcomeModal } from '$components/WelcomeModal';
  import { startWatching, stopWatching } from '$libs/wagmi';

  onMount(startWatching);
  onDestroy(stopWatching);
</script>

<!-- App components -->
<SideNavigation>
  <Header />
  <main>
    <slot />
  </main>
</SideNavigation>

<!--
  The following UI is global and should be rendered 
  at the root of the app.
-->

<NotificationToast />

<AccountConnectionToast />

<SwitchChainModal />

<BridgePausedModal />

<WelcomeModal />
