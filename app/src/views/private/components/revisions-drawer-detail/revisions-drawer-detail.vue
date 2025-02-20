<template>
	<sidebar-detail
		:title="t('revisions')"
		icon="change_history"
		:badge="!loading && revisions ? abbreviateNumber(revisionsCount) : null"
	>
		<v-progress-linear v-if="loading" indeterminate />

		<template v-else>
			<template v-for="group in revisionsByDate" :key="group.date.toString()">
				<v-divider>{{ group.dateFormatted }}</v-divider>

				<template v-for="(item, index) in group.revisions" :key="item.id">
					<revision-item :revision="item" :last="index === group.revisions.length - 1" @click="openModal(item.id)" />
				</template>
			</template>

			<v-divider v-if="revisionsCount > 100" class="other">
				{{ t('count_other_revisions', revisionsCount - 101) }}
			</v-divider>

			<template v-if="created">
				<revision-item :revision="created" last @click="openModal(created.id)" />
			</template>

			<template v-else>
				<v-divider v-if="revisionsByDate.length > 0" />

				<div class="external">
					{{ t('revision_delta_created_externally') }}
				</div>
			</template>
		</template>

		<revisions-drawer
			v-if="revisions"
			v-model:current="modalCurrentRevision"
			v-model:active="modalActive"
			:revisions="revisions"
			@revert="$emit('revert', $event)"
		/>
	</sidebar-detail>
</template>

<script lang="ts">
import { useI18n } from 'vue-i18n';
import { defineComponent, ref } from 'vue';
import { Revision, RevisionsByDate } from './types';

import api from '@/api';
import { groupBy, orderBy } from 'lodash';
import { isToday, isYesterday, isThisYear } from 'date-fns';
import formatLocalized from '@/utils/localized-format';
import RevisionItem from './revision-item.vue';
import RevisionsDrawer from './revisions-drawer.vue';
import { unexpectedError } from '@/utils/unexpected-error';
import { abbreviateNumber } from '@/utils/abbreviate-number';

export default defineComponent({
	components: { RevisionItem, RevisionsDrawer },
	props: {
		collection: {
			type: String,
			required: true,
		},
		primaryKey: {
			type: [String, Number],
			required: true,
		},
	},
	emits: ['revert'],
	setup(props) {
		const { t } = useI18n();

		const { revisions, revisionsByDate, loading, refresh, revisionsCount, created } = useRevisions(
			props.collection,
			props.primaryKey
		);

		const modalActive = ref(false);
		const modalCurrentRevision = ref<number | null>(null);

		return {
			t,
			revisions,
			revisionsByDate,
			loading,
			refresh,
			modalActive,
			modalCurrentRevision,
			openModal,
			revisionsCount,
			created,
			abbreviateNumber,
		};

		function openModal(id: number) {
			modalCurrentRevision.value = id;
			modalActive.value = true;
		}

		function useRevisions(collection: string, primaryKey: number | string) {
			const revisions = ref<Revision[] | null>(null);
			const revisionsByDate = ref<RevisionsByDate[] | null>(null);
			const loading = ref(false);
			const revisionsCount = ref(0);
			const created = ref<Revision>();

			getRevisions();

			return { created, revisions, revisionsByDate, loading, refresh, revisionsCount };

			async function getRevisions() {
				loading.value = true;

				try {
					const response = await api.get(`/revisions`, {
						params: {
							filter: {
								collection: {
									_eq: collection,
								},
								item: {
									_eq: primaryKey,
								},
							},
							sort: '-id',
							limit: 100,
							fields: [
								'id',
								'data',
								'delta',
								'collection',
								'item',
								'activity.action',
								'activity.timestamp',
								'activity.user.id',
								'activity.user.email',
								'activity.user.first_name',
								'activity.user.last_name',
								'activity.ip',
								'activity.user_agent',
							],
							meta: ['filter_count'],
						},
					});

					const createdResponse = await api.get(`/revisions`, {
						params: {
							filter: {
								collection: {
									_eq: collection,
								},
								item: {
									_eq: primaryKey,
								},
								activity: {
									action: {
										_eq: 'create',
									},
								},
							},
							sort: '-id',
							limit: 1,
							fields: [
								'id',
								'data',
								'delta',
								'collection',
								'item',
								'activity.action',
								'activity.timestamp',
								'activity.user.id',
								'activity.user.email',
								'activity.user.first_name',
								'activity.user.last_name',
								'activity.ip',
								'activity.user_agent',
							],
							meta: ['filter_count'],
						},
					});

					created.value = createdResponse.data.data?.[0];

					const revisionsGroupedByDate = groupBy(
						response.data.data.filter((revision: any) => !!revision.activity),
						(revision: Revision) => {
							// revision's timestamp date is in iso-8601
							const date = new Date(new Date(revision.activity.timestamp).toDateString());
							return date;
						}
					);

					const revisionsGrouped: RevisionsByDate[] = [];

					for (const [key, value] of Object.entries(revisionsGroupedByDate)) {
						const date = new Date(key);
						const today = isToday(date);
						const yesterday = isYesterday(date);
						const thisYear = isThisYear(date);

						let dateFormatted: string;

						if (today) dateFormatted = t('today');
						else if (yesterday) dateFormatted = t('yesterday');
						else if (thisYear) dateFormatted = await formatLocalized(date, String(t('date-fns_date_short_no_year')));
						else dateFormatted = await formatLocalized(date, String(t('date-fns_date_short')));

						revisionsGrouped.push({
							date: date,
							dateFormatted: String(dateFormatted),
							revisions: orderBy(value, ['activity.timestamp'], ['desc']),
						});
					}

					revisionsByDate.value = orderBy(revisionsGrouped, ['date'], ['desc']);
					revisions.value = orderBy(response.data.data, ['activity.timestamp'], ['desc']);
					revisionsCount.value = response.data.meta.filter_count;
				} catch (err: any) {
					unexpectedError(err);
				} finally {
					loading.value = false;
				}
			}

			async function refresh() {
				await getRevisions();
			}
		}
	},
});
</script>

<style lang="scss" scoped>
.v-progress-linear {
	margin: 24px 0;
}

.v-divider {
	--v-divider-color: var(--background-normal-alt);

	position: sticky;
	top: 0;
	z-index: 3;
	margin-top: 8px;
	margin-right: -8px;
	margin-bottom: 6px;
	margin-left: -8px;
	padding-top: 8px;
	padding-right: 8px;
	padding-left: 8px;
	background-color: var(--background-normal);
	box-shadow: 0 0 2px 2px var(--background-normal);

	&:first-of-type {
		margin-top: 0;
	}
}

.empty {
	margin-top: 16px;
	margin-bottom: 16px;
	color: var(--foreground-subdued);
	font-style: italic;
}

.external {
	margin-left: 20px;
	color: var(--foreground-subdued);
	font-style: italic;
}

.other {
	--v-divider-label-color: var(--foreground-subdued);

	font-style: italic;
}
</style>
