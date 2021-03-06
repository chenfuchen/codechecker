<template>
  <select-option
    :id="id"
    title="Tag Filter"
    :bus="bus"
    :fetch-items="fetchItems"
    :selected-items="selectedItems"
    :search="search"
    :loading="loading"
    @clear="clear(true)"
    @input="setSelectedItems"
  >
    <template v-slot:icon>
      <v-icon color="grey">
        mdi-tag
      </v-icon>
    </template>
  </select-option>
</template>

<script>
import { ccService, handleThriftError } from "@cc-api";
import {
  ReportFilter,
  RunFilter,
  RunHistoryFilter
} from "@cc/report-server-types";

import SelectOption from "./SelectOption/SelectOption";
import BaseSelectOptionFilterMixin from "./BaseSelectOptionFilter.mixin";

export default {
  name: "BaselineTagFilter",
  components: {
    SelectOption
  },
  mixins: [ BaseSelectOptionFilterMixin ],

  data() {
    return {
      id: "run-tag",
      search: {
        placeHolder : "Search for run tags...",
        regexLabel: "Filter by wildcard pattern (e.g.: mytag*, myrun*:mytag*)",
        filterItems: this.filterItems
      }
    };
  },

  methods: {
    getSelectedItems(tagWithRunNames) {
      return tagWithRunNames.map(async s => ({
        id: s,
        tagIds: await this.getTagIds(s),
        title: s,
        count: "N/A"
      }));
    },

    initByUrl() {
      return new Promise(resolve => {
        const state = [].concat(this.$route.query[this.id] || []);
        if (state.length) {
          const selectedItems = this.getSelectedItems(state);
          Promise.all(selectedItems).then(async res => {
            await this.setSelectedItems(res, false);
            resolve();
          });
        } else {
          resolve();
        }
      });
    },

    async getSelectedTagIds() {
      return [].concat(...await Promise.all(
        this.selectedItems.map(async item => {
          if (!item.tagIds) {
            item.tagIds = await this.getTagIds(item.title);
          }

          return Promise.resolve(item.tagIds);
        })));
    },

    async updateReportFilter() {
      const tagIds = await this.getSelectedTagIds();
      this.setReportFilter({ runTag: tagIds.length ? tagIds : null });
    },

    onReportFilterChange(key) {
      if (key === "runTag") return;
      this.update();
    },

    async fetchItems(opt={}) {
      this.loading = true;

      const reportFilter = new ReportFilter(this.reportFilter);

      const limit = opt.limit || this.defaultLimit;
      const offset = 0;

      reportFilter.runTag = opt.query
        ? (await Promise.all(opt.query?.map(s => this.getTagIds(s)))).flat()
        : null;

      return new Promise(resolve => {
        ccService.getClient().getRunHistoryTagCounts(this.runIds, reportFilter,
          null, limit, offset, handleThriftError(res => {
            resolve(res.map(tag => {
              const title = tag.runName + ":" + tag.name;
              return {
                id: title,
                tagIds: [ tag.id.toNumber() ],
                title: title,
                count: tag.count.toNumber()
              };
            }));
            this.loading = false;
          }));
      });
    },

    async getTagIds(runWithTagName) {
      const index = runWithTagName.indexOf(":");

      let runName, tagName;
      if (index !== -1) {
        runName = runWithTagName.substring(0, index);
        tagName = runWithTagName.substring(index + 1);
      } else {
        tagName = runWithTagName;
      }

      const runIds = runName ? await this.getRunIds(runName) : null;
      const limit = null;
      const offset = 0;
      const runHistoryFilter = new RunHistoryFilter({
        tagNames: [ tagName ]
      });

      return new Promise(resolve => {
        ccService.getClient().getRunHistory(runIds, limit, offset,
          runHistoryFilter, handleThriftError(res => {
            resolve(res.map(history => history.id.toNumber()));
          }));
      });
    },

    getRunIds(runName) {
      return new Promise(resolve => {
        const limit = null;
        const offset = null;
        const sortMode = null;
        const runFilter = new RunFilter({ names: [ runName ] });

        ccService.getClient().getRunData(runFilter, limit, offset, sortMode,
          handleThriftError(runs => {
            resolve(runs.map(run => run.runId.toNumber()));
          }));
      });
    }
  }
};
</script>
