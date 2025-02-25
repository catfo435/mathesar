<script lang="ts">
  import { _ } from 'svelte-i18n';
  import {
    ControlledModal,
    type ModalController,
  } from '@mathesar-component-library';
  import { upgradeApi } from '@mathesar/api/upgrade';
  import type { Release } from '@mathesar/stores/releases';
  import { getErrorMessage } from '@mathesar/utils/errors';
  import { assertExhaustive } from '@mathesar/utils/typeUtils';
  import connectionsApi from '@mathesar/api/connections';
  import UpgradeConfirm from './UpgradeConfirm.svelte';
  import UpgradeError from './UpgradeError.svelte';
  import UpgradeProcessing from './UpgradeProcessing.svelte';

  export let controller: ModalController;
  export let release: Release;

  type State =
    | { status: 'confirm' }
    | { status: 'processing' }
    | { status: 'error'; errorMsg: string };
  type Status = State['status'];

  function getInitialState(): State {
    return { status: 'confirm' };
  }

  let state: State = getInitialState();
  let reloadTimeout: number | undefined;

  $: version = `${$_('mathesar')} ${release.tagName}`;
  $: titleMap = ((): Record<Status, string> => ({
    confirm: $_('upgrade_to_version', { values: { version } }),
    processing: $_('upgrading_to_version', { values: { version } }),
    error: $_('error_upgrading'),
  }))();
  $: title = titleMap[state.status];

  function init() {
    if (state.status !== 'processing') {
      window.clearTimeout(reloadTimeout);
      state = getInitialState();
    }
  }

  async function reloadOnServerAvailability(): Promise<void> {
    try {
      await connectionsApi.list();
      window.location.reload();
    } catch {
      reloadTimeout = window.setTimeout(() => {
        void reloadOnServerAvailability();
      }, 2000);
    }
  }

  async function performUpgrade() {
    state = { status: 'processing' };
    try {
      await upgradeApi.upgrade();
      void reloadOnServerAvailability();
    } catch (e) {
      state = { status: 'error', errorMsg: getErrorMessage(e) };
    }
  }
</script>

<ControlledModal
  {controller}
  {title}
  on:open={init}
  size="large"
  allowClose={state.status !== 'processing'}
>
  {#if state.status === 'confirm'}
    <UpgradeConfirm
      {release}
      onProceed={performUpgrade}
      onClose={() => controller.close()}
    />
  {:else if state.status === 'processing'}
    <UpgradeProcessing />
  {:else if state.status === 'error'}
    <UpgradeError
      message={state.errorMsg}
      onClose={() => controller.close()}
      onRetry={performUpgrade}
    />
  {:else}
    {assertExhaustive(state)}
  {/if}
</ControlledModal>
