<template>
  <ZDatatableGeneric
    buttonActionDelete
    includeActionsColumn
    :includeActionDeleteList="!readSearch"
    selectable
    :items="items"
    :columns="columns"
    :loading="loading"
    :paginatorInfo="paginatorInfo"
    :filter="true"
    :optionSearch="false"
    textButtonDelete="Ler"
    @search="searchNotification"
    @actionSearch="getNotifications({ fetchPolicy: 'network-only' })"
    @actionClear="clearSearch"
    @delete="readNotification"
    @deletes="readNotifications"
    @update:currentPageActive="updateCurrentPageActive"
  >
    <!-- ACTIONS -->
    <template #extra-actions-top>
      <ZButton color="danger" class="mr-3" @click="readAllNotifications"
        >Ler Todas</ZButton
      >
    </template>
    <!-- FILTER -->
    <template #filter>
      <!-- TODO - Ajustar distribuição dos itens dentro deste componente -->
      <div class="row">
        <div class="flex flex-col md6">
          <div class="item mr-2">
            <va-switch
              v-model="read"
              true-inner-label="Lidas"
              false-inner-label="Não lidas"
            />
          </div>
        </div>
      </div>
    </template>

    <!-- CELL -->
    <template #cell(notification)="{ rowKey }">
      <component
        :is="getNotificationComponent(rowKey.type)"
        :notification="rowKey"
      />
    </template>
    <template #cell(userAction)="{ rowKey }">
      <div v-if="parseData(rowKey.data).userAction">
        <ZUser
          :data="parseData(rowKey.data).userAction"
          showEmail
          :showConfirmTraining="
            rowKey.type ===
            'App\\Notifications\\Training\\ConfirmationTrainingNotification'
          "
          :showCancelTraining="
            rowKey.type ===
            'App\\Notifications\\Training\\CancelTrainingNotification'
          "
        />
      </div>
    </template>
  </ZDatatableGeneric>
</template>

<script>
import { defineComponent } from "vue";
import NOTIFICATIONS from "~/graphql/notification/query/notifications.graphql";
import ZDatatableGeneric from "~/components/molecules/Datatable/ZDatatableGeneric";
import ZUser from "~/components/molecules/Datatable/Slots/ZUser";
import NOTIFICATIONREAD from "~/graphql/notification/mutation/notificationsRead.graphql";
import { confirmSuccess, confirmError } from "~/utils/sweetAlert2/swalHelper";
import ZTrainingNotification from "~/components/molecules/Datatable/Slots/ZTrainingNotification";
import ZNotificationConfirmationTraining from "~/components/molecules/Datatable/Slots/ZNotificationConfirmationTraining";
import ZNotificationCancelTraining from "~/components/molecules/Datatable/Slots/ZNotificationCancelTraining";
import ZListItemNotification from "~/components/molecules/List/ZListItemNotification";
import ZButton from "~/components/atoms/Buttons/ZButton";

