<script>
  import {
    Layout,
    Heading,
    Body,
    Button,
    ButtonGroup,
    Modal,
    Tag,
    Tags,
    Table,
    Divider,
    Search,
    notifications,
  } from "@budibase/bbui"
  import { groups, auth, licensing, admin } from "stores/portal"
  import { onMount } from "svelte"
  import CreateEditGroupModal from "./_components/CreateEditGroupModal.svelte"
  import { cloneDeep } from "lodash/fp"
  import GroupAppsTableRenderer from "./_components/GroupAppsTableRenderer.svelte"
  import UsersTableRenderer from "./_components/UsersTableRenderer.svelte"
  import GroupNameTableRenderer from "./_components/GroupNameTableRenderer.svelte"
  import { goto } from "@roxi/routify"

  const DefaultGroup = {
    name: "",
    icon: "UserGroup",
    color: "var(--spectrum-global-color-blue-600)",
    users: [],
    apps: [],
    roles: {},
  }

  let modal
  let searchString
  let group = cloneDeep(DefaultGroup)
  let customRenderers = [
    { column: "name", component: GroupNameTableRenderer },
    { column: "users", component: UsersTableRenderer },
    { column: "roles", component: GroupAppsTableRenderer },
  ]

  $: schema = {
    name: {},
    users: { sortable: false },
    roles: { sortable: false, displayName: "Apps" },
  }
  $: filteredGroups = filterGroups($groups, searchString)

  const filterGroups = (groups, searchString) => {
    if (!searchString) {
      return groups
    }
    searchString = searchString.toLocaleLowerCase()
    return groups?.filter(group => {
      return group.name?.toLowerCase().includes(searchString)
    })
  }

  async function saveGroup(group) {
    try {
      group = await groups.actions.save(group)
      $goto(`./${group._id}`)
      notifications.success(`User group created successfully`)
    } catch (error) {
      if (error.status === 400) {
        notifications.error(error.message)
      } else {
        notifications.error(`Failed to save group`)
      }
    }
  }

  const showCreateGroupModal = () => {
    group = cloneDeep(DefaultGroup)
    modal?.show()
  }

  onMount(async () => {
    try {
      // always load latest
      await licensing.init()
      if ($licensing.groupsEnabled) {
        await groups.actions.init()
      }
    } catch (error) {
      notifications.error("Error getting user groups")
    }
  })
</script>

<Layout noPadding gap="M">
  <Layout gap="XS" noPadding>
    <Heading size="M">User groups</Heading>
    {#if !$licensing.groupsEnabled}
      <Tags>
        <div class="tags">
          <div class="tag">
            <Tag icon="LockClosed">Pro plan</Tag>
          </div>
        </div>
      </Tags>
    {/if}
    <Body>
      Easily assign and manage your users' access with user groups.
      {#if !$auth.accountPortalAccess && !$licensing.groupsEnabled && $admin.cloud}
        Contact your account holder to upgrade your plan.
      {/if}
    </Body>
  </Layout>
  <Divider />
  <div class="controls">
    <ButtonGroup>
      {#if $licensing.groupsEnabled}
        <!--Show the group create button-->
        <Button
          newStyles
          icon={"UserGroup"}
          cta
          on:click={showCreateGroupModal}
        >
          Create user group
        </Button>
      {:else}
        <Button
          newStyles
          disabled={!$auth.accountPortalAccess && $admin.cloud}
          on:click={$licensing.goToUpgradePage()}
        >
          Upgrade
        </Button>
        <!--Show the view plans button-->
        <Button
          newStyles
          secondary
          on:click={() => {
            window.open("https://budibase.com/pricing/", "_blank")
          }}
        >
          View Plans
        </Button>
      {/if}
    </ButtonGroup>
    <div class="controls-right">
      <Search bind:value={searchString} placeholder="Search" />
    </div>
  </div>
  <Table
    on:click={({ detail }) => $goto(`./${detail._id}`)}
    {schema}
    data={filteredGroups}
    allowEditColumns={false}
    allowEditRows={false}
    {customRenderers}
  />
</Layout>

<Modal bind:this={modal}>
  <CreateEditGroupModal bind:group {saveGroup} />
</Modal>

<style>
  .controls {
    display: flex;
    flex-direction: row;
    justify-content: space-between;
    align-items: center;
  }
  .controls-right {
    display: flex;
    flex-direction: row;
    justify-content: flex-end;
    align-items: center;
    gap: var(--spacing-xl);
  }
  .controls-right :global(.spectrum-Search) {
    width: 200px;
  }
  .tag {
    margin-top: var(--spacing-xs);
    margin-left: var(--spacing-m);
  }
</style>
