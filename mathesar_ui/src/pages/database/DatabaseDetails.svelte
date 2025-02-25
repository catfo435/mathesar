<script lang="ts">
  import { _ } from 'svelte-i18n';
  import {
    Button,
    Help,
    Icon,
    DropdownMenu,
    ButtonMenuItem,
  } from '@mathesar-component-library';
  import { reflectApi } from '@mathesar/api/reflect';
  import type { Database, SchemaEntry } from '@mathesar/AppTypes';
  import AppSecondaryHeader from '@mathesar/components/AppSecondaryHeader.svelte';
  import {
    iconAddNew,
    iconDatabase,
    iconManageAccess,
    iconRefresh,
    iconMoreActions,
    iconEdit,
    iconDeleteMajor,
  } from '@mathesar/icons';
  import { confirmDelete } from '@mathesar/stores/confirmation';
  import { modal } from '@mathesar/stores/modal';
  import type { DBSchemaStoreData } from '@mathesar/stores/schemas';
  import {
    deleteSchema as deleteSchemaAPI,
    schemas as schemasStore,
  } from '@mathesar/stores/schemas';
  import { removeTablesInSchemaTablesStore } from '@mathesar/stores/tables';
  import { toast } from '@mathesar/stores/toast';
  import { getUserProfileStoreFromContext } from '@mathesar/stores/userProfile';
  import EntityContainerWithFilterBar from '@mathesar/components/EntityContainerWithFilterBar.svelte';
  import {
    EditConnectionModal,
    DeleteConnectionModal,
  } from '@mathesar/systems/connections';
  import { CONNECTIONS_URL } from '@mathesar/routes/urls';
  import { RichText } from '@mathesar/components/rich-text';
  import { router } from 'tinro';
  import AddEditSchemaModal from './AddEditSchemaModal.svelte';
  import DbAccessControlModal from './DbAccessControlModal.svelte';
  import SchemaRow from './SchemaRow.svelte';

  const addEditModal = modal.spawnModalController();
  const accessControlModal = modal.spawnModalController();
  const editConnectionModal = modal.spawnModalController();
  const deleteConnectionModal = modal.spawnModalController();

  const userProfileStore = getUserProfileStoreFromContext();
  $: userProfile = $userProfileStore;

  export let database: Database;

  $: schemasMap = $schemasStore.data;

  $: canExecuteDDL = userProfile?.hasPermission({ database }, 'canExecuteDDL');
  $: canEditPermissions = userProfile?.hasPermission(
    { database },
    'canEditPermissions',
  );

  let filterQuery = '';
  let targetSchema: SchemaEntry | undefined;
  let isReflectionRunning = false;

  function filterSchemas(
    schemaData: DBSchemaStoreData['data'],
    filter: string,
  ): SchemaEntry[] {
    const filtered: SchemaEntry[] = [];
    schemaData.forEach((schema) => {
      if (schema.name?.toLowerCase().includes(filter.toLowerCase())) {
        filtered.push(schema);
      }
    });
    return filtered;
  }

  $: displayList = filterSchemas(schemasMap, filterQuery);

  function addSchema() {
    targetSchema = undefined;
    addEditModal.open();
  }

  function editSchema(schema: SchemaEntry) {
    targetSchema = schema;
    addEditModal.open();
  }

  function deleteSchema(schema: SchemaEntry) {
    void confirmDelete({
      identifierType: $_('schema'),
      identifierName: schema.name,
      body: [$_('schema_delete_warning'), $_('are_you_sure_to_proceed')],
      onProceed: async () => {
        await deleteSchemaAPI(database.id, schema.id);
        // TODO: Create common util to handle data clearing & sync between stores
        removeTablesInSchemaTablesStore(schema.id);
      },
    });
  }

  function manageAccess() {
    accessControlModal.open();
  }

  function handleClearFilterQuery() {
    filterQuery = '';
  }

  async function reflect() {
    try {
      isReflectionRunning = true;
      await reflectApi.reflect();
      window.location.reload();
    } catch (e) {
      toast.fromError(e);
    } finally {
      isReflectionRunning = false;
    }
  }
</script>