export default defineComponent({
  components: {
    ZDatatableGeneric,
    ZUser,
    ZTrainingNotification,
    ZNotificationConfirmationTraining,
    ZNotificationCancelTraining,
    ZListItemNotification,
    ZButton,
  },

  mounted() {
    this.getNotifications({ fetchPolicy: "network-only" });
  },

  data() {
    let loading = false;

    const columns = [
      { key: "id", name: "id", sortable: true },
      {
        key: "notification",
        name: "notification",
        label: "Notificações",
        sortable: true,
      },
      {
        key: "userAction",
        name: "userAction",
        label: "Usuário",
        sortable: true,
      },
    ];

    return {
      read: false,
      readSearch: false,
      items: [],
      loading,
      columns,
      paginatorInfo: {
        currentPage: 1,
        lastPage: 1,
        total: 0,
      },
      variablesGetNotifications: {
        page: 1,
        filter: {
          search: "%%",
          positionsIds: [],
          teamsIds: [],
        },
        orderBy: "id",
        sortedBy: "desc",
      },
      selectedItems: [],
      selectedItemsEmitted: [],
      selectMode: "multiple",
      selectedColor: "primary",
      selectModeOptions: ["single", "multiple"],
      selectColorOptions: ["primary", "danger", "warning", "#EF467F"],
    };
  },

  methods: {
    parseData(rowKeyData) {
      try {
        return JSON.parse(rowKeyData);
      } catch (error) {
        console.error("Erro ao converter dados para JSON", error);
        // Retorne um valor padrão ou lide com o erro conforme necessário
        return {};
      }
    },

    unselectItem(item) {
      this.selectedItems = this.selectedItems.filter(
        (selectedItem) => selectedItem !== item
      );
    },

    async readAllNotifications() {
      try {
        this.loading = true;

        const query = gql`
          ${NOTIFICATIONREAD}
        `;

        const variables = {
          markAllAsRead: true,
        };

        const { mutate } = await useMutation(query, { variables });

        const { data } = await mutate();

        confirmSuccess("Notificação(ões) lida(s) com sucesso!", () => {
          this.items = [];
        });
      } catch (error) {
        console.error(error);
        this.error = true;

        if (
          error.graphQLErrors &&
          error.graphQLErrors[0] &&
          error.graphQLErrors[0].extensions &&
          error.graphQLErrors[0].extensions.validation
        ) {
          this.errors = error.graphQLErrors[0].extensions.validation;

          const errorMessages = Object.values(this.errors).map((item) => {
            return item[0];
          });

          this.errorFields = Object.keys(this.errors);

          const footer = errorMessages.join("<br>");

          confirmError("Ocorreu um erro ao ler todas as notificações!", footer);
        } else {
          confirmError("Ocorreu um erro ao ler todas as notificações!");
        }
      }
      this.loading = false;
    },

    async readItems(ids) {
      try {
        this.loading = true;

        const query = gql`
          ${NOTIFICATIONREAD}
        `;

        const variables = {
          id: ids,
        };

        const { mutate } = await useMutation(query, { variables });

        const { data } = await mutate();

        confirmSuccess("Notificação(ões) lida(s) com sucesso!", () => {
          this.items = this.items.filter((item) => !ids.includes(item.id));
        });

        this.getNotifications({ fetchPolicy: "network-only" });
      } catch (error) {
        console.error(error);
        this.error = true;

        if (
          error.graphQLErrors &&
          error.graphQLErrors[0] &&
          error.graphQLErrors[0].extensions &&
          error.graphQLErrors[0].extensions.validation
        ) {
          this.errors = error.graphQLErrors[0].extensions.validation;

          const errorMessages = Object.values(this.errors).map((item) => {
            return item[0];
          });

          this.errorFields = Object.keys(this.errors);

          const footer = errorMessages.join("<br>");

          confirmError("Ocorreu um erro ao ler a notificação!", footer);
        } else {
          confirmError("Ocorreu um erro ao ler a notificação!");
        }
      }
      this.loading = false;
    },

    async readNotification(id) {
      await this.readItems([id]);
    },

    async readNotifications(items) {
      await this.readItems(items);
    },

    updateCurrentPageActive(page) {
      this.variablesGetNotifications.page = page;
      this.getNotifications({ fetchPolicy: "network-only" });
    },

    searchNotification(search) {
      this.variablesGetNotifications.filter.search = `%${search}%`;
    },

    clearSearch() {
      this.variablesGetNotifications.filter = {
        search: "%%",
      };
      this.read = false;
    },

    getNotificationComponent(type) {
      if (type === "App\\Notifications\\Training\\TrainingNotification") {
        return ZTrainingNotification;
      } else if (
        type ===
        "App\\Notifications\\Training\\ConfirmationTrainingNotification"
      ) {
        return ZNotificationConfirmationTraining;
      } else if ("App\\Notifications\\Training\\CancelTrainingNotification") {
        return ZNotificationCancelTraining;
      }
      // Aqui você pode adicionar mais condições para outros tipos de notificações
      return ZListItemNotification; // Componente padrão para notificações desconhecidas
    },

    getNotifications(fetchPolicyOptions = {}) {
      this.loading = true;
      this.items = [];

      const query = gql`
        ${NOTIFICATIONS}
      `;

      this.readSearch = this.read;

      const consult = {
        read: this.read,
        first: 10,
        page: this.variablesGetNotifications.page,
      };

      const {
        result: { value },
      } = useQuery(query, consult);

      const { onResult } = useQuery(query, consult, {
        fetchPolicy: fetchPolicyOptions.fetchPolicy || "cache-first", // Usa 'network-only' quando quer buscar nova consulta, senão 'cache-first'
      });

      onResult((result) => {
        if (result?.data?.notifications?.data.length > 0) {
          this.paginatorInfo = result.data.notifications.paginatorInfo;
          this.items = result.data.notifications.data;
        }
      });

      if (value) {
        if (value?.notifications?.data.length > 0) {
          this.paginatorInfo = value.notifications.paginatorInfo;
          this.items = value.notifications.data;
        }
      }
      this.loading = false;
    },
  },
});
</script>
