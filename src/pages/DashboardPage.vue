<template>
	<div class="">
		<div class="">
			<q-table :rows="tableRows">
				<template v-slot:header="props">
					<q-tr :props="props">
						<q-th
							v-if="currentUserRole === 'TEACHER'"
						>
							Студент
						</q-th>
						<q-th
							v-if="currentUserRole === 'STUDENT'"
						>
							Предмет
						</q-th>
					</q-tr>
				</template>

				<template v-slot:body="props">
					<q-tr :props="props">
						<q-td
							v-if="currentUserRole === 'TEACHER'"
						>
							{{ props.row.id }}
						</q-td>
						<q-td
							v-if="currentUserRole === 'STUDENT'"
						>
							{{ props.row.name }}
						</q-td>
					</q-tr>
				</template>
			</q-table>
		</div>
	</div>
</template>

<script>
import axios from 'axios'

export default {
	name: "DashboardPage",

	data: () => ({
		currentUserRole: "",
		tableRows: []
	}),

	computed: {},

	methods: {},

	watch: {}
}
</script>

<style scoped>
.table-param {
	display: flex;
	gap: 10px;
	align-items: center;
}

.table-param-text {
	display: flex;
	justify-content: center;
}

.dashboard {
	min-height: 100vh;
}

.grade-input input::-webkit-outer-spin-button,
.grade-input input::-webkit-inner-spin-button {
	display: none;
	-webkit-appearance: none;
	margin: 0;
}

.grade-input input[type="number"] {
	-moz-appearance: textfield;
	text-align: center;
}

:deep(.q-table th) {
	font-weight: bold;
}
</style>