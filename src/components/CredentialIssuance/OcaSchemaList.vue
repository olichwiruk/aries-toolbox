<template >
  <div>
    <nav class="navbar navbar-expand-lg navbar-light bg-light">
      <a class="navbar-brand" href="#">{{ title }}</a>
      <vue-bootstrap-typeahead 
        v-model="retrieve_schema_query"
        :minMatchingChars="0"
        :data="ocaSchemaSearch || []"
        :serializer="s => s.schemaName"
        @hit="getOcaSchema"
        style="width: 400px;">
        <template slot="append">
          <el-button
          slot="append"
          icon="el-icon-search"
          v-show="ocaSchema && retrieve_schema_query"
          @click="preview">Preview</el-button>
        </template>

        <template slot="suggestion" slot-scope="{ data, htmlText }">
          <small>{{ data.namespace }}/</small><span v-html="htmlText"></span>
        </template>
      </vue-bootstrap-typeahead>
      <el-button
        type="primary"
        icon="el-icon-plus"
        @click="createFormActive = true">Create</el-button>
      <el-button
        type="primary"
        icon="el-icon-refresh"
        @click="$emit('schema-refresh',)"></el-button>
    </nav>
    <el-collapse v-model="expanded_items">
      <ul class="list">
        <el-collapse-item
          v-for="schema in list"
          v-bind:title="schema.schema_name+' '+schema.schema_version"
          :name="schema.schema_id"
          :key="schema.schema_id">
          <el-row>
            <div>
              <vue-json-pretty
                :deep=1
                :data="schema">
              </vue-json-pretty>
            </div>
            <el-button v-on:click="collapse_expanded(schema)">^</el-button>
          </el-row>
        </el-collapse-item>
      </ul>
    </el-collapse>
    <el-dialog title="Create New Schema" :visible.sync="createFormActive">
      <el-form :model="createForm">
        <el-form-item label="Name:" :label-width="formLabelWidth">
          <el-input v-model="createForm.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="Version:" :label-width="formLabelWidth">
          <el-input v-model="createForm.version" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item
          v-for="(attribute, index) in createForm.attributes"
          :label="'Attribute ' + index"
          :label-width="formLabelWidth"
          :key="index">
          <el-input
            v-model="createForm.attributes[index]"
            label="'Attribute ' + index">
            <el-button
              slot="append"
              icon="el-icon-close"
              @click="remove_attribute(index)"></el-button>
          </el-input>
        </el-form-item>
        <el-button
          type="primary"
          icon="el-icon-plus"
          @click="add_attribute">Add attribute</el-button>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="createFormActive = false">Cancel</el-button>
        <el-button type="primary" @click="create">Confirm</el-button>
      </span>
    </el-dialog>

    <preview-component ref="PreviewComponent" :readonly="true" :form="ocaForm"></preview-component>
  </div>
</template>

<script>
import VueJsonPretty from 'vue-json-pretty';
import VueBootstrapTypeahead from 'vue-bootstrap-typeahead'
import axios from 'axios'

import { renderForm, PreviewComponent } from 'odca-form'

export default {
  name: 'oca-schema-list',
  props: ['title', 'list','editable'],
  components: {
    VueJsonPretty,
    VueBootstrapTypeahead,
    PreviewComponent
  },
  data () {
    return {
      expanded_items:[],
      createFormActive: false,
      createForm: {
        name: '',
        version: '',
        attributes: [],
      },
      retrieve_schema_query: '',
      ocaSchemaSearch: [],
      ocaSchema: null,
      ocaForm: null,
      formLabelWidth: '100px'
    }
  },
  watch: {
    retrieve_schema_query: function(input) {
      if(input.length == 0) {
        this.ocaSchema = null
      }
      this.fetch_oca_schemas(input)
    },
  },
  mounted() {
    this.ocaSchemaSearch = this.fetch_oca_schemas('')
  },
  methods: {
    fetch_oca_schemas: function(input) {
        axios.get(`http://localhost:9292/v2/schemas?_index=schema_base&q=${input}`)
        .then(r => {
          this.ocaSchemaSearch = r.data.map(x => {
            return {
              namespace: x.namespace,
              DRI: x.DRI,
              schemaName: x.schema.name
            }
          })
        })
    },
    collapse_expanded: function(schema){
      this.expanded_items = this.expanded_items.filter(
        item => item != schema.schema_id
      );
    },
    add_attribute: function() {
      this.createForm.attributes.push('');
    },
    remove_attribute: function(index) {
      this.createForm.attributes.splice(index, 1);
    },
    create: function() {
      let values = {
        name: this.createForm.name,
        version: this.createForm.version,
        attributes: this.createForm.attributes
      }
      this.createForm.name = '';
      this.createForm.version = '';
      this.createForm.attributes = [];
      this.$emit('schema-send', values);
      this.createFormActive = false;
    },
    getOcaSchema: async function(schema) {
      const result = await axios.get(`http://localhost:9292/v2/schemas?_index=branch&schema_base=${schema.DRI}`)
      const branch = result.data.find(e => e.namespace == schema.namespace)

      axios.get(`http://localhost:9292/v2/schemas/${branch.namespace}/${branch.DRI}`)
        .then(r => this.ocaSchema = r.data)
    },
    preview() {
      this.$refs.PreviewComponent.openModal({ label: 'Loading...', sections: [] });
      try {
          this.ocaForm = renderForm([this.ocaSchema.schema_base, ...this.ocaSchema.overlays]).form
          this.$refs.PreviewComponent.openModal(this.ocaForm);
      } catch {
          this.$refs.PreviewComponent.closeModal()
          this.$noty.error("ERROR! Form data are corrupted.", {
            timeout: 1000
          })
      }
      // this.$emit('schema-get', this.retrieve_schema_id);
      // this.retrieve_schema_id = '';
    }
  }
}
</script>
