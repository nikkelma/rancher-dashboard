<script>
import { cleanForNew } from '@/plugins/steve/normalize';
import CreateEditView from '@/mixins/create-edit-view/impl';
import Loading from '@/components/Loading';
import ResourceYaml from '@/components/ResourceYaml';
import {
  _VIEW, _EDIT, _CLONE, _STAGE, _CREATE,
  AS, _YAML, _DETAIL, _CONFIG,
} from '@/config/query-params';
import { SCHEMA } from '@/config/types';
import { createYaml } from '@/utils/create-yaml';
import Masthead from '@/components/ResourceDetail/Masthead';
import DetailTop from '@/components/DetailTop';
import Vue from 'vue';
import isEqual from 'lodash/isEqual';

function modeFor(route) {
  if ( route.params.id ) {
    return route.query.mode || _VIEW;
  } else {
    return _CREATE;
  }
}

async function getYaml(model) {
  let yaml;
  const opt = { headers: { accept: 'application/yaml' } };

  if ( model.hasLink('rioview') ) {
    yaml = (await model.followLink('rioview', opt)).data;
  } else if ( model.hasLink('view') ) {
    yaml = (await model.followLink('view', opt)).data;
  }

  return yaml;
}

// Components can't have asyncData, only pages.
// So you have to call this in the page and pass it in as a prop.
export async function asyncData(ctx) {
  const { params, store } = ctx;
  const resource = params.resource;
  const hasCustomEdit = store.getters['type-map/hasCustomEdit'](resource);
  const hasCustomDetail = store.getters['type-map/hasCustomDetail'](resource);
  let editComponent, detailComponent;

  if ( hasCustomDetail ) {
    const importer = store.getters['type-map/importDetail'](resource);

    detailComponent = (await importer())?.default;
  }

  if ( hasCustomEdit ) {
    const importer = store.getters['type-map/importEdit'](resource);

    editComponent = (await importer())?.default;
  }

  return { editComponent, detailComponent };
}

