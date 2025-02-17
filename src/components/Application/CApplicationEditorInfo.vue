<template>
  <b-card
    class="shadow-sm"
    header-bg-variant="white"
    footer-bg-variant="white"
  >
    <b-form
      @submit.prevent="$emit('submit', application)"
    >
      <b-form-group
        v-if="application.applicationID"
        :label="$t('id')"
        label-cols="2"
      >
        <b-form-input
          v-model="application.applicationID"
          plaintext
          disabled
        />
      </b-form-group>

      <b-form-group
        :label="$t('name')"
        label-cols="2"
      >
        <b-form-input
          v-model="application.name"
          required
        />

        <!--
          include hidden input to enable
          trigger submit event w/ ENTER
        -->
        <input
          type="submit"
          class="d-none"
        >
      </b-form-group>

      <b-form-group
        label-cols="2"
        :class="{ 'mb-0': !application.applicationID }"
      >
        <b-form-checkbox
          v-model="application.enabled"
        >
          {{ $t('enabled') }}
        </b-form-checkbox>
      </b-form-group>

      <b-form-group
        v-if="application.updatedAt"
        :label="$t('updatedAt')"
        label-cols="2"
      >
        <b-form-input
          v-model="application.updatedAt"
          plaintext
          disabled
        />
      </b-form-group>

      <b-form-group
        v-if="application.deletedAt"
        :label="$t('deletedAt')"
        label-cols="2"
      >
        <b-form-input
          v-model="application.deletedAt"
          plaintext
          disabled
        />
      </b-form-group>

      <b-form-group
        v-if="application.createdAt"
        :label="$t('createdAt')"
        label-cols="2"
        class="mb-0"
      >
        <b-form-input
          v-model="application.createdAt"
          plaintext
          disabled
        />
      </b-form-group>
    </b-form>

    <template #header>
      <h3 class="m-0">
        {{ $t('title') }}
      </h3>
    </template>

    <template #footer>
      <c-submit-button
        class="float-right"
        :processing="processing"
        :success="success"
        :disabled="!canCreate"
        @submit="$emit('submit', application)"
      />

      <confirmation-toggle
        v-if="application && application.applicationID"
        @confirmed="$emit('delete')"
      >
        {{ getDeleteStatus }}
      </confirmation-toggle>
    </template>
  </b-card>
</template>

<script>
import ConfirmationToggle from 'corteza-webapp-admin/src/components/ConfirmationToggle'
import CSubmitButton from 'corteza-webapp-admin/src/components/CSubmitButton'

export default {
  name: 'CApplicationEditorInfo',

  i18nOptions: {
    namespaces: 'system.applications',
    keyPrefix: 'editor.info',
  },

  components: {
    ConfirmationToggle,
    CSubmitButton,
  },

  props: {
    application: {
      type: Object,
      required: true,
    },

    processing: {
      type: Boolean,
      value: false,
    },

    success: {
      type: Boolean,
      value: false,
    },

    canCreate: {
      type: Boolean,
      required: true,
    },
  },

  computed: {
    getDeleteStatus () {
      return this.application.deletedAt ? this.$t('undelete') : this.$t('delete')
    },
  },
}
</script>
