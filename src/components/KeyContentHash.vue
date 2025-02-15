<template>
  <div>
    <div>
      <!-- add button -->
      <el-form :inline="true" size="small">
        <el-form-item>
          <el-button size="small" type="primary" @click="dialogFormVisible = true">{{ $t('message.add_new_line') }}</el-button>
        </el-form-item>
      </el-form>

      <!-- add dialog -->
      <el-dialog :title="$t('message.add_new_line')" :visible.sync="dialogFormVisible">
        <el-form>
          <el-form-item label="Field">
            <el-input v-model="newLineItem.field" autocomplete="off"></el-input>
          </el-form-item>

          <el-form-item label="Value">
            <el-input type="textarea" :rows="2" v-model="newLineItem.value" autocomplete="off"></el-input>
          </el-form-item>
        </el-form>

        <div slot="footer" class="dialog-footer">
          <el-button @click="dialogFormVisible = false">{{ $t('el.messagebox.cancel') }}</el-button>
          <el-button type="primary" @click="addLine">{{ $t('el.messagebox.confirm') }}</el-button>
        </div>
      </el-dialog>

      <!-- edit dialog -->
      <el-dialog :title="$t('message.edit_line')" :visible.sync="editDialog">
        <el-form>
          <el-form-item label="Field">
            <el-input v-model="editLineItem.key" autocomplete="off"></el-input>
          </el-form-item>

          <el-form-item label="Value">
            <el-input type="textarea" :rows="2" v-model="editLineItem.value" autocomplete="off"></el-input>
          </el-form-item>
        </el-form>

        <div slot="footer" class="dialog-footer">
          <el-button @click="editDialog = false">{{ $t('el.messagebox.cancel') }}</el-button>
          <el-button type="primary" @click="editLine">{{ $t('el.messagebox.confirm') }}</el-button>
        </div>
      </el-dialog>

    </div>

    <!-- content table -->
    <PaginationTable :data="hashData" :filterValue="filterValue" filterKey="key">
      <el-table-column
        type="index"
        label="ID"
        sortable
        width="150">
      </el-table-column>
      <el-table-column
        prop="key"
        sortable
        resizable
        label="Key"
        width=150
        >
      </el-table-column>
      <el-table-column
        prop="value"
        resizable
        sortable
        show-overflow-tooltip
        label="Value"
        >
      </el-table-column>

      <el-table-column label="Operation">
        <template slot="header" slot-scope="scope">
          <input
            class="el-input__inner key-detail-filter-value"
            v-model="filterValue"
            :placeholder="$t('message.key_to_search')"
            />
        </template>
        <template slot-scope="scope">
          <el-button type="text" @click="showEditDialog(scope.row)" icon="el-icon-edit" circle></el-button>
          <el-button type="text" @click="deleteLine(scope.row)" icon="el-icon-delete" circle></el-button>
        </template>
      </el-table-column>
    </PaginationTable>
  </div>
</template>

<script>
import PaginationTable from '@/components/PaginationTable';

export default {
  data() {
    return {
      filterValue: '',
      dialogFormVisible: false,
      editDialog: false,
      hashData: [], // {key: xxx, value: xxx}
      newLineItem: {},
      beforeEditItem: {},
      editLineItem: {},
    };
  },
  components: {PaginationTable},
  props: ['redisKey', 'syncKeyParams'],
  methods: {
    initShow() {
      const key = this.syncKeyParams.keyName;

      if (!key) {
        return;
      }

      this.$util.get('client').hgetallAsync(key).then((reply) => {
        const hashData = [];

        for (const i in reply) {
          hashData.push({ key: i, value: reply[i] });
        }

        this.hashData = hashData;
      });
    },
    showEditDialog(row) {
      this.editLineItem = row;
      this.beforeEditItem = JSON.parse(JSON.stringify(row));
      this.editDialog = true;
    },
    editLine() {
      const key = this.syncKeyParams.keyName;
      const client = this.$util.get('client');

      const before = this.beforeEditItem;
      const after = this.editLineItem;

      client.hsetAsync(key, after.key, after.value).then((reply) => {
        // key changed
        if (before.key !== after.key) {
          client.hdelAsync(key, before.key).then((reply) => {
            this.initShow();
          });
        } else {
          this.initShow();
        }
      });

      this.$message.success({
        message: `${after.key} ${this.$t('message.modify_success')}`,
        duration: 1000,
      });

      this.editDialog = false;
    },
    deleteLine(row) {
      this.$confirm(this.$t('message.confirm_to_delete_row_data'), {
        type: 'warning',
      }).then(() => {
        const key = this.syncKeyParams.keyName;

        this.$util.get('client').hdelAsync(key, row.key).then((reply) => {
          if (reply === 1) {
            this.$message.success({
              message: `${row.key} ${this.$t('message.delete_success')}`,
              duration: 1000,
            });

            this.initShow();
          }
        });
      });
    },
    addLine() {
      const key = this.syncKeyParams.keyName;
      const ttl = this.syncKeyParams.keyTTL;
      const client = this.$util.get('client');

      this.dialogFormVisible = false;

      if (!key) {
        this.$parent.$parent.emptyKeyWhenAdding();
        return false;
      }

      if (!this.newLineItem.field || !this.newLineItem.value) {
        return;
      }

      client.hsetAsync(key, this.newLineItem.field, this.newLineItem.value).then((reply) => {
        if (ttl > 0) {
          client.expireAsync(key, ttl).then((reply) => {});
        }

        // reply==1:new field; reply==0 field exists
        this.$message.success({
          message: reply ? this.$t('message.add_success') : this.$t('message.modify_success'),
          duration: 1000,
        });

        // if in new key mode, exec refreshAfterAdd
        this.redisKey ? this.initShow() : this.$parent.$parent.refreshAfterAdd(key);
      });
    },
  },
  mounted() {
    this.initShow();
  },
};
</script>
