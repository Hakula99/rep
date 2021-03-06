<template>
    <loading-view :loading="loading">
        <form v-if="panels" @submit.prevent="createResource" autocomplete="off">
            <form-panel
                class="mb-8"
                v-for="panel in panelsWithFields"
                :panel="panel"
                :name="panel.name"
                :key="panel.name"
                :resource-name="resourceName"
                :fields="panel.fields"
                mode="form"
                :validation-errors="validationErrors"
                :via-resource="viaResource"
                :via-resource-id="viaResourceId"
                :via-relationship="viaRelationship"
            />

            <!-- Create Button -->
            <div class="flex items-center">
                <cancel-button />

                <progress-button
                    class="mr-3"
                    dusk="create-and-add-another-button"
                    @click.native="createAndAddAnother"
                    :disabled="isWorking"
                    :processing="submittedViaCreateAndAddAnother"
                >
                    {{ __('Create & Add Another') }}
                </progress-button>

                <progress-button
                    dusk="create-button"
                    type="submit"
                    :disabled="isWorking"
                    :processing="submittedViaCreateResource"
                >
                    {{ __('Create :resource', { resource: singularName }) }}
                </progress-button>
            </div>
        </form>
    </loading-view>
</template>

<script>
import { Errors, Minimum, InteractsWithResourceInformation } from 'laravel-nova'

export default {
    mixins: [InteractsWithResourceInformation],

    props: {
        resourceName: {
            type: String,
            required: true,
        },
        viaResource: {
            default: '',
        },
        viaResourceId: {
            default: '',
        },
        viaRelationship: {
            default: '',
        },
    },

    data: () => ({
        relationResponse: null,
        loading: true,
        submittedViaCreateAndAddAnother: false,
        submittedViaCreateResource: false,
        fields: [],
        panels: [],
        validationErrors: new Errors(),
    }),

    async created() {
        if (Nova.missingResource(this.resourceName)) return this.$router.push({ name: '404' })

        // If this create is via a relation index, then let's grab the field
        // and use the label for that as the one we use for the title and buttons
        if (this.isRelation) {
            const { data } = await Nova.request(
                '/nova-api/' + this.viaResource + '/field/' + this.viaRelationship
            )
            this.relationResponse = data
        }

        this.getFields()
    },

    methods: {
        /**
         * Get the available fields for the resource.
         */
        async getFields() {
            this.panels = []
            this.fields = []

            const {
                data: { panels, fields },
            } = await Nova.request().get(`/nova-api/${this.resourceName}/creation-fields`, {
                params: {
                    editing: true,
                    editMode: 'create',
                    viaResource: this.viaResource,
                    viaResourceId: this.viaResourceId,
                    viaRelationship: this.viaRelationship,
                },
            })

            this.panels = panels
            this.fields = fields
            this.loading = false
        },

        /**
         * Create a new resource instance using the provided data.
         */
        async createResource() {
            this.submittedViaCreateResource = true

            try {
                const {
                    data: { redirect },
                } = await this.createRequest()

                this.submittedViaCreateResource = false

                this.$toasted.show(
                    this.__('The :resource was created!', {
                        resource: this.resourceInformation.singularLabel.toLowerCase(),
                    }),
                    { type: 'success' }
                )

                this.$router.push({ path: redirect })
            } catch (error) {
                this.submittedViaCreateResource = false

                if (error.response.status == 422) {
                    this.validationErrors = new Errors(error.response.data.errors)
                }
            }
        },

        /**
         * Create a new resource and reset the form
         */
        async createAndAddAnother() {
            this.submittedViaCreateAndAddAnother = true

            try {
                const response = await this.createRequest()

                this.submittedViaCreateAndAddAnother = false

                this.$toasted.show(
                    this.__('The :resource was created!', {
                        resource: this.resourceInformation.singularLabel.toLowerCase(),
                    }),
                    { type: 'success' }
                )

                // Reset the form by refetching the fields
                this.getFields()

                this.validationErrors = new Errors()
            } catch (error) {
                this.submittedViaCreateAndAddAnother = false

                if (error.response.status == 422) {
                    this.validationErrors = new Errors(error.response.data.errors)
                }
            }
        },

        /**
         * Send a create request for this resource
         */
        createRequest() {
            return Nova.request().post(
                `/nova-api/${this.resourceName}`,
                this.createResourceFormData()
            )
        },

        /**
         * Create the form data for creating the resource.
         */
        createResourceFormData() {
            return _.tap(new FormData(), formData => {
                // _.each(this.panels, panel => {
                _.each(this.fields, field => {
                    field.fill(formData)
                })
                // })

                formData.append('viaResource', this.viaResource)
                formData.append('viaResourceId', this.viaResourceId)
                formData.append('viaRelationship', this.viaRelationship)
            })
        },
    },

    computed: {
        singularName() {
            if (this.relationResponse) {
                return this.relationResponse.singularLabel
            }

            return this.resourceInformation.singularLabel
        },

        isRelation() {
            return Boolean(this.viaResourceId && this.viaRelationship)
        },

        /**
         * Determine if the form is being processed
         */
        isWorking() {
            return this.submittedViaCreateResource || this.submittedViaCreateAndAddAnother
        },

        panelsWithFields() {
            return _.map(this.panels, panel => {
                return {
                    name: panel.name,
                    fields: _.filter(this.fields, field => field.panel == panel.name),
                }
            })
        },
    },
}
</script>
