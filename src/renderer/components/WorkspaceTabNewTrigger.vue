<template>
   <div v-show="isSelected" class="workspace-query-tab column col-12 columns col-gapless">
      <div class="workspace-query-runner column col-12">
         <div class="workspace-query-runner-footer">
            <div class="workspace-query-buttons">
               <button
                  class="btn btn-primary btn-sm"
                  :disabled="!isChanged"
                  :class="{'loading':isSaving}"
                  title="CTRL+S"
                  @click="saveChanges"
               >
                  <i class="mdi mdi-24px mdi-content-save mr-1" />
                  <span>{{ $t('word.save') }}</span>
               </button>
               <button
                  :disabled="!isChanged"
                  class="btn btn-link btn-sm mr-0"
                  :title="$t('message.clearChanges')"
                  @click="clearChanges"
               >
                  <i class="mdi mdi-24px mdi-delete-sweep mr-1" />
                  <span>{{ $t('word.clear') }}</span>
               </button>
            </div>
            <div class="workspace-query-info">
               <div class="d-flex" :title="$t('word.schema')">
                  <i class="mdi mdi-18px mdi-database mr-1" /><b>{{ schema }}</b>
               </div>
            </div>
         </div>
      </div>
      <div class="container">
         <div class="columns">
            <div class="column col-auto">
               <div class="form-group">
                  <label class="form-label">{{ $t('word.name') }}</label>
                  <input
                     ref="firstInput"
                     v-model="localTrigger.name"
                     class="form-input"
                     type="text"
                  >
               </div>
            </div>
            <div v-if="customizations.definer" class="column col-auto">
               <div class="form-group">
                  <label class="form-label">{{ $t('word.definer') }}</label>
                  <select
                     v-if="workspace.users.length"
                     v-model="localTrigger.definer"
                     class="form-select"
                  >
                     <option value="">
                        {{ $t('message.currentUser') }}
                     </option>
                     <option v-if="!isDefinerInUsers" :value="originalTrigger.definer">
                        {{ originalTrigger.definer.replaceAll('`', '') }}
                     </option>
                     <option
                        v-for="user in workspace.users"
                        :key="`${user.name}@${user.host}`"
                        :value="`\`${user.name}\`@\`${user.host}\``"
                     >
                        {{ user.name }}@{{ user.host }}
                     </option>
                  </select>
                  <select v-if="!workspace.users.length" class="form-select">
                     <option value="">
                        {{ $t('message.currentUser') }}
                     </option>
                  </select>
               </div>
            </div>
            <fieldset class="column columns mb-0" :disabled="customizations.triggerOnlyRename">
               <div class="column col-auto">
                  <div class="form-group">
                     <label class="form-label">{{ $t('word.table') }}</label>
                     <select v-model="localTrigger.table" class="form-select">
                        <option v-for="table in schemaTables" :key="table.name">
                           {{ table.name }}
                        </option>
                     </select>
                  </div>
               </div>
               <div class="column col-auto">
                  <div class="form-group">
                     <label class="form-label">{{ $t('word.event') }}</label>
                     <div class="input-group">
                        <select v-model="localTrigger.activation" class="form-select">
                           <option>BEFORE</option>
                           <option>AFTER</option>
                        </select>
                        <select
                           v-if="!customizations.triggerMultipleEvents"
                           v-model="localTrigger.event"
                           class="form-select"
                        >
                           <option v-for="event in Object.keys(localEvents)" :key="event">
                              {{ event }}
                           </option>
                        </select>
                        <div v-if="customizations.triggerMultipleEvents" class="px-4">
                           <label
                              v-for="event in Object.keys(localEvents)"
                              :key="event"
                              class="form-checkbox form-inline"
                              @change.prevent="changeEvents(event)"
                           >
                              <input :checked="localEvents[event]" type="checkbox"><i class="form-icon" /> {{ event }}
                           </label>
                        </div>
                     </div>
                  </div>
               </div>
            </fieldset>
         </div>
      </div>
      <div class="workspace-query-results column col-12 mt-2 p-relative">
         <BaseLoader v-if="isLoading" />
         <label class="form-label ml-2">{{ $t('message.triggerStatement') }}</label>
         <QueryEditor
            v-show="isSelected"
            ref="queryEditor"
            :value.sync="localTrigger.sql"
            :workspace="workspace"
            :schema="schema"
            :height="editorHeight"
         />
      </div>
   </div>
</template>

<script>
import QueryEditor from '@/components/QueryEditor';
import { mapGetters, mapActions } from 'vuex';
import BaseLoader from '@/components/BaseLoader';
import Triggers from '@/ipc-api/Triggers';

