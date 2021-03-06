<script>
import cronstrue from 'cronstrue';
import { clone } from '@/utils/object';
import { CONFIG_MAP, SECRET, WORKLOAD, NODE } from '@/config/types';
import LoadDeps from '@/mixins/load-deps';
import Tab from '@/components/Tabbed/Tab';
import Tabbed from '@/components/Tabbed';
import CreateEditView from '@/mixins/create-edit-view';
import { allHash } from '@/utils/promise';
import NameNsDescription from '@/components/form/NameNsDescription';
import LabeledSelect from '@/components/form/LabeledSelect';
import LabeledInput from '@/components/form/LabeledInput';
import HealthCheck from '@/components/form/HealthCheck';
import Command from '@/edit/workload/Command';
import Security from '@/edit/workload/Security';
import Scheduling from '@/edit/workload/Scheduling';
import Upgrading from '@/edit/workload/Upgrading';
import Networking from '@/edit/workload/Networking';
import Footer from '@/components/form/Footer';
import Job from '@/edit/workload/Job';
import Labels from '@/components/form/Labels';
import WorkloadPorts from '@/edit/workload/WorkloadPorts';

export default {
  name:       'CruWorkload',
  components: {
    NameNsDescription,
    LabeledSelect,
    LabeledInput,
    Tabbed,
    Tab,
    HealthCheck,
    Command,
    Security,
    Scheduling,
    Upgrading,
    Labels,
    Networking,
    Footer,
    Job,
    WorkloadPorts
  },

  mixins:     [CreateEditView, LoadDeps],
  props:  {
    value: {
      type:     Object,
      required: true,
    },
    mode:     {
      type:    String,
      default: 'create'
    }
  },

  data() {
    const typeOpts = [];
    const workloadMap = {
      [WORKLOAD.DEPLOYMENT]:             'Deployment',
      [WORKLOAD.DAEMON_SET]:             'Daemon Set',
      [WORKLOAD.STATEFUL_SET]:           'Stateful Set',
      [WORKLOAD.REPLICA_SET]:            'Replica Set',
      [WORKLOAD.JOB]:                    'Job',
      [WORKLOAD.CRON_JOB]:               'Cron Job',
      [WORKLOAD.REPLICATION_CONTROLLER]: 'Replication Controller'
    };

    for (const key in workloadMap) {
      typeOpts.push({ value: key, label: workloadMap[key] });
    }

    const selectNode = false;

    if ( !this.value.spec ) {
      this.value.spec = {};
    }
    let type = this.value._type || this.value.type || WORKLOAD.DEPLOYMENT;

    if (type === 'workload') {
      type = WORKLOAD.DEPLOYMENT;
    }

    const spec = this.value.spec ? clone(this.value.spec) : {};
    const metadata = this.value.metadata ? clone(this.value.metadata) : {};

    if (!spec.template) {
      spec.template = { spec: { restartPolicy: this.isJob ? 'Never' : 'Always' } };
    }

    return {
      selectNode,
      type,
      typeOpts,
      spec,
      metadata,
      allConfigMaps: null,
      allSecrets:    null,
      allNodes:      null,
      showTabs:      false
    };
  },

  computed: {
    schema() {
      return this.$store.getters['cluster/schemaFor']( this.type );
    },

    namespace() {
      return this.metadata.namespace;
    },

    container: {
      get() {
        let template = this.spec.template;

        if (this.isCronJob) {
          template = this.spec.jobTemplate.spec.template;
        }
        const { containers } = template.spec;

        if (!containers) {
          this.$set(template.spec, 'containers', [{ name: this.metadata.name }]);
        }

        return template.spec.containers[0];
      },

      set(neu) {
        let template = this.spec.template;

        if (this.isCronJob) {
          template = this.spec.jobTemplate.spec.template;
        }
        this.$set(template.spec.containers, 0, { ...neu, name: this.metadata.name });
      }
    },

    containerImage: {
      get() {
        return this.container.image;
      },
      set(neu) {
        this.container = { ...this.container, image: neu };
      }
    },

    containerPorts: {
      get() {
        return this.container.ports || [];
      },
      set(neu) {
        this.container = { ...this.container, ports: neu };
      }
    },

    canReplicate() {
      return (this.type === WORKLOAD.DEPLOYMENT || this.type === WORKLOAD.REPLICA_SET || this.type === WORKLOAD.REPLICATION_CONTROLLER || this.type === WORKLOAD.STATEFUL_SET);
    },

    isJob() {
      return this.type === WORKLOAD.JOB || this.isCronJob;
    },

    isCronJob() {
      return this.type === WORKLOAD.CRON_JOB;
    },

    cronLabel() {
      const { schedule } = this.spec;

      if (!this.isCronJob || !schedule) {
        return null;
      }

      try {
        const hint = cronstrue.toString(schedule);

        return hint;
      } catch (e) {
        return 'invalid cron expression';
      }
    },

    workloadSelector() {
      return { 'workload.user.cattle.io/workloadselector': `${ 'deployment' }-${ this.metadata.namespace }-${ this.metadata.name }` };
    },

  },

  watch: {
    type(neu, old) {
      const template = old === WORKLOAD.CRON_JOB ? this.spec?.jobTemplate?.spec?.template : this.spec?.template;

      if (!template.spec) {
        template.spec = {};
      }
      const restartPolicy = this.isJob ? 'Never' : 'Always';

      this.$set(template.spec, 'restartPolicy', restartPolicy);

      if (!this.canReplicate) {
        delete this.spec.replicas;
      }

      if (old === WORKLOAD.CRON_JOB) {
        this.$set(this.spec, 'template', { ...template });
        delete this.spec.jobTemplate;
        delete this.spec.schedule;
      } else if (neu === WORKLOAD.CRON_JOB) {
        this.$set(this.spec, 'jobTemplate', { spec: { template } });
        this.$set(this.spec, 'schedule', '0 * * * *');
        delete this.spec.template;
      }

      this.$set(this.value, 'type', neu);
      delete this.value.apiVersion;
    }
  },

  methods: {
    async loadDeps() {
      const hash = await allHash({
        configMaps: this.$store.dispatch('cluster/findAll', { type: CONFIG_MAP }),
        secrets:    this.$store.dispatch('cluster/findAll', { type: SECRET }),
        nodes:      this.$store.dispatch('cluster/findAll', { type: NODE })
      });

      this.allSecrets = hash.secrets;
      this.allConfigMaps = hash.configMaps;
      this.allNodes = hash.nodes.map(node => node.id);
    },

    toggleTabs() {
      this.showTabs = !this.showTabs;
    },

    saveWorkload(cb) {
      if (!this.spec.selector && this.type !== WORKLOAD.JOB) {
        this.spec.selector = { matchLabels: this.workloadSelector };
      }

      let template;

      if (this.type === WORKLOAD.CRON_JOB) {
        template = this.spec.jobTemplate;
      } else {
        template = this.spec.template;
      }

      if (!template.metadata && this.type !== WORKLOAD.JOB) {
        template.metadata = { labels: this.workloadSelector };
      }

      delete this.value.kind;
      this.value.spec = this.spec;
      this.value.metadata = this.metadata;
      this.save(cb);
    },
  },
};
</script>

