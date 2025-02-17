<template>
  <b-card
    class="shadow-sm"
    header-bg-variant="white"
    footer-bg-variant="white"
  >
    <b-form>
      <b-form-group>
        <b-form-checkbox
          v-model="external.enabled"
          :value="true"
          :unchecked-value="false"
        >
          {{ $t('enabled') }}
        </b-form-checkbox>
      </b-form-group>

      <b-button
        class="float-right mb-3"
        @click="newOIDC()"
      >
        {{ $t('oidc.add') }}
      </b-button>

      <b-table
        :items="providers.items"
        :fields="providers.fields"
        :tbody-tr-class="(i) => i.rowBackground"
      >
        <template #cell(enabled)="{ item }">
          <b-checkbox
            :checked="item.enabled"
            @change="item.enable($event)"
          />
        </template>

        <template #cell(provider)="{ item }">
          {{ item.provider }}
          <b-badge
            v-if="item.tag"
            class="ml-1"
          >
            {{ item.tag }}
          </b-badge>
        </template>

        <template #cell(editor)="{ item }">
          <confirmation-toggle
            v-if="item.delete"
            cta-class="link"
            @confirmed="item.delete()"
          >
            <font-awesome-icon :icon="['far', 'trash-alt']" />
          </confirmation-toggle>
          <b-button
            variant="link"
            @click="openEditor(item.editor)"
          >
            <font-awesome-icon
              :icon="['fas', 'wrench']"
            />
          </b-button>
        </template>
      </b-table>

      <b-modal
        v-model="modal.open"
        :title="modal.title"
        size="lg"
        title-class="text-capitalize"
        @ok="modal.updater(modal.data)"
      >
        <component
          :is="modal.component"
          v-model="modal.data"
        />
      </b-modal>
    </b-form>

    <template #header>
      <h3 class="m-0">
        {{ $t('title') }}
      </h3>
    </template>

    <template #footer>
      <c-submit-button
        class="float-right"
        :disabled="!dirty || !canManage"
        :processing="processing"
        :success="success"
        @submit="$emit('submit', changes)"
      />
    </template>
  </b-card>
</template>

<script>
import _ from 'lodash'
import CSubmitButton from 'corteza-webapp-admin/src/components/CSubmitButton'
import OidcExternal from 'corteza-webapp-admin/src/components/Settings/System/Auth/ExternalOIDC'
import StandardExternal from 'corteza-webapp-admin/src/components/Settings/System/Auth/ExternalStd'
import SamlExternal from 'corteza-webapp-admin/src/components/Settings/System/Auth/ExternalSAML'
import ConfirmationToggle from 'corteza-webapp-admin/src/components/ConfirmationToggle'

const idpStandard = [
  'google',
  'github',
  'facebook',
  'linkedin',
]

const idpSecurity = {
  permittedRoles: [],
  forbiddenRoles: [],
  forcedRoles: [],
}

/**
 * Give some structure to key-val settings
 */