export default {
   name: 'WorkspaceTabNewTrigger',
   components: {
      BaseLoader,
      QueryEditor
   },
   props: {
      connection: Object,
      tab: Object,
      isSelected: Boolean,
      schema: String
   },
   data () {
      return {
         isLoading: false,
         isSaving: false,
         originalTrigger: {},
         localTrigger: {},
         lastTrigger: null,
         sqlProxy: '',
         editorHeight: 300,
         localEvents: { INSERT: false, UPDATE: false, DELETE: false }
      };
   },
   computed: {
      ...mapGetters({
         selectedWorkspace: 'workspaces/getSelected',
         getWorkspace: 'workspaces/getWorkspace'
      }),
      workspace () {
         return this.getWorkspace(this.connection.uid);
      },
      tabUid () {
         return this.$vnode.key;
      },
      customizations () {
         return this.workspace.customizations;
      },
      isChanged () {
         return JSON.stringify(this.originalTrigger) !== JSON.stringify(this.localTrigger);
      },
      isDefinerInUsers () {
         return this.originalTrigger ? this.workspace.users.some(user => this.originalTrigger.definer === `\`${user.name}\`@\`${user.host}\``) : true;
      },
      schemaTables () {
         const schemaTables = this.workspace.structure
            .filter(schema => schema.name === this.schema)
            .map(schema => schema.tables);

         return schemaTables.length ? schemaTables[0].filter(table => table.type === 'table') : [];
      }
   },
   watch: {
      isSelected (val) {
         if (val)
            this.changeBreadcrumbs({ schema: this.schema });
      },
      isChanged (val) {
         this.setUnsavedChanges({ uid: this.connection.uid, tUid: this.tabUid, isChanged: val });
      }
   },
   created () {
      this.originalTrigger = {
         sql: this.customizations.triggerSql,
         definer: '',
         table: this.schemaTables.length ? this.schemaTables[0].name : null,
         activation: 'BEFORE',
         event: this.customizations.triggerMultipleEvents ? ['INSERT'] : 'INSERT',
         name: ''
      };

      this.localTrigger = JSON.parse(JSON.stringify(this.originalTrigger));

      setTimeout(() => {
         this.resizeQueryEditor();
      }, 50);

      window.addEventListener('keydown', this.onKey);
   },
   mounted () {
      if (this.isSelected)
         this.changeBreadcrumbs({ schema: this.schema });

      setTimeout(() => {
         this.$refs.firstInput.focus();
      }, 100);

      window.addEventListener('resize', this.resizeQueryEditor);
   },
   destroyed () {
      window.removeEventListener('resize', this.resizeQueryEditor);
   },
   beforeDestroy () {
      window.removeEventListener('keydown', this.onKey);
   },
   methods: {
      ...mapActions({
         addNotification: 'notifications/addNotification',
         refreshStructure: 'workspaces/refreshStructure',
         changeBreadcrumbs: 'workspaces/changeBreadcrumbs',
         setUnsavedChanges: 'workspaces/setUnsavedChanges',
         newTab: 'workspaces/newTab',
         removeTab: 'workspaces/removeTab',
         renameTabs: 'workspaces/renameTabs'
      }),
      changeEvents (event) {
         if (this.customizations.triggerMultipleEvents) {
            this.localEvents[event] = !this.localEvents[event];
            this.localTrigger.event = [];
            for (const key in this.localEvents) {
               if (this.localEvents[key])
                  this.localTrigger.event.push(key);
            }
         }
      },
      async saveChanges () {
         if (this.isSaving) return;
         this.isSaving = true;
         const params = {
            uid: this.connection.uid,
            schema: this.schema,
            ...this.localTrigger
         };

         try {
            const { status, response } = await Triggers.createTrigger(params);

            if (status === 'success') {
               await this.refreshStructure(this.connection.uid);

               this.newTab({
                  uid: this.connection.uid,
                  schema: this.schema,
                  elementName: this.localTrigger.name,
                  elementType: 'trigger',
                  type: 'trigger-props'
               });

               this.removeTab({ uid: this.connection.uid, tab: this.tab.uid });
               this.changeBreadcrumbs({ schema: this.schema, trigger: this.localTrigger.name });
            }
            else
               this.addNotification({ status: 'error', message: response });
         }
         catch (err) {
            this.addNotification({ status: 'error', message: err.stack });
         }

         this.isSaving = false;
      },
      clearChanges () {
         this.localTrigger = JSON.parse(JSON.stringify(this.originalTrigger));
         this.$refs.queryEditor.editor.session.setValue(this.localTrigger.sql);

         Object.keys(this.localEvents).forEach(event => {
            this.localEvents[event] = false;
         });

         if (this.customizations.triggerMultipleEvents) {
            this.originalTrigger.event.forEach(e => {
               this.localEvents[e] = true;
            });
         }
      },
      resizeQueryEditor () {
         if (this.$refs.queryEditor) {
            const footer = document.getElementById('footer');
            const size = window.innerHeight - this.$refs.queryEditor.$el.getBoundingClientRect().top - footer.offsetHeight;
            this.editorHeight = size;
            this.$refs.queryEditor.editor.resize();
         }
      },
      onKey (e) {
         if (this.isSelected) {
            e.stopPropagation();
            if (e.ctrlKey && e.keyCode === 83) { // CTRL + S
               if (this.isChanged)
                  this.saveChanges();
            }
         }
      }
   }
};
</script>