<template>
  <form>
    <slot :value="value" name="top">
      <NameNsDescription :value="{metadata}" :mode="mode" :extra-columns="['type']" @input="e=>metadata=e">
        <template v-slot:type>
          <LabeledSelect v-model="type" label="Type" :options="typeOpts" />
        </template>
      </NameNsDescription>

      <div class="row">
        <div class="col span-4">
          <LabeledInput v-model="containerImage" label="Container Image" placeholder="eg nginx:latest" required />
        </div>
        <div class="col span-4" />
        <template v-if="isCronJob">
          <div class="col span-4">
            <LabeledInput v-model="spec.schedule" label="cron Schedule" />
            <span class="cron-hint text-small">{{ cronLabel }}</span>
          </div>
        </template>
        <template v-if="canReplicate">
          <div class="col span-4">
            <LabeledInput v-model.number="spec.replicas" label="Replicas" />
          </div>
        </template>
      </div>

      <div class="row">
        <WorkloadPorts v-model="containerPorts" :mode="mode" />
      </div>
    </slot>
    <Tabbed :default-tab="isJob ? 'job' : 'command'">
      <Tab v-if="isJob" label="Job Configuration" name="job">
        <Job v-model="spec" :mode="mode" :type="type" />
      </Tab>
      <Tab label="Command" name="command">
        <Command
          v-if="allConfigMaps"
          v-model="container"
          :spec="container"
          :secrets="allSecrets"
          :config-maps="allConfigMaps"
          :mode="mode"
          :namespace="namespace"
        />
      </Tab>
      <Tab label="Networking" name="networking">
        <Networking v-if="isCronJob" v-model="spec.jobTemplate.spec.template.spec" :mode="mode" />
        <Networking v-else v-model="spec.template.spec" :mode="mode" />
      </Tab>
      <Tab label="Health" name="health">
        <HealthCheck :spec="container" :mode="mode" />
      </Tab>
      <Tab label="Security" name="security">
        <Security v-if="isCronJob" v-model="spec.jobTemplate.spec.template.spec" :mode="mode" />
        <Security v-else v-model="spec.template.spec" :mode="mode" />
      </Tab>

      <Tab label="Node Scheduling" name="scheduling">
        <Scheduling v-if="isCronJob" v-model="spec.jobTemplate.spec.template.spec" :nodes="allNodes" :mode="mode" />
        <Scheduling v-else v-model="spec.template.spec" :mode="mode" />
      </Tab>
      <Tab label="Scaling/Upgrade Policy" name="upgrading">
        <Upgrading v-model="spec" :mode="mode" />
      </Tab>

      <Tab label="Labels" name="labelsAndAnnotations">
        <Labels :spec="{metadata}" :mode="mode" />
      </Tab>
    </Tabbed>
    <Footer :errors="errors" :mode="mode" @save="saveWorkload" @done="done" />
  </form>
</template>

<style>
  .cron-hint{
    color: var(--muted);
    padding: 3px;
  }
</style>