<AppSecondaryHeader
  pageTitleAndMetaProps={{
    name: database.nickname,
    type: 'database',
    icon: iconDatabase,
  }}
>
  <svelte:fragment slot="action">
    {#if canExecuteDDL || canEditPermissions}
      <div>
        {#if canEditPermissions}
          <Button on:click={manageAccess} appearance="secondary">
            <Icon {...iconManageAccess} />
            <span>{$_('manage_access')}</span>
          </Button>
        {/if}
        <DropdownMenu
          showArrow={false}
          triggerAppearance="plain"
          label=""
          closeOnInnerClick={false}
          icon={iconMoreActions}
          preferredPlacement="bottom-end"
          menuStyle="--spacing-y:0.8em;"
        >
          <ButtonMenuItem
            icon={{ ...iconRefresh, spin: isReflectionRunning }}
            disabled={isReflectionRunning}
            on:click={reflect}
          >
            <div class="reflect">
              {$_('sync_external_changes')}
              <Help>
                <p>
                  {$_('sync_external_changes_structure_help')}
                </p>
                <p>
                  {$_('sync_external_changes_data_help')}
                </p>
              </Help>
            </div>
          </ButtonMenuItem>
          {#if userProfile?.isSuperUser}
            <ButtonMenuItem
              icon={iconEdit}
              on:click={() => editConnectionModal.open()}
            >
              {$_('edit_connection')}
            </ButtonMenuItem>
            <ButtonMenuItem
              icon={iconDeleteMajor}
              danger
              on:click={() => deleteConnectionModal.open()}
            >
              {$_('delete_connection')}
            </ButtonMenuItem>
          {/if}
        </DropdownMenu>
      </div>
    {/if}
  </svelte:fragment>
</AppSecondaryHeader>

<div class="schema-list-wrapper">
  <div class="schema-list-title-container">
    <h2 class="schema-list-title">{$_('schemas')} ({schemasMap.size})</h2>
  </div>
  <EntityContainerWithFilterBar
    searchPlaceholder={$_('search_schemas')}
    bind:searchQuery={filterQuery}
    on:clear={handleClearFilterQuery}
  >
    <svelte:fragment slot="action">
      {#if canExecuteDDL}
        <Button on:click={addSchema} appearance="primary">
          <Icon {...iconAddNew} />
          <span>{$_('create_schema')}</span>
        </Button>
      {/if}
    </svelte:fragment>
    <p slot="resultInfo">
      <RichText
        text={$_('schemas_matching_search', {
          values: { count: displayList.length },
        })}
        let:slotName
      >
        {#if slotName === 'searchValue'}
          <strong>{filterQuery}</strong>
        {/if}
      </RichText>
    </p>
    <ul class="schema-list" slot="content">
      {#each displayList as schema (schema.id)}
        <li class="schema-list-item">
          <SchemaRow
            {database}
            {schema}
            canExecuteDDL={userProfile?.hasPermission(
              { database, schema },
              'canExecuteDDL',
            )}
            on:edit={() => editSchema(schema)}
            on:delete={() => deleteSchema(schema)}
          />
        </li>
      {/each}
    </ul>
  </EntityContainerWithFilterBar>
</div>

<AddEditSchemaModal
  controller={addEditModal}
  {database}
  schema={targetSchema}
/>

<DbAccessControlModal controller={accessControlModal} {database} />

<EditConnectionModal controller={editConnectionModal} connection={database} />
<DeleteConnectionModal
  controller={deleteConnectionModal}
  connection={database}
  on:delete={() => router.goto(CONNECTIONS_URL)}
/>

<style lang="scss">
  .schema-list-wrapper {
    display: flex;
    flex-direction: column;
    width: 100%;

    .schema-list-title-container {
      display: flex;
      justify-content: space-between;
      align-items: center;
    }

    .schema-list-title {
      font-size: var(--text-size-x-large);
      font-weight: 500;
      margin-top: var(--size-super-ultra-small);
    }

    .schema-list {
      width: 100%;
      list-style: none;
      padding: 0;
      display: flex;
      flex-direction: column;
      margin-top: var(--size-x-large);

      .schema-list-item + .schema-list-item {
        margin-top: var(--size-base);
      }
    }
  }
</style>
