<template>
    <tr
        :data-pivot-id="resource['id'].pivotValue"
        :dusk="resource['id'].value + '-row'"
        class="group"
        :class="{
            'o1-divide-x divide-gray-100 dark:divide-gray-700':
             shouldShowColumnBorders,
        }"
        @click.stop.prevent="navigateToDetail"
    >
        <!-- Resource Selection Checkbox -->
        <td
            v-if="shouldShowCheckboxes || canSeeReorderButtons"
            :class="{
                'o1-py-2': !shouldShowTight,
                'o1-border-t border-gray-100 dark:border-gray-700 o1-px-2': true,
                'o1-cursor-pointer': resource.authorizedToView,
            }"
            class="td-fit o1-pl-5 o1-pr-5 dark:bg-gray-800 group-hover:bg-gray-50 dark:group-hover:bg-gray-900"
            @click.stop
        >
            <ReorderButtons
                :resource="resource"
                :relationship-type="relationshipType"
                :via-resource-id="viaResourceId"
                :via-relationship="viaRelationship"
                :resource-name="resourceName"
                @moveToEnd="$emit('moveToEnd')"
                @moveToStart="$emit('moveToStart')"
            >
                <Checkbox
                    v-if="shouldShowCheckboxes"
                    :aria-label="__('Select Resource :title', { title: resource.title })"
                    :checked="checked"
                    :data-testid="`${testId}-checkbox`"
                    :dusk="`${resource['id'].value}-checkbox`"
                    @input="toggleSelection"
                />
            </ReorderButtons>
        </td>

        <!-- Fields -->
        <td
            v-for="(field, index) in resource.fields"
            :key="field.uniqueKey"
            :class="{
                'o1-px-6': index == 0 && !shouldShowCheckboxes,
                'o1-px-2': index != 0 || shouldShowCheckboxes,
                'o1-py-2': !shouldShowTight,
                'o1-whitespace-nowrap': !field.wrapping,
                'o1-cursor-pointer': clickableRow,
            }"
            class="dark:bg-gray-800 group-hover:bg-gray-50 dark:group-hover:bg-gray-900"
        >
            <component
                :is="'index-' + field.component"
                :class="`text-${field.textAlign}`"
                :field="field"
                :resource="resource"
                :resource-name="resourceName"
                :via-resource="viaResource"
                :via-resource-id="viaResourceId"
            />
        </td>

        <td
            :class="{
                'o1-py-2': !shouldShowTight,
                'o1-cursor-pointer': resource.authorizedToView,
            }"
            class="o1-px-2 td-fit o1-text-right o1-align-middle dark:bg-gray-800 group-hover:bg-gray-50 dark:group-hover:bg-gray-900"
        >
            <div
                class="o1-flex o1-items-center o1-justify-end o1-space-x-0 text-gray-400"
            >
                <InlineActionDropdown
                    v-if="shouldShowActionDropdown"
                    :actions="availableActions"
                    :endpoint="actionsEndpoint"
                    :resource="resource"
                    :resource-name="resourceName"
                    :via-many-to-many="viaManyToMany"
                    :via-resource="viaResource"
                    :via-resource-id="viaResourceId"
                    :via-relationship="viaRelationship"
                    @actionExecuted="$emit('actionExecuted')"
                    @show-preview="navigateToPreviewView"
                />

                <!-- Start CDE Override -->
                <!-- Preview Resource Link -->
                <a
                    v-if="authorizedToViewAnyResources"
                    v-tooltip.click="__('Preview')"
                    :aria-label="__('Preview')"
                    :data-testid="`${testId}-preview-button`"
                    :dusk="`${resource['id'].value}-preview-button`"
                    class="inline-flex items-center justify-center h-9 w-9"
                    :class="
                         resource.authorizedToView
                            ? 'text-gray-500 dark:text-gray-400 hover:[&:not(:disabled)]:text-primary-500 dark:hover:[&:not(:disabled)]:text-primary-500'
                            : 'disabled:cursor-not-allowed disabled:opacity-50'
                    "
                    :disabled="!resource.authorizedToView"
                    @click.prevent="openPreviewModal"
                    @click.stop
                >
                    <Icon name="magnifying-glass-plus" type="outline" />
                </a>

                <!-- View Resource Link -->
                <a
                    v-if="authorizedToViewAnyResources"
                    v-tooltip.click="__('View')"
                    target="_blank"
                    :aria-label="__('View')"
                    :data-testid="`${testId}-view-button`"
                    :dusk="`${resource['id'].value}-view-button`"
                    :href="$url(`/resources/${resourceName}/${resource['id'].value}`)"
                    class="inline-flex items-center justify-center h-9 w-9"
                    :class="
                         resource.authorizedToView
                            ? 'text-gray-500 dark:text-gray-400 hover:[&:not(:disabled)]:text-primary-500 dark:hover:[&:not(:disabled)]:text-primary-500'
                            : 'disabled:cursor-not-allowed disabled:opacity-50'
                    "
                    :disabled="!resource.authorizedToView"
                    @click.stop
                >
                    <Icon name="eye" type="outline" />
                </a>
                <!-- End CDE Override -->

                <!-- Edit Button -->
                <Link
                    v-if="authorizedToUpdateAnyResources"
                    :as="!resource.authorizedToUpdate ? 'button' : 'a'"
                    :href="updateURL"
                    :disabled="!resource.authorizedToUpdate ? true : null"
                    @click.stop
                    class="inline-flex items-center justify-center h-9 w-9"
                    :class="
                        resource.authorizedToUpdate
                        ? 'text-gray-500 dark:text-gray-400 hover:[&:not(:disabled)]:text-primary-500 dark:hover:[&:not(:disabled)]:text-primary-500'
                        : 'disabled:cursor-not-allowed disabled:opacity-50'
                    "
                    :dusk="
                        viaManyToMany
                            ? `${resource['id'].value}-edit-attached-button`
                            : `${resource['id'].value}-edit-button`
                    "
                    :aria-label="viaManyToMany ? __('Edit Attached') : __('Edit')"
                    v-tooltip.click="viaManyToMany ? __('Edit Attached') : __('Edit')"
                >
                    <span class="flex items-center gap-1">
                        <span>
                            <Icon name="pencil-square"/>
                        </span>
                    </span>
                </Link>

                <!-- Delete Resource Link -->
                <Button
                    v-if="
                        authorizedToDeleteAnyResources &&
                        (!resource.softDeleted || viaManyToMany)
                    "
                    @click.stop="openDeleteModal"
                    v-tooltip.click="__(viaManyToMany ? 'Detach' : 'Delete')"
                    :aria-label="__(viaManyToMany ? 'Detach' : 'Delete')"
                    :dusk="`${resource.id.value}-delete-button`"
                    icon="trash"
                    variant="action"
                    :disabled="!resource.authorizedToDelete"
                />

                <!-- Restore Resource Link -->
                <Button
                    v-if="
                        authorizedToRestoreAnyResources &&
                        resource.softDeleted &&
                        !viaManyToMany
                    "
                    v-tooltip.click="__('Restore')"
                    :aria-label="__('Restore')"
                    :disabled="!resource.authorizedToRestore"
                    :dusk="`${resource.id.value}-restore-button`"
                    type="button"
                    @click.stop="openRestoreModal"
                    icon="arrow-path"
                    variant="action"
                />

                <DeleteResourceModal
                    :mode="viaManyToMany ? 'detach' : 'delete'"
                    :show="deleteModalOpen"
                    @close="closeDeleteModal"
                    @confirm="confirmDelete"
                />

                <RestoreResourceModal
                    :show="restoreModalOpen"
                    @close="closeRestoreModal"
                    @confirm="confirmRestore"
                >
                    <ModalHeader v-text="__('Restore Resource')"/>
                    <ModalContent>
                        <p class="leading-normal">
                            {{ __('Are you sure you want to restore this resource?') }}
                        </p>
                    </ModalContent>
                </RestoreResourceModal>
            </div>
        </td>
    </tr>

    <PreviewResourceModal
        v-if="previewModalOpen"
        :resource-id="resource.id.value"
        :resource-name="resourceName"
        :show="previewModalOpen"
        @close="closePreviewModal"
        @confirm="closePreviewModal"
    />