export default {
  components: {
    Loading,
    DetailTop,
    ResourceYaml,
    Masthead,
  },

  mixins: [CreateEditView],

  props: {
    // ** NOTE: These are provided by asyncData() above, you must call asyncData page using this
    // component and then pass the output into this component as props. **
    detailComponent: {
      type:    Object,
      default: null,
    },
    editComponent: {
      type:    Object,
      default: null,
    }
  },

  async fetch() {
    const store = this.$store;
    const route = this.$route;
    const params = route.params;
    const inStore = store.getters['currentProduct']?.inStore;
    const realMode = this.realMode;

    // eslint-disable-next-line prefer-const
    let { namespace, id } = params;
    let resource = params.resource;

    // There are 5 "real" modes that can be put into the query string
    // These are mapped down to the 3 regular page "mode"s that create-edit-view components
    // know about:  view, edit, create (stage and clone become "create")
    const mode = ((realMode === _STAGE || realMode === _CLONE) ? _CREATE : realMode);

    const hasCustomDetail = store.getters['type-map/hasCustomDetail'](resource);
    const hasCustomEdit = store.getters['type-map/hasCustomEdit'](resource);
    const yamlOnlyDetail = store.getters['type-map/isYamlOnlyDetail'](resource);
    const schemas = store.getters[`${ inStore }/all`](SCHEMA);

    // As determines what component will be rendered
    const requested = route.query[AS];
    let as;

    if ( mode === _VIEW && hasCustomDetail && (!requested || requested === _DETAIL) ) {
      as = _DETAIL;
    } else if ( hasCustomEdit && (!requested || requested === _CONFIG) ) {
      as = _CONFIG;
    } else {
      as = _YAML;
    }

    this.as = as;

    const override = this.parentOverride;

    if ( override.resource ) {
      resource = override.resource;
    }

    const schema = store.getters[`${ inStore }/schemaFor`](resource);
    let originalModel, model, yaml;

    if ( realMode === _CREATE ) {
      if ( !namespace ) {
        namespace = store.getters['defaultNamespace'];
      }

      const data = { type: resource };

      if ( schema.attributes?.namespaced ) {
        data.metadata = { namespace };
      }

      originalModel = await store.dispatch(`${ inStore }/create`, data);
      model = originalModel;

      if ( as === _YAML ) {
        yaml = createYaml(schemas, resource, data);
      }
    } else {
      let fqid = id;

      if ( schema.attributes?.namespaced && namespace ) {
        fqid = `${ namespace }/${ fqid }`;
      }

      originalModel = await store.dispatch(`${ inStore }/find`, {
        type: resource,
        id:   fqid,
        opt:  { watch: true }
      });

      if (realMode === _VIEW) {
        model = originalModel;
      } else {
        model = await store.dispatch(`${ inStore }/clone`, { resource: originalModel });
      }

      if ( as === _YAML ) {
        yaml = await getYaml(originalModel);

        if ( realMode === _CLONE || realMode === _STAGE ) {
          cleanForNew(model);
          yaml = model.cleanYaml(yaml, realMode);
        }
      }
    }

    // Ensure labels & annotations exists, since lots of things need them
    if ( !model.metadata ) {
      Vue.set(model, 'metadata', {});
    }

    if ( !model.metadata.annotations ) {
      Vue.set(model, 'metadata.annotations', {});
    }

    if ( !model.metadata.labels ) {
      Vue.set(model, 'metadata.labels', {});
    }

    const out = {
      hasCustomDetail,
      hasCustomEdit,
      yamlOnlyDetail,
      resource,
      as,
      yaml,
      originalModel,
      mode,
      value: model,
    };

    for ( const key in out ) {
      this[key] = out[key];
    }

    if ( this.mode === _CREATE ) {
      this.value.applyDefaults(this, this.mode);
    }
  },

  data() {
    return {
      resourceSubtype: null,

      // Set by asyncData & passed in as props
      // detailComponent: null,
      // editComponent:   null,

      // Set by fetch
      hasCustomDetail: null,
      hasCustomEdit:   null,
      yamlOnlyDetail:  null,
      resource:        null,
      asYaml:          null,
      yaml:            null,
      originalModel:   null,
      mode:            null,
      as:              null,
      value:           null,
      model:           null,
    };
  },

  computed: {
    realMode() {
      // There are 5 "real" modes that you can start in: view, edit, create, stage, clone
      const realMode = modeFor(this.$route);

      return realMode;
    },

    isView() {
      return this.mode === _VIEW;
    },

    isYaml() {
      return this.as === _YAML;
    },

    isDetail() {
      return this.as === _DETAIL;
    },

    offerPreview() {
      return this.as === _YAML && [_EDIT, _CLONE, _STAGE].includes(this.mode);
    },

    showComponent() {
      switch ( this.as ) {
      case _DETAIL: return this.detailComponent;
      case _CONFIG: return this.editComponent;
      }

      return null;
    },

    parentOverride() {
      const over = this.showComponent?.parentOverride;

      if ( !over ) {
        return {};
      }

      if ( typeof over === 'function' ) {
        return over.apply(this) || {};
      }

      return over || {};
    },

    yamlSave() {
      return this.parentOverride?.yamlSave;
    },
  },

  watch: {
    '$route.query'(neu, old) {
      if ( !isEqual(neu, old) ) {
        this.$fetch();
      }
    },

    // Auto refresh YAML when the model changes
    async 'value.metadata.resourceVersion'(a, b) {
      if ( this.mode === _VIEW && this.as === _YAML && a && b && a !== b) {
        this.yaml = await getYaml(this.originalModel);
      }
    }
  },

  methods: {
    setSubtype(subtype) {
      this.resourceSubtype = subtype;
    }
  }
};
</script>

<template>
  <Loading v-if="$fetchState.pending" />
  <div v-else>
    <Masthead
      :value="originalModel"
      :mode="mode"
      :real-mode="realMode"
      :as="as"
      :parent-override="parentOverride"
      :has-detail="hasCustomDetail"
      :has-edit="hasCustomEdit"
      :resource-subtype="resourceSubtype"
    />

    <DetailTop
      v-if="isView && isDetail"
      :value="originalModel"
    />

    <ResourceYaml
      v-if="isYaml"
      ref="resourceyaml"
      v-model="value"
      :mode="mode"
      :yaml="yaml"
      :offer-preview="offerPreview"
      :done-route="doneRoute"
      :done-override="value.doneOverride"
      :save-override="yamlSave"
    />

    <component
      :is="showComponent"
      v-else
      ref="comp"
      v-model="value"
      v-bind="_data"
      :done-params="doneParams"
      :done-route="doneRoute"
      :mode="mode"
      :original-value="originalModel"
      :real-mode="realMode"
      @set-subtype="setSubtype"
    />
  </div>
</template>

<style lang='scss' scoped>
  .actions > * {
    display: inline-block;
  }

  .flat {
    border-collapse: collapse;
    table-layout: fixed;
    width: 100%;
    & th{
      padding-bottom: 1rem;
      text-align: left;
      font-weight: normal;
      color: var(--secondary);
    }

    & :not(THEAD) tr{
      border-bottom: 1px solid var(--border);

      & td {
        padding: 10px 0 10px 0;
      }
    }
    & tr td:last-child, th:last-child{
      text-align: right;
    }

    & tr td:first-child, th:first-child{
      text-align: left;
      margin-left: 15px;
    }

    & .click-row a{
      color: var(--input-text);
    }

    & .click-row:hover{
      @extend .faded;
    }

    & .faded {
      opacity: 0.3
    }
  }
</style>
