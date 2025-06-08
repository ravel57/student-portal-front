<template>
	<div class="">
		<div class="table-param">
			<q-btn
				v-if="currentUserRole === 'ADMIN'"
				outline
				@click="showModal = true"
				label="Добавить занятие"
			/>
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
						<q-th v-for="item in this.lessons">
							{{ item.date }}
						</q-th>
					</q-tr>
				</template>

				<template v-slot:body="props">
					<q-tr :props="props">
						<q-td
							v-if="currentUserRole === 'TEACHER'"
						>
							{{ props.row.firstname }} {{ props.row.lastname }}
						</q-td>
						<q-td
							v-if="currentUserRole === 'STUDENT'"
						>
							{{ props.row.name }}
						</q-td>
						<q-td>
							{{ props.row.name }}
						</q-td>
					</q-tr>
				</template>
			</q-table>
		</div>
	</div>
	<q-dialog v-model="this.showModal">
		<q-card>
			<q-card-section>
				Добавить занятие
			</q-card-section>
			<q-card-section>
				<q-select
					:options="groups"
					label="Группа"
					option-label="name"
					option-value="id"
					emit-value map-options
					v-model="selectedGroup"
					dense style="min-width: 120px"
				/>
				<q-select
					:options="subjects"
					label="Предмет"
					option-label="name"
					option-value="id"
					emit-value
					map-options
					v-model="selectedSubject"
					dense
					style="min-width: 120px"
				/>
				<q-input
					mask="##.##.####"
					type="date"
					v-model="this.newLessonDate"
				/>
			</q-card-section>
			<q-card-actions>
				<q-btn
					v-close-popup
					label="Отмена"
				/>
				<q-btn
					@click="this.addNewLesson"
					label="Добавить"
				/>
			</q-card-actions>
		</q-card>
	</q-dialog>
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
		user: {},
		showModal: false,
		newLessonDate: "",
		lessons: []
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
					this.tableRows = response.data
					// this.students = response.data
				})
		},
		addNewLesson() {
			axios.post("/api/v1/lesson", {
				subjectId: this.selectedSubject,
				groupId: this.selectedGroup,
				date: this.newLessonDate
			}).then(() => {
				this.showModal = false
			})
		},
		getLessonsBy() {
			if (this.selectedSubject !== "" && this.selectedGroup !== "") {
				axios.post("/api/v1/lessons-by", {
					subjectId: this.selectedSubject,
					groupId: this.selectedGroup,
				}).then(response => {
					this.lessons = response.data
				})
			}
		}
	},

	watch: {
		selectedGroup() {
			this.fetchStudents()
			if (this.currentUserRole !== 'TEACHER') return
			this.getLessonsBy()
		},
		selectedSubject() {
			this.getLessonsBy()
		}
	},

	mounted() {
		axios.get("/api/v1/groups")
			.then(response => {
				console.log(response.data)
				this.groups = response.data
			})
		this.fetchSubjects()
		axios.get("/api/v1/me")
			.then(response => {
				console.log(response.data)
				this.user = response.data
				this.currentUserRole = this.user.role
			})
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