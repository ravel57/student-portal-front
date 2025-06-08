<template>
	<div class="">
		<div class="table-param">
			<div
				v-if="currentUserRole === 'TEACHER'"
				class="table-param-text"
				v-text="'Группа'"
			/>
			<q-select
				v-if="currentUserRole === 'TEACHER'"
				:options="groups"
				option-label="name"
				option-value="id"
				emit-value map-options
				v-model="selectedGroup"
				dense style="min-width: 120px"
			/>
			<div
				v-if="currentUserRole === 'TEACHER'"
				class="table-param-text"
				v-text="'Предмет'"
			/>
			<q-select
				v-if="currentUserRole === 'TEACHER'"
				:options="subjects"
				option-label="name"
				option-value="id"
				emit-value
				map-options
				v-model="selectedSubject"
				dense
				style="min-width: 120px"
			/>
		</div>
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
		tableRows: [],
		groups: [],
		selectedGroup: "",
		subjects: [],
		selectedSubject: "",
		students: [],
		student: {}
	}),

	computed: {},

	methods: {
		fetchSubjects() {
			axios.get("/api/v1/subjects")
				.then(res => this.subjects = res.data)
		},
		fetchStudents() {
			axios.post("/api/v1/students", {id: this.selectedGroup})
				.then(response => {
					this.students = response.data
				})
		},
	},

	watch: {
		selectedGroup() {
			this.fetchStudents()
		}
	},

	mounted() {
		axios.get("/api/v1/groups")
			.then(response => {
				this.groups = response.data
			})
		this.fetchSubjects()
		if (this.currentUserRole !== "TEACHER") {
			axios.get("/api/v1/me")
				.then(response => {
					console.log(response.data)
					this.student = response.data
				})
		}
	},
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