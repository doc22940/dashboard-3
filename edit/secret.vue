<script>
import { DOCKER_JSON, OPAQUE, TLS } from '@/models/secret';
import { base64Encode, base64Decode } from '@/utils/crypto';
import { get } from '@/utils/object';
import { NAMESPACE, SECRET } from '@/config/types';
import { DESCRIPTION } from '@/config/labels-annotations';
import CreateEditView from '@/mixins/create-edit-view';
import Footer from '@/components/form/Footer';
import KeyValue from '@/components/form/KeyValue';
import LabeledInput from '@/components/form/LabeledInput';
import RadioGroup from '@/components/form/RadioGroup';
import NameNsDescription from '@/components/form/NameNsDescription';
import LabeledSelect from '@/components/form/LabeledSelect';

export default {
  name: 'CruSecret',

  components: {
    KeyValue,
    Footer,
    LabeledInput,
    LabeledSelect,
    RadioGroup,
    NameNsDescription
  },
  mixins:     [CreateEditView],
  data() {
    const types = [
      { label: 'Certificate', value: TLS },
      { label: 'Registry', value: DOCKER_JSON },
      { label: 'Secret', value: OPAQUE },
    ];
    const registryAddresses = [
      'DockerHub', 'Quay.io', 'Artifactory', 'Custom'
    ];
    const isNamespaced = !!this.value.metadata.namespace;

    let username;
    let password;
    let registryFQDN;

    if (this.value._type === DOCKER_JSON) {
      const json = base64Decode(this.value.data['.dockerconfigjson']);

      const { auths } = JSON.parse(json);

      registryFQDN = Object.keys(auths)[0];

      username = auths[registryFQDN].username;
      password = auths[registryFQDN].password;
    }

    return {
      // define 'type' as secret so .save() function uses secret schema instead of looking for subtype schema
      type:             SECRET,
      types,
      isNamespaced,
      registryAddresses,
      newNS:            false,
      registryProvider: registryAddresses[0],
      username,
      password,
      registryFQDN,
      toUpload:         null,
      key:              null,
      cert:             null
    };
  },

  computed: {
    certificate() {
      return TLS;
    },

    name: {
      get() {
        return get(this.value, 'metadata.name');
      },
      set(neu) {
        this.$set(this.value.metadata, 'name', neu);
      }
    },

    description: {
      get() {
        const { metadata:{ annotations = {} } } = this.value;

        return annotations[DESCRIPTION] || '';
      },
      set(neu) {
        this.$set(this.value.metadata.annotations, DESCRIPTION, neu );
      }
    },

    secretSubType: {
      get() {
        return this.value._type || OPAQUE;
      },
      set(neu) {
        this.$set(this.value, '_type', neu);
      }
    },

    namespace: {
      get() {
        if (this.isNamespaced) {
          return get(this.value, 'metadata.namespace') || 'default';
        } else {
          return 'n/a';
        }
      },
      set(neu) {
        this.$set(this.value.metadata, 'namespace', neu);
      }
    },

    dockerconfigjson() {
      let dockerServer = this.registryProvider === 'DockerHub' ? 'index.dockerhub.io/v1/' : 'quay.io';

      if (this.needsDockerServer) {
        dockerServer = this.registryFQDN;
      }

      if (this.isRegistry && dockerServer) {
        const config = {
          auths: {
            [dockerServer]: {
              username: this.username,
              password: this.password,
            }
          }
        };
        const json = JSON.stringify(config);

        return json;
      } else {
        return null;
      }
    },

    namespaces() {
      return this.$store.getters['cluster/all'](NAMESPACE).map((obj) => {
        return {
          label: obj.nameDisplay,
          value: obj.id,
        };
      });
    },

    isRegistry() {
      return this.secretSubType === DOCKER_JSON;
    },

    needsDockerServer() {
      return this.registryProvider === 'Artifactory' || this.registryProvider === 'Custom';
    },
  },

  methods: {
    saveSecret(buttonCB) {
      if (this.secretSubType === DOCKER_JSON) {
        const data = { '.dockerconfigjson': base64Encode(this.dockerconfigjson) };

        this.$set(this.value, 'data', data);
      } else if (this.secretSubType === TLS) {
        const data = { 'tls.cert': base64Encode(this.cert), 'tls.key': base64Encode(this.key) };

        this.$set(this.value, 'data', data);
      }
      this.save(buttonCB);
    },

    fileUpload(field) {
      this.toUpload = field;
      this.$refs.uploader.click();
    },

    fileChange(event) {
      const input = event.target;
      const handles = input.files;
      const names = [];

      if ( handles ) {
        for ( let i = 0 ; i < handles.length ; i++ ) {
          const reader = new FileReader();

          reader.onload = (loaded) => {
            const value = loaded.target.result;

            this[this.toUpload] = value;
          };

          reader.onerror = (err) => {
            this.$dispatch('growl/fromError', { title: 'Error reading file', err }, { root: true });
          };

          names[i] = handles[i].name;
          reader.readAsText(handles[i]);
        }

        input.value = '';
      }
    },
  }
};
</script>

<template>
  <form>
    <NameNsDescription v-model="value" :mode="mode" :extra-columns="['type']">
      <template v-slot:type>
        <LabeledSelect v-model="secretSubType" label="Type" :options="types" />
      </template>
    </NameNsDescription>
    <template v-if="isRegistry">
      <div id="registry-type" class="row">
        Provider: &nbsp; <RadioGroup :style="{'display':'flex'}" :options="registryAddresses" :value="registryProvider" @input="e=>registryProvider = e" />
      </div>
      <div v-if="needsDockerServer" class="row">
        <LabeledInput v-model="registryFQDN" label="Registry Domain Name" placeholder="e.g. index.docker.io" />
      </div>
      <div class="row">
        <div class="col span-6">
          <LabeledInput v-model="username" label="Username" />
        </div>
        <div class="col span-6">
          <LabeledInput v-model="password" label="Password" />
        </div>
      </div>
    </template>

    <div v-else-if="secretSubType===certificate" class="row">
      <div class="col span-6">
        <LabeledInput v-model="key" label="Private Key" />
        <button type="button" class="btn btn-sm bg-primary mt-10" @click="fileUpload('key')">
          READ FROM FILE
        </button>
      </div>
      <div class="col span-6">
        <LabeledInput v-model="cert" label="CA Certificate" />
        <button type="button" class="btn btn-sm bg-primary mt-10" @click="fileUpload('cert')">
          READ FROM FILE
        </button>
      </div>
    </div>

    <div v-else class="row">
      <KeyValue
        key="data"
        v-model="value.data"
        :mode="mode"
        title="Data"
        :initial-empty-row="true"
        :value-base64="true"
        read-icon=""
        add-icon=""
        add-label="ADD"
        read-label="READ FROM A FILE"
      />
    </div>

    <input
      ref="uploader"
      type="file"
      class="hide"
      @change="fileChange"
    />

    <Footer :mode="mode" :errors="errors" @save="saveSecret" @done="done" />
  </form>
</template>

<style>
#registry-type {
  display: flex;
  align-items:center;
}
</style>
