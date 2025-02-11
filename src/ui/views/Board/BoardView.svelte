<script lang="ts">
  import type { ProjectViewProps } from "src/customViewApi";
  import { createDataRecord } from "src/lib/dataApi";
  import type {
    DataField,
    DataFrame,
    DataRecord,
  } from "src/lib/dataframe/dataframe";
  import { updateRecordValues } from "src/lib/datasources/helpers";
  import { notUndefined } from "src/lib/helpers";
  import { i18n } from "src/lib/stores/i18n";
  import { app } from "src/lib/stores/obsidian";
  import type { ViewApi } from "src/lib/viewApi";
  import type { ProjectDefinition } from "src/settings/settings";
  import { CreateNoteModal } from "src/ui/modals/createNoteModal";
  import { EditNoteModal } from "src/ui/modals/editNoteModal";
  import {
    getRecordColorContext,
    sortRecordsContext,
  } from "src/ui/views/helpers";
  import BoardOptionsProvider from "./BoardOptionsProvider.svelte";
  import { getColumns, getFieldByName } from "./board";
  import { Board } from "./components";
  import type {
    OnRecordAdd,
    OnRecordClick,
    OnRecordUpdate,
    OnSortColumns,
  } from "./components/Board/types";
  import type { BoardConfig } from "./types";

  export let project: ProjectDefinition;
  export let frame: DataFrame;
  export let readonly: boolean;
  export let api: ViewApi;
  export let getRecordColor: ProjectViewProps["getRecordColor"];
  export let sortRecords: ProjectViewProps["sortRecords"];
  export let getRecord: ProjectViewProps["getRecord"];

  export let config: BoardConfig | undefined;
  export let onConfigChange: (cfg: BoardConfig) => void;

  export let hasSort: boolean;
  export let hasFilter: boolean;

  $: ({ fields, records } = frame);
  $: orderSyncField =
    (config?.orderSyncField && getFieldByName(fields, config.orderSyncField)) ||
    undefined;

  getRecordColorContext.set(getRecordColor);
  sortRecordsContext.set((records) =>
    hasSort ? sortRecords(records) : [...records]
  );

  const handleRecordClick: OnRecordClick = (record) => {
    new EditNoteModal(
      $app,
      fields,
      (record) => api.updateRecord(record, fields),
      record
    ).open();
  };

  const handleRecordUpdate =
    (groupByField: DataField | undefined): OnRecordUpdate =>
    (record, { id: column, records }, trigger) => {
      // Update record groupByField
      if (trigger === "addToColumn" && groupByField?.name) {
        record = updateRecordValues(record, {
          [groupByField.name]: column,
        });
      }

      // If sort or filter is active, get records from settings hand
      // and insert record in correct position or remove it
      let isConfigRecordsLoaded = false;
      if (hasSort || hasFilter) {
        const configRecords = config?.columns?.[column]?.records
          ?.map((r) => getRecord(r))
          .filter(notUndefined);

        if (configRecords) {
          const configRecordIndex = configRecords.findIndex(
            (r) => r.id === record.id
          );
          const isRecordInConfig = configRecordIndex !== -1;

          if (
            isRecordInConfig &&
            (trigger === "removeFromColumn" || !hasSort)
          ) {
            configRecords.splice(configRecordIndex, 1);
          }

          if (trigger === "addToColumn") {
            if (hasSort) {
              // Sort is active, insert record at end of column if it's
              // not already in the config, else keep position
              if (!isRecordInConfig) {
                configRecords.push(record);
              }
            } else if (hasFilter) {
              // If filter is active insert record in correct position
              // relative to records before or after it
              const index = records.findIndex((r) => r.id === record.id);
              const before = records[index - 1];
              const after = records[index + 1];
              let position: number = -1;

              if (before) {
                const beforeIndex = configRecords.findIndex(
                  (r) => r.id === before.id
                );
                if (beforeIndex !== -1) position = beforeIndex + 1;
              }
              if (after && position === -1) {
                position = configRecords.findIndex((r) => r.id === after.id);
              }
              if (position === -1) {
                position = configRecords.length;
              }

              configRecords.splice(position, 0, record);
            }
          }

          records = [
            ...configRecords,
            // In case the config is out of sync, insert any
            // remaining records at the end
            ...records.filter(
              (r1) =>
                r1.id !== record.id &&
                !configRecords.find((r2) => r2.id === r1.id)
            ),
          ];
          isConfigRecordsLoaded = true;
        }
      }

      saveConfig({
        ...config,
        columns: {
          ...config?.columns,
          [column]: {
            ...config?.columns?.[column],
            records: records.map((r) => r.id),
          },
        },
      });

      // If record was removed from column, there is no need to update
      // the groupBy or orderSync fields, since order is preserved on removal
      // and the groupBy field is set on adding the record to the other column
      if (trigger !== "removeFromColumn") {
        const recordsToUpdate: DataRecord[] = [];

        // If a sync field is defined, update it according to the order. Only
        // update if no sort is active or records are loaded from the config,
        // since they are sorted correctly according to custom order
        const orderSyncFieldName = orderSyncField?.name;
        if (orderSyncFieldName && (isConfigRecordsLoaded || !hasSort)) {
          records.forEach((r, i) => {
            const recordToUpdate =
              r.id === record.id
                ? record
                : r.values[orderSyncFieldName] !== i && r;
            if (recordToUpdate) {
              recordsToUpdate.push(
                updateRecordValues(recordToUpdate, {
                  [orderSyncFieldName]: i,
                })
              );
            }
          });
        } else {
          recordsToUpdate.push(record);
        }

        recordsToUpdate.forEach((r, i) => {
          api.updateRecord(r, fields);
        });
      }
    };

  const handleRecordAdd =
    (field: DataField | undefined): OnRecordAdd =>
    (column: string) => {
      new CreateNoteModal($app, project, (name, templatePath) => {
        api.addRecord(
          createDataRecord(
            name,
            project,
            field
              ? {
                  [field.name]:
                    column !== $i18n.t("views.board.no-status")
                      ? column
                      : undefined,
                }
              : {}
          ),
          templatePath
        );
      }).open();
    };

  const handleSortColumns: OnSortColumns = (names) => {
    saveConfig({
      ...config,
      columns: Object.fromEntries(
        names.map((name, i) => [
          name,
          { ...config?.columns?.[name], weight: i },
        ])
      ),
    });
  };

  function saveConfig(cfg: BoardConfig) {
    config = cfg;
    onConfigChange(cfg);
  }
</script>

<!--
  @component

  BoardView is the main board component that gets mounted inside the project view.
-->
<BoardOptionsProvider
  {frame}
  config={config ?? {}}
  onConfigChange={saveConfig}
  let:columnWidth
  let:groupByField
  let:includeFields
>
  <Board
    columns={getColumns(
      records,
      config?.columns ?? {},
      groupByField,
      orderSyncField,
      !hasSort
    )}
    {columnWidth}
    includeFields={fields.filter((field) => includeFields.includes(field.name))}
    onRecordClick={handleRecordClick}
    onRecordAdd={handleRecordAdd(groupByField)}
    onRecordUpdate={handleRecordUpdate(groupByField)}
    onSortColumns={handleSortColumns}
    {readonly}
    richText={groupByField?.typeConfig?.richText ?? false}
  />
</BoardOptionsProvider>
