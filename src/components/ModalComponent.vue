<template>
    <div class="modal">
        <div @click="handleClick">
            <div class="modal-container">
                <button class="pp-square btn pp-color m-0" style="padding: 0" @click="$emit('close')">
                    <i class="i-fa6-solid:xmark m-0" />
                </button>
                <slot></slot>
            </div>
        </div>
    </div>
</template>

<script>
export default {
    emits: ["close"],
    mounted() {
        window.addEventListener("keydown", this.handleKeyDown);
    },
    unmounted() {
        window.removeEventListener("keydown", this.handleKeyDown);
    },
    methods: {
        handleKeyDown(event) {
            if (event.code === "Escape") {
                this.$emit("close");
            } else return;
            event.preventDefault();
        },
        handleClick(event) {
            if (event.target !== event.currentTarget) return;
            this.$emit("close");
        },
    },
};
</script>

<style>
.modal {
    @apply fixed z-50 top-0 left-0 w-full h-full bg-dark-900 bg-opacity-80 transition-opacity table;
}
.modal > div {
    @apply table-cell align-middle;
}
.modal-container {
    @apply w-100% m-auto max-w-[421rem] relative;
}
.modal-container > button {
    @apply absolute right-[12rem] top-[15rem];
}
</style>
