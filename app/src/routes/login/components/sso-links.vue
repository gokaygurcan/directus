<template>
	<div class="sso-links">
		<template v-if="providers && providers.length > 0">
			<v-divider />

			<a v-for="provider in providers" :key="provider.name" class="sso-link" :href="provider.link">
				{{ t('log_in_with', { provider: provider.name }) }}
			</a>
		</template>
	</div>
</template>

<script lang="ts">
import { useI18n } from 'vue-i18n';
import { defineComponent, ref, onMounted } from 'vue';
import api from '@/api';
import { getRootPath } from '@/utils/get-root-path';
import { unexpectedError } from '@/utils/unexpected-error';

export default defineComponent({
	setup() {
		const { t } = useI18n();

		const providers = ref([]);
		const loading = ref(false);

		onMounted(() => fetchProviders());

		return { t, providers };

		async function fetchProviders() {
			loading.value = true;

			try {
				const response = await api.get('/auth/oauth/');

				providers.value = response.data.data?.map((providerName: string) => {
					return {
						name: providerName,
						link: `${getRootPath()}auth/oauth/${providerName.toLowerCase()}?redirect=${window.location.href}`,
					};
				});
			} catch (err: any) {
				unexpectedError(err);
			} finally {
				loading.value = false;
			}
		}
	},
});
</script>

<style lang="scss" scoped>
.v-divider {
	margin: 24px 0;
}

.sso-link {
	display: block;
	display: flex;
	align-items: center;
	justify-content: center;
	width: 100%;
	height: var(--input-height);
	text-align: center;
	background-color: var(--background-normal);
	border-radius: var(--border-radius);
	transition: background var(--fast) var(--transition);

	&:hover {
		background-color: var(--background-normal-alt);
	}

	& + & {
		margin-top: 12px;
	}
}
</style>