function prepareExternal (external) {
  const extractKey = (settings = {}, name, t = 'string') => {
    const v = settings.find(v => v.name === `auth.external.${name}`)

    switch (t) {
      case 'string':
        return (v || { value: null }).value || ''
      case 'boolean':
        return !!(v || { value: null }).value
      case 'array':
        return (v || { value: [] }).value || []
    }
  }

  const extractKeys = (provider, settings = {}, base = {}) => {
    let out = { ...base }

    for (let k in base) {
      out[k] = extractKey(settings, `providers.${provider}.${k}`, typeof out[k])
    }

    return out
  }

  const extractSec = (prefix) => ({
    permittedRoles: extractKey(external, `${prefix}.permittedRoles`, 'array'),
    forbiddenRoles: extractKey(external, `${prefix}.forbiddenRoles`, 'array'),
    forcedRoles: extractKey(external, `${prefix}.forcedRoles`, 'array'),
  })

  const data = {
    enabled: !!(external.find(v => v.name === 'auth.external.enabled') || {}).value,

    oidc: [],
    standard: [],

    saml: {
      enabled: extractKey(external, 'saml.enabled'),
      cert: extractKey(external, 'saml.cert'),
      name: extractKey(external, 'saml.name'),
      key: extractKey(external, 'saml.key'),
      'sign-method': extractKey(external, 'saml.sign-method'),
      'sign-requests': extractKey(external, 'saml.sign-requests', 'boolean'),
      binding: extractKey(external, 'saml.binding'),
      idp: {
        url: extractKey(external, 'saml.idp.url'),
        'ident-name': extractKey(external, 'saml.idp.ident-name'),
        'ident-handle': extractKey(external, 'saml.idp.ident-handle'),
        'ident-identifier': extractKey(external, 'saml.idp.ident-identifier'),
      },
      security: extractSec('saml.security'),
    },
  }

  data.standard = idpStandard.map(handle => ({
    handle,
    ...extractKeys(handle, external, {
      enabled: false,
      secret: '',
      key: '',
      security: {},
    }),
    security: extractSec(handle),
  }))

  const prefix = `auth.external.providers.openid-connect.`

  data.oidc =
    [...new Set(external
      // Filter out all keys that start with prefix
      .filter(v => v.name.indexOf(prefix) === 0)
      // trim off the prefix and everything after next.
      .map(({ name }) => name.substring(prefix.length).split('.', 2)[0]))]
      .map(handle => ({
        ...extractKeys('openid-connect.' + handle, external, {
          enabled: false,
          issuer: '',
          key: '',
          secret: '',
          scope: '',
          security: {},
        }),
        handle,
        security: extractSec('openid-connect.' + handle),
        deleted: false,
      }))

  return data
}

