<template>
    <footer class="efy_trans_filter efy_shadow_trans efy_shadow_button_off">
        <a aria-label="GitHub" href="https://github.com/TeamPiped/Piped" target="_blank">
            <i class="i-fa6-brands:github" />
            <span v-t="'actions.source_code'" />
        </a>
        <a href="https://docs.piped.video/" target="_blank">
            <i class="i-fa6-solid:book" />
            <span v-t="'actions.documentation'" />
        </a>
        <a href="https://github.com/TeamPiped/Piped#donations" target="_blank">
            <i class="i-fa6-brands:bitcoin" />
            <span v-t="'actions.donations'" />
        </a>
        <a v-if="statusPageHref" :href="statusPageHref">
            <i class="i-fa6-solid:server" />
            <span v-t="'actions.status_page'" />
        </a>
        <a v-if="donationHref" :href="donationHref">
            <i class="i-fa6-solid:money-check" />
            <span v-t="'actions.instance_donations'" />
        </a>
        <a v-if="privacyPolicyHref" :href="privacyPolicyHref" target="_blank">
            <i class="i-fa6-solid:eye" />
            <span v-t="'actions.instance_privacy_policy'" />
        </a>
    </footer>
</template>

<script>
export default {
    data() {
        return {
            donationHref: null,
            statusPageHref: null,
            privacyPolicyHref: null,
        };
    },
    mounted() {
        this.fetchConfig();
    },
    methods: {
        async fetchConfig() {
            this.fetchJson(this.apiUrl() + "/config").then(config => {
                this.donationHref = config?.donationUrl;
                this.statusPageHref = config?.statusPageUrl;
                this.privacyPolicyHref = config?.privacyPolicyUrl;
            });
        },
    },
};
</script>

<style>
footer {
    display: flex;
    flex-wrap: wrap;
    gap: 15rem;
    justify-content: center;
    width: 100%;
    margin: 15rem 0 0 0;
    background: var(--efy_bg1);
    border-radius: var(--efy_radius);
    padding: 15rem 5rem;
    border: var(--efy_border);
}
footer a {
    color: var(--efy_text) !important;
    -webkit-text-fill-color: var(--efy_text) !important;
    background: transparent !important;
    margin: 0;
    display: flex;
    gap: 8rem;
    align-items: center;
}
</style>
