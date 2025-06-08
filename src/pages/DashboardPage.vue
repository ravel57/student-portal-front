<template>
	<q-layout view="hHh lpR fFf" class="dashboard">
		<q-header elevated class="bg-primary text-white">
			<q-toolbar>
				<q-toolbar-title>Учебный портал</q-toolbar-title>
				<q-space/>
				<q-btn flat icon="logout" @click="logout"/>
			</q-toolbar>
		</q-header>
		<q-page-container>
			<q-page padding>
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
				<!-- Для TEACHER обычная таблица -->
				<q-table v-if="currentUserRole !== 'STUDENT'" :rows="tableRows">
					<template v-slot:header="props">
						<q-tr :props="props">
							<q-th v-if="currentUserRole === 'TEACHER'">
								Студент
							</q-th>
							<q-th v-for="item in lessons" :key="item.date">
								{{ item.date }}
							</q-th>
						</q-tr>
					</template>
					<template v-slot:body="props">
						<q-tr :props="props">
							<q-td v-if="currentUserRole === 'TEACHER'">
								{{ props.row.firstname }} {{ props.row.lastname }}
							</q-td>
							<q-td v-for="lesson in lessons" :key="lesson.date">
								<q-input
									dense
									type="number"
									min="2"
									max="5"
									mask="#"
									:model-value="localMarks[props.row.id]?.[lesson.date] ?? ''"
									@update:model-value="val => onMarkInput(props.row.id, lesson.date, val)"
								/>
							</q-td>
						</q-tr>
					</template>
				</q-table>

				<!-- Для STUDENT специальная таблица: строки — предметы, колонки — даты -->
				<q-table
					v-else
					:rows="studentRows"
					:columns="studentColumns"
					row-key="subject"
				>
					<template v-slot:body="props">
						<q-tr :props="props">
							<q-td>{{ props.row.subject }}</q-td>
							<q-td
								v-for="date in allDates"
								:key="date"
								class="text-center"
							>
								{{ props.row.marksByDate[date] ?? '' }}
							</q-td>
						</q-tr>
					</template>
				</q-table>

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
			</q-page>
		</q-page-container>
	</q-layout>
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
		lessons: [],
		localMarks: {},
		studentRows: [],
		studentColumns: [],
		allDates: [],
		studentMarksBySubject: [],
	}),

	methods: {
		logout() {
			axios.post('/logout')
				.then(() => location.reload())
				.catch(() => {
					axios.post('/logout')
						.then(() => location.reload())
				})
		},
		fetchSubjects() {
			axios.get("/api/v1/subjects")
				.then(res => this.subjects = res.data)
		},
		fetchStudents() {
			axios.post("/api/v1/students", {id: this.selectedGroup})
				.then(response => {
					this.tableRows = response.data
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
		},
		fillMarks() {
			this.localMarks = {};
			for (const student of this.tableRows) {
				this.localMarks[student.id] = {};
				if (student.studentsMarks && student.studentsMarks.length > 0) {
					for (const sm of student.studentsMarks) {
						for (const mark of sm.marks) {
							this.localMarks[student.id][mark.date] = mark.value;
						}
					}
				}
				for (const lesson of this.lessons) {
					if (!(lesson.date in this.localMarks[student.id])) {
						this.localMarks[student.id][lesson.date] = '';
					}
				}
			}
		},
		onMarkInput(studentId, date, value) {
			if (!this.localMarks[studentId]) this.localMarks[studentId] = {};
			this.localMarks[studentId][date] = value;
			axios.post('/api/v1/update-mark', {
				date: date,
				mark: value,
				subjectId: this.selectedSubject,
				studentId: studentId,
			})
		},

		/** Формирует таблицу для STUDENT: строки — предметы, колонки — даты */
		fillStudentTable() {
			if (!this.studentMarksBySubject) return;
			// Собрать все уникальные даты по всем предметам
			const dateSet = new Set();
			this.studentMarksBySubject.forEach(item => {
				item.marks.forEach(mark => {
					dateSet.add(mark.date);
				});
			});
			this.allDates = Array.from(dateSet).sort();

			// Строим строки таблицы
			this.studentRows = this.studentMarksBySubject.map(subjectItem => {
				const marksByDate = {};
				subjectItem.marks.forEach(mark => {
					marksByDate[mark.date] = mark.value;
				});
				return {
					subject: subjectItem.subject.name,
					marksByDate,
				};
			});

			// Строим колонки: "Предмет", далее по датам
			this.studentColumns = [
				{ name: 'subject', label: 'Предмет', align: 'left', field: 'subject' },
				...this.allDates.map(date => ({
					name: date,
					label: date,
					align: 'center',
					field: row => row.marksByDate[date] ?? '',
				}))
			];
		},
	},

	watch: {
		selectedGroup() {
			this.fetchStudents()
			if (this.currentUserRole !== 'TEACHER') return
			this.getLessonsBy()
		},
		selectedSubject() {
			this.getLessonsBy()
		},
		tableRows: {
			handler() {
				this.fillMarks();
			},
			immediate: true
		},
		lessons: {
			handler() {
				this.fillMarks();
			},
			immediate: true
		},
		studentMarksBySubject: {
			handler() {
				this.fillStudentTable();
			},
			immediate: true
		}
	},

	mounted() {
		axios.get("/api/v1/groups")
			.then(response => {
				this.groups = response.data
			})
		this.fetchSubjects()
		axios.get("/api/v1/me")
			.then(response => {
				this.user = response.data
				this.currentUserRole = this.user.role
				if (this.currentUserRole === 'STUDENT') {
					axios.get(`/api/v1/students-marks-by-subjects?studentId=${this.user.id}`).then(response => {
						this.studentMarksBySubject = response.data.marksBySubject;
					})
				}
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