export default {
  name: 'CSystemEditorExternal',

  i18nOptions: {
    namespaces: 'system.settings',
    keyPrefix: 'editor.external',
  },

  components: {
    CSubmitButton,
    OidcExternal,
    StandardExternal,
    SamlExternal,
    ConfirmationToggle,
  },

  props: {
    value: {
      type: Array,
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

    canManage: {
      type: Boolean,
      required: true,
    },
  },

  data () {
    return {
      // current open modal
      modal: {
        open: false,
        editor: null,
        title: null,
        data: null,
      },

      // working copy
      external: prepareExternal(this.value),

      roles: [],
    }
  },

  computed: {
    dirty () {
      return this.changes.length > 0
    },

    // copy of all values for comparison
    original () {
      return Object.freeze(prepareExternal(this.value))
    },

    /**
     * Configuration for table that shows external providers
     *
     * @returns object
     */
    providers () {
      return {
        fields: [
          { key: 'enabled', label: '', thStyle: { width: '50px' } },
          { key: 'provider', label: this.$t('table.header.provider'), thStyle: { width: '200px' }, tdClass: 'text-capitalize' },
          { key: 'info', label: this.$t('table.header.info') },
          { key: 'editor', label: '', thStyle: { width: '200px' }, tdClass: 'text-right' },
        ],
        items: [
          {
            rowBackground: _.isEqual(this.original.saml, this.external.saml) ? '' : 'bg-warning',
            provider: this.external.saml.name,
            info: this.external.saml.idp.url,
            tag: 'SAML',

            enabled: this.external.saml.enabled,
            enable: (val) => this.$set(this.external.saml, 'enabled', val),

            editor: {
              component: 'saml-external',
              data: this.external.saml,
              title: this.$t('saml.title'),
              updater: (changed) => this.updater('saml', changed),
            },
          },
          ...this.external.oidc
            .map((p, i) => ({
              rowBackground: (() => {
                if (_.isEqual(this.original.oidc[i], p)) {
                  return ''
                }

                if (p.deleted) {
                  return 'text-light deleted'
                }

                return 'bg-warning'
              })(),
              provider: p.handle,
              tag: 'OIDC',
              info: p.issuer,

              enabled: p.enabled,
              enable: (val) => this.$set(this.external.oidc[i], 'enabled', val),
              delete: () => this.$set(this.external.oidc[i], 'deleted', !p.deleted),

              editor: {
                component: 'oidc-external',
                data: p,
                title: p.handle,
                updater: (changed) => this.updater('oidc', changed, i),
              },
            })),
          ...this.external.standard.map((p, i) => ({
            rowBackground: _.isEqual(this.original.standard[i], p) ? '' : 'bg-warning',
            provider: p.handle,
            info: p.key,

            enabled: p.enabled,
            enable: (val) => this.$set(this.external.standard[i], 'enabled', val),

            editor: {
              component: 'standard-external',
              data: p,
              title: p.handle,
              updater: (changed) => this.updater('standard', changed, i),
            },
          })),
        ],
      }
    },

    /**
     * Converts changed settings back to values
     */
    changes () {
      let name, value
      let c = []

      const prefix = 'auth.external.providers'
      const o = this.original
      const e = this.external

      if (!_.isEqual(o.enabled, e.enabled)) {
        c.push({ name: 'auth.external.enabled', value: e.enabled })
      }

      /**
       * General purpose key mapper
       */
      const mapKeys = (prefix, wc, org, keys) => {
        for (const k of keys) {
          if (wc[k] === undefined) {
            console.error(`potential issue - unknown key "${prefix}.${k}" used`, wc)
          }

          if (_.isEqual(wc[k], org[k])) {
            continue
          }

          name = `${prefix}.${k}`
          value = wc[k]
          c.push({ name, value })
        }
      }

      e.standard.forEach((p, i) => {
        mapKeys(
          `${prefix}.${p.handle}`,
          p,
          o.standard[i],
          ['key', 'secret', 'enabled', 'security'],
        )
      })

      // @todo how do we remove OIDC?
      const oidcKeys = ['key', 'secret', 'enabled', 'issuer', 'scope', 'security']
      e.oidc.forEach((p, i) => {
        if (p.deleted) {
          // use base set of keys used for OIDC provider
          // plus add a few extra ones that are usually set by the backend
          //
          // @todo it would probably smarter to trigger the backend that all settings
          //       with a certain key prefix should be deleted.
          //       Unfortunately, backend does not support that for now + could be a bit dangerous
          [...oidcKeys, 'weight', 'redirect', 'label']
            .forEach(name => c.push({ name: `${prefix}.openid-connect.${p.handle}.${name}`, value: null }))
        } else {
          mapKeys(
            `${prefix}.openid-connect.${p.handle}`,
            p,
            o.oidc[i] || {},
            oidcKeys
          )
        }
      })

      mapKeys(
        `auth.external.saml`,
        e.saml,
        o.saml,
        ['enabled', 'name', 'key', 'cert', 'sign-method', 'sign-requests', 'binding', 'security']
      )

      mapKeys(
        `auth.external.saml.idp`,
        e.saml.idp,
        o.saml.idp,
        ['url', 'ident-name', 'ident-handle', 'ident-identifier']
      )

      return c
    },
  },

  watch: {
    value: {
      immediate: true,
      handler () {
        this.external = prepareExternal(this.value)
      },
    },
  },

  methods: {
    openEditor ({ component, title, data, updater }) {
      this.modal.open = true
      this.modal.component = component
      this.modal.title = title
      this.modal.updater = updater

      // deref
      this.modal.data = JSON.parse(JSON.stringify(data))
    },

    newOIDC () {
      this.openEditor({
        component: 'oidc-external',
        title: this.$t('oidc.add'),
        data: {
          handle: '',
          enabled: true,
          issuer: '',
          key: '',
          secret: '',
          scope: '',
          fresh: true,
          security: { ...idpSecurity },
        },
        updater: (changed) => {
          this.updater('oidc', changed, -1)
        },
      })
    },

    /**
     * Please note that we're assuming that array items will not get changed!
     *
     * @param key
     * @param i
     * @param val
     */
    updater (key, val, i = undefined) {
      if (i === undefined) {
        this.$set(this.external, key, val)
      } else if (i < 0) {
        this.external[key].push(val)
      } else {
        this.$set(this.external[key], i, val)
      }
    },
  },
}
</script>
<style lang="scss">
.deleted {
  text-decoration: line-through
}
</style>
