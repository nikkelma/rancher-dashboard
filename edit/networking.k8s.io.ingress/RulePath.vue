<script>
import InputWithSelect from '@/components/form/InputWithSelect';
import LabeledInput from '@/components/form/LabeledInput';
import Select from '@/components/form/Select';
import { get, set } from '@/utils/object';
import debounce from 'lodash/debounce';

export default {
  components: {
    InputWithSelect, LabeledInput, Select
  },
  props:      {
    value: {
      type:    Object,
      default: () => {
        return {};
      }
    },
    serviceTargets:  {
      type:    Array,
      default: () => []
    },
    ingress: {
      type:     Object,
      required: true
    }
  },
  data() {
    const pathTypes = [
      'Prefix',
      'Exact',
      'ImplementationSpecific'
    ];

    set(this.value, 'backend', this.value.backend || {});
    set(this.value, 'path', this.value.path || '');
    set(this.value, 'pathType', this.value.pathType || pathTypes[0]);
    set(this.value.backend, this.ingress.serviceNamePath, get(this.value.backend, this.ingress.serviceNamePath) || '');
    set(this.value.backend, this.ingress.servicePortPath, get(this.value.backend, this.ingress.servicePortPath) || '');

    const serviceName = get(this.value.backend, this.ingress.serviceNamePath);
    const servicePort = get(this.value.backend, this.ingress.servicePortPath);

    return {
      pathTypes, serviceName, servicePort, pathType: this.value.pathType, path: this.value.path
    };
  },
  computed: {
    portOptions() {
      const service = this.serviceTargets.find(s => s.label === this.serviceName);

      return service?.ports || [];
    },
    serviceTargetStatus() {
      const serviceName = this.serviceName.label || this.serviceName;
      const isValueAnOption = !serviceName || this.serviceTargets.find(target => serviceName === target.value);

      return isValueAnOption ? null : 'warning';
    },
    serviceTargetTooltip() {
      return this.serviceTargetStatus === 'warning' ? this.t('ingress.rules.target.doesntExist') : null;
    }
  },
  created() {
    this.queueUpdate = debounce(this.update, 500);
    this.queueUpdatePathTypeAndPath = debounce(this.updatePathTypeAndPath, 500);
  },
  methods: {
    update() {
      const servicePort = Number.parseInt(this.servicePort) || this.servicePort;
      const serviceName = this.serviceName.label || this.serviceName;
      const out = {
        backend: {}, path: this.path, pathType: this.pathType
      };

      set(out.backend, this.ingress.servicePortPath, servicePort);
      set(out.backend, this.ingress.serviceNamePath, serviceName);

      this.$emit('input', out);
    },
    updatePathTypeAndPath(values) {
      this.path = values.text;
      this.pathType = values.selected;
      this.update();
    }
  }
};
</script>
<template>
  <div class="rule-path row">
    <div v-if="ingress.showPathType" class="col span-6">
      <InputWithSelect
        class="path-type"
        :options="pathTypes"
        :placeholder="t('ingress.rules.path.placeholder', undefined, true)"
        :select-value="value.pathType"
        :text-value="value.path"
        :searchable="false"
        @input="queueUpdatePathTypeAndPath"
      />
    </div>
    <div v-else class="col span-4">
      <input v-model="path" :placeholder="t('ingress.rules.path.placeholder', undefined, true)" @input="queueUpdate" />
    </div>
    <div class="col" :class="{'span-3': ingress.showPathType, 'span-4': !ingress.showPathType}">
      <Select
        v-model="serviceName"
        option-label="label"
        option-key="label"
        :options="serviceTargets"
        :status="serviceTargetStatus"
        :taggable="true"
        :searchable="true"
        :tooltip="serviceTargetTooltip"
        :hover-tooltip="true"
        @input="queueUpdate(); servicePort = ''"
      />
    </div>
    <div class="col" :class="{'span-2': ingress.showPathType, 'span-3': !ingress.showPathType}" :style="{'margin-right': '0px'}">
      <LabeledInput
        v-if="portOptions.length === 0"
        v-model="servicePort"
        :placeholder="t('ingress.rules.port.placeholder')"
        @input="queueUpdate"
      />
      <Select
        v-else
        v-model="servicePort"
        :options="portOptions"
        :placeholder="t('ingress.rules.port.placeholder')"
        @input="queueUpdate"
      />
    </div>
    <button class="btn btn-sm role-link col" @click="$emit('remove')">
      {{ t('ingress.rules.removePath') }}
    </button>
  </div>
</template>
<style lang="scss" scoped>
.rule-path ::v-deep {
  .path-type {
    .unlabeled-select {
      min-width: 200px;
    }
  }

  &, .input-container {
    height: $input-height;
  }

  .input-container .in-input.unlabeled-select {
    width: initial;
  }

  button {
    line-height: 40px;
  }

  .v-select INPUT {
    height: 50px;
  }
  .labeled-input {
    padding-top: 6px;
  }
}
</style>