</template>

<script>
import filter from 'lodash/filter'
import ReordersResources from '../mixins/ReordersResources'
import {mapGetters} from 'vuex'
import {Button, Icon} from 'laravel-nova-ui'

export default {
    emits: ['actionExecuted'],
    mixins: [ReordersResources],
    components: {Button, Icon},

    inject: [
        'authorizedToViewAnyResources',
        'authorizedToUpdateAnyResources',
        'authorizedToDeleteAnyResources',
        'authorizedToRestoreAnyResources',
    ],

    props: [
        'testId',
        'deleteResource',
        'restoreResource',
        'resource',
        'resourcesSelected',
        'resourceName',
        'relationshipType',
        'viaRelationship',
        'viaResource',
        'viaResourceId',
        'viaManyToMany',
        'checked',
        'actionsAreAvailable',
        'actionsEndpoint',
        'shouldShowCheckboxes',
        'shouldShowColumnBorders',
        'tableStyle',
        'updateSelectionStatus',
        'queryString',
        'clickAction',
    ],

    data: () => ({
        commandPressed: false,
        deleteModalOpen: false,
        restoreModalOpen: false,
        previewModalOpen: false,
    }),

    mounted() {
        window.addEventListener('keydown', this.handleKeydown)
        window.addEventListener('keyup', this.handleKeyup)
    },

    beforeUnmount() {
        window.removeEventListener('keydown', this.handleKeydown)
        window.removeEventListener('keyup', this.handleKeyup)
    },

    methods: {
        /**
         * Select the resource in the parent component
         */
        toggleSelection() {
            this.updateSelectionStatus(this.resource)
        },

        handleKeydown(e) {
            if (e.key === 'Meta') {
                this.commandPressed = true
            }
        },

        handleKeyup(e) {
            if (e.key === 'Meta') {
                this.commandPressed = false
            }
        },

        navigateToDetail(e) {
            if (this.clickAction === 'edit') {
                return this.navigateToEditView(e)
            } else if (this.clickAction === 'select') {
                return this.toggleSelection()
            } else if (this.clickAction === 'ignore') {
                return
            } else if (this.clickAction === 'detail') {
                return this.navigateToDetailView(e)
            } else if (this.clickAction === 'preview') {
                return this.navigateToPreviewView(e)
            } else {
                return this.navigateToDetailView(e)
            }
        },

        navigateToDetailView(e) {
            if (!this.resource.authorizedToView) {
                return
            }
            this.commandPressed
                ? window.open(this.viewURL, '_blank')
                : this.$inertia.visit(this.viewURL) // use this instead of the Inertia.visit to avoid the following issue: https://github.com/outl1ne/nova-sortable/issues/128#issuecomment-1171264760
        },

        navigateToEditView(e) {
            if (!this.resource.authorizedToUpdate) {
                return
            }
            this.commandPressed
                ? window.open(this.updateURL, '_blank')
                : this.$inertia.visit(this.updateURL) // use this instead of the Inertia.visit to avoid the following issue: https://github.com/outl1ne/nova-sortable/issues/128#issuecomment-1171264760
        },

        navigateToPreviewView(e) {
            if (!this.resource.authorizedToView) {
                return
            }
            this.openPreviewModal()
        },

        openPreviewModal() {
            this.previewModalOpen = true
        },

        closePreviewModal() {
            this.previewModalOpen = false
        },

        openDeleteModal() {
            this.deleteModalOpen = true
        },

        confirmDelete() {
            this.deleteResource(this.resource)
            this.closeDeleteModal()
        },

        closeDeleteModal() {
            this.deleteModalOpen = false
        },

        openRestoreModal() {
            this.restoreModalOpen = true
        },

        confirmRestore() {
            this.restoreResource(this.resource)
            this.closeRestoreModal()
        },

        closeRestoreModal() {
            this.restoreModalOpen = false
        },
    },

    computed: {
        ...mapGetters(['currentUser']),

        updateURL() {
            if (this.viaManyToMany) {
                return this.$url(
                    `/resources/${this.viaResource}/${this.viaResourceId}/edit-attached/${this.resourceName}/${this.resource.id.value}`,
                    {
                        viaRelationship: this.viaRelationship,
                        viaPivotId: this.resource.id.pivotValue,
                    }
                )
            }

            return this.$url(
                `/resources/${this.resourceName}/${this.resource.id.value}/edit`,
                {
                    viaResource: this.viaResource,
                    viaResourceId: this.viaResourceId,
                    viaRelationship: this.viaRelationship,
                }
            )
        },

        viewURL() {
            return this.$url(
                `/resources/${this.resourceName}/${this.resource.id.value}`
            )
        },

        availableActions() {
            return filter(this.resource.actions, a => a.showOnTableRow)
        },

        shouldShowTight() {
            return this.tableStyle === 'tight'
        },

        clickableRow() {
            if (this.clickAction === 'edit') {
                return this.resource.authorizedToUpdate
            } else if (this.clickAction === 'select') {
                return this.shouldShowCheckboxes
            } else if (this.clickAction === 'ignore') {
                return false
            } else if (this.clickAction === 'detail') {
                return this.resource.authorizedToView
            } else if (this.clickAction === 'preview') {
                return this.resource.authorizedToView
            } else {
                return this.resource.authorizedToView
            }
        },

        shouldShowActionDropdown() {
            return this.availableActions.length > 0 || this.userHasAnyOptions
        },
        shouldShowPreviewLink() {
            return this.resource.authorizedToView && this.resource.previewHasFields
        },
        userHasAnyOptions() {
            return (
                this.resource.authorizedToReplicate ||
                this.shouldShowPreviewLink ||
                this.canBeImpersonated
            )
        },
        canBeImpersonated() {
            return (
                this.currentUser.canImpersonate && this.resource.authorizedToImpersonate
            )
        },
    },
}
</script>
