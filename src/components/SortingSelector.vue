<template>
    <label v-t="'actions.sort_by'" for="ddlSortBy" class="m-0" />
    <select id="ddlSortBy" v-model="selectedSort" class="m-0 w-auto">
        <option v-for="(value, key) in options" :key="key" v-t="`actions.${key}`" :value="value" />
    </select>
</template>

<script setup>
import { ref, watch } from "vue";

const options = {
    most_recent: "descending",
    least_recent: "ascending",
    channel_name_asc: "channel_ascending",
    channel_name_desc: "channel_descending",
};

const selectedSort = ref("descending");

const props = defineProps({
    byKey: {
        type: String,
        required: true,
    },
});

const emit = defineEmits(["apply"]);

watch(selectedSort, value => {
    switch (value) {
        case "ascending":
            emit("apply", (a, b) => a[props.byKey] - b[props.byKey]);
            break;
        case "descending":
            emit("apply", (a, b) => b[props.byKey] - a[props.byKey]);
            break;
        case "channel_ascending":
            emit("apply", (a, b) => a.uploaderName.localeCompare(b.uploaderName));
            break;
        case "channel_descending":
            emit("apply", (a, b) => b.uploaderName.localeCompare(a.uploaderName));
            break;
        default:
            console.error("Unexpected sort value");
    }
});
</script>